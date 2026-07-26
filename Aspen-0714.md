# Aspen实验复现

完成代码的适配

https://gitee.com/zhuweipu/aspen/commit/b6380e3ee3176b0841494a4711bb002116931d2a


```bash
sudo apt install make gcc cmake pkg-config libnl-3-dev libnl-route-3-dev libnuma-dev uuid-dev libssl-dev libaio-dev libcunit1-dev libclang-dev libncurses-dev meson python3-pyelftools

make submodules

make clean && make

# 构建 ksched 内核模块并加载
pushd ksched
make clean && make
popd
sudo ./scripts/setup_machine.sh

# 构建 C++ 绑定和 shim（若只需构建 Rust 应用可跳过）
pushd bindings/cc
make clean && make
popd

pushd shim
make clean && make
popd

curl https://sh.rustup.rs -sSf | sh
rustup default nightly

pushd apps/synthetic
cargo clean
cargo update
cargo build --release
popd
```

```bash
# 终端 1：启动 iokerneld
sudo ./iokerneld simple noht nobw

# 终端 2：启动 local-client
./apps/synthetic/target/release/synthetic 192.168.31.100:5000 --config server.config --mode local-client
```

```
sudo ./iokerneld simple noht nobw
CPU 07| <5> cpu: detected 16 cores, 1 nodes
CPU 07| <5> time: detected 3686 ticks / us
[  0.000535] CPU 07| <5> sched: CPU configuration...
        node 0: [0][1][2][3][4][5][6][7][8][9][10][11][12][13][14][15]
[  0.000546] CPU 07| <5> sched: dataplane on 1, control on 0
[  0.042916] CPU 07| <5> control: spawning control thread
[  0.042973] CPU 07| <5> dpdk arg:
./iokerneld -l 1 --socket-mem=128 --vdev=net_tap0 
EAL: Detected CPU lcores: 16
EAL: Detected NUMA nodes: 1
EAL: Detected static linkage of DPDK
EAL: Multi-process socket /var/run/dpdk/rte/mp_socket
EAL: Selected IOVA mode 'VA'
TELEMETRY: No legacy callbacks, legacy socket not created
[  0.123476] CPU 01| <5> dpdk: driver: net_tap port 0 MAC: ba a3 6f b8 df 55
[  0.123499] CPU 01| <5> main: core 1 running dataplane. [Ctrl+C to quit]
[8451.542151] CPU 01| <2> Duplicate IP address detected.

 ./apps/synthetic/target/release/synthetic 192.168.31.100:5000 --config server.config --mode local-client
CPU 12| <5> cpu: detected 16 cores, 1 nodes
[  0.000361] CPU 12| <5> loading configuration from 'server.config'
[  0.000379] CPU 12| <5> cfg: provisioned 4 cores (4 guaranteed, 0 burstable, 4 spinning)
[  0.000381] CPU 12| <5> cfg: task is latency critical (LC)
[  0.000382] CPU 12| <5> cfg: THRESH_QD: 10, THRESH_HT: 0 THRESH_QUANTUM: 100
[  0.000383] CPU 12| <5> cfg: storage disabled, directpath disabled
[  0.000387] CPU 12| <5> process pid: 3894828
[  0.024607] CPU 12| <5> shm: using 270532608 bytes
[  0.024623] CPU 12| <5> net: started network stack
[  0.024625] CPU 12| <5> net: using the following configuration:
[  0.024626] CPU 12| <5>   addr:        192.168.31.100
[  0.024628] CPU 12| <5>   netmask:     255.255.255.0
[  0.024628] CPU 12| <5>   gateway:     192.168.31.1
[  0.024629] CPU 12| <5>   mac:         BA:A3:6F:B8:DF:55
[  0.024631] CPU 12| <5>   mtu:         1500
[  0.024759] CPU 12| <5> spawning 4 kthreads
Distribution, Target, Actual, Dropped, Never Sent, Median, 90th, 99th, 99.9th, 99.99th, Start, StartTsc
zero, 1000, 0, 0, 0, 1784022301
zero, 2000, 0, 0, 0, 1784022314
zero, 3000, 0, 0, 0, 1784022327
zero, 4000, 0, 0, 0, 1784022340
zero, 5000, 0, 0, 0, 1784022354
zero, 6000, 0, 0, 0, 1784022367
zero, 7000, 0, 0, 0, 1784022380
zero, 8000, 0, 0, 0, 1784022393
zero, 9000, 0, 0, 0, 1784022406
zero, 10000, 0, 0, 0, 1784022419
zero, 11000, 0, 0, 0, 1784022432
zero, 12000, 0, 0, 0, 1784022445
zero, 13000, 0, 0, 0, 1784022458
zero, 14000, 0, 0, 0, 1784022472
zero, 15000, 0, 0, 0, 1784022485
zero, 16000, 0, 0, 0, 1784022498
zero, 17000, 0, 0, 0, 1784022511
zero, 18000, 0, 0, 0, 1784022524
zero, 19000, 0, 0, 0, 1784022537
zero, 20000, 0, 0, 0, 1784022550
[262.135851] CPU 04| <5> init: shutting down -> SUCCESS
```

