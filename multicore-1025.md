## 网络通信

| 调用 | 作用 |
|---|---|
| socket | 创建套接字，每个请求 1 个 |
| bind | 绑定地址端口 |
| listen | 监听 |
| accept | 接收连接 |
| connect | 非阻塞连接 |
| setsockopt | 设置socket属性 |
| getsockopt | 非阻塞 connect 后查连接结果 |
| read | 读 HTTP 响应 |
| write | 发 HTTP 请求 |
| close | 关闭连接 |


## 运行时

| 调用 | 作用 |
|---|---|
| epoll_create | 创建epoll |
| epoll_ctl | 增删改 fd 在 epoll 实例中的监听关系 |
| epoll_wait | 从 epoll 实例的就绪链表中取出已经就绪的 fd 事件列表 |


## 进程生命周期

| 调用 | 功能 |
|------|------|
| execve | 用新程序替换当前进程映像 |
| arch_prctl | 设置/读取特定于架构的线程状态 |
| mmap | 在进程虚拟地址空间中映射文件或匿名内存页；用于分配堆、线程栈、映射 vdso、加载共享库等 |
| mprotect | 修改一段已映射内存页的访问权限 |
| rt_sigaction | 为指定信号注册/修改处理函数 |
| rt_sigprocmask | 屏蔽信号 |
| prlimit64 | get/set 进程级资源限制 |
| openat | 打开文件/目录并返回 fd；启动阶段用于动态链接器加载 .so、读 /proc/self/exe、打开配置文件 |
| statx / fstat | get/set 进程级资源限制 |
| set_tid_address | 线程退出通知 |
| set_robust_list | 注册 robust futex 链表；线程异常退出时内核遍历链表自动释放该线程持有的 robust mutex，防止死锁 |
| exit_group | 终止进程内所有线程并回收资源 |

### 线程生命周期

```
std::thread::spawn(closure)
  └─ std::sys::unix::thread::Thread::new
       ├─ Box::new(包住 closure 和 Thread 结构)  # Rust 的线程参数传递
       ├─ pthread_attr_setstacksize(2MB)
       ├─ pthread_create(thread_start, packet)   # NPTL，用户态
       │    ├─ mmap 栈 / stack cache
       │    ├─ _dl_allocate_tls
       │    ├─ rt_sigprocmask
       │    └─ clone3(...)                       
       ├─ (子线程) prctl(PR_SET_NAME)            
       └─ (子线程) 握手：写 packet → futex wake   # 父线程 spawn 返回
```

| # | 调用 | 作用 |
|---|---|---|
| 1 | mmap | 预留线程栈虚拟空间 |
| 2 | mprotect | 栈顶初始可写页 |
| 3 | clone3 | 内核创建线程 |
| 4 | rt_sigprocmask | clone 后恢复信号掩码 |
| 5 | rseq | 注册 restartable sequences |
| 6 | set_robust_list | 注册 robust futex 链表 |
| 7 | rt_sigprocmask | 设置线程信号掩码 |
| 8 | sched_getaffinity | 查 CPU 亲和性 |
| 9 | sigaltstack  | 查备用信号栈现状 |
| 10 | sigaltstack | 信号栈 |
| 11 | munmap 12KB | 释放备用栈 |
| 12 | rt_sigprocmask 屏蔽 glibc 内部信号，准备退出 |
| 13 | madvise  | 释放 2MB 栈物理页  |
| 14 | futex | 子线程还没退出时父线程在此睡眠，子线程退出后内核唤醒 |

## 裸 clone3

如果自己实现线程创建，那么只需要调用clone3和mmap，mprotect，munmap

| syscall | 说明 |
|---|---|
| clone3 | 生成线程 |
| mmap | 2MB 栈、glibc arena、 加载器 |
| mprotect | 栈保护页、arena 提交页 |
| munmap | 栈回收、arena 裁剪 |

