
# AeoFS 子系统

fsutils文件系统工具集（cat cp mkdir mv ls rm echof dummy ftruncate touch）

## Aeolia-src/AeoFS/kfs

kfs 是 AeoFS 的内核辅助模块，生成sufs.ko

用户态 libfs 做不到的事情（特权操作）全部下沉到这里，kfs 内核模块完全绕开 AeoDriver（直连主线 NVMe）

用户态 libfs 负责数据面高频读写，内核 kfs 负责格式化/分配/授权等低频控制面。


```c
 用户态 (libsupremefs.so)
    │ ioctl(/dev/supremefs, cmd, arg)
    ▼
 ┌─────────────────────────── kfs (sufs.ko) ──────────────────────────┐
 │ device.c: sufs_kfs_ioctl() switch → 路由到 10+ 子模块               │
 │                                                                     │
 │  inode.c  inode 分配/回收 + 读写 sinode(shadow inode)到 NVMe        │
 │  balloc.c block 分配/回收（per-CPU 红黑树空闲区间）                 │
 │  lease.c  按 inode 粒度的 READ/WRITE Lease(最多 4 持有者, 5ms 租期) │
 │  mmap.c   文件映射 / Delegation 映射 / chmod/chown 占位             │
 │  super.c  布局计算、格式化、root inode 初始化、fs_init/fini         │
 │  dev_nvme.c 找 NVMe PCI dev + per-CPU DMA buffer / PRP list          │
 │  pagecache.c 读缓存 chunk 分配 & mmap 到 0x3F0000000000             │
 │  ring.c   Lease Ring / Mapped Ring 在固定地址的共享内存              │
 │  util.h   __sufs_kfs_send_nvme_rw() 直接用 nvme_submit_sync_cmd     │
 └─────────────────────────────────────────────────────────────────────┘
 ```

## Aeolia-src/AeoFS/libfs

生成sufs.so，

