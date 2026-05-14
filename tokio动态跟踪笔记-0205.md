关于理解tokio的worker

除了主线程执行block_on() 会经过不同的路口最后调用 run以外，其他的worker线程全部都是使用blocing pool的 spawn创建出一个新的线程，然后由这个新的线程执行run，在run中进行local queue任务的调度，以及定期进入turn 查询外部事件。

一定要区分多个run，worker真正的run是Context的run()

```rust

tokio\src\runtime\scheduler\multi_thread\worker.rs

impl Context {
    fn run(&self, mut core: Box<Core>) -> RunResult {
        // Reset `lifo_enabled` here in case the core was previously stolen from
        // a task that had the LIFO slot disabled.
        self.reset_lifo_enabled(&mut core);

        // Start as "processing" tasks as polling tasks from the local queue
        // will be one of the first things we do.
        core.stats.start_processing_scheduled_tasks();

        while !core.is_shutdown {
            self.assert_lifo_enabled_is_correct(&core);

            if core.is_traced {
                core = self.worker.handle.trace_core(core);
            }

            // Increment the tick
            core.tick();

            // Run maintenance, if needed
            maintenance 中会检查tick 是否到达event_interval，如果是的话调用park_yield 查询外部的事件是否就绪
            core = self.maintenance(core);

            // First, check work available to the current worker.
            if let Some(task) = core.next_task(&self.worker) {
                core = self.run_task(task, core)?;
                continue;
            }

            // We consumed all work in the queues and will start searching for work.
            core.stats.end_processing_scheduled_tasks();

            // There is no more **local** work to process, try to steal work
            // from other workers.
            if let Some(task) = core.steal_work(&self.worker) {
                // Found work, switch back to processing
                core.stats.start_processing_scheduled_tasks();
                core = self.run_task(task, core)?;
            } else {
                // Wait for work
                core = if !self.defer.is_empty() {
                    self.park_yield(core)
                } else {
                    self.park(core)
                };
                core.stats.start_processing_scheduled_tasks();
            }
        }
    }
```


run函数中唯一负责的就是不断通过core.next_task或者core.steal_work拿到下一个任务，然后运行。

至于任务如何被唤醒并且放回到队列中，需要等到 worker 没有任务运行后进入 park 然后再进入turn 通过poll 得到就绪event，通过evant 和readiness 找到对应waker，然后国通waker唤醒任务。

waker如何唤醒任务，通过wake_by_val 调用self.schedule(); 这个schedule 会把任务唤醒然后重新放回队列尾部。

tokio spawn 对应生成一个任务Task 和一个上下文context，以及一个waker。不管这个spawn future中调用了多少个子future，都是共享这个waker，只用这个waker就可以把整个任务唤醒。


tokio\src\runtime\task\harness.rs
```rust
impl RawTask {
    pub(super) fn wake_by_val(&self) {
        use super::state::TransitionToNotifiedByVal;

        match self.state().transition_to_notified_by_val() {
            TransitionToNotifiedByVal::Submit => {
                // 调用wake后，如果可以提交，就调用schedule把任务放回队列中
                // 不同的rt对应不同的scheduler
                self.schedule();

                // Now that we have completed the call to schedule, we can
                // release our ref-count.
                self.drop_reference();
            }
        }
    }
```

# 用户态中断调度方案1

```rust
uintr(token).await?;

pub async fn uintr(token: UintrToken) -> std::io::Result<()> {
    UintrFuture { token }.await
}

use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Waker};

/// 表示某个 UINTR 中断源的句柄（可以理解为你说的 TOKEN）
#[derive(Clone)]
pub struct UintrToken {
    inner: Arc<Inner>,
}

struct Inner {
    /// 是否已经收到一次中断
    pending: Mutex<bool>,
    /// 当前在等这个中断的任务的 waker（最多一个）
    waker: Mutex<Option<Waker>>,
}

impl UintrToken {
    pub fn new() -> Self {
        Self {
            inner: Arc::new(Inner {
                pending: Mutex::new(false),
                waker: Mutex::new(None),
            }),
        }
    }

    /// 这个函数稍后会在 UINTR 中断处理路径里调用，用来“通知有中断到了”
    pub fn notify(&self) {
        // 标记 pending = true
        {
            let mut pending = self.inner.pending.lock().unwrap();
            *pending = true;
        }

        // 把 waker 取出来并唤醒
        if let Some(waker) = self.inner.waker.lock().unwrap().take() {
            waker.wake();
        }
    }
}
```

