首先在主线程找不到task运行以后，或者在运行了handle.shared.config.event_interval（默认好像是61个）个任务后，执行一个park
就会进入park准备休眠，在park执行中会检查和处理IO事件（时间，signal的执行都被集成到IO stack中了，实现完全的事件驱动，尽管他们不是一般意义上的IO）

Tokio 运行时在没有任务可执行时，会进入 park 状态等待事件。park 会计算下一个需要处理的事件（IO、计时器、信号等）的时间，然后通过一次系统调用等待这些事件。当事件触发时，处理相应的事件并唤醒等待的任务。

早期的Timer都是有一个单独的Timer thread负责的，并且直接调用系统调用的libc::nanosleep()

现在的tokio采用 分层时间轮 + 共享驱动(Timerfd + epoll) ，而且是通过mio库使用

Context::park()->Context::park_internal()->driver::Driver::park()->TimerDriver::park()->time::Driver::park()->time::Driver::park_internal()->IoStack::park_timeout()->process::Driver::park_timeout()->singal::Driver::park_timeout()->io::driver::Driver::park_timeout()->io::driver::Driver::turn()->

turn()是 Tokio 事件驱动架构的引擎，将阻塞的 I/O 等待转换为非阻塞的事件通知。
它连接了：操作系统的 I/O 多路复用，Tokio 的任务调度，用户代码的执行
turn()的作用是执行一次事件循环迭代：
1.等待事件：通过 poll等待 I/O、计时器或信号
2.处理事件：将就绪的事件分发给对应的处理程序
3.唤醒任务：唤醒等待这些事件的异步任务
4.推动执行：使被唤醒的任务可以被调度执行
5.维护状态：更新内部状态和指标

```rust
fn turn(&mut self, handle: &Handle, max_wait: Option<Duration>) {

        handle.release_pending_registrations(); // 处理在上一轮事件循环中新增的 I/O 资源注册

        let events = &mut self.events;

        // 阻塞等待事件发生，并统计发生了多少个事件
        // 调用 Mio 的 poll
        match self.poll.poll(events, max_wait) {
            Ok(()) => {}
            Err(ref e) if e.kind() == io::ErrorKind::Interrupted => {}
            #[cfg(target_os = "wasi")]
            Err(e) if e.kind() == io::ErrorKind::InvalidInput => {
                // In case of wasm32_wasi this error happens, when trying to poll without subscriptions
                // just return from the park, as there would be nothing, which wakes us up.
            }
            Err(e) => panic!("unexpected error when polling the I/O driver: {e:?}"),
        }

        // 处理返回的Event列表
        let mut ready_count = 0;
        for event in events.iter() {
            let token = event.token();

            if token == TOKEN_WAKEUP {
                // 无需特殊处理，事件已完成了它的使命（解除阻塞）
                // TOKEN_WAKEUP的唯一作用是解除 I/O 驱动的阻塞状态，让 poll()提前返回。一旦 poll()返回，它的使命就完成了。
            } else if token == TOKEN_SIGNAL {
                self.signal_ready = true; // 设置标志位，稍后处理信号
            }        {
                // 下面的情况是tcp 唤醒之类会走的

                let ready = Ready::from_mio(event); // 将 mio::event::Event转换为 Tokio 的 Ready标志
                let ptr = super::EXPOSE_IO.from_exposed_addr(token.0);

                // Safety: we ensure that the pointers used as tokens are not freed
                // until they are both deregistered from mio **and** we know the I/O
                // driver is not concurrently polling. The I/O driver holds ownership of
                // an `Arc<ScheduledIo>` so we can safely cast this to a ref.
                let io: &ScheduledIo = unsafe { &*ptr };

                io.set_readiness(Tick::Set, |curr| curr | ready); // 设置 I/O 资源的就绪状态
                io.wake(ready); // 唤醒等待的任务，内部调用waker

                ready_count += 1;
            }
        }

        {
            let mut guard = handle.get_uring().lock();
            let ctx = &mut *guard;
            ctx.dispatch_completions();
        }

        handle.metrics.incr_ready_count_by(ready_count);
    }
```

如果turn中有收到signal Token，那么在返回到singal::Driver::park_timeout()->singal::Driver::process()时会处理signal

```rust
fn process(&mut self) {
        // If the signal pipe has not received a readiness event, then there is
        // nothing else to do.
        if !self.io.consume_signal_ready() {
            return;
        }

        // Drain the pipe completely so we can receive a new readiness event
        // if another signal has come in.
        let mut buf = [0; 128];
        // clippy::unused_io_amount：忽略"未使用读取字节数"的警告,因为我们只关心是否有数据，不关心具体内容
        #[allow(clippy::unused_io_amount)]
        loop {
            // 当信号到达时，会写入一个管道（或 eventfd），工作线程从这个管道读取数据来知道有信号需要处理。
            match self.receiver.read(&mut buf) {
                Ok(0) => panic!("EOF on self-pipe"),
                Ok(_) => continue, // Keep reading
                Err(e) if e.kind() == std_io::ErrorKind::WouldBlock => break, // 无数据可读（非阻塞） 退出循环
                Err(e) => panic!("Bad read on self-pipe: {e}"),
            }
        }

        // 通知所有监听者信号已到达
        globals().broadcast();
    }
```

返回到time::Driver::park_internal()时候调用Handle:process()->Handle::process_at_time()处理已经到期的计时器，并唤醒等待的任务。

```rust
pub(self) fn process_at_time(&self, mut now: u64) {
        let mut waker_list = WakeList::new();

        if now < lock.wheel.elapsed() { // 时间回退场景
            now = lock.wheel.elapsed();
        }

        // 不断从时间轮中取出在 now时刻（或之前）到期的计时器
        while let Some(entry) = lock.wheel.poll(now) {
            // entry.fire 标记计时器为已触发 返回关联的 Waker
            if let Some(waker) = unsafe { entry.fire(Ok(())) } {
                waker_list.push(waker);
            }
        }
        // 唤醒列表中剩余的所有任务
        waker_list.wake_all();
    }
```

```rust
fn fire(&self, result: TimerResult) -> Option<Waker> {
        let cur_state = self.state.load(Ordering::Relaxed);
        if cur_state == STATE_DEREGISTERED {
            return None;
        }
        self.state.store(STATE_DEREGISTERED, Ordering::Release);

        self.waker.take_waker()
    }
```

目前的问题是，单步调试的时候，wake_all()不能够跳转到 Waker的wake函数()
```rust
pub(crate) fn wake_all(&mut self) {
        while !ptr::eq(guard.start, guard.end) {
            // SAFETY: `start` is always initialized if `start != end`.
            let waker = unsafe { ptr::read(guard.start) };
            // SAFETY: The resulting pointer is in bounds or one after the length of the same object.
            guard.start = unsafe { guard.start.add(1) };
            // If this panics, then `guard` will clean up the remaining wakers.
            waker.wake();
        }
    }
```

## 

每个线程定义了一个CURRENT_PARKER TLS，通过CachedParkThread  提供的API可以快速访问这个ParkThread
tokio_thread_local! {
    static CURRENT_PARKER: ParkThread = ParkThread::new();
}