```bash
# 终端 1：启动 iokerneld
sudo ./iokerneld simple noht nobw

# 终端 2：启动 server（使用 server.config，IP 192.168.31.100）
./apps/synthetic/target/release/synthetic 192.168.31.100:5000 --config server.config --mode spawner-server

# 终端 3：启动 client（使用 server.config，IP 192.168.31.100）
./apps/synthetic/target/release/synthetic 192.168.31.100:5000 --config server.config --mode runtime-client
```

## 代码分析

```c
Aspen (主项目)
├── Aspen-KB: 内核旁路用户态运行时
│   ├── iokernel/     # 调度器核心
│   ├── ksched/       # 内核模块
│   ├── runtime/      # 用户态运行时
│   ├── apps/         # 应用层
│   └── breakwater/   # DataFrame 应用
```

### IOKernel 层（调度器核心）

```c
iokernel/main.c          # iokerneld 入口，启动流程
iokernel/sched.c         # 调度策略（EDF/P-Fair）
iokernel/control.c       # 控制平面，管理 runtime 注册
iokernel/rx.c            # 网络接收路径
iokernel/tx.c            # 网络发送路径
iokernel/dpdk.c          # DPDK 网卡驱动封装
```

运行模式 simple 模式
简化版调度器，来自 Caladan。README 中提到：
 Since Aspen does not require Caladan's methods for addressing CPU interference, this simple mode is sufficient for experimentation.
核心逻辑 ：

- 每个 runtime 独占一组 CPU 核心
- 不处理跨 runtime 的 CPU 干扰
- 不支持动态核心分配

main线程与控制线程的同步方式-pthread_barrier_wait
```
主线程 (main.c)                控制线程 (control.c)
      │                            │
      │  pthread_barrier_init(..., 2)  ← 初始化 barrier，需要 2 个线程到达
      │                            │
      │  run_init_handlers()       │
      │  ├── 初始化 DPDK           │
      │  ├── 初始化调度器          │
      │  └── 初始化网络栈          │
      │                            │
      │                            │  control_thread() 启动
      │                            │  ├── 创建 epoll_fd
      │                            │  └── 注册事件
      │                            │
      │  pthread_barrier_wait()    │  pthread_barrier_wait()
      │  ──────────────────────────┼────────────────────────
      │                            │  两个线程都到达 barrier
      │                            │
      │  dataplane_loop()          │  epoll_wait() 主循环
      │  ← 数据平面开始运行        │  ← 控制平面开始运行
```

主线程初始化各种模块，init_entry展开得到各种init
```
main()
    │
    └── run_init_handlers("iokernel", iok_init_handlers, ARRAY_SIZE(...))
            │
            ├── base_init()        ← 基础库
            ├── ksched_init()      ← 内核模块
            ├── sched_init()       ← 调度器核心
            ├── simple_init()      ← 调度模式
            ├── numa_init()
            ├── ias_init()
            ├── control_init()     ← 控制平面（创建并启动控制线程）
            ├── dpdk_init()        ← 数据平面
            ├── rx_init()
            ├── tx_init()
            ├── dp_clients_init()
            ├── dpdk_late_init()   ← DPDK 端口初始化
            ├── directpath_init()  ← 直接路径（可选）
            └── hw_timestamp_init()← 硬件时间戳（可选）
```

#### TLS

