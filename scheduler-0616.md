# tokio 任务队列的修改尝试

尝试在 用户态中断处理上下文中调用wake唤醒。

由于任务队列定义为 Mutex<RefCell<T>> ，强行借用会发生下面的错误。

thread 'main' (xxxx) panicked at tokio/tokio/src/runtime/scheduler/current_thread/mod.rs:662:40: RefCell already borrowed

比如刚好当前working thread 刚好拿到scheduler准备调度，然后来了一个用户态中断，进入wake 也准备借用scheduler就会出现这样的报错。
core.borrow_mut() [已借用]。

Embassy采用UnsafeCell
UnsafeCell 没有任何运行时检查 ， .get() 直接返回裸指针。这里靠的不是"借检查"，而是程序员+临界区来保证安全。

## 直接将tokio 中的任务队列改成RefCell 是否可行。

```
Running 30s test @ http://127.0.0.1:8080/api/echo
  8 threads and 16 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   399.22us  173.18us  14.38ms   92.95%
    Req/Sec     4.96k   509.97     5.76k    75.82%
  Latency Distribution
     50%  352.00us
     75%  397.00us
     90%  518.00us
     99%    0.95ms
  241095 requests in 30.04s, 41.62MB read
Requests/sec:   8026.32
Transfer/sec:      1.39MB
./scripts/run_shm_uintr.sh: line 30: 447599 Segmentation fault      (core dumped) $backend_cmd
```

UnsafeCell 产生 aliased &mut 引用
信号处理器(UINTR 中断)可以在 任意指令边界 抢占主线程。

## 无锁 Treiber 栈 替代对 core 的直接访问

| | Embassy RunQueue | 我们的方案 |
|---|---|---|
| 栈头 | `AtomicPtr<TaskRef>` | Core 中新增 `AtomicPtr<Header>` |
| 节点链接 | Task 内置 `next: AtomicPtr<Self>` | 复用 Header 已有的 `queue_next` |
| push | `head.fetch_and_set()` + CAS | 同模式 |
| pop_all | `head.swap(null)` | 同模式 |
| 信号安全 | 仅原子操作 | 仅原子操作 |

核心思路：
1. 信号 handler 不再走 `waker.wake()` → `schedule_task()` → `schedule_local()`（会碰 `&mut core`）
2. 改为直接 push 到 Treiber 栈（纯原子操作，信号安全）
3. 主 loop** 在 `next_task()` 前 drain Treiber 栈到 local queue

Context 新增 `signal_head: AtomicPtr<Header>`；`push_signal`/`drain_signal_to` 方法；主 loop 每次迭代 drain Treiber 栈 

| 操作 | 旧代码 | 新代码 |
|------|--------|--------|
| enqueue | `schedule_task` → `schedule_local` → 访问 `core.run_queue` (`&mut Core`) | `push_signal` → `AtomicPtr::compare_exchange` (纯原子) |
| dequeue | `next_task` → `run_queue.pop()` | `drain_signal_to` → `AtomicPtr::swap(null)` + 推入 local queue |

```
Running 30s test @ http://127.0.0.1:8080/api/echo
  8 threads and 16 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   725.56us    0.90ms   4.07ms   87.36%
    Req/Sec     1.82k     0.88k    3.63k    68.18%
  Latency Distribution
     50%  354.00us
     75%  439.00us
     90%    2.71ms
     99%    3.71ms
  3987 requests in 30.05s, 704.74KB read
Requests/sec:    132.66
Transfer/sec:     23.45KB

```

## 为什么tokio不能像Embassy一样用原子链表

Embassy的任务在编译时候即可确定，群补存在静态数组，每个任务的 TaskHeader 里嵌了一个链表节点，wake 时通过 CAS 把自己链入 executor 的运行链表；executor 轮询时 CAS 取出头部执行。任务内存是静态的，但"谁在就绪队列里"是动态变化的，靠链表而非数组来组织。

tokio的任务是动态的，虽然也可以用链表串接起来。但是 Tokio 的 **work-stealing 调度器设计**依赖数组队列的三个特性：

1. 批量窃取 (`steal_into`)：一次 CAS 拿走半队列，侵入式链表做不到，必须逐个 CAS
2. 溢出机制 (`push_overflow`)：本地队列满时把一半+当前任务推入 inject，链表没有"一半"的概念
3. LIFO slot：单 slot 缓存，依赖"只有一个生产者"。链表结构下这个优化无意义

而是work-stealing 设计选择了必须使用数组，而不能使用原子链表。


工程能力不够，改不了tokio的代码，牵一发动全身，魔改完发现性能很差。甚至跑不起来。

除了maintainer能够改改，其他人动不了。不亚于要参考ArceOS 把linux改成异步操作系统。

把中断和event同时放在一个tokio runtime中就是无法做到多核的唤醒。

必须要通过产生一个event的方式才能把epoll_wait 唤醒，才能执行epoll。这个就是tokio的根基。

          
## 可行性分析

### 技术上可以做到，但改了之后能工作的部分和不能工作的部分

可以做到：

| 步骤 | 说明 |
|------|------|
| 新增 Header 字段 | 加一个 `run_queue_next` 指针，侵入式链表需要 |
| 原子链表 push/pop | CAS 操作，单步原子，信号安全 |
| 原子 LIFO slot | 改为 `AtomicPtr` 后也能 CAS 操作 |

做不到（或代价很大）：

### 1. LIFO slot 存在根本性竞态（即使原子化）

```rust
// schedule_local 里 LIFO 的两步操作：
let prev = lifo_slot.take();    // 步骤1
if let Some(prev) = prev {
    run_queue.push_back(prev);   // 步骤2（原子，OK）
}
lifo_slot = Some(task);          // 步骤3
```

信号在步骤1之后、步骤3之前中断 → 信号也做 `take()` → 拿到 `None` → 写入自己的 task → **主线程的步骤3 覆盖了信号的 task → 任务丢失**。

解决：信号处理器永远不走 LIFO，直接 push 到链表。但需要识别"当前在信号上下文"。

### 2. 批量窃取丢失

Chase-Lev 支持 `steal_into` 一次 CAS 偷走半队列。换成链表之后只能一个个 CAS 偷 — 高竞争下性能掉几个量级。

### 3. 溢出检测需要额外计数字段

数组天然有容量上限（256），满载时触发溢出。链表无上限，需要显式维护原子计数器或定期扫描 `queue_next` 链表长度来判断是否溢出到 inject。

### 4. 缓存局部性恶化

数组队列（连续内存） vs 链表（任务散布在堆上）。高吞吐场景下 cache miss 显著增加。

### 5. 每步 push/pop 从非原子变成 CAS

当前 `push` 热路径：`tail` 用 `unsync_load`（直接读内存，~1ns），只需一次 `Release store`。改成 CAS 链表后，每次 push/pop 都是 CAS（~20-30ns），热路径慢 10-20 倍。

# 一定要使用epoll吗

Linux 没有把"socket 可读"转成用户态中断的机制。 epoll 是内核通知用户态 IO 事件的唯一接口。

```
用户态:    epoll_wait(fd1, fd2, ...)
               ↑
内核:     TCP 栈 → socket ready → 唤醒 epoll
          定时器 → epoll 超时
          eventfd → epoll 返回
          （IO 就绪通知只走 epoll 这一条路）
```