# 性能调优

从 process_uintr_wakeup的角度

1. 使用专门定时的协程调用process_uintr_wakers进行唤醒，协程唤醒间隔在一定程度上会影响其性能，尝试不同唤醒间隔，找到最优。

2. 在turn中唤醒，runtime会以固定周期进入 turn()，可以在其中调用 process_global_uintr_wakers

3. 在每次协程切换时调用 process_global_uintr_wakers唤醒，即run()

方案 唤醒位置 关键参数 吞吐量 (Req/sec) 平均延迟 
方案 1a 专用定时协程 5μs 间隔 197,455 5.18ms 
方案 1b 专用定时协程 10μs 间隔 193,427 5.28ms 
方案 2 I/O Driver turn() 随事件循环 197,517 5.16ms 
方案 3 Scheduler run() 随任务切换 197,802 5.18ms



传输方式 低并发 (64 conn) 中并发 (256 conn) 高并发 (1024 conn) 
shm-uds 25,176 87,891 203,505 
shm-eventfd 25,426 89,111 191,374 
shm-uintr (方案三) 17,837 63,446 194,181

# 有几点没有验证

现在的用户态中断会打断epoll_wait 吗？

# 原生问题思考

相较于UDS和epoll通过事件方式将通知暂存到内核中，再集中通过epoll_wait从内核读取的方式，

用户态中断虽然能够起到及时通知的作用，但是过于频繁的打断去执行中断处理程序也反而带来性能开销，

而且在现有的tokio开发框架下，依然需要定期进入epoll_wait读取外部事件，这部分开销也没有减少。

可以考虑backend去掉epoll_wait看看能否运行起来，gateway必须保留epoll接受http请求。

实际做法是可以将backend的event_interval 无限增大，这样可以减少进入turn的次数。

默认
测试结果（ 1024 connections / high concurrency）

- event_interval=64：rps=190300.77，avg=5.17ms，p99=11.85ms
- event_interval=128：rps=190281.87，avg=5.34ms，p99=11.67ms
- event_interval=256：rps=188400.78，avg=5.21ms，p99=9.91ms
- event_interval=512：rps=187287.00，avg=5.47ms，p99=12.34ms
- event_interval=1024：rps=194493.17，avg=5.26ms，p99=12.34ms
- event_interval=2048：rps=190233.07，avg=5.37ms，p99=11.77ms


fn main() -> Result<()> {
    tracing_subscriber::fmt()
        .with_env_filter(tracing_subscriber::EnvFilter::from_default_env())
        .init();

    let args = Args::parse();


    let event_interval: u32 = std::env::var("TOKIO_EVENT_INTERVAL")
        .ok()
        .and_then(|v| v.parse().ok())
        .unwrap_or(64);

    let rt = Builder::new_multi_thread()
        .event_interval(64)
        .enable_all()
        .build()?;

    rt.block_on(async_main(args))
}

