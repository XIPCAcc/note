# Embassy的Reactor

Embassy 的Reactor没有一个中心化的事件循环，每种中断的上下文对应该中断自己的Reactor处理。

## EXTI GPIO

对于EXTI GPIO而言，16 个 EXTI 通道共用一个静态 AtomicWaker 数组EXTI_WAKERS。

每次中断到来后，在EXTI 的中断处理程序on_irq()中，使用pin作为下标，从EXTI_WAKERS数组找到对应的waker，然后即可唤醒等待该pin的协程。
```rust
embassy-stm32/src/exti/mod.rs
static EXTI_WAKERS: [AtomicWaker; EXTI_COUNT]

unsafe fn on_irq() {
    let bits = read_pending() & 0x0000FFFF;

    // Wake the tasks
    for pin in BitIter(bits) {
        EXTI_WAKERS[pin as usize].wake();
    }

    // Clear pending
    low_level::clear_exti_pending_mask(bits);

```

ExtiInputFuture的poll会将waker存入EXTI_WAKERS对应下标的位置，等待中断上下文唤醒。
```rust
embassy-stm32/src/exti/mod.rs

struct ExtiInputFuture<'a> {
    pin: PinNumber,
    drop: bool,
    phantom: PhantomData<&'a mut AnyPin>,
}

impl<'a> Future for ExtiInputFuture<'a> {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // 把当前任务的 waker 写入 EXTI_WAKERS[self.pin]
        EXTI_WAKERS[self.pin as usize].register(cx.waker());

        // 然后检查事件是否已经发生
        let imr = cpu_regs().imr(0).read();
        if !imr.line(self.pin as _) {
            Poll::Ready(())   // 中断已触发（IMR 被 on_irq 清掉）, 事件已发生
        } else {
            Poll::Pending     // 还没发生，挂起，等中断唤醒
        }
    }
}
```

## usart

usart 则通过State存储waker，然后在中断上下文中拿到State从而唤醒协程。

```rust
embassy-stm32/src/usart/buffered.rs

pub(super) struct State {
    rx_waker: AtomicWaker,  // 读等待者
    rx_buf: RingBuffer,
    tx_waker: AtomicWaker,  // 写等待者
    tx_buf: RingBuffer,
    ...
}


unsafe fn on_interrupt(r: Regs, state: &'static State) {

    if eager > 0 {
        if state.rx_buf.available() >= eager {
            state.rx_waker.wake();
        }
    } else {
        if state.rx_buf.is_half_full() {
            state.rx_waker.wake();
        }
    }

    if sr_val.idle() {
        state.rx_waker.wake();
    }

    state.tx_done.store(true, Ordering::Release);
    state.tx_waker.wake();

}
```

usart调用BufferedUartRx read()读取数据时，如果数据未就绪，就将waker存入state中，等待中断上下文唤醒。

```rust
embassy-stm32/src/usart/buffered.rs

impl<'d> BufferedUartRx<'d> {
    async fn read(&self, buf: &mut [u8]) -> Result<usize, Error> {
        poll_fn(move |cx| {
            let state = self.state;
            
            if buf_len != 0 {
                Poll::Ready(Ok(buf_len))
            } else {
                state.rx_waker.register(cx.waker());
                Poll::Pending
            }
        })
        .await

```


## Timer

Timer的中断上下文中调用trigger_alarm()，trigger_alarm()通过遍历整个Timer queue，判断Timer到期情况，然后取出到期的struct Timer中的waker并唤醒等待的协程。

```rust
embassy-stm32/src/time_driver/gp16.rs

pub(crate) struct RtcDriver {
    alarm: Mutex<CriticalSectionRawMutex, AlarmState>,
    queue: Mutex<CriticalSectionRawMutex, RefCell<Queue>>,
}

pub(crate) fn on_interrupt(&self) {
    critical_section::with(|cs| {
        if sr.ccif(1) && dier.ccie(1) {
            self.trigger_alarm(cs);  //  queue.next_expiration → 唤醒 Timer future
        }
    })
}

fn trigger_alarm(&self, cs: CriticalSection) {
    let mut next = self.queue.borrow(cs).borrow_mut().next_expiration(self.now());
    while !self.set_alarm(cs, next) {
        next = self.queue.borrow(cs).borrow_mut().next_expiration(self.now());
    }
}
```

Timer queue

```rust
embassy-time-queue-utils/src/queue_generic.rs

pub struct Queue {
    queue: ConstGenericQueue<QUEUE_SIZE>,
}

pub struct ConstGenericQueue<const QUEUE_SIZE: usize> {
    queue: Vec<Timer, QUEUE_SIZE>,
}

struct Timer {
    at: u64,
    waker: Waker,
}

/// Dequeues expired timers and returns the next alarm time.
pub fn next_expiration(&mut self, now: u64) -> u64 {
    let mut next_alarm = u64::MAX;

    let mut i = 0;
    while i < self.queue.len() {
        let timer = &self.queue[i];
        if timer.at <= now {
            let timer = self.queue.swap_remove(i);
            timer.waker.wake();
        } else {
            next_alarm = min(next_alarm, timer.at);
            i += 1;
        }
    }

    next_alarm
}

    pub fn schedule_wake(&mut self, at: u64, waker: &Waker) -> bool {
        self.queue
            .find(|timer| timer.waker.will_wake(waker))
            .map(|timer| {
                if timer.at > at {
                    timer.at = at;
                    true
                } else {
                    false
                }
            })
            .unwrap_or_else(|| {
                let mut timer = Timer {
                    waker: waker.clone(),
                    at,
                };

                loop {
                    match self.queue.push(timer) {
                        Ok(()) => break,
                        Err(e) => timer = e,
                    }

                    self.queue.pop().unwrap().waker.wake();
                }

                true
            })
    }
```

应用层Timer是如何注册并景waker放入队列的
```rust
embassy-time/src/timer.rs

pub struct Timer {
    expires_at: Instant,   // 绝对到期时间（ticks）
    yielded_once: bool,    // 防止首次 poll 立即返回的标志
}

impl Future for Timer {
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.yielded_once && self.expires_at <= Instant::now() {
            Poll::Ready(())
        } else {
            // 关键：向 time driver 注册唤醒，将waker放入RtcDriver 的queue队列
            embassy_time_driver::schedule_wake(self.expires_at.as_ticks(), cx.waker());
            self.yielded_once = true;
            Poll::Pending
        }
    }
}
```

总而言之，对于Embassy的reactor模式，如何从中断事件源反向查找到等待任务分成四种模式。

1. 全局数组按索引 — EXTI
2. 每外设实例静态单例 — USART
3. 静态单例内的动态数组 - Timer (ConstGenericQueue)
3. 内联在Task自身 — Timer (TimerQueueItem)
