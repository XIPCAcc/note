Wake任务调度
Harness是对Future和Schedule的封装，负责维护任务状态
tokio\src\runtime\task\harness.rs
```rust
impl<T, S> Harness<T, S>
    fn poll_inner(&self) -> PollFuture {
        use super::state::{TransitionToIdle, TransitionToRunning};

        match self.state().transition_to_running() {
            TransitionToRunning::Success => {
                // 如果任务被调度进入运行态，那么开始创建waker准备轮询任务
                let header_ptr = self.header_ptr();
                // 这里使用的waker_ref 创建WakerRef
                let waker_ref = waker_ref::<S>(&header_ptr);
                let cx = Context::from_waker(&waker_ref);
                let res = poll_future(self.core(), cx);

                if res == Poll::Ready(()) {
                    // The future completed. Move on to complete the task.
                    return PollFuture::Complete;
                }

                // 如果返回的其他状态，那么转换返回状态
                let transition_res = self.state().transition_to_idle();
                if let TransitionToIdle::Cancelled = transition_res {
                    // The transition to idle failed because the task was
                    // cancelled during the poll.
                    cancel_task(self.core());
                }
                transition_result_to_poll_future(transition_res)
            }
        }
    }
```

waker_ref 创建WakerRef

tokio\src\runtime\task\waker.rs
```rust
pub(super) fn waker_ref<S>(header: &NonNull<Header>) -> WakerRef<'_, S>
{
    let waker = unsafe { ManuallyDrop::new(Waker::from_raw(raw_waker(*header))) };

    WakerRef {
        waker,
        _p: PhantomData,
    }
}

使用raw_waker创建出RawWaker
fn raw_waker(header: NonNull<Header>) -> RawWaker {
    let ptr = header.as_ptr() as *const ();
    RawWaker::new(ptr, &WAKER_VTABLE)
}
RawWaker中的虚函数表是WAKER_VTABLE
static WAKER_VTABLE: RawWakerVTable =
    RawWakerVTable::new(clone_waker, wake_by_val, wake_by_ref, drop_waker);

其中的wake和wake_by_ref 分别是RawTask的wake_by_val和wake_by_ref 
unsafe fn wake_by_val(ptr: *const ()) {
    let ptr = unsafe { NonNull::new_unchecked(ptr as *mut Header) };
    let raw = unsafe { RawTask::from_raw(ptr) };
    raw.wake_by_val();
}

```

tokio\src\runtime\task\harness.rs
```rust
impl RawTask {
    pub(super) fn wake_by_val(&self) {
        use super::state::TransitionToNotifiedByVal;

        match self.state().transition_to_notified_by_val() {
            TransitionToNotifiedByVal::Submit => {
                // 调用wake后，如果可以提交，就调用schedule把任务放回队列中
                self.schedule();

                // Now that we have completed the call to schedule, we can
                // release our ref-count.
                self.drop_reference();
            }
        }
    }
```

tokio\src\runtime\task\harness.rs
```rust
pub(super) fn poll(self) {
        // poll调用了poll_inner
        match self.poll_inner() {
            PollFuture::Notified => {
                // 如果poll_inner内部调用future的poll返回的是Pending，那么这里调用yield_now，将任务放回任务队列
                self.core()
                    .scheduler
                    .yield_now(Notified(self.get_new_task()));

                // The remaining ref-count is now dropped. We kept the extra
                // ref-count until now to ensure that even if the `yield_now`
                // call drops the provided task, the task isn't deallocated
                // before after `yield_now` returns.
                self.drop_reference();
            }
            PollFuture::Complete => {
                self.complete();
            }
```

```rust
    fn yield_now(&self, task: Notified<Self>) {
        self.schedule_task(task, true);
    }
```

以上是Tokio 的 Waker是任务级唤醒器
Driver中使用的是mio::Waker，是系统级唤醒器
Tokio 中使用 mio::Waker的场景：

系统级唤醒：唤醒整个事件循环

跨线程通知：从任何线程唤醒 I/O 驱动

定时器触发：定时器到期时唤醒

信号处理：收到信号时唤醒

外部事件：自定义事件源通知

触发时机：

新任务到达且工作线程都在休眠

定时器到期需要处理

收到系统信号

自定义事件就绪

需要重新计算超时时间

与 tokio::task::Waker的关系：

mio::Waker：唤醒驱动处理系统事件

tokio::task::Waker：唤醒任务继续执行

先有系统事件，再有任务唤醒

有外部事件到达后，调用的wake是任务的waker
```rust
fn turn() {
    ......
    for event in events.iter() {
                let token = event.token();

                if token == TOKEN_WAKEUP {
                    // Nothing to do, the event is used to unblock the I/O driver
                } else if token == TOKEN_SIGNAL {
                    self.signal_ready = true;
                } else {
                    let ready = Ready::from_mio(event);
                    let ptr = super::EXPOSE_IO.from_exposed_addr(token.0);

                    // Safety: we ensure that the pointers used as tokens are not freed
                    // until they are both deregistered from mio **and** we know the I/O
                    // driver is not concurrently polling. The I/O driver holds ownership of
                    // an `Arc<ScheduledIo>` so we can safely cast this to a ref.
                    let io: &ScheduledIo = unsafe { &*ptr };

                    io.set_readiness(Tick::Set, |curr| curr | ready);
                    io.wake(ready);

                    ready_count += 1;
                }
            }
}

```

tokio\src\runtime\io\scheduled_io.rs
```rust
pub(super) fn wake(&self, ready: Ready) {
        let mut wakers = WakeList::new();
        let mut waiters = self.waiters.lock();
        // check for AsyncRead slot
        if ready.is_readable() {
            if let Some(waker) = waiters.reader.take() {
                wakers.push(waker);
            }
        }

        // check for AsyncWrite slot
        if ready.is_writable() {
            if let Some(waker) = waiters.writer.take() {
                wakers.push(waker);
            }
        }

        'outer: loop {
            let mut iter = waiters.list.drain_filter(|w| ready.satisfies(w.interest));

            while wakers.can_push() {
                match iter.next() {
                    Some(waiter) => {
                        let waiter = unsafe { &mut *waiter.as_ptr() };

                        if let Some(waker) = waiter.waker.take() {
                            waiter.is_ready = true;
                            wakers.push(waker);
                        }
                    }
                    None => {
                        break 'outer;
                    }
                }
            }

            drop(waiters);

            wakers.wake_all();

            // Acquire the lock again.
            waiters = self.waiters.lock();
        }

        // Release the lock before notifying
        drop(waiters);

        wakers.wake_all();
    }
```

tokio\src\util\wake_list.rs
```rust
pub(crate) fn wake_all(&mut self) {

        let mut guard = {
            let start = self.inner.as_mut_ptr().cast::<Waker>();
            // SAFETY: The resulting pointer is in bounds or one after the length of the same object.
            let end = unsafe { start.add(self.curr) };
            // Transfer ownership of the wakers in `inner` to `DropGuard`.
            self.curr = 0;
            DropGuard { start, end }
        };
        while !ptr::eq(guard.start, guard.end) {
            // SAFETY: `start` is always initialized if `start != end`.
            let waker = unsafe { ptr::read(guard.start) };
            // SAFETY: The resulting pointer is in bounds or one after the length of the same object.
            guard.start = unsafe { guard.start.add(1) };
            // If this panics, then `guard` will clean up the remaining wakers.
            waker.wake();
        }
    }
}
```