
# AeoTrusted 子系统（可信文件系统层）

AeoTrusted(libtrusted.so)是 AeoFS 的"可信域运行时",用 MPK 硬件隔离保护文件系统元数据/位图/journal 等敏感状态,是 AeoFS 与 AeoDriver 之间的"中间层"。

## 文件数据 read/write

### 1. `__tfs_blk_read` / `__tfs_blk_write`

[trusted-fs.c#L173-210](Aeolia/Aeolia-src/AeoTrusted/trusted-fs.c#L173):

```c
static int __tfs_blk_read(unsigned long arg) {
    struct tfs_blk_io_arg *blk_io_arg = (struct tfs_blk_io_arg *)arg;
    int i, tot_bfn;
    tot_bfn = (blk_io_arg->count + SUFS_PAGE_SIZE - 1) / SUFS_PAGE_SIZE;
    // 1. Lease 权限校验:逐页检查 TFS_TEST_R 位图
    for (i = 0; i < tot_bfn; i++)
        if (!TFS_TEST_R((blk_io_arg->lba >> SUFS_PAGE_SHIFT) + i, tfs_state))
            return -1;
    // 2. 直接转发到 libstorage 异步 API
    ret = read_blk_async(tfs_tls_ls_nvme_qp(), blk_io_arg->lba, blk_io_arg->count,
                         blk_io_arg->buf, blk_io_arg->callback, blk_io_arg->data);
    return ret;
}
```

`__tfs_blk_write` 结构完全对称,只是用 `TFS_TEST_W` 检查写权限,调 `write_blk_async`。

**关键点**:
- AeoTrusted 在 read/write 热路径上**不直接做 NVMe I/O**,只是 **(a) 检查 Lease 权限位图 (b) 转发到 libstorage**。
- 真正的 NVMe 命令提交、CQE 轮询都在 libstorage 里;AeoTrusted 只是"门卫 + 转发"。
- 走 `trampoline_call` 进可信域(切栈 + 理论上切 PKRU),每次 IO 都付这个开销。

### 2. `__tfs_blk_idle`(POLLING 入口)

[trusted-fs.c#L212-215](Aeolia/Aeolia-src/AeoTrusted/trusted-fs.c#L212):

```c
static int __tfs_blk_idle(unsigned long arg){
    block_idle(tfs_tls_ls_nvme_qp());   // → ls_qpair_process_completions
    return 0;
}
```

`block_idle` 在 [tls.h#L40](Aeolia/Aeolia-src/AeoTrusted/include/tls.h#L40) 内联展开为 `ls_qpair_process_completions(qp, 0)`,即扫一次 CQ。AeoFS 的 `while (num_io==0) sufs_libfs_cmd_blk_idle()` 每次循环都经过 trampoline 调到这里。

### 3. DMA buffer 管理

[trusted-fs.c#L217-235](Aeolia/Aeolia-src/AeoTrusted/trusted-fs.c#L217):

- `__tfs_create_dma_buffer` → `create_dma_buffer`(libstorage)
- `__tfs_delete_dma_buffer` → `delete_dma_buffer`

AeoFS 读文件时用的 `page_cache_entry->buffer` 就是 AeoTrusted 在文件 open 时调 `tfs_create_dma_buffer` 分配的 NVMe DMA 内存,这样 NVMe 控制器可以直接 DMA 写入,AeoFS 用户态也能直接 `memcpy` 读。

## 索引页/目录项页读取

这部分是 AeoFS 在 `readm` 内部 page cache 未命中时调用的。**注意:这些函数不直接读 NVMe,而是从 AeoTrusted 的内存缓存里 memcpy**。

### 1. `tfs_do_read_fidx_page`(读文件索引页)

[fsop.c#L378-394](Aeolia/Aeolia-src/AeoTrusted/fsop.c#L378):

```c
int tfs_do_read_fidx_page(int inode, unsigned long lba, void *buf) {
    struct tfs_inode_cache *inode_cache = tfs_inode_cache_array[inode];
    struct tfs_fidx_buffer_list *fidx =
        tfs_radix_array_find(inode_cache->fidx_array, lba >> SUFS_PAGE_SHIFT, 0, 0);
    if (fidx == NULL) abort();
    memcpy(buf, fidx->fidx_buffer.buf->buf, SUFS_PAGE_SIZE);   // 直接 memcpy,不读 NVMe
    return 0;
}
```
