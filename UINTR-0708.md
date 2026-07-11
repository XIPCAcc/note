# Extended User Interrupts (xUI): Fast and Flexible  Notification without Polling

xUI是如何拓展UIPI 使其能够支持硬件的Interrupt Forwarding的。

传统中断系统只知道：“把中断路由到某个 APIC ID（通常在线程启动时静态绑定到某个核）”
• 设备中断无法自然地投递给当前正在运行的用户态线程
• UIPI 的 posted‑interrupt 机制与传统中断路由不兼容

Interrupt Forwarding 的核心思想

Interrupt Forwarding 是对 Local APIC 的扩展，其目标是：

让 Local APIC 在收到设备中断时，能够“转发”该中断，直接投递给当前正在运行的用户态线程，而不是走传统的内核 IDT 路径。

软件接口与映射模型

Interrupt Forwarding 允许内核建立如下映射关系：

[每核 APICID] × [中断向量 V]  ↔  [用户态中断向量 uv]

映射由内核在线程注册接收设备中断时建立

用户态注册流程

1. 线程通过 OS 系统调用声明：  
   “我希望接收来自某设备的用户态中断”
2. 内核分配一个 notification vector（uv）
3. 内核在 Local APIC 中建立映射：
   设备中断向量 V → 用户态向量 uv


硬件微架构设计

Local APIC 新增 两个 256‑bit 寄存器（每核）：

寄存器 位宽 含义
forwarding_enabled 256 bit 指示本核上哪些中断向量允许被转发
forwarded_active 256 bit 指示当前运行的线程是哪些向量的合法接收者

中断到达时的硬件判定逻辑

假设设备中断到达，向量号为 V = 8：
```
if (forwarding_enabled[8] == 0)
    → 普通中断（IDT）
    return

// forwarding_enabled[8] == 1
UIRR[8] = 1   // APIC 直接置 UIRR 对应位

if (forwarded_active[8] == 1) {
    // Fast Path
    // 说明当前线程是注册接收者，投递一个self UIPI 硬件在 return‑to‑user 时自动 deliver UI handler
} else {
    // Slow Path
    // 说明目标线程不在 CPU 上，发起常规中断，走内核 trap handler
    1. 识别中断原因
    2. 从 UIRR 中读出 pending 的向量
    3. 将该 pending 信息存入 DUPID（Deferred User Posted Interrupt Descriptor）
    4. 唤醒目标线程
}
```

# 总结

需要拓展APIC的功能，让它能够设置UIRR寄存器，以及实现一个映射。


## KB_timer

KB_Timer 是一个每核独立的定时器机制，被集成到 Aspen（基于 Caladan 的用户态运行时）中。

该定时器可向调度器或运行时发送用户态众人用来驱动用户态抢占.

实验运行 RocksDB [8]（v5.15.10），采用 99.5% GET（1.2 μs）/ 0.5% SCAN（580 μs）的双峰负载。Caladan 开环负载生成器按泊松分布发送 UDP 请求，生成器与 RocksDB 分别置于单核 Aspen 实例中。为适应 gem5 环境，Aspen 进行了三点调整：(1) 添加转发核心模拟网络流量；(2) 移除 ksched并绑核（pinning）以消除噪声；(3) 集成 KB_Timer，赋予每个核心独立的本地定时器。