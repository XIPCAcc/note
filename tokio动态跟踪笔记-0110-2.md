compio如何监听Ctrl+C信号并在收到信号时退出程序

等待 signal转变为 等待对 signalfd 的一次 异步 read 操作完成。
signal 唤醒过程 signal 到达 → signalfd 变为可读 → 异步 read 完成 → io_uring 完成事件触发 → runtime 调度器唤醒 Future。

```rust
use compio::runtime;
use compio::signal::ctrl_c;

fn main() {
    // 初始化一个 compio 运行时（thread-per-core、基于完成的 IO 驱动）
    let rt = compio::runtime::Runtime::new().unwrap();
    // runtime 开始一个“事件循环”：不断从 OS / 驱动那里取完成事件，唤醒任务。
    rt.block_on(async {
        println!("等待 Ctrl+C 信号...");
        
        match compio::signal::ctrl_c().await {
            Ok(()) => println!("收到 Ctrl+C 信号，程序退出"),
            Err(e) => eprintln!("错误: {}", e),
        }
    });
}


// [dependencies]
// compio = { version = "0.17", features = ["signal", "runtime"] }

// kill -INT 线程PID （pa aux | grep 程序名 查看 PID）发送信号，直接按ctrl-c会终止整个进程
```

```rust
pub async fn ctrl_c() -> std::io::Result<()> {
    #[cfg(windows)]
    {
        windows::ctrl_c().await
    }
    #[cfg(unix)]
    {
        unix::signal(libc::SIGINT).await
    }
}

pub async fn signal(sig: i32) -> io::Result<()> {
    let fd = SignalFd::new(sig)?;
    fd.wait().await?;
    Ok(())
}

impl SignalFd {
    fn new(sig: i32) -> io::Result<Self> {
        // 1. 把这个 signal 加入当前线程的屏蔽集
        let set = register_signal(sig)?;  // 内部用 pthread_sigmask(SIG_BLOCK, ...)

        // 2. 创建 signalfd 信号不再以传统“异步打断”的方式送到线程，而是通过 signalfd 这个 fd 送达
        let mut flags = libc::SFD_CLOEXEC;
        if Runtime::with_current(|rt| rt.driver_type().is_polling()) {
            flags |= libc::SFD_NONBLOCK;
        }

        let raw_fd = libc::signalfd(-1, &set, flags);
        // 3. 把 raw_fd 封成 OwnedFd -> SharedFd，交给 compio 的 driver 管理
        ...
        Ok(SignalFd { fd, sig })
    }

    async fn wait(self) -> io::Result<()> {

        let info = SignalInfo(MaybeUninit::<libc::signalfd_siginfo>::uninit());
        // 构造一个读操作
        let op = Recv::new(self.fd.clone(), info);
        // 提交给 runtime 等待结果
        let BufResult(res, op) = compio_runtime::submit(op).await;
       
        // 取出结果
        let info = op.into_inner();
        let info = unsafe { info.0.assume_init() };
        Ok(())
    }
}
```