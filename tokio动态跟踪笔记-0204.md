Contex 的run_task 拿到具体的任务运行是，如何转换任务状态的
```rust
tokio\src\runtime\scheduler\multi_thread\worker.rs

fn run_task(&self, task: Notified, mut core: Box<Core>) -> RunResult {
    // Run the task
    coop::budget(|| {
        ...
        task.run();
        ...
        // 然后是 LIFO 循环，可能继续执行 lifo_slot 里的其他任务
        ...
    })
```

这里的 task.run() 会走到 Tokio 内部的任务驱动逻辑，最终调用到：

RawTask::poll()
Harness::poll()
Harness::poll_inner()
→ 里面真正 poll 用户的 Future

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

                // 这里真正 poll 用户的 Future
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
                // 如果是Pending而且可以转换成Idle任务，这里返回PollFuture::Done
                transition_result_to_poll_future(transition_res)
            }
        }
    }

    pub(super) fn poll(self) {
        // We pass our ref-count to `poll_inner`.
        match self.poll_inner() {
            PollFuture::Notified => {
                // The `poll_inner` call has given us two ref-counts back.
                // We give one of them to a new task and call `yield_now`.
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
            PollFuture::Dealloc => {
                self.dealloc();
            }
            // 返回PollFuture::Done 表明任务被挂起
            PollFuture::Done => (),
        }
    }
```