Aspen 在base_init实现了自定义的线程本地存储（TLS）机制。
每个线程单独申请一块匿名的内存，通过全局定义一个链接器label地址__perthread_start，计算出偏移量赋给gs寄存器，然后所有的线程都可以通过__perthread_start访问TLS。DEFINE_PERTHREAD 定义的TLS变量链接时候都会保存在perthread section，加载的时候不使用标准的glibc 的 FS 寄存器机制，变量存在 gs:[链接地址]的内存。

#### control线程

main线程在control_init 中创建两块共享内存，并分别用INGRESS_MBUF_SHM_KEY和IOKERNEL_INFO_KEY标识 代表网络数据包缓冲区（mbuf 池）和IOKernel 配置信息。通过UDS发送 runtime 的共享内存KEY，方便建立IOkernel和Runtime的连接。

将UDS和eventfd两个fd注册到epoll_fd的EPOLL_CONTROLFD_COOKIE和EPOLL_EFD_COOKIE两个事件上面，UDS代表新的runtime和 control 线程的连接请求，eventfd则是 LRPC的请求。

通过control_init_dataplane_comm 创建 LRPC 共享内存轻量RPC，contrl plane和data plane通过LRC通信。（后续也会创建IO kernel和runtime的 LRPC通道）。

主线程随后创建新的线程作为控制线程，控制线程从control_thread开始运行，control_thread绑定到固定的核是运行，最后进入control_loop，epoll_wait 等待事件。比如

EPOLL_CONTROLFD_COOKIE -- runtime通过uds发起的连接请求。连接成功后，在control_add_client中调用control_create_proc执行映射内存，创建pcb等代码

```
control_create_proc(key, len, pid)
    │
    ├── 1. 映射 runtime 的共享内存 通过 runtime 发来的 key ，映射同一块共享内存。这块内存是 runtime 创建的，包含所有通信队列。
    │
    ├── 2. 读取 control header，校验版本 读取共享内存头部的控制信息，校验 magic number 和版本号，确保 IOKernel 和 runtime 编译自同一份代码。
    │
    ├── 3. 创建 struct proc 结构体 创建 struct proc ，这是 IOKernel 中代表一个 runtime 进程的核心数据结构。
    │
    ├── 4. 为每个 kthread 初始化 3 个 LRPC 队列 为每个 kthread 初始化 3 个 LRPC 队列 ，全部在共享内存中：
    │
    ├── 5. 记录物理页地址 获取共享内存所有页的 物理地址 ，用于 DPDK 零拷贝网络：
    │
    └── 6. 初始化 overflow 队列 当 runtime 的发送队列满了时，IOKernel 用 overflow 队列暂存多余的 mbuf 地址。
```

| 队列 | 方向 | 用途 | 数据内容 |
|------|------|------|----------|
| **rxq** | IOKernel → Runtime | 通知有数据包到达 | mbuf 物理地址 |
| **txpktq** | Runtime → IOKernel | 请求发送数据包 | mbuf 物理地址 |
| **txcmdq** | Runtime → IOKernel | 发送控制命令 | 命令类型 + 参数 |


EPOLL_EFD_COOKIE -- 数据平面给控制平面的消息通知，控制平面被这个事件唤醒后，读取lrpc_data_to_control，目前唯一支持的命令是CONTROL_PLANE_REMOVE_CLIENT，runtime 断开连接的 最后一步 ：数据平面清理完自己的资源后，通知控制平面可以安全地销毁 struct proc。

调用dpdk 库的api初始化

#### dataplane_loop

主线程最后执行dataplane_loop，变成data plane专用线程， 轮询处理数据包和调度任务。数据平面线程 永远不睡眠 ，100% 占用一个 CPU 核心。这看起来"浪费 CPU"。

优先级 ：数据平面工作（rx/tx/commands）> 控制平面工作（LRPC 控制消息）控制平面消息（runtime 注册/注销）频率很低（毫秒级），数据包频率很高（微秒级），所以这种优先级保证了热路径的延迟。

1. rx_burst() — 收取数据包
- 调用 DPDK rte_eth_rx_burst 从网卡 RX 队列批量收取数据包
- 解析包头（IP、UDP/TCP），根据 flow 分发到对应 runtime 的 LRPC rxq 队列

