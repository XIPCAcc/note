
handle.waker.wake()向一个 eventfd 写入数据，把阻塞在 epoll/poll 上的线程（通常是 I/O 驱动 / runtime 线程）从睡眠中唤醒。​作用对象是：阻塞在 epoll/poll/kqueue 上的 I/O 驱动线程（mio 的 Poll 所在那条线程）。
写 eventfd → I/O 线程的 poll() 立刻返回 → I/O 驱动醒来，去处理 I/O 事件、定时器等。
unpark作用对象是：Tokio 的 worker 线程（调度任务的那些线程）。
调用 unpark → 把一个正在 Condvar::wait() 上睡的 worker 叫醒 → 继续跑它的调度循环，从队列里取任务执行。
可以简单理解为：

wake() 是用来叫醒 I/O 反应器线程，
unpark 是用来叫醒执行任务的 worker 线程。

```rust
pub(crate) fn wake(&self) -> io::Result<()> {
        // The epoll emulation on some illumos systems currently requires
        // the eventfd to be read before an edge-triggered read event is
        // generated.
        // See https://www.illumos.org/issues/16700.
        #[cfg(target_os = "illumos")]
        self.reset()?;

        let buf: [u8; 8] = 1u64.to_ne_bytes();
        match (&self.fd).write(&buf) {
            Ok(_) => Ok(()),
            Err(ref err) if err.kind() == io::ErrorKind::WouldBlock => {
                // Writing only blocks if the counter is going to overflow.
                // So we'll reset the counter to 0 and wake it again.
                self.reset()?;
                self.wake()
            }
            Err(err) => Err(err),
        }
    }
```