async fn async_main(args: Args) -> Result<()> {

TOKIO_EVENT_INTERVAL=256 bash scripts/run_shm_uintr.sh 1000 10

需要稍微校正/补充的两点

- UDS/eventfd + epoll 并不总是“更集中更省”
   内核事件暂存 + epoll_wait 的优势是“批处理/合并”，但它的代价是：
  
  - 通知路径至少一次 syscall/内核态切换（写 eventfd/写 socket + epoll 取出）
  - 可能引入“等到下一次 epoll_wait 返回”的调度延迟（尤其在 runtime 进入 park/深睡时）
     UINTR 的价值点通常是把“唤醒路径”从 syscall 变成用户态更短路径（前提是中断频率不会爆炸），并降低“需要一个额外 fd 来被 epoll 监视”的依赖。
- 关键瓶颈往往不是 epoll_wait 本身，而是“谁在驱动 wakeup + work stealing + 队列竞争”
   你之前测到“wakeup 放哪里差不多”，说明在你的负载下：
  
  - runtime 的主循环已经足够热，检查点密度够高
  - 主要成本可能在共享内存读写、序列化/反序列化、wrk 压测端、以及 tokio 多线程队列/锁竞争，而不是唤醒函数本身
     这也解释了为什么 UINTR 方案没有显著超过 UDS：你并没有减少 Tokio 的 driver/调度成本，只是替换了“通知机制”。
如何把你的结论变成更强的“工程决策”

- 如果目标是“吞吐最大化”：更倾向于 合并通知/降低中断频率 （例如把多个请求合并成一次 senduipi、或仅在从空变为非空时发中断）。
- 如果目标是“低负载/空闲时快速响应”：UINTR 可能更有优势（避免等 epoll 超时/减少额外 fd 唤醒链路），但要控制中断风暴。
总体来说：你这段分析的方向是对的——在 Tokio 现有框架下，UINTR 很难单靠“更快唤醒”就打赢“epoll 批处理 + 成熟路径”，除非你进一步做“减少中断次数/合并通知/减少 runtime 进入 epoll 轮询的必要性”的架构调整。

# 性能分析方法

目标“算出三种 SHM（shm-uds / shm-eventfd / shm-uintr）端到端延迟里各部分占比、定位瓶颈、找到可优化点”，一般要同时做两类分析：**路径分段（in-code 分段计时）** + **系统级剖析（perf/trace 看 CPU/内核/调度）**。只靠 wrk 的 P99/P50 只能看到总延迟，无法知道占比。

下面给你一套可落地的方法论（按这个做，基本就能得到“占比饼图 + 瓶颈归因 + 优化方向”）。

---

## 1) 先把“端到端延迟”拆成统一时间线

以一次请求为例（wrk → gateway HTTP → SHM → backend → SHM → gateway HTTP 返回），建议至少拆成这些段：

### Gateway 侧（HTTP 入口到 transport.call 结束）
- **G1: HTTP ingress**：接收连接/解析 header/读取 body（[gateway/src/main.rs](file:///home/zwp/gateway-gRPC/gateway/src/main.rs) 的 `handle_request`）
- **G2: 解析/组包**：JSON parse + 构造 `EchoRequest`
- **G3: 发送到后端（transport.call 内部）**：按三种 SHM 进一步拆
- **G4: HTTP egress**：构造 response + 写回 socket

### Transport 层（每种 SHM 都有）
对于 `shm-uds / shm-eventfd / shm-uintr`，你现在的实现结构基本一致（都有 request_id、pending map、响应 listener、SHM ring buffer + 通知机制），因此可以统一分段：

- **T1: request serialize**：`Uuid` + `bincode::serialize`（例如 [shm_transport_uds.rs](file:///home/zwp/gateway-gRPC/gateway/src/shm_transport_uds.rs#L132-L159)、[shm_transport_eventfd.rs](file:///home/zwp/gateway-gRPC/gateway/src/shm_transport_eventfd.rs#L206-L227)）
- **T2: shm write**：`request_buffer.write(...)`
- **T3: notify send**：  
  - shm-uds：`notify.write_u8(1)`  
  - shm-eventfd：`write(eventfd)`  
  - shm-uintr：`senduipi(...)`（[shm_transport_uintr.rs](file:///home/zwp/gateway-gRPC/gateway/src/shm_transport_uintr.rs#L205-L216)）
- **T4: wait response (oneshot)**：`rx.await`（这一段往往包含了“后端等待通知 + 读 SHM + 处理 + 写 SHM + 回通知 + gateway listener 读 SHM”的全部时间）
- **T5: response deserialize**：`bincode::deserialize`（在 listener 中）

### Backend 侧（通知到达到写回通知）
同样建议拆：
- **B1: wait notify**：等待 UDS/eventfd/uintr（或读 socket/AsyncFd/uintr_wait 返回）
- **B2: shm read/drain**：从 request ring buffer 读出（可能循环 drain 多条）
- **B3: request deserialize**
- **B4: compute/delay**：你现在有 `delay_us`，这是一个“已知成本”，要么设为 0 用于测 transport overhead，要么保留用于“业务占比”
- **B5: response serialize**
- **B6: shm write**
- **B7: notify send back**

---

## 2) 怎么“算占比”：两条路，建议都做

### 路线 A：代码内“分段计时 + 直方图/采样输出”（最直观）
做法要点：
1. **用跨进程可比的时钟**：同机跨进程推荐 `clock_gettime(CLOCK_MONOTONIC[_RAW])` 得到 `u64 ns`，不要用 `Instant::now().elapsed()` 这种相对值（目前你 request.timestamp_ns 就是相对值，没法跨进程对齐）。
2. **给每个 request 一个 id**：你已有 `Uuid` request_id（SHM 三种 transport 都有），很好。
3. **把时间戳跟着消息走**：你现在 shm-uds/eventfd/uintr 的 payload 是 `(request_id, EchoRequest)` 的 bincode；可以扩成 `(request_id, TraceTimestamps, EchoRequest)`，TraceTimestamps 里存 `t_gw_send_ns / t_be_recv_ns / t_be_send_ns / t_gw_recv_ns` 等。
4. **统计方式**：不要每次都打印日志（会改变性能），用：
   - 每段一个 histogram（p50/p90/p99）
   - 或者 1/N 采样，把样本写入文件

这样你可以得到每种 transport 的“平均占比”和 tail 占比，比如：
- notify wait 占比 20%
- shm read/write 占比 5%
- serialize/deserialize 占比 30%
- runtime/scheduling（off-CPU）占比 40%

### 路线 B：系统级剖析（定位 CPU/内核/调度瓶颈）
它回答的是“时间耗在哪里”，尤其是你怀疑的：
- epoll_wait/park/unpark 成本
- futex/锁竞争（pending map、tokio 内部、malloc）
- context switch / wakeup 延迟

常用组合：
- **perf stat**：对比三种方式 syscall、ctx switch、cache miss  
  重点看：`context-switches`, `cpu-migrations`, `cycles`, `instructions`, `cache-misses`，以及 `syscalls:sys_enter_epoll_wait`, `syscalls:sys_enter_futex`, `syscalls:sys_enter_read`, `syscalls:sys_enter_write`
- **perf record + flamegraph**：看热点函数（bincode、uuid、hashmap、shared-memory read/write、tokio task 调度）
- **offcpu time / sched**（BPF 或 perf sched）：看尾延迟是不是被“睡眠/等待/锁”吃掉


## 3) 实验设计：先“测清楚”再“优化”

为了分离不同来源的延迟，建议至少跑两组：

1. **delay_us=0**：测“纯框架/传输/调度”开销（这能直接暴露 shm-uintr 为什么比 shm-uds 慢，是通知机制还是用户态逻辑）
2. **delay_us=1000**：测“业务占比 + 传输占比”，看传输优化还有没有意义

并且建议：
- 固定并发（比如 256 或 1024），先把变量收敛，否则噪声太大
- 关掉高频日志：尤其你 `shm-uintr` response listener 里有 `info!` 在循环里（见 [shm_transport_uintr.rs](file:///home/zwp/gateway-gRPC/gateway/src/shm_transport_uintr.rs#L99-L107)），如果默认 `RUST_LOG=info`，这会直接主导延迟和吞吐

---

## 4) 你这份代码里“很可能的瓶颈点”（基于结构推断）

在 shm-uds/eventfd/uintr 三种 transport 的共同热点：
- **Uuid + HashMap + oneshot**：每个请求都要生成 UUID、锁 pending map、oneshot 通道调度，这些会产生不少分配和锁竞争（见 [shm_transport_uds.rs](file:///home/zwp/gateway-gRPC/gateway/src/shm_transport_uds.rs#L132-L165)、[shm_transport_eventfd.rs](file:///home/zwp/gateway-gRPC/gateway/src/shm_transport_eventfd.rs#L206-L227)）
- **bincode serialize/deserialize**：payload 编解码成本在高 QPS 下会很明显
- **SharedMemoryRingBuffer read/write**：如果内部用了原子/内存屏障/拷贝，会在 perf flamegraph 里体现

三种通知机制差异点（用系统剖析能看到）：
- shm-uds：read/write + epoll（通常 syscall 多，但批处理好）
- shm-eventfd：eventfd read/write + AsyncFd（syscall + readiness）
- shm-uintr：用户态中断路径（可能减少 syscall，但会引入频繁打断/调度抖动；如果再叠加“定时协程 process_uintr_wakers”，会带来额外轮询成本）

---

## 5) 最推荐的分析落地顺序

- **第一步（最快能出结论）**：delay_us=0，把三种 transport 分别跑一次，配合 `perf stat`（看 syscall、ctx switch、cache miss 谁高）
- **第二步（找到热点）**：对 gateway 和 backend 分别 perf record，出火焰图，看是不是 UUID/bincode/pending map/shm read/write
- **第三步（算占比）**：加“跨进程 timestamps + 直方图”，得到每段 p50/p99 占比（用于指导你到底该优化通知、序列化还是调度）

如果你希望我直接把“分段计时/trace payload”那套 instrumentation 按这三种 transport 都接进去（并给你输出每段 p50/p99 的统计），你告诉我想看的分段列表（比如只关心 T2/T3/T4/B1/B2/B4），我可以在现有代码结构上最小侵入实现。

# 多个中断的覆盖问题

现在只有一个pending表示中断到达，那么在哪里将pending再次设置为false合适。

如何process_uintr_waker 中将协程唤醒并已经将pending设置为true，但是poll中读取的pending却是false从而导致返回Pending。

还有多个中断到达，如何确保后面的中断不会覆盖前面的中断。



## Pending 改成原子结构，避免锁

两个经典问题： 丢唤醒（lost wakeup） 和 覆盖/竞争（producer/consumer 互相覆盖） 。
把实现改成了“ 原子序号 seq + 已消费序号 consumed_seq ”的结构

## 1) pending 什么时候置回 false 合适？
结论： 只能在“消费者确认自己观察到中断”时置回（也就是 poll / 消费路径），不能在 process_uintr_wakers 里清。

原因是： process_uintr_wakers 的职责是“把等待者唤醒去 poll”，它不等于“事件已被消费”。如果它先清掉 pending，就会出现“任务被唤醒了，但 poll 看到 false → Pending”的事件丢失。

在新实现里，不再用 pending=false ，而是：

- 中断到达： seq += 1
- poll 观察到 seq != consumed_seq 后，把 consumed_seq = seq （等价于“把 pending 清空”）


