
```rust
async fn run_server(args: &Args) -> Result<(), Box<dyn Error>> {
        // 发送信号到客户端
        let _ = send_signal(0, NixSignal::SIGUSR2);
        // 等待客户端的响应信号
        signal(libc::SIGUSR1).await?;
}

async fn run_client(args: &Args) -> Result<(), Box<dyn Error>> {
        // 等待来自服务器的信号
        signal(libc::SIGUSR2).await?;
        // 向进程组发送信号（使用PID 0）
        let _ = send_signal(0, NixSignal::SIGUSR1);
}
```

```rust
/// 发送信号到目标 PID
fn send_signal(pid: u32, signal: NixSignal) -> Result<(), Box<dyn Error>> {
    let nix_pid = Pid::from_raw(pid as i32);
    kill(nix_pid, signal)?;
    Ok(())
}
```

## Unix 实现

use compio::signal::unix::signal;

compio = { version = "0.8.0", features = ["signal"] }

早期的compio没有使用io uring，所以创建了一套机制等待信号。

Unix signal的本质是注册signal handler负责接受对应的信号。

那么如何将这个信号和原本的协程做绑定呢？或者说如何让这个信号负责唤醒异步等待该信号的协程呢？

会创建一个os_pipe管道，写端交给signal handler，当有信号到达时，signal handler向 pipe写入数据

管道的另一端则交给专门创建出来等待信号的线程，该线程执行read阻塞等待管道的数据。

一旦等待信号的线程被唤醒，则找到对应的协程Event handler，调用其notify（底层是wake）唤醒协程。

和tokio处理信号的异同，都是通过signal handler接受信号，然后通过管道读写通知信号到达。

不同的是，tokio会将管道的读端注册到epoll，由epoll负责返回事件负责唤醒协程，不需要专门的信号线程阻塞等待.

```rust
pub async fn signal(sig: i32) -> io::Result<()> {
    let fd = SignalFd::new(sig)?;
    fd.wait().await?;
    Ok(())
}

impl SignalFd {
    fn new(sig: i32) -> io::Result<Self> {
        let event = Event::new();
        let key = register(sig, &event)?;
        Ok(Self {
            sig,
            key,
            event: Some(event),
        })
}
```


```rust
fn register(sig: i32, fd: &Event) -> io::Result<usize> {
    unsafe { init(sig)? };
    // 注册EventHandle，方便调用notify 唤醒当前协程
    let handle = fd.handle();
    let key = HANDLER
        .lock()
        .unwrap()
        .entry(sig)
        .or_default()
        .insert(handle);
    Ok(key)
}

unsafe fn init(sig: i32) -> io::Result<()> {
    // 定义一个全局的static PIPE: LazyLock<Pipe> = LazyLock::new(|| Pipe::new().unwrap());
    // 并在此处检查其初始化
    let _ = PIPE.deref();
    // 为信号注册一个处理函数
    if libc::signal(sig, signal_handler as *const () as usize) == libc::SIG_ERR {
        Err(io::Error::last_os_error())
    } else {
        Ok(())
    }
}

static PIPE: LazyLock<Pipe> = LazyLock::new(|| Pipe::new().unwrap());
// 执行的Pipe new() 会spawn出一个线程负责作为 Pipe receiver 
// real_signal_handler 负责调用Eventhandler notify唤醒线程
impl Pipe {
    pub fn new() -> io::Result<Self> {
        let (receiver, sender) = os_pipe::pipe()?;

        std::thread::spawn(move || {
            real_signal_handler(receiver);
        });

        Ok(Self { sender })
    }

    pub fn send(&self, sig: i32) -> io::Result<()> {
        (&self.sender).write_all(&sig.to_ne_bytes())?;
        Ok(())
    }
}

// 信号处理函数只负责向PIPE写入信号
unsafe extern "C" fn signal_handler(sig: i32) {
    PIPE.send(sig).unwrap();
}

// 处理信号的实际函数，在单独的线程中运行，会阻塞读取 pipe 中的数据
fn real_signal_handler(mut receiver: PipeReader) {
    loop {
        let mut buffer = [0u8; 4];
        // 此处会阻塞读取 pipe 中的数据
        let res = receiver.read_exact(&mut buffer);
        if let Ok(()) = res {
            let sig = i32::from_ne_bytes(buffer);
            let mut handler = HANDLER.lock().unwrap();
            if let Some(fds) = handler.get_mut(&sig)
                && !fds.is_empty()
            {
                let fds = std::mem::take(fds);
                for (_, fd) in fds {
                    // 唤醒等待该信号的任务
                    fd.notify();
                }
            }
        } else {
            break;
        }
    }
}

```

