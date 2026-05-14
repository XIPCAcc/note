
为了降低跟踪多任务的难度，改成运行单线程版本的tokio

```rust
#[tokio::main(flavor = "current_thread")]
async fn main() {
    println!("Hello World!");
    
    // 任务1 - 在可能的 worker 线程1 上执行
    let task1 = tokio::spawn(async {
        println!("[任务1] 在 {:?} 上启动", std::thread::current().id());
        
        for i in 1..=2 {
            println!("[任务1] 步骤 {}", i);
            tokio::time::sleep(Duration::from_millis(100)).await;
            println!("[任务1] 步骤 {} 唤醒", i);
        }
    });
    
    // 任务2 - 在可能的 worker 线程2 上执行
    let task2 = tokio::spawn(async {
        println!("[任务2] 在 {:?} 上启动", std::thread::current().id());
        tokio::time::sleep(Duration::from_millis(200)).await;
        println!("[任务2] 唤醒");
    });
    
    // 同时等待两个任务
    println!("等待两个任务");
    let (_result1, _result2) = tokio::join!(task1, task2);

    println!("Finish");

}

```


```rust
创建runtime后进入Runtime::block_on()->Runtime::block_on_inner()->CurrentThread::block_on()->runtime::context::enter_runtime()->CoreGuard::block_on()
->future.poll()

tokio\src\runtime\scheduler\current_thread\mod.rs
CoreGuard::block_on(future) {
    loop {
    // 未返回Ready则选择下一个任务(Notified类型)
    if handle.reset_woken() {
                    let (c, res) = context.enter(core, || {
                        crate::task::coop::budget(|| future.as_mut().poll(&mut cx))
                    });

                    core = c;

                    if let Ready(v) = res {
                        return (core, Some(v));
                    }
                }
 // 一批任务调度循环：最多执行 event_interval 个任务
    for _ in 0..handle.shared.config.event_interval {
    // Handle to the current thread scheduler
    let entry = core.next_task(handle); // 选择任务
    // 运行任务
    let (c, ()) = context.run_task(core, || {
                            #[cfg(tokio_unstable)]
                            context.handle.task_hooks.poll_start_callback(&task_meta);

                            task.run();

                            #[cfg(tokio_unstable)]
                            context.handle.task_hooks.poll_stop_callback(&task_meta);
                        });
    // 批次结束后，主动把执行权让给 driver：检查 I/O、定时器
    core = context.park_yield(core, handle);

}

CoreGuard::block_on()->Context::run_task()->LocalNotified::poll()->Core::poll() 然后开始运行由tokio:spawn()生成的其他任务
```

poll()执行完成后运行返回到poll_inner()完成任务状态的记录和切换（transition_result_to_poll_future()）
如果是返回Pending，能够转换为PollFuture::Done
```rust
enum PollFuture {
    Done,        // 轮询完成，任务空闲
    Notified,    // 轮询完成，但已被通知需要再次运行
    Dealloc,     // 需要释放任务内存
    Complete,    // Future 已完成
}

pub(super) fn poll(self) {
        // We pass our ref-count to `poll_inner`.
        match self.poll_inner() {
            PollFuture::Notified => {                
                self.core()
                    .scheduler
                    .yield_now(Notified(self.get_new_task()));

                self.drop_reference();
            }
            PollFuture::Complete => {
                self.complete();
            }
            PollFuture::Dealloc => {
                self.dealloc();
            }
            PollFuture::Done => (),
        }
    }

```

返回到CoreGuard::block_on()后，如果task队列空了，调用 context.park(core, handle)
CoreGuard::block_on()->Context::park()->Context::park_internal()->driver::Driver::park()->TimeDriver::park()->time::Driver::park()->Driver::park_internal()


sleep()

```rust
pub fn sleep(duration: Duration) -> Sleep {
    match Instant::now().checked_add(duration) {
        Some(deadline) => Sleep::new_timeout(deadline, location),
        None => Sleep::new_timeout(Instant::far_future(), location),
    }
}

impl Sleep {
    pub(crate) fn new_timeout(deadline: Instant, location: Option<&'static Location<'static>>, ) -> Sleep {
        let handle = scheduler::Handle::current();
        let entry = Timer::new(handle, deadline);
        Sleep { inner, entry }
    }

pub(crate) enum Timer {
        Traditional(time::TimerEntry),
        #[cfg(all(tokio_unstable, feature = "rt-multi-thread"))]
        Alternative(time_alt::Timer),
    }
```

sleep()创建struct Sleep后返回，然后调用await()进入Sleep future的poll()

Sleep::poll_elapsed() -> TimerEntry::poll_elapsed()
```rust
fn poll(mut self: Pin<&mut Self>, cx: &mut task::Context<'_>) -> Poll<Self::Output> {
        match ready!(self.as_mut().poll_elapsed(cx)) {
            Ok(()) => Poll::Ready(()),
            Err(e) => panic!("timer error: {e}"),
        }
}

Sleep
fn poll_elapsed(self: Pin<&mut Self>, cx: &mut task::Context<'_>) -> Poll<Result<(), Error>> {
        let coop = ready!(crate::task::coop::poll_proceed(cx));

        let result = me.entry.poll_elapsed(cx).map(move |r| {
            coop.made_progress();
            r
        });
}

TimerEntry
pub(crate) fn poll_elapsed( mut self: Pin<&mut Self>, cx: &mut Context<'_>, ) -> Poll<Result<(), super::Error>> {
    inner.state.poll(cx.waker())
}

fn poll(&self, waker: &Waker) -> Poll<TimerResult> {
    // We must register first. This ensures that either `fire` will
    // observe the new waker, or we will observe a racing fire to have set
    // the state, or both.
    self.waker.register_by_ref(waker);

    self.read_state()
}
```

poll()->register_by_ref()->do_register()

```rust
fn do_register<W>(&self, waker: W)
{
    // 通过compare_exchange获取锁
    match self.state.compare_exchange(WAITING, REGISTERING, Acquire, Acquire)
    {
        WAITING => {
                // 其实就是调用 waker.into_waker() 将waker从RawWaker传唤为Waker 并处理可能得异常
                let new_waker_or_panic = catch_unwind(move || waker.into_waker());

                match new_waker_or_panic {
                    // 如果生成new_waker就把new_waker 保存到self.waker
                    Ok(new_waker) => {
                        old_waker = self.waker.with_mut(|t| (*t).take());
                        self.waker.with_mut(|t| *t = Some(new_waker));
                    }
                    Err(panic) => maybe_panic = Some(panic),
                }
            }
        }
        WAKING => {
            // Currently in the process of waking the task, i.e.,
            // `wake` is currently being called on the old waker.
            // So, we call wake on the new waker.
            //
            // If this panics, someone else is responsible for restoring the
            // state of the waker.
            waker.wake();

            // This is equivalent to a spin lock, so use a spin hint.
            hint::spin_loop();
        }
    }
}
```

poll()->read_state()
```rust
检查计时器是否已经完成，并返回相应的结果
fn read_state(&self) -> Poll<TimerResult> {
    let cur_state = self.state.load(Ordering::Acquire);

    if cur_state == STATE_DEREGISTERED {
        Poll::Ready(unsafe { self.result.with(|p| *p) })
    } else {
        Poll::Pending
    }
}
```

AI说 Timer的定时功能是靠 turn()中传入的max_wait，然后self.poll.poll(events, max_wait) 控制的？而且和TOKEN_WAKEUP没关系？