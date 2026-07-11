# The Benefits and Limitations of User Interrupts  for Preemptive Userspace Scheduling

Aspen 用户态调度器，基于Linux，使用Go语言编写，支持goroutine的调度。并且用于解决网络栈处理数据包的长尾延迟和Head of line blocking问题

实现了Aspen-KB（C 协程调度）和 Aspen-Go（Go coroutine调度）

Aspen-KB 通过扩展 Caladan（一个采用 C 语言实现的用户态调度器），引入了定时器核心、支持用户中断的调度器特性，以及双队列调度机制。

Aspen-Go 则是通过扩展 Go 运行时构建而成。

专门预留一个核运行定时器并向其他核发送用户态中断。

## Head-of-line blocking

光把 goroutine 抢占做得更细，并不能解决 Go 网络应用的 Head-of-Line Blocking。

原生的go 协程调度器优先跑 goroutine，除非应用主动 read（非阻塞）收包；或者调度器空闲才调 netpoller，只有没 goroutine 可跑，才会去检查有没有网络包；或者sysmon 每 10ms 检查一次；

如果队头正在跑的 CPU-bound goroutine堵住，网络包的tail latency 暴涨，即使抢得非常频繁，也无济于事（HTAP / 微服务里经典问题）

为尽可能减轻网络栈中的队头阻塞问题，调度器可以在每次抢占发生后都调用 netpoller，以检查是否有新到达的数据包。然而在 Go 中，这需要一次系统调用，会为每次抢占额外带来至少 2 μs 的开销。这会严重损害性能，尤其对很少进行网络 I/O 的应用而言更是如此。

为此，Aspen-Go 改为让 sysmon 线程以更高的频率调用 netpoller，周期为 100 μs。虽然这一改动使得积压的数据包能够被更快地感知和处理，但并未彻底消除队头阻塞。原因在于：一旦 sysmon 处理完到达的数据包并将对应的 goroutine 标记为可运行，这些 goroutine 会被加入全局运行队列（global runqueue）（以避免对每核本地队列的竞争）；而它们在该队列中，仍然不得不排在已被抢占的 goroutine 之后，无法立即得到调度。

## 测试

我们对多种内核旁路系统进行了评估。理想情况下，我们希望将 Aspen-KB 与使用其他抢占机制的先进系统（如 Shinjuku [39] 和 Concord [37]）进行直接对比。然而，这种对比面临较大挑战。首先，Shinjuku 与 Concord 均基于较旧的 Linux 内核（4.4.185）设计，而该版本不被 Aspen-KB 支持。其次，三者采用的核间负载均衡策略各不相同：Shinjuku 使用单一集中式队列分发任务，Concord 采用 JBSQ（Joint Borrow-and-Steal Queue）[40]，而 Aspen-KB 使用工作窃取（work stealing）。这些负载均衡策略的差异在多核实验中会对性能产生显著影响 [44]，从而使得抢占机制本身的贡献难以被单独剥离。因此，我们选择评估 Aspen-KB 的五种变体（§3.2）。这包括基于信号的抢占调度器和基于用户中断的抢占调度器（Aspen-KB 的默认配置）。在编译器插桩方面，我们评估了 Concord 的默认配置（同时对函数调用和循环回边进行插桩）[37]，以及经过精细调优的 Concord：我们针对每个应用仔细配置了子循环参数、循环展开程度以及是否对函数调用进行插桩，以获得最佳性能。此外，我们还评估了非抢占的“运行至完成（run-to-completion）”基线，即通过禁用 Aspen-KB 的抢占功能实现。

除非另有说明，非抢占基线配置为使用 25 个核运行被测应用，而其他所有系统均配置为使用 24 个核运行应用，并保留 1 个核专用于定时器线程。

Aspen-KB 测试程序收UDP请求，然后创建协程读RocksDB内容返回。

Aspen-Go 让 goroutine 收HTTP 请求，然后在 goroutine 里直接读 BadgerDB。

GET operations 代表短任务，taking about 5 μs and RangeSCAN operations 代表长任务，around 800 μs.

因为 Go 运行时本身的结构问题，UINTR 带来的提升没有 Aspen-KB 那么明显。主要有三个原因：第一，新来的网络请求被塞进了全局队列，得排队等前面的任务跑完，而 Aspen-KB 能直接把它们放进 CPU 本地队列，马上就能跑。第二，sysmon看不到内核网络栈里到底有没有包，就算没活干，时间到了也会抢，这就产生了很多无效的抢占。第三，Go 的上下文切换要管 GC、管栈，比 Aspen-KB 轻量的切换要重不少。总的来说，Go 的设计决定了它很难照搬 Aspen-KB 那种“插队”式的激进优化，不然就没法保证现在的兼容性和部署便利性了。

在 Go 等并未完全为细粒度抢占设计的系统中，抢占机制对性能的影响并不显著。下文我们将总结针对 Aspen-KB 及同类低开销、内核旁路抢占系统的主要经验。对于给定的应用，抢占机制的最佳选择取决于最优抢占时间片（quantum）的设定。确定合理时间片的一个启发式原则是：其长度至少应不小于“短任务”（即不应被抢占的任务）的尾部执行时长。结合 §4.3 的指导意见，可以更精确地识别最优时间片——为避免过度抢占，该值甚至可能需长于短任务本身的执行时间。无论何种情况，只要选定的时间片不低于 10 μs，用户中断（UINTR）的性能通常均优于或与编译器插桩持平（如图 1 中的 DataFrame 结果及部分基准测试所示）。而在必须使用极低时间片（低于 10 μs）的场景下，只要应用中不存在大量紧密循环（§2.2），编译器插桩可能会提供更好的性能。然而，正如 RocksDB 的实验结果所示，这种性能提升十分有限。这是因为编译器插桩仅能降低抢占机制自身的开销，却无法解决上下文切换的开销（§4.2）。综上所述，我们认为对于大多数应用场景而言，用户中断可能是更优的选择。