注册信号处理函数后，回到signal 开始wait
```rust
    async fn wait(mut self) {
        self.event
            .take()
            .expect("event could not be None")
            .wait()
            .await
    }
}
```

本质上是在等待Event的wait
```rust
compio-runtime\src\event.rs

impl Event {
    /// Wait for [`EventHandle::notify`] called.
    // 本身wait就是一个async块，所以可以被 await
    // 编译器将wait转换future，其poll中调用了self.flag.await
    pub async fn wait(self) {
        self.flag.await
    }
}

impl Future for Flag {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        // quick check to avoid registration if already done.
        if self.0.set.load(Ordering::Relaxed) {
            return Poll::Ready(());
        }

        self.0.waker.register(cx.waker());

        if self.0.set.load(Ordering::Relaxed) {
            Poll::Ready(())
        } else {
            Poll::Pending
        }
    }
}

self.0拿到的是Arc<Inner>，
struct Flag(Arc<Inner>);

self.0.set拿到的是set: AtomicBool
struct Inner {
    waker: AtomicWaker,
    set: AtomicBool,
}

self.0.set.load 访问到set: AtomicBool得到其值如果是true，返回ready

Flag的notify会负责将set 设为true
impl Flag {
    pub fn notify(&self) {
        self.0.set.store(true, Ordering::Relaxed);
        self.0.waker.wake();
    }
```

一层层Pending返回后，来到runtime的 run
```rust
compio-runtime-0.2.0/src/runtime/mod.rs

    pub fn run(&self) {
        loop {
            let next_task = self.runnables.pop();
            if let Some(task) = next_task {
                task.run();
            } else {
                break;
            }
        }
    }
```
如果不再有任务在返回到block_on
```rust
    pub fn block_on<F: Future>(&self, future: F) -> F::Output {
        let mut result = None;
        unsafe { self.spawn_unchecked(async { result = Some(future.await) }) }.detach();
        loop {
            self.run();
            if let Some(result) = result.take() {
                return result;
            }
            self.poll();
        }
    }
```
接着执行poll
```rust
compio-runtime-0.2.0/src/runtime/mod.rs

    pub fn poll(&self) {
        instrument!(compio_log::Level::DEBUG, "poll");
        let timeout = self.current_timeout();
        debug!("timeout: {:?}", timeout);
        self.poll_with(timeout)
    }
```


## linux 实现

linux 下基于io_uring的实现本质上将signal读写通过libc::signalfd()转换为文件的读写并构造io_uring的读操作并提交至submit queue。

compio runtime通过sys::io_uring_enter 提交请求并且阻塞等待completion queue，直到内核通过等待队列机制将进程唤醒。

compio runtime再根据completion queue entry 中的user data 找到对应future的handle并唤醒future。