## 可行性分析

这个方案想要在UINTR 中断处理路径 调用waker.wake() 把任务唤醒，调度的事情就交个运行时。

相当于把中断处理和唤醒放到一起，调度另外。

但是有一个问题，就是中断上下文中能否调用waker

UINTR 的 handler 跟传统的信号处理函数非常像：

随时打断当前线程正在执行的任意代码；
运行在一个特殊的中断上下文（独立栈，不能睡眠）；
只能调用 async-signal-safe 的操作，不能做锁、内存分配等复杂行为[1]。


风险类别	具体表现
异步信号安全	wake() 不是 async-signal-safe，可能调用锁、分配内存、系统调用等
死锁	中断打断持锁代码，再在 handler 中重入同一锁
数据结构损坏	调度器内部状态在“不变式被破坏的中间态”被再次修改
栈/性能问题	handler 栈小、要求极短，wake() 可能调用链过长或遍历队列
可重入问题	多个 UINTR 嵌套导致对调度结构的并发修改，Tokio 并未设计为中断可重入
调试困难	问题常表现为“随机挂死/偶尔崩溃/偶尔任务不再被调度”，极难重现和排查


谁来在“正常上下文”中把 waker 叫起来？​
常见两种方式：

### 方式 A：由轮询任务负责 wake
写一个 Tokio 任务，周期性检查 pending，看到 true 就把保存的 waker 唤醒：

```rust
async fn uintr_driver(token: UintrToken) {
    loop {
        if token.inner.pending.swap(false, Ordering::Acquire) {
            if let Some(w) = token.inner.waker.lock().unwrap().take() {
                w.wake();   // 这里已经在正常 async 上下文中了
            }
        }
        // 稍微 sleep 一下避免 busy-loop
        tokio::time::sleep(std::time::Duration::from_micros(10)).await;
    }
}
```
优点：实现简单，不涉及信号安全问题；
缺点：有一点轮询开销。

### 方式 B：用 async-signal-safe 的“桥接原语”（pipe / eventfd）
在 handler 里只做 async-signal-safe 的操作，例如：

写一个 eventfd；
写一个管道的写端（这是 POSIX 保证 async-signal-safe 的少数操作之一）；
Tokio 这边用 AsyncFd 监听这个 fd，收到“可读”事件后，再去读取并 wake() 实际的 waker。
这样 wake 发生在普通 poll 路径下，完全避开中断上下文。

## 与Embassy的对比

Embassy 把“等待硬件事件的 Future”背后都挂在一个 AtomicWaker 上；
硬件中断触发 → 中断服务函数里调用对应驱动的 AtomicWaker::wake() → 执行器在中断上下文里 poll 被唤醒的任务 → Future 变为 Ready，await 返回。

Embassy 就是在中断上下文中wake的方式。



# ideas

模仿embasssy设计一个独立的运行时，有单独的线程负载waker，平时休眠，然后 handler中唤醒这个线程

```rust
async fn uintr_driver(token: UintrToken) {
    loop {
        token.wake_if_pending();
        // 避免 busy loop，小睡一会儿
        tokio::time::sleep(std::time::Duration::from_micros(10)).await;
    }
}

// 对外 API：你想要的 uintr(token).await? 形式
pub async fn uintr(token: UintrToken) -> std::io::Result<()> {
    UintrFuture { token }.await
}

use std::sync::{Arc, Mutex};
use std::sync::atomic::{AtomicBool, Ordering};
use std::task::{Waker, Context, Poll};
use std::future::Future;
use std::pin::Pin;

pub struct UintrToken {
    inner: Arc<Inner>,
}

struct Inner {
    pending: AtomicBool,          // UINTR handler 设置标志
    waker: Mutex<Option<Waker>>,  // 只在普通上下文访问
}

impl UintrToken {
    pub fn new() -> Self {
        Self {
            inner: Arc::new(Inner {
                pending: AtomicBool::new(false),
                waker: Mutex::new(None),
            }),
        }
    }

    // 在普通上下文中供“驱动任务”调用，用来真正唤醒
    pub fn wake_if_pending(&self) {
        if self.inner.pending.swap(false, Ordering::Acquire) {
            if let Some(waker) = self.inner.waker.lock().unwrap().take() {
                waker.wake();  // 安全：在普通上下文
            }
        }
    }
}
```