- entry.c	- constructor 初始化；通过 dlsym 保存 glibc 原版 syscall；重写 open / open64 / __open64_2 / opendir / readdir / closedir / read / write / lseek / close / fstat / ... 等几十个 POSIX 符号。每个符号都先判断路径是否 /sufs/* ，是则走 sufs_libfs_sys_*，否则走原版 glibc。

- syscall.c	- 23 个 sufs_libfs_sys_*（模拟原本的 syscall，不过现在最终调用用户态的AeoDriver）：openat / close / read / pread / pwrite / write / lseek / fstat / lstat / stat / rename / mkdirat / unlink / chown / chmod / ftruncate / chdir / readdir / getdents / fsync / fdatasync。负责参数校验 + 路径解析(namei) + 调 mfs/mnode/ cmd 实际干活。

## sufs_libfs_sys_openat

sufs_libfs_sys_openat 最后会调用 AeoTrusted提供的 tfs_alloc_inode_in_directory

```c
用户 test_rw: open("/sufs/test", O_CREAT|O_RDWR, 0644)
 │ （glibc 调 open → 已经被 sufs.so 的符号 hook，不 syscall）
 ▼
entry.c: open()
 ├─ sufs_libfs_upath_to_lib_path() → 命中 /sufs/*  用户态，无 syscall
 ▼
sufs_libfs_sys_openat(proc, dirfd=AT_FDCWD, newpath, O_CREAT|O_RDWR, 0644)
 [syscall.c#L333]
 ├─ 1. cwd_m = proc->cwd_m (root mnode = /sufs/)                  纯内存
 ├─ 2. sufs_libfs_namei(cwd_m, "test") → 在 chainhash 找有无同名 mnode  纯内存
 ├─ 3. O_CREAT + 没找到 → 调 mnode 创建子 mnode（dir 下挂 dentry）
 │       sufs_libfs_create() → sufs_libfs_cmd_alloc_inode_in_directory()
 │          └─ cmd.c#L83 → tfs_alloc_inode_in_directory(&arg)
 │                └─ trampoline_call(__tfs_alloc_inode_in_directory, arg)
 │                      ├─ MPK wrpkru（用户态寄存器，无 syscall）
 │                      ├─ 栈切换（纯寄存器操作）
 │                      ├─ 可信域: 查本线程 inode 段缓存够不够？
 │                      │       够 → 原子位分配 inode 号 
 │                      │       不够 → tfs_cmd_alloc_inodes()
 │                      │              └─ syscall(SYS_ioctl, dev_fd, SUFS_CMD_ALLOC_INODE, &entry)
 │                      │                  这一步才真正陷入 kfs sufs.ko
 │                      │                 （从内核 per-CPU rbtree 大段分配 inode 号段）
 │                      ├─ 调 tfs_do_alloc_inode_in_directory() 写 bitmap + dentry
 │                      └─ return ino_num
 ├─ 4. sufs_libfs_map_file(m, writable=O_RDWR ? 1:0)
 │       → cmd_map_file → tfs_mmap_file → ioctl(SUFS_CMD_MAP) → sufs.ko
 │        Lease/Delegation 申请（可能进内核 1 次）
 ├─ 5. 建 struct sufs_libfs_file_mnode{mnode, off=0, flags, refcnt}
 ├─ 6. filetable_allocfd(ftable, f, percpu=true, cloexec=false)
 │       → per-CPU slot CAS 分配 fd（纯原子）
 └─ 返回 fd + 1048576
```
### AeoFS 和 AeoTrusted的关系

AeoFS/libfs（外层） = syscall hook + VFS 仿真层。收到 open()/read()/write() 后做路径解析、dentry 查找、锁、偏移计算；然后把 I/O/分配等"特权"操作交给 AeoTrusted。

AeoTrusted（内层） = 可信执行域（Trusted Computing Base）。收到参数后做权限校验、元数据实际落盘、调 AeoDriver 发 NVMe 异步 I/O、管理 ialloc/balloc/journal 等。

两者之间通过一个精心设计的 MPK(Memory Protection Keys) + stack trampoline 机制进入"保护域"

AeoFS libfs 的 "下原语"层是 cmd.c（libfs/include/cmd.h 声明了 19 个 sufs_libfs_cmd_*）。这些函数全部是薄封装：把参数打包成 AeoTrusted 的 arg 结构体，然后调用对应的 tfs_*()。

libfs 外层函数	→	直接调用 AeoTrusted API
sufs_libfs_cmd_read_blk(lba, size, buf, data)	→	tfs_blk_read(&tfs_blk_io_arg{lba,count,cb,data,buf})
sufs_libfs_cmd_write_blk(lba, size, buf, data)	→	tfs_blk_write(...)
sufs_libfs_cmd_map_file(ino, writable, *index)	→	tfs_mmap_file(&tfs_map_arg)


```
            ┌──────────────────────────────────────────────────────┐
            │                    AeoFS 应用层                       │
            │   test_create / test_rw / ls、cp、rm、cat 等 fsutils  │
            └───────────────┬──────────────────────────────────────┘
                            │ 链接 sufs.so（constructor hook syscall）
                            ▼
            ┌──────────────────────────────────────────────────────┐
            │   libfs + AeoTrusted（用户态）                        │
            │                                                      │
            │   open/read/write/mkdir/unlink → sufs_libfs_sys_*   │
            │         │                                            │
            │         └──► readm / writem (mfs.c)                 │
            │                   │                                  │
            │                   └──► cmd.c: sufs_libfs_cmd_rw     │
            │                          │                           │
            │                          ▼                           │
            │              AeoTrusted trusted-fs.c:                │
            │              tfs_blk_read / tfs_blk_write            │
            │                    (tfs_tls_ls_nvme_qp)              │
            │                          │                           │
            │          ┌───────────────┴───────────────┐           │
            │          │  AeoTrusted/tls.c 初始化 QP:  │           │
            │          │  open_device("dev/nvme0")     │           │
            │          │  create_qp(POLLING 模式)      │           │
            │          │  /dev/ls_nvme/nvme0instanceX  │           │
            │          └───────────────┬───────────────┘           │
            └──────────────────────────┼───────────────────────────┘
                                       │ 调用 libstorage API
                                       ▼
            ┌──────────────────────────────────────────────────────┐
            │           AeoDriver（用户态 + 内核 ko）               │
            │   libstorage.so: open_device / create_qp             │
            │                 read_blk_async / write_blk_async     │
            │                 ls_qpair_process_completions         │
            │                                                      │
            │   libstorage_kernel.ko: /dev/libdriver               │
            │     /dev/ls_nvme/nvme0cq0 cq sq db 字符设备 mmap     │
            │     UINTR 机制 / MSI-X 向量分配 / UPID page          │
            └─────────────────────────────────────────────────────┘
                         
                         
            ┌──────────────────────────────────────────────────────┐
            │   kfs sufs.ko（此路径不经过 AeoDriver）           │
            │                                                      │
            │   dev_nvme.c: pci_get_class(0x010802)               │
            │      └─► 拿主线 nvme.ko 的 struct nvme_dev/ns        │
            │                                                      │
            │   util.h: __sufs_kfs_send_nvme_rw()                 │
            │      └─► 手动构造 NVMe cmd + nvme_submit_sync_cmd    │
            │                                                      │
            │   inode.c: 写回 shadow inode                        │
            │   super.c: 格式化清零 sinode/bitmap                  │
            └────────────┬─────────────────────────────────────────┘
                         │ 直接调主线 nvme.ko 导出符号
                         ▼
                    NVMe 硬件 SSD
```
