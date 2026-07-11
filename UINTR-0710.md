# Aeolia: A Fast and Secure Userspace Interrupt-Based Storage Stack

Aeolia 基于UINTR实现在用户态响应NVMe设备。

Aeolia 利用 Intel User Interrupts 将 NVMe 设备完成中断直接投递到用户态，配合内核分配的 NVMe 队列 mmap 实现完全 kernel-bypass 的 I/O 提交与完成处理；并结合硬件隔离与 sched_ext 实现多任务安全资源共享，在此基础上构建了高性能用户态库文件系统 AeoFS。

## 如何将设备中断映射为用户态中断

具体而言，注册用户态中断时使用设备中断号，这样APIC就不会将设备中断当成普通中断。其次，将每个进程的UPID映射到用户态内存，每次在用户态中断处理程序结束前，将UPID.PIR[UV] = 1，然后保持。

等到下一次设备中断来到，APIC会自动识别UPID.PIR[UV] = 1，然后将其加载到UIRR寄存器触发用户态中断处理逻辑。

Q: 每次从内核态切回到用户态，检测到UPID.PIR[UV] = 1，即使没有用户态中断或者设备中断，都会自己触发一次self IPI，这个问题是怎么解决的。

猜想：确实会触发spurious UINTR，但是这个不影响程序的正确性。因为中断处理程序是通过读取 hardware completion queues entries判断是否真的有中断发生的。

其次，如果接收方进程处于切出状态，那么此时 UINV MSR的值和设备中断号就不同了，依然走原本的内核中断处理逻辑。

## 协调调度（Coordinated Scheduling）

​Aeolia 通过协调线程调度，使多个任务能够共享同一 CPU 核心。为实现这一目标：

Aeolia 采用了 可扩展调度类 sched_ext[12]——一种基于 eBPF 的最新内核调度框架。sched_ext用于在内核中定义调度算法，并将调度策略暴露给用户态；Aeolia 借助它将当前调度策略同步至用户空间。具体而言，由于 sched_ext使用 eBPF map 存储全部调度状态，Aeolia 直接通过 mmap将 eBPF map 映射到用户态地址空间，使 Aeolia 的可信组件可读取调度状态。得益于 sched_ext，Aeolia 能够精确判断何时必须让出核心，从而实现仅在必要时 yield。

采用了sched_ext后，当前进程仅在必要时候让出（不受内核默认调度器的时间片影响）

当前的设计旨在证明，线程调度并非用户态存储栈的固有缺陷；因此，其在实现上紧密效仿了内核栈的调度机制。所以sched_ext的调度策略是EEVDF

## 实现

在 Linux 6.12.20 内核上实现了 Aeolia。其中 AeoKern、AeoDriver 和 AeoFS 的代码行数分别为 3992、1889 和 11870 行。

（Linux 6.12.20 第一次引入 sched_ext） ，然后把用户态中断的支持也移植了过来。

## 性能

单线程性能。​ 图 10 展示了 AeoDriver 在不同 I/O 大小下的单线程性能。

在较小访问粒度下，AeoDriver 显著优于传统的基于中断的内核存储栈。具体而言，在 512B 访问粒度下，与 POSIX 相比，AeoDriver 实现了 2 倍的吞吐量提升、中位延迟降低 48%​ 以及 尾延迟降低 26%；在较大的 8KB 粒度下，AeoDriver 实现了 1.54 倍吞吐量提升、中位延迟降低 36%​ 以及 尾延迟降低 21%。

在大多数 I/O 大小下，AeoDriver 的性能与 SPDK 相当，但在小于 4KB 的微小请求场景下存在一定差距。在最坏情况下（512B 读请求），AeoDriver 的吞吐量降低 10.7%，中位延迟提高 18.2%，尾延迟提高 6.1%。这是因为此时中断开销（0.6μs）相对于访问延迟（3.2μs）已变得不可忽略。

AeoDriver 之所以优于内核存储栈，是因为消除了多层软件栈与内核陷入（kernel trapping）的开销；而其性能略逊于 SPDK，则源于中断机制所带来的微小开销。

多线程性能。​ 图 11 展示了 4KB I/O 粒度下的多线程性能。AeoDriver 与 SPDK 均表现出良好的可扩展性，在 8 线程时即可使磁盘达到饱和；POSIX 与 io_uring 性能相当。AeoDriver 相较 POSIX/io_uring 最高可实现 1.18 倍的吞吐量提升。相比之下，iou_poll 在 16 线程时出现性能瓶颈。小结。​ 在各种 I/O 大小与线程数配置下，基于中断的 AeoDriver 凭借直达磁盘访问的优势，始终优于 Linux 传统栈，且性能表现与 SPDK 相近。