或者结合epoll
```rust
async fn uintr_driver(token: UintrToken, eventfd_fd: RawFd) -> io::Result<()> {
    use tokio::io::unix::AsyncFd;
    use tokio::io::Interest;

    let mut afd = AsyncFd::with_interest(eventfd_fd, Interest::READABLE)?;

    loop {
        afd.readable().await?;
        // 读出 eventfd 中的计数，清零
        let mut buf = [0u8; 8];
        let _ = nix::unistd::read(eventfd_fd, &mut buf)?;
        // 然后真正唤醒 Future
        token.wake_if_pending();
    }
}
```

## UINTR handler为什么 要遵守async‑signal‑safe

UINTR handler 在 Linux 用户态里和普通 signal handler 的危险程度是一样的：

也可以在任意时刻打断用户态线程；
也运行在同一个用户栈上；
也共享同一个 heap、同一套锁、同一个语言运行时。
所以，哪怕它的“触发源”是 CPU 新指令集，而不是传统的 POSIX 信号机制，它的执行语义对用户态来说几乎就是“一个高优先级的 SIG*** 信号处理函数”。

执行语义一样 → 风险一样 → 约束也必须一样
这就是为什么要按 async‑signal‑safe 的规则来约束 UINTR handler。

## ISR 里 Embassy 都可以直接 AtomicWaker::wake() 啊，为什么 UINTR 不行？

区别在于运行环境完全不同​：

MCU / 内核中的硬件中断（比如 Embassy 用的）
运行在 特权态​（内核态 / 裸机），有独立的中断栈或异常栈。
中断优先级、屏蔽关系由 NVIC/中断控制器严格控制。
整个系统的代码（包括 allocator、锁、任务调度）​都是为“可能在中断里被调用”设计和验证的​，可以用 cortex_m::interrupt::free 之类的原语建立受控的临界区。
没有 POSIX 线程库、glibc malloc、JIT、Tokio 这类复杂运行时。
所以 Embassy 敢说：

“这个 GenericAtomicWaker::wake() 是 designed for ISR，用原子 + 临界区包装好，可以在中断里安全调用。”
UINTR handler（Linux 用户态）
运行在 用户态​，用的就是当前线程的用户栈。
可能在任何用户代码中断，包括：
glibc 的 malloc / free 内部；
Rust runtime / C++ runtime / Go runtime 的调度/GC代码；
你自己的互斥锁保护的临界区里。
用户态的各种库 完全没按“可在 signal handler 里被重入调用”这个目标设计​。

UINTR handler 要遵守 async‑signal‑safe，是因为在 Linux 用户态，它的执行语义和普通 signal handler 一样：可以在任意时刻打断任意用户代码，跑在同一个用户栈和堆上。为了不破坏内存分配器、锁、运行时调度器等内部不变式，只能做 POSIX 规定的那类极小、可重入的操作。这跟 MCU/内核中的硬件 ISR 是完全不同的安全模型。

# 用户态中断方案二

只在中断handler中设置标志位，然后在tokio的调度路径中加入检测该标志位的代码，在该代码中执行wake()

目前有两个地方可以尝试，一个实在run()中，另一个是在turn()中。

如果是在run()中就是每一个任务调度运行返回后都检查一次，

如果在turn()中的话，就是多轮任务运行后检查一次。

如果在run中，还可以实现自己的wake函数，实现wake and run，改schedule()，增加一个标志说明需要yield就行。

但是都会带来一个问题，就是如果当前所有任务都运行完成，core进入 park后调用poll等待外部事件，但是即使有用户态中断到达也不会触发中断，无法让poll返回。

意思是epoll和用户态中断是不相兼容的，或者要把用户态中断注册到epoll去。

目前最大的问题是，中断等待的client线程因为没有任务可以运行了，进入了park，然后一直处于futex_wait。server发送用户态中断是无法唤醒client的，必须等待client从内核返回才会检查用户态中断。（不是阻塞在match self.poll.poll(events, max_wait), 因为会有一个max_wait的timeout）

还有被唤醒的任务唤醒是不一定运行在原本注册中断的线程上面。