# read的实现

## 一、入口层(libfs)

用户程序比如 fio 调用 glibc 的 `read()`,被 libsufs.so 劫持,经 fd 偏移(1024*1024)识别为 AeoFS fd 后转入:

- [entry.c#L373](Aeolia/Aeolia-src/AeoFS/libfs/entry.c#L373) `read()` → `sufs_libfs_sys_read(proc, fd, buf, count)`
- [syscall.c#L447](Aeolia/Aeolia-src/AeoFS/libfs/syscall.c#L447) `sufs_libfs_sys_read()`:
  1. `sufs_libfs_getfile(proc, fd)` —— 查 per-CPU filetable 拿到 `file_mnode *f`
  2. 调 `sufs_libfs_file_mnode_read(f, p, n)`

## 二、文件抽象层(libfs)

- [file.c#L22](Aeolia/Aeolia-src/AeoFS/libfs/file.c#L22) `sufs_libfs_file_mnode_read()`:
  - 检查 `readable`、`m != NULL`、`m` 类型为 `SUFS_FILE_TYPE_REG`
  - 检查 `f->off < file_size`,越界返回 0(EOF)
  - 调 `sufs_libfs_readm(f->m, addr, off, n)`

## 三、页缓存层(libfs)

在这里调用异步的sufs_libfs_cmd_read_blk 读取，然后sufs_libfs_cmd_blk_idle()进入忙等待。

- [mfs.c#L228](Aeolia/Aeolia-src/AeoFS/libfs/mfs.c#L228) `sufs_libfs_readm()`:
  - 取 `inode_read_lock`(rwlock,允许并发读)
  - 按 `SUFS_FILE_BLOCK_SIZE` 分页,循环:
    1. `sufs_libfs_mnode_file_get_page_cache(m, pgidx)` —— 查页缓存命中?
    2. **未命中**:`sufs_libfs_mnode_file_get_page(m, pgidx)` 取 LBA,分配新 `page_cache_entry`,挂入 mnode 的页缓存链表
    3. 调 `sufs_libfs_cmd_read_blk(lba, BLOCK_SIZE, &entry->buffer, &num_io)` —— **触发 NVMe 读**
    4. **忙等**:`while (*((volatile int*)&num_io) == 0) sufs_libfs_cmd_blk_idle();` —— POLLING 模式扫描 CQ 直到回调递增 `num_io`
    5. **命中**:`memcpy(buf + off, entry->buffer.buf + pgoff, len)` 直接拷贝
  - 释放 range lock、inode read unlock

## 四、命令封装层(libfs → AeoTrusted)

- [cmd.c#L165](Aeolia/Aeolia-src/AeoFS/libfs/cmd.c#L165):
  - `cmd_io_callback(data)` —— 回调函数,原子递增 `*(volatile int*)data`(即 `num_io`)
  - `sufs_libfs_cmd_read_blk()` 构造 `tfs_blk_io_arg{lba, count, buf, callback, data}`,调 `tfs_blk_read(arg)`

## 五、可信域层(AeoTrusted,MPK 隔离)

- [trusted-fs.c#L342](Aeolia/Aeolia-src/AeoTrusted/trusted-fs.c#L342):
  - `tfs_blk_read(arg)` → `trampoline_call(__tfs_blk_read, arg)` —— 通过 trampoline 切换 PKRU/栈,进入可信域
  - [__tfs_blk_read](Aeolia/Aeolia-src/AeoTrusted/trusted-fs.c#L173): 用 `TFS_TEST_R` 校验 Lease 是否授予读权限,然后调 `read_blk_async(tfs_tls_ls_nvme_qp(), lba, count, buf, callback, data)`

  - `tfs_blk_idle()` → `trampoline_call(__tfs_blk_idle, 0)` → [block_idle(qp)](Aeolia/Aeolia-src/AeoTrusted/include/tls.h#L40) → `ls_qpair_process_completions(qp, 0)` 扫描 CQ

## 六、I/O 引擎层(libstorage.so → 硬件)

- [io.c#L325](Aeolia/Aeolia-src/AeoDriver/libstorage/io.c#L325) `read_blk_async()`:
  - 构造 `io_request{lba, lba_count, prp1/prp2(指向 dma_buffer->buf), opcode=nvme_cmd_read, callback=cmd_io_callback}`
  - 调 `ls_submit_req(qp, req)`:
    - 在 SQE ring 中找空闲 `iod` 槽
    - 填充 `cmd->rw.{opcode, command_id=iod_id, nsid, prp1, prp2, slba, length}`
    - `writel(qp->sq_tail, qp->sq_db)` —— **写 doorbell 寄存器通知 NVMe 控制器**
- `ls_qpair_process_completions(qp, 0)`(在 `block_idle` 中调用):
  - 轮询 CQ,根据 phase 位判断新 CQE
  - 匹配 `command_id` 找回 `iod`,调 `iod->req->callback(iod->req->data)` —— 即 `cmd_io_callback(&num_io)`,`num_io` 递增跳出忙等循环
  - 更新 CQ doorbell

## sufs_libfs_cmd_blk_idle

AeoFS 的 read 调用栈所有"NVMe 读"都直接调 `read_blk_async`，核心等待点在 [mfs.c#L285](Aeolia/Aeolia-src/AeoFS/libfs/mfs.c#L285)

```c
ret = sufs_libfs_cmd_read_blk(lba, SUFS_FILE_BLOCK_SIZE,
                              &page_cache_entry->buffer, &num_io);
if (ret < 0) ...
while (*((volatile int*)(&num_io)) == 0) {
    sufs_libfs_cmd_blk_idle();   // ← 等待异步结果的地方
}
```

- `num_io` 是栈局部变量(初始为 0),作为 `data` 透传到 NVMe 命令的 `iod->req->data`。
- `cmd_io_callback(void *data) { (*((volatile int *)data))++; }` 递增它。
- 忙等循环里调 `block_idle` → `ls_qpair_process_completions` 主动扫 CQ,扫到 CQE 后调 callback,`num_io` 变 1,循环退出。
- **等待点就是 `while` 循环本身**,即"`sufs_libfs_readm` 函数体内"。

等待的本质:都是 POLLING 忙等

无论哪条路径,等待结果的核心动作都是 `ls_qpair_process_completions(qp, 0)` 主动扫 CQ:

- **AeoFS 文件读**:`sufs_libfs_readm` → `sufs_libfs_cmd_blk_idle` → `block_idle` → 扫 CQ。
- **AeoTrusted 位图/journal**:函数内 `block_idle(qp)` → 扫 CQ。
- **AeoDriver 自测**:`read_blk` → `while` 内 `ls_qpair_process_completions` → 扫 CQ。

由于 [tls.c#L40](Aeolia/Aeolia-src/AeoTrusted/tls.c#L40) 把 `qp->interrupt` 硬编码为 `LS_INTERRPUT_POLLING`,这些 `ls_qpair_process_completions` 调用永远不会进入 UINTR 的 `puir=1` 分支——**当前 AeoFS 全栈实际上不存在 UINTR 等待路径,所有"等待"都是 CPU 自旋扫 CQ**。