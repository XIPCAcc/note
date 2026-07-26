# IOKernel

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

主线程最后执行dataplane_loop，变成data plane专用线程， 轮询处理数据包和调度任务。数据平面线程 永远不睡眠 ，100% 占用一个 CPU 核心。

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