```bash
5次请求，每次请求32条数据

原始std::thread::spawn 方案

./tools/bench/syscall.sh thread   ./target/bench/thread/release/crawler-daemon    32 5

--- syscall 明细（strace -c 原始表，按耗时降序）---
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ------------------
 16.18    0.008952          55       160       160 connect
 15.88    0.008783          26       326           write
  8.69    0.004808          14       342           epoll_ctl
  8.69    0.004807          13       353        10 read
  8.16    0.004514          28       161           socket
  7.09    0.003921          22       174           close
  4.88    0.002697          36        74           mmap
  4.69    0.002597          19       132           sigaltstack
  4.37    0.002418          15       160           getsockopt
  3.63    0.002006          11       173           rt_sigprocmask
  3.60    0.001990          45        44           rseq
  2.93    0.001620          37        43           clone3
  2.75    0.001519          34        44           set_robust_list
  2.43    0.001343          20        65           mprotect
  2.08    0.001153          22        51           munmap
  1.35    0.000748          16        45           sched_getaffinity
  0.86    0.000477          28        17         2 futex
  0.78    0.000433          10        43           madvise
  0.73    0.000403          17        23         1 epoll_wait
  0.15    0.000081           7        11         6 accept4
  0.02    0.000011           2         4           brk
  0.01    0.000008           8         1           bind
  0.01    0.000007           7         1           epoll_create1
  0.01    0.000006           6         1           listen
  0.01    0.000005           0         8           rt_sigaction
  0.01    0.000005           5         1           setsockopt
  0.01    0.000004           2         2           getrandom
  0.00    0.000000           0         4           fstat
  0.00    0.000000           0         1           poll
  0.00    0.000000           0         4           lseek
  0.00    0.000000           0         1         1 rt_sigreturn
  0.00    0.000000           0         2           pread64
  0.00    0.000000           0         1         1 access
  0.00    0.000000           0         1           execve
  0.00    0.000000           0         1           arch_prctl
  0.00    0.000000           0         1           set_tid_address
  0.00    0.000000           0         9         1 openat
  0.00    0.000000           0         2           prlimit64
  0.00    0.000000           0         5           statx
------ ----------- ----------- --------- --------- ------------------

 ./tools/bench/syscall.sh thread   ./target/bench/raw/release/crawler-daemon    32 5
=== thread: burst=32 rounds=5 requests=160 total_syscalls=1960 ===
每请求: 12.25 次
--- syscall 明细（strace -c 原始表，按耗时降序）---
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ------------------
 20.83    0.008615          53       160       160 connect
 19.67    0.008133          24       326           write
 11.39    0.004710          26       178           close
 11.12    0.004600          12       363        16 read
 11.12    0.004598          13       348           epoll_ctl
  9.70    0.004013          24       161           socket
  3.80    0.001570           9       160           getsockopt
  3.06    0.001264          35        36           clone3
  2.60    0.001076          39        27           munmap
  2.38    0.000984          18        54           mmap
  2.34    0.000967          21        45           mprotect
  1.23    0.000508          18        28         1 epoll_wait
  0.44    0.000184          16        11         6 accept4
  0.08    0.000032           2        13         1 openat
  0.06    0.000023           4         5           statx
  0.04    0.000015           2         7           rt_sigaction
  0.02    0.000010           2         4           brk
  0.02    0.000010           5         2           sched_getaffinity
  0.02    0.000009           3         3           sigaltstack
  0.02    0.000008           2         4           lseek
  0.01    0.000005           5         1           bind
  0.01    0.000005           5         1           epoll_create1
  0.01    0.000004           4         1         1 rt_sigreturn
  0.01    0.000004           4         1           listen
  0.01    0.000003           3         1           setsockopt
  0.01    0.000003           1         2           getrandom
  0.00    0.000000           0         4           fstat
  0.00    0.000000           0         1           poll
  0.00    0.000000           0         2           pread64
  0.00    0.000000           0         1         1 access
  0.00    0.000000           0         1           execve
  0.00    0.000000           0         1           arch_prctl
  0.00    0.000000           0         3         1 futex
  0.00    0.000000           0         1           set_tid_address
  0.00    0.000000           0         1           set_robust_list
  0.00    0.000000           0         2           prlimit64
  0.00    0.000000           0         1           rseq
------ ----------- ----------- --------- --------- ------------------
```