```rust
compio = { version = "0.17.0", features = ["io-uring", "signal"] }

struct SignalFd {
    fd: SharedFd<OwnedFd>,
    sig: i32,
}

pub async fn signal(sig: i32) -> io::Result<()> {
    let fd = SignalFd::new(sig)?;
    fd.wait().await?;
    Ok(())
}

impl SignalFd {
    fn new(sig: i32) -> io::Result<Self> {
        // 1. 把这个 signal 加入当前线程的屏蔽集，避免跳转到信号处理函数
        let set = register_signal(sig)?;  // 内部用 pthread_sigmask(SIG_BLOCK, ...)

        
        let mut flags = libc::SFD_CLOEXEC;
        if Runtime::with_current(|rt| rt.driver_type().is_polling()) {
            flags |= libc::SFD_NONBLOCK;
        }

        // 2. 创建 signalfd 信号不再以传统“异步打断”的方式送到线程，而是通过 signalfd 这个 fd 送达
        let raw_fd = libc::signalfd(-1, &set, flags);
        // 3. 把 raw_fd 封成 OwnedFd -> SharedFd，交给 compio 的 driver 管理
        ...
        Ok(SignalFd { fd, sig })
    }

    async fn wait(self) -> io::Result<()> {

        let info = SignalInfo(MaybeUninit::<libc::signalfd_siginfo>::uninit());
        // 构造一个读操作
        let op = Recv::new(self.fd.clone(), info);
        // 提交给 runtime 等待结果，此时如果没有ready，就返回Pending，退出到block
        let BufResult(res, op) = compio_runtime::submit(op).await;
       
        // 取出结果
        let info = op.into_inner();
        let info = unsafe { info.0.assume_init() };
        Ok(())
    }
}
```

```rust
compio_runtime::submit(op)位于 compio-runtime-0.10.1/src/runtime/mod.rs
```rust
/// Submit an operation to the current runtime, and return a future for it.
pub async fn submit<T: OpCode + 'static>(op: T) -> BufResult<usize, T> {
    submit_with_flags(op).await.0
}

pub async fn submit_with_flags<T: OpCode + 'static>(op: T) -> (BufResult<usize, T>, u32) {
    Runtime::with_current(|r| r.submit_with_flags(op)).await
}

fn submit_with_flags<T: OpCode + 'static>(
    &self,
    op: T,
) -> impl Future<Output = (BufResult<usize, T>, u32)> + use<T> {
    match self.submit_raw(op) {
        // 这里返回的是Either future（use futures_util::future::Either）
        PushEntry::Pending(user_data) => Either::Left(OpFuture::new(user_data)),
        PushEntry::Ready(res) => {
            // submit_flags won't be ready immediately, if ready, it must be error without
            // flags
            Either::Right(ready((res, 0)))
        }
    }
}

fn submit_raw<T: OpCode + 'static>(&self, op: T) -> PushEntry<Key<T>, BufResult<usize, T>> {
    self.driver.borrow_mut().push(op)
}

/// Push an operation into the driver, and return the unique key, called
/// user-defined data, associated with it.
pub struct Proactor {
    driver: Driver,
}

impl Proactor {
    pub fn push<T: OpCode + 'static>(&mut self, op: T) -> PushEntry<Key<T>, BufResult<usize, T>> {
        let mut op = self.driver.create_op(op);
        match self
            .driver
            .push(&mut unsafe { Key::<dyn OpCode>::new_unchecked(op.user_data()) })
        {
            Poll::Pending => PushEntry::Pending(op),
            Poll::Ready(res) => {
                op.set_result(res);
                // SAFETY: just completed.
                PushEntry::Ready(unsafe { op.into_inner() })
            }
        }
    }
}
```

block_on 检查当前没有任务可以运行，则进入poll
```rust
    pub fn block_on<F: Future>(&self, future: F) -> F::Output {
        self.enter(|| {
            let opt_waker = self.opt_waker();
            let waker = Waker::from(opt_waker.clone());
            let mut context = Context::from_waker(&waker);
            let mut future = std::pin::pin!(future);
            loop {
                if let Poll::Ready(result) = future.as_mut().poll(&mut context) {
                    self.run();
                    return result;
                }
                // We always want to reset the waker here.
                let remaining_tasks = self.run() | opt_waker.reset();
                if remaining_tasks {
                    self.poll_with(Some(Duration::ZERO));
                } else {
                    self.poll();
                }
            }
        })
    }

```

最终进入iour Driver的poll
```rust
impl Driver {