## 原生的Go调度

原生Go 的 sysmon（system monitor）后台线程每 ~10ms 扫描运行中的 G，发现超时被标记抢占并且发送发信号抢占。

```
Go 的调度器是 GMP 模型：

G：goroutine

M：OS thread

P：processor（逻辑调度单元，绑定一个 CPU 核）

每个 P​ 有一个 本地 runq（per-core queue)，还有一个 全局 runq（global queue）

调度采用任务窃取算法：先扫当前 P 的本地 runq，本地空了，再去全局 runq 偷一个，再空，去别的 P 偷（work stealing）
```

## future work

Aspen-Go 通过用户中断显著降低了抢占本身的开销，但并没有改变 Go 调度器“优先跑 goroutine、后看网络包”的根本策略。这也导致了两个没能彻底解决的问题：一是网络包在内核里最多等 100 μs，二是被唤醒的网络 goroutine 仍需在全局队列中排队。沿着这个方向，可以从以下四个方面继续深入：

第一，引入内核旁路网络栈，消除轮询开销。​

现在 sysmon 每 100 μs 才去 poll 一次网络，是因为 epoll_wait是一次系统调用，代价太高。如果能像 Junction 或 Caladan 那样，把网络栈搬到用户态，轮询网络就不再是 syscall，而是读一块共享内存，开销可以从微秒级降到纳秒级。这样，我们就有机会在每一次抢占之后都检查网络包，而不再受限于 100 μs 的轮询周期，从机制上大幅缓解队头阻塞。

第二，设计网络感知的调度策略。​

即便轮询变快了，Go 仍然把网络 goroutine 放进全局队列，让它们排在 CPU 任务后面。下一步，我希望在调度器中显式引入“网络优先级”：当 sysmon 发现有新包到达时，被唤醒的网络 goroutine 可以直接插入 per-core 队列的前端，或者预留一部分 runqueue 容量专门给 I/O 任务。这样可以避免网络任务在全局队列中长期排队，真正做到“网络一就绪，任务马上跑”。

第三，优化抢占启发式，减少无意义切换。​

现在 sysmon 只要发现 goroutine 超时就会抢占，即使系统里根本没有别的 runnable 任务。后续工作可以结合 netpoller 和 runqueue 的状态，让 sysmon 学会“判断局势”：只有当存在更紧急的任务（比如新到的网络包或短事务）时才触发抢占，否则就让当前 goroutine 继续跑。这能在保持响应性的同时，进一步降低不必要的上下文切换。

第四，把 UINTR 预置推广到 Rust 等无 safepoint 的语言。​

Go 至少有编译器插桩和信号抢占作为基础，而 Rust 的 async 任务是纯协作式的，完全没有语言级的抢占能力。这反而让 UINTR 的价值更大。接下来，我计划设计并实现一个UINTR 感知的 Rust M:N 执行器，在不修改编译器、不依赖 safepoint 的情况下，为 Rust 引入真正的细粒度抢占。这不仅是对 Aspen 思想的泛化，也是对 Rust 异步生态的一个重要补充。

# 对比Tokio的思考

在tokio中加入抢占，工程上极难直接在Tokio现有API内“透明地”加入真抢占；最合理的做法是写一个 UINTR-aware 自定义 executor，或退而求其次做“标记+yield”式伪抢占。

Tokio 的调度核心（Multi-threaded Work Stealer）是：

• LocalQueue（per-worker）

• Injector（全局）

• waker（按 RawWaker 规范）

- 任务状态机由 Future::poll() 推进

难点在于：无 safepoint async Fn 只 .await 时 yield，无编译器插桩，中断上下文限制 UINTR / SIGURG handler 不能调 wake() / Injector::push() / lock

Future 不挂起自身 Tokio 没有 "保存 regs + 切出 mid-Future" 的原语

TLS / LocalSet worker-local task 假设在 own worker thread 执行

Tokio 的 poll loop 假设任务会自己让出，不提供 mid-execution 强制切出接口。


自己写最小 M:N executor：

Task = { saved_regs, stack_ptr, Pin<&mut Future> }
UINTR handler:
  - 保存 callee-saved regs → cur_task.ctx
  - scheduler.select_next()
  - restore regs ← next_task.ctx


最适合 Aspen-Rust Design

Design / Motivation

Rust’s dominant async runtime, Tokio, implements cooperative scheduling: tasks yield only at .await points and provide no facility for mid-execution preemption. Injecting true preemption directly into Tokio is impractical, as its scheduler and waker machinery must not be invoked from an interrupt context. We therefore realize Aspen-Rust as a UINTR-aware M:N executor that manages task contexts independently of Tokio, demonstrating that user interrupts can retrofit fine-grained preemption to languages lacking built-in safepoints—something even cooperative runtimes like Tokio cannot achieve on their own.

Limitation / Future Work

An alternative would be to extend Tokio with deferred preemption—where a user interrupt merely sets a flag checked at .await boundaries—but this remains cooperative and cannot preempt CPU-bound futures that omit .await, unlike the UINTR-based approach presented here.


Tokio 本身不能透明加入真抢占（无 safepoint + 中断上下文限制），想做 Rust 上 UINTR 抢占最干净的做法是自建 UINTR-aware executor；退而求其次是让 UINTR 设标志、在 .await 点配合 yield（伪抢占，仍有饥饿风险）。
