# Embassy Timer

Embassy 定时器队列的两条实现路径：TimerQueueItem 是"集成式队列"（槽位内联在任务里），ConstGenericQueue 是"通用队列"（独立固定容量数组）

```rust
#[cfg(all(not(feature = "integrated-timers"), feature = "_generic-queue"))]
type QueueImpl = queue_generic::Queue;      // 用 ConstGenericQueue

#[cfg(any(feature = "integrated-timers", not(feature = "_generic-queue")))]
type QueueImpl = queue_integrated::Queue;   // 用 TimerQueueItem 集成式
```

## TimerQueueItem —— 任务自带的原始槽位

TimerQueueItem 是 TaskHeader 的一个字段。

```rust
struct QueueItem {
    pub next: Cell<Option<NonNull<QueueItem>>>,  // 指向下一个节点的侵入式指针
    pub expires_at: u64,                          // 绝对到期时间
    pub waker: Option<Waker>,                     // None = 不在队列中
}

#[repr(align(8))]
pub struct TimerQueueItem {
    data: [usize; ITEM_WORDS],  // 保存QueueItem的数组
}

pub(crate) struct TaskHeader {
    pub(crate) state: State,
    pub(crate) run_queue_item: RunQueueItem,
    pub(crate) executor: AtomicPtr<SyncExecutor>,
    poll_fn: SyncUnsafeCell<Option<unsafe fn(TaskRef)>>,
    /// Integrated timer queue storage. This field should not be accessed outside of the timer queue.
    pub(crate) timer_queue_item: TimerQueueItem,   // 每个任务自带一个定时器槽位
    pub(crate) metadata: Metadata,
}
```

## ConstGenericQueue —— 独立固定容量的通用队列

```rust
embassy-time-queue-utils/src/queue_generic.rs

pub struct ConstGenericQueue<const QUEUE_SIZE: usize> {
    queue: Vec<Timer, QUEUE_SIZE>,
}

struct Timer {
    at: u64,
    waker: Waker,
}

pub fn schedule_wake(&mut self, at: u64, waker: &Waker) -> bool {
    self.queue
        .iter_mut()
        .find(|timer| timer.waker.will_wake(waker))   // ① 已存在则更新时间
        .map(|timer| {
            if timer.at > at { timer.at = at; true } else { false }
        })
        .unwrap_or_else(|| {
            // ② 不存在则 push；满了就 pop 最老的一个并 wake 它（驱逐）
            let mut timer = Timer { waker: waker.clone(), at };
            loop {
                match self.queue.push(timer) {
                    Ok(()) => break,
                    Err(e) => timer = e,
                }
                self.queue.pop().unwrap().waker.wake();  // ★ 容量满了，唤醒最老的
            }
            true
        })
}

pub fn next_expiration(&mut self, now: u64) -> u64 {
    let mut next_alarm = u64::MAX;
    let mut i = 0;
    while i < self.queue.len() {
        let timer = &self.queue[i];
        if timer.at <= now {
            let timer = self.queue.swap_remove(i);   // 移除并保持紧凑
            timer.waker.wake();                       // 到期唤醒
        } else {
            next_alarm = min(next_alarm, timer.at);
            i += 1;
        }
    }
    next_alarm
}
```

两者殊途同归，都实现了同一个 schedule_wake(at, waker) + next_expiration(now) 接口，向上被 RtcDriver::schedule_wake / trigger_alarm 调用。

它们的本质差异在于槽位放在哪里：

集成式（TimerQueueItem）：槽位跟着任务走，用 waker 作为"钥匙"反查任务内存。这是 Embassy 的标志性零分配方案。
通用式（ConstGenericQueue）：槽位跟着队列走，waker 作为值被复制进队列。牺牲了无上限和无堆，换来了与外部异步运行时兼容的灵活性。