    pub fn poll(&mut self, timeout: Option<Duration>) -> io::Result<()> {
        instrument!(compio_log::Level::TRACE, "poll", ?timeout);
        // Anyway we need to submit once, no matter there are entries in squeue.
        trace!("start polling");

        if self.need_push_notifier {
            #[allow(clippy::useless_conversion)]
            self.push_raw(
                // PollAdd = 用 io_uring 提交“监听某个 fd 是否可读/可写”的操作码，相当于把 poll/epoll 那种就绪监听，变成一个 io_uring 的 SQE
                // 轮询这个文件描述符，当它满足指定事件（可读 / 可写等）时，往完成队列里塞一个 CQE 通知
                // 不需要自己调用 poll 或 epoll_wait，而是像等待 read/write 完成一样等待 CQE
                PollAdd::new(Fd(self.notifier.as_raw_fd()), libc::POLLIN as _)
                    .multi(true)
                    .build()
                    .user_data(Self::NOTIFY)
                    .into(),
            )?;
            self.need_push_notifier = false;
        }

        if !self.poll_entries() {
            self.submit_auto(timeout)?;
            self.poll_entries();
        }

        Ok(())
    }
```

poll->submit_auto->submit_and_wait->submit_and_wait->enter->sys::io_uring_enter
```rust
// 对 Linux io_uring 系统调用的安全封装 向内核提交 IO 操作和/或等待操作完成。
// 会在此阻塞
pub unsafe fn enter<T: Sized>(
        &self,
        to_submit: u32,
        min_complete: u32,
        flag: u32,
        arg: Option<&T>,
    ) -> io::Result<usize> {
        let arg = arg
            .map(|arg| cast_ptr(arg).cast())
            .unwrap_or_else(ptr::null);
        let size = mem::size_of::<T>();
        sys::io_uring_enter(
            self.fd.as_raw_fd(),
            to_submit,
            min_complete,
            flag,
            arg,
            size,
        )
        .map(|res| res as _)
    }
```

Linux内核是如何唤醒io_uring_enter 阻塞等待的进程的

1. 设备驱动完成 IO 操作
    ↓
2. 调用 io_req_complete(req, result)
    ↓
3. io_put_cqe() 将完成项放入 CQ 环
    ↓
4. io_cqring_wake(ctx) 检查等待队列
    ↓
5. 如果等待队列非空，调用 wake_up_all()
    ↓
6. 内核调度器将进程标记为可运行
    ↓
7. 进程从 schedule() 返回
    ↓
8. 继续执行 io_uring_enter 系统调用

唤醒的进程返回后，拿到完成的entry，调用notify唤醒future
```rust
pub unsafe fn notify(self) {
        let user_data = self.user_data();
        // 利用 user_data 找回对应的底层操作对象（RawOp）
        let mut op = unsafe { Key::<()>::new_unchecked(user_data) };
        op.set_flags(self.flags());
        // set_result把完成结果 res 存入 RawOp 的 result 字段中； 如果之前挂着 waker，就唤醒对应的 future；
        if op.set_result(self.into_result()) {
            // SAFETY: completed and cancelled.
            let _ = unsafe { op.into_box() };
        }
    }
```

```rust
pub fn block_on<F: Future>(&self, future: F) -> F::Output {
    self.enter(|| {
        let opt_waker = self.opt_waker();
        let waker = Waker::from(opt_waker.clone());
        let mut context = Context::from_waker(&waker);
        let mut future = std::pin::pin!(future);
        loop {
            if let Poll::Ready(result) = future.as_mut().poll(&mut context) {
                self.run();
                return result;
            }
            // self.run()提交尚未提交的 io_uring 请求；从完成队列（CQE）里取出已经完成的 I/O；调用它们的 Waker::wake()，把这些 Future 标记为“就绪任务”。
            // opt_waker.reset()：检查在这段时间里有没有 wake() 发生；如果有，返回 true，说明“有 Future 需要重新 poll”。
            let remaining_tasks = self.run() | opt_waker.reset();
            if remaining_tasks {
                // poll_with(Duration::ZERO) 做零超时轮询，快速触发 io_uring 的处理，但不长时间阻塞；
                self.poll_with(Some(Duration::ZERO));
            } else {
                // 调用 self.poll()，在内核里真正阻塞等待新的 I/O 事件；一旦有新的 I/O 完成，唤醒对应 Future，再回到上面的流程
                self.poll();
            }
        }
    })
}
```