2. sched_poll() — 调度决策
- Caladan 抢占式调度的核心
- 检查每个 runtime 的负载、优先级、队列长度
- 决定是否需要抢占某个 kthread 的 CPU 核心，分配给更高优先级的 runtime
- 通过 ksched 内核模块（mwait + 共享内存）通知 kthread 切换

sched_poll 收集各个核上面的拥塞信心，决定是否要调度新的kthread运行，是否发送抢占信息，退出前调用 /dev/ksched 设备的ioctl函数ksched_ioctl()，并调用smp_call_function_many 内核api发送IPI。

### Runtime 层（用户态运行时）

```c
runtime/init.c           # runtime 初始化
runtime/sched.c          # 用户态线程调度
runtime/kthread.c        # kthread 管理（绑定到 CPU）
runtime/net/tcp.c        # TCP 协议栈
runtime/preempt.c        # 抢占机制
runtime/timer.c          # 定时器
```

一个runtime就是一个程序，里面可以有多个kthread内核线程和多个uthread用户线程。

应用程序需要静态链接libruntime.a，然后在main中调用runtime_init。

Runtime 功能总览

核心调度

| 子系统 | 源文件 | 头文件 | 功能 |
|---|---|---|---|
| **kthread** | [runtime/kthread.c](file:///home/zwp/aspen/runtime/kthread.c) | — | 内核级线程管理，每个 kthread 绑定一个 CPU 核 |
| **ioqueues** | [runtime/ioqueues.c](file:///home/zwp/aspen/runtime/ioqueues.c) | — | 与 IOKernel 通信的共享内存队列，收发网络/存储包 |
| **sched** | [runtime/sched.c](file:///home/zwp/aspen/runtime/sched.c) | — | 用户态线程调度器，在 kthread 上调度 uthread |

用户态线程（user-level thread）

轻量级用户态线程：

```c
thread_t *thread_create(thread_fn_t fn, void *arg);   // 创建线程
int       thread_spawn(thread_fn_t fn, void *arg);    // 创建并启动
void      thread_yield(void);                          // 主动让出 CPU
void      thread_exit(void);                           // 退出当前线程
thread_t *thread_self(void);                          // 获取当前线程
void      thread_ready(thread_t *t);                  // 唤醒线程
```

#### runtime_init

runtime_init 接收真正的应用程序main函数运行。初始化TLS。


根据配置文件，创建kthread。解析配置文件，maxks代表runtime最大可创建的kthread数量。

在sched_init_thread中创建了0号 uthread，继续运行runtime_init，最后0号uthread会调用sched_start，成为调度器uthread。

在timer_init_thread()  创建 timer_softirq uthread

调用pthread_create 创建线程，入口函数为pthread_entry，每个kthread上面有单独创建的 timer_softirq uthread

runtime init调用ioqueues_register_iokernel 通过UDS 连接io kernel。

调用thread_spawn_main 创建uthread，负责执行真正的main函数，并设置为主thread。

最后late_init_handlers 调用uintr_init_late 创建 uintr_timer pthread，启动抢占计时。

所有的kthread启动后，最终都进入sched_start，跳转到schedule_start的同时切换到runtime 栈，然后在kthread_wait_to_attach调用ioctl像 /dev/sched 发送KSCHED_IOC_START请求。

ksched_ioctl对于KSCHED_IOC_START请求，则调用ksched_start，设置线程状态为TASK_INTERRUPTIBLE，并schedule。

#### schdule

thread_spawn_main 会根据 init时候传入的真正的main，创建一个uthread。thread_create 分配出一个thread_t 结构，设置uthread的堆栈和寄存器。thread_ready则将其放入当前kthread 0的rq队列。IOKernel检测到 runqueue 有积压,simple_notify_congested 检测到 parked_thread_busy == true → simple_add_kthread → ksched_run → 唤醒 kthread 0
```c
int thread_spawn_main(thread_fn_t fn, void *arg)
{
	th = thread_create(fn, arg);
	thread_ready(th);
	return 0;
}
```

所有的kthread创建初始，都通过kthread_wait_to_attach iotcl阻塞了。被唤醒后执行schedule。


```c
static __noreturn void schedule_start(void)
{
	struct kthread *k = myk();
	k->parked = false;
	schedule();
}
```

schedule() — 调度器核心主循环，清理旧 uthread + 抢占检查。

```
uthread A 运行中
    │
    ├─ timer 到期 → 信号抢占 → handle_sigusr2()
    │     └─ thread_yield()                         ← A 被触发切出
    │           └─ enter_schedule(A)
    │                 ├─ 保存 A 的寄存器到 A->tf     ← 真正的"保存"
    │                 ├─ thread_ready(A)              ← A 放回就绪队列
    │                 └─ jmp_runtime → schedule()     ← 切 runtime 栈
    │
    ▼
schedule():                                          ← 此时 A 已经保存完毕
    ├─ __self->thread_running = false   ← 只是改个 bool
    ├─ __self = NULL                     ← 只是清个指针
    └─ 选下一个 uthread → 跳转
```

l->rq_head != l->rq_tail 表示 本地就绪队列不为空 ，有 uthread 等待运行，从队列头拿到任务jmp_thread(th)运行。

如果队列为空，从其他 kthread 偷 uthread。先偷 HT 兄弟的（cache 局部性好），再随机遍历所有 kthread。

没有任务则调用 kthread_park()休眠。



被抢占（cede） IOKernel 发 SIGUSR1 要收回核心时，thread_ready_head(myth);    // 被抢占的任务放回就绪队列头部

主动 yield thread_ready(th);           // 任务放回尾部


```
handle_sigusr1：thread_cede():   uthread 保存 → kthread park → 核心归还 IOKernel
                     ↑                          ↑
                  uthread 层面              kthread 层面

handle_sigusr2：thread_yield():  uthread 入队 → 选下一个 uthread → 继续运行
                     ↑
                 仅 uthread 层面（kthread 不释放核心）
```


| | `thread_cede` | `thread_yield` |
|---|---|---|
| uthread | 保存并放回就绪队列头部 | 保存并放回就绪队列尾部 |
| kthread | **park（阻塞），归还核心给 IOKernel** | 继续运行，不归还核心 |
| 触发 | SIGUSR1（IOKernel 要收回核心） | SIGUSR2 或主动 `thread_yield()` |
| 核心使用权 | **丢失**（核心可能被分配给其他 runtime） | **保留**（仍在当前 runtime） |

所以 `thread_cede` 是**双重让出**：当前 uthread 暂停运行，同时整个 kthread 把 CPU 核心交还给 IOKernel 重新分配。

### uintr timer

每个 runtime 进程都有自己独立的 uintr_timer 线程

用户态抢占（preemption）的三种机制 ：UINTR 硬件中断、Concord 标志、Signal。核心是一个独立的 timer pthread ，定期检查每个 kthread 是否需要被抢占。

定时器线程 uintr_timer 是一个 独立的 Linux pthread （不是 uthread），绑定到专用的 timer_core CPU。

```
timer_core 上的 pthread 主循环:
   │
   ├─ 遍历所有 kthread (i = 0..maxks-1)
   │     │
   │     ├─ 当前 uthread 运行时间未超 TIMESLICE → 跳过
   │     │
   │     └─ 超时 → 发送抢占信号:
   │           ├─ UINTR_PREEMPT:  _senduipi() → 硬件用户中断
   │           ├─ CONCORD_PREEMPT: 写标记位
   │           └─ SIGNAL_PREEMPT:  pthread_kill(SIGUSR1)
   │
   └─ loop 继续

              timer_core (独立 pthread)
              ┌─────────────────────────┐
              │  uintr_timer() 主循环    │
              │  rdtsc() 读取时间        │
              │  遍历 maxks 个 kthread  │
              │  超 TIMESLICE?           │
              │    ├─ UINTR: _senduipi() │
              │    ├─ Concord: 写标志    │
              │    └─ Signal: pthread_kill│
              └──────┬──────────────────┘
                     │
    ┌────────────────┼────────────────┐
    ▼                ▼                ▼
 kthread 0      kthread 1      kthread N-1
    │
    ├─ UINTR → ui_handler → thread_yield()
    ├─ Signal → signal_handler → thread_yield()
    └─ Concord → 检查标志 → thread_yield()
```

```c
// UINTR 的处理函数
ui_handler(...) {
    if (preempt_enabled()) {
        thread_yield();        // 让出当前 uthread
    } else {
        set_upreempt_needed(); // 临界区中，标记延迟处理
    }
}
```
uintr_timer 是一个 独立的 Linux pthread ，绑在 timer_core 上， 持续运行不死循环 ：
```c
void* uintr_timer(void*) {
    while (uintr_timer_flag != -1) {
        current = rdtsc();                        // 不停读 TSC
        for (i = 0; i < maxks; ++i) {
            // 检测 uthread 是否切换了
            long long start_ts = ks[i]->uthread_start_ts;
            if (last_check[i] < start_ts)
                last_check[i] = start_ts;         // 新 uthread → 重置计时

            if (current - last_check[i] < TIMESLICE)
                continue;                          // 未超时 → 跳过

            // 超时！发送抢占信号
            _senduipi(uipi_index[i]);              // UINTR 硬件中断
        }
    }
}
```

### Base 层（基础库）

```c
base/thread.c            # 用户态线程实现
base/lrpc.c              # Lightweight RPC（跨 CPU 通信）
base/mem.c               # 内存分配
base/time.c              # 时间管理
base/cpu.c               # CPU 拓扑检测
```

### Kernel 模块

```c
ksched/ksched.c          # /dev/ksched 设备驱动实现
```

ksched 内核模块，作用是通过注册一个字符设备/dev/ksched驱动，为系统提供线程的抢占调度功能。

它通过 劫持 cpuidle  + mwait 指令 + 共享内存 的方式，让 IOKernel 能够直接控制 CPU 核心上的线程调度。能够让runtime的kthread 主动休眠。

最主要的就是驱动的ksched_ioctl函数和ksched_idle函数。



```
┌─────────────────────────────────────────────────────────────┐
│  用户态                                                       │
│  ┌──────────────────────────────────────────────────┐       │
│  │  IOKernel 数据平面                               │       │
│  │  ├── mmap(/dev/ksched) → ksched_shm 共享内存     │       │
│  │  ├── ksched_run() → 写 s->tid, s->gen            │       │
│  │  ├── ksched_enqueue_intr() → 写 s->signum, s->sig│       │
│  │  └── ioctl(KSCHED_IOC_INTR) → 批量发中断         │       │
│  └──────────────────────────────────────────────────┘       │
│                          ↕                                   │
├─────────────────────────────────────────────────────────────┤
│  内核态 (ksched.ko)                                          │
│  ┌──────────────────────────────────────────────────┐       │
│  │  1. 字符设备 /dev/ksched                         │       │
│  │     ├── mmap: 导出 shm 共享内存                  │       │
│  │     └── ioctl: KSCHED_IOC_START/PARK/INTR        │       │
│  │                                                  │       │
│  │  2. cpuidle 劫持                                 │       │
│  │     └── 替换 idle 驱动为 ksched_idle             │       │
│  │                                                  │       │
│  │  3. mwait 等待 + 调度                            │       │
│  │     └── 在 idle 循环中 mwait, 收到 gen 变化唤醒  │       │
│  │                                                  │       │
│  │  4. IPI 中断处理                                 │       │
│  │     └── smp_call_function_many → ksched_ipi      │       │
│  └──────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

#### ksched_cpuidle_hijack

Linux 内核的 cpuidle 子系统负责管理 CPU 空闲状态（C-states）。当 CPU 没有可运行任务时，会调用 cpuidle_driver.states[i].enter 进入低功耗状态。

ksched 直接替换了 idle 入口函数 ：

- 原来：进入 C1/C2/C3 深度睡眠，等待中断唤醒（毫秒级延迟）
- 现在：调用 ksched_idle ，用 mwait 监控共享内存（亚微秒级唤醒）
为什么这么干？ 因为标准的 cpuidle 机制唤醒延迟太高（几十微秒），无法满足 Aspen 微秒级调度的需求。通过劫持，ksched 用 mwait 实现了 亚微秒级唤醒 ，同时还能直接感知 IOKernel 写入共享内存的调度请求。

MONITOR 是 x86 的一条指令，用于 武装一个监控地址范围 。武装后，如果该地址范围内的任何内存被写入，CPU 会记录一个"监控事件"。随后执行 MWAIT 时，CPU 会进入低功耗状态，当监控事件发生时自动唤醒。

作用 ：和 MWAIT 配合，实现"地址变化即唤醒"的高效等待机制。

INTERRUPT_BREAK 中断会导致 MWAIT 退出，sched 在 ksched_idle() 中调用 mwait 时，中断是关闭的（ lockdep_assert_irqs_disabled() ）。正常情况下，MWAIT 在中断关闭时不会被中断唤醒。但设置了 INTERRUPT_BREAK 位后， 即使中断被屏蔽，到达的中断也会让 MWAIT 退出 （尽管中断处理程序要等到 STI 之后才会执行）。

MONITOR 监控的 不是一个字节 ，而是以 addr 为中心的一个**缓存行（cache line）**大小的区域（通常 64 字节）。只要这个缓存行被任何写入操作修改，就会触发监控事件。

MWAIT 指令让 CPU 进入一个 可选的低功耗状态 ，同时持续监控之前 MONITOR 武装的地址范围。当地址被写入（或发生中断）时，CPU 立即恢复执行。

hint 值 C-state 功耗 唤醒延迟 
0x00 C0 (monitor 状态，非睡眠) 高 极低 (~ns) 
0x10 C1 中低 低 (~us) 
0x20 C2 更低 较高 (~10us) 
0x30 C3 最低 最高 (~100us)

#### ksched_next_tid

释放当前线程，唤醒下一个线程，参数 tid=0 表示"释放当前 task，不唤醒新的"。

**Aspen 语境下的 "kthread" = 内核调度的线程 = 普通 Linux 用户态线程（pthread）**

| 术语 | 含义 | 对应实体 |
|------|------|---------|
| **kthread**（Aspen 中） | "kernel thread" 的缩写，意思是"由内核调度的线程"（区别于用户态调度的 uthread） | `struct task_struct`（Linux 内核的线程描述符） |
| **uthread**（Aspen 中） | "user thread"，用户态线程/协程，由 runtime 的协程库调度 | 纯用户态数据结构 |

**容易混淆的点**：Linux 内核也有 "kernel thread"（内核线程，如 `ksoftirqd`、`kworker`），但 Aspen 的 kthread **不是**那种内核线程。Aspen 的 kthread 是**运行在用户态的普通 pthread**，只是它通过 `ksched` 模块把调度控制权交给了 IOKernel。

```
┌──────────────────────────────────────────────────────────────┐
│  用户态                                                       │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  runtime 进程                                        │    │
│  │   ├── kthread 1 (pthread)                            │    │
│  │   │    ├── uthread A                                 │    │
│  │   │    ├── uthread B                                 │    │
│  │   │    └── uthread C                                 │    │
│  │   └── kthread 2 (pthread)                            │    │
│  │        └── ...                                       │    │
│  └──────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          ▼                                   │
├──────────────────────────────────────────────────────────────┤
│  内核态                                                       │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Linux 内核调度器                                    │    │
│  │   schedule() / wake_up_process()                     │    │
│  │   runqueue / context switch                          │    │
│  └──────────────────────────────────────────────────────┘    │
│                          ▲                                   │
│                          │ 调用                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  ksched.ko（内核模块）                               │    │
│  │   ├── 劫持 cpuidle → mwait 快速唤醒                  │    │
│  │   ├── ksched_park → schedule() 让出                 │    │
│  │   └── ksched_next_tid → wake_up_process() 唤醒       │    │
│  └──────────────────────────────────────────────────────┘    │
│                          ▲                                   │
└──────────────────────────┼───────────────────────────────────┘
                           │
                   IOKernel (用户态)
                   共享内存 + ioctl
```

三个层级的调度

Aspen 实际上有**三层调度**：

| 层级 | 调度器 | 调度对象 | 粒度 |
|------|--------|---------|------|
| L1（最底层） | **Linux 内核调度器 CFS** | kthread（Linux 线程） | 毫秒级 |
| L2（中间层） | **ksched + IOKernel** | kthread（抢占式分配 CPU 核心） | 微秒级 |
| L3（最上层） | **runtime 协程调度器** | uthread（用户态协程） | 亚微秒级 |


```
状态 1：运行中（running）
  kthread 正在 CPU 上跑 uthread
  ↓
  （IOKernel 决定抢占）
  ↓
  收到 SIGUSR1 → 检测到 cede_gen 变化 → 调用 ioctl(KSCHED_IOC_PARK)
  ↓
状态 2：parked（停泊）
  ksched_park() 中执行 schedule()
  ↓
  Linux 调度器把它从 CPU 上换下
  ↓
  它在 runqueue 里以 TASK_INTERRUPTIBLE 状态睡眠
  ↓
  （IOKernel 决定让它跑）
  ↓
  ksched_next_tid() 调用 wake_up_process()
  ↓
  线程被唤醒，设为 TASK_RUNNING
  ↓
  ksched_park() 从 schedule() 返回
  ↓
  返回核心号，kthread 继续跑 uthread
```


ksched 做的事情是**控制"什么时候让谁跑"**，但实际的"让它跑/让它停"的动作，仍然通过内核原语完成：
- **让它停**：`schedule()` —— 让当前线程让出 CPU
- **让它跑**：`wake_up_process()` —— 把目标线程加入运行队列

ksched 替换的是**调度决策**（选哪个线程跑），而不是**调度机制**（怎么切换上下文）。


#### smp_call_function_many

smp_call_function_many() 也是 Linux 内核本身提供的核心 API ，是 Linux SMP（对称多处理）子系统的一部分。

```
smp_call_function_many(mask, ksched_ipi, NULL, false);
```

- func = ksched_ipi ：目标 CPU 收到 IPI 后执行 ksched_ipi() 函数
- info = NULL ：不传参数
- wait = false ： 异步发送 ，发完就返回，不等待执行完成

```
CPU 0 (IOKernel)                    CPU 1 (运行 kthread)
    │                                    │
    │ smp_call_function_many             │
    │   (发起 IPI)                        │
    │                                    │
    └───────────────────────────────────►│ IPI 中断到达
                                         │
                                     打断当前执行
                                     保存上下文
                                     调用 ksched_ipi()
                                     发送 SIGUSR1 给 kthread
                                     恢复上下文
                                     继续执行
```


OKernel 运行在 CPU 0（数据平面核心） 上，而要抢占的 kthread 运行在 其他 CPU 核心 上。通过 IPI + 信号才能抢占kthread。

1. IPI 打断目标 CPU ：让目标 CPU 立即停下正在做的事，进入内核
2. 内核中发送信号 ：在 IPI 回调 ksched_ipi() 中调用 send_sig(SIGUSR1, task)
3. 线程收到信号 ：kthread 的信号处理函数被触发，检测到 cede_gen 变化，调用 KSCHED_IOC_PARK 让出


#### ksched_park

ksched_park() 是 kthread 让出 CPU 核心的统一入口。它不仅是简单的让出，还负责检查下一个要跑的线程、完成切换、然后自己进入睡眠。

Runtime检测到IO kernel写入共享内存的抢占标志，cede_gen == wake_gen。调用ioctl(ksched_fd, KSCHED_IOC_PARK, 0) -> ksched_ioctl -> ksched_park 

被 IOKernel 抢占
```
IOKernel 决定抢占 runtime B 的核心给 runtime A
    │
    └── ksched_enqueue_intr(core, KSCHED_INTR_CEDE)
            │
            └── 设置 s->signum = SIGUSR1, s->sig = gen
                    │
                    └── ioctl(KSCHED_IOC_INTR)
                            │
                            └── smp_call_function_many → ksched_ipi
                                    │
                                    └── send_sig(SIGUSR1, running_task)
                                            │
                                            ▼
                                    runtime B 的 kthread 收到 SIGUSR1
                                    信号处理函数被调用
                                            │
                                            ▼
                                    检查 preempt_cede_needed()
                                    （q_ptrs->cede_gen == curr_grant_gen?）
                                            │
                                            ▼
                                    是 → 让出 CPU
                                            │
                                            ▼
                                    ioctl(KSCHED_IOC_PARK)  ← park 调用
                                            │
                                            ▼
                                    ksched_park() [L257]
```

kthread 主动让出
```
kthread 运行完所有 uthread
    （没有待处理的网络包、没有定时器、没有 IO）
    │
    ▼
sched_yield() / sched_sleep()
    │
    ▼
ioctl(KSCHED_IOC_PARK)  ← park 调用
    │
    ▼
ksched_park() [L257]
```