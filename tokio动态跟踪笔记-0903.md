# tokio的文件异步读写

大多数操作系统不提供异步文件系统 API。
因此，Tokio 会在底层使用普通的阻塞文件操作。通过 [spawn_blocking] 线程池在后台运行这些操作来实现的。
tokio::fs 模块只应用于普通文件。尝试将其用于例如 Linux 上的命名管道，可能会导致意外行为，例如在运行时关闭期间挂起。对于特殊文件，应该使用专门的类型，例如 [tokio::net::unix::pipe] 或 [AsyncFd]。
目前，Tokio 在所有平台上都会使用 [spawn_blocking]，但在未来可能会改为使用 io_uring 等异步文件系统 API。

为什么普通文件读写不能采用 epoll等待，而是使用 阻塞线程。普通文件在 select/poll/epoll 上"always ready"。对普通文件 open 时 O_NONBLOCK，Linux 会"收下这个标志但不理它"。磁盘文件的 read/write 永远表现得像阻塞式调用：它会阻塞到内核把数据从盘（或 page cache）搬完才返回。

如果想要"提交一个读请求，之后可以干别的，内核读好后再通知我，全程不占线程"，只能靠内核异步接口（io_uring），不是 POSIX 传统的非阻塞 I/O。Linux 上有三类方案：

接口	是否真异步	现状
Linux AIO（libaio / io_submit）	是，但前提是必须 O_DIRECT、对齐、特定文件系统…	限制多、生态差，很少直接用
io_uring（5.1+）	是，提交-完成队列模型，可以 buffer I/O，不强制 O_DIRECT	现在的主流真异步方案，tokio 有 feature 支持
posix_fadvise(WILLNEED) + 稍后再读	半套（内核预读后，RWF_NOWAIT 命中）	自己实现的折中式


IPC读写和普通文件读写的区别

tokio::fs 只用于普通文件（磁盘上的常规 inode）。
Linux 命名管道（FIFO）会出问题：读 FIFO 没有数据时 spawn_blocking 线程卡在内核 read，runtime 关闭时这些线程退不出来，导致 shutdown hang。
tokio::net::unix::pipe针对 FIFO/pipe 这种特殊文件的专用封装，真 epoll 异步
AsyncFd: 通用型，把任意 fd 注册进 reactor，自己调 syscall
tokio 官方认为 FIFO 不应再被"包装成普通 File 来异步化"，而是要 fd 语义匹配的类型。
tokio::fs 的实现是 spawn_blocking + 同步 std::fs::File（fs/mod.rs:19-21 也写了这点）。
对于普通文件：
read/write 基本都会很短时间内返回（哪怕是机械盘，内核也有 O(1) 时间语义——不会因为"没对端"而永远阻塞）；
shutdown 时即使有读没完成，等待一会儿也就结束了。
但 FIFO 属于带配对语义的流：没有对端写，read 永远不返回；对端永远不连，open 永远不返回。这种"无限期阻塞"一旦发生在 blocking 线程里：
blocking 池线程耗尽 → 其他 tokio::fs 操作排队挂起；
runtime 进入 shutdown，blocking 池会等所有正在执行的 blocking task 退出来再关 → 某个 task 卡在阻塞 read 上，runtime 关不掉（就是文档说的 "hangs during runtime shutdown"）。


在tokio中，文件的异步读写实现为struct File。

```rust
tokio-1.49.0/src/fs/file.rs

pub struct File {
    std: Arc<StdFile>,  // std::fs::File 对应 Linux的文件句柄fd的封装
    inner: Mutex<Inner>,
    max_buf_size: usize,
}

struct Inner {
    state: State,

    /// Errors from writes/flushes are returned in write/flush calls. If a write
    /// error is observed while performing a read, it is saved until the next
    /// write / flush call.
    last_write_err: Option<io::ErrorKind>,

    pos: u64,
}

enum State {
    Idle(Option<Buf>),
    Busy(JoinHandle<(Operation, Buf)>),
}

enum Operation {
    Read(io::Result<usize>),
    Write(io::Result<()>),
    Seek(io::Result<u64>),
}
```

tokio定义不同的异步操作trait，比如AsyncRead，AsyncWrite，AsyncSeek等。

File 必须通过实现AsyncRead，AsyncWrite，AsyncSeek以实现异步文件读写。


以AsyncRead为例，File 实现AsyncRead 只需要实现poll_read()


```rust
impl AsyncRead for File {
    fn poll_read(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
        dst: &mut ReadBuf<'_>,
    ) -> Poll<io::Result<()>> {

        loop {
            match inner.state {
                State::Idle(ref mut buf_cell) => {
                    let mut buf = buf_cell.take().unwrap();

                    // 若 Buf 里还有预读缓存 → copy_to(dst) 直接返回（省系统调用）
                    if !buf.is_empty() || dst.remaining() == 0 {
                        buf.copy_to(dst);
                        return Poll::Ready(Ok(()));
                    }

                    // Buf没有数据，spawn_blocking() 创建出阻塞线程，拿到阻塞线程的 JoinHandle
                    inner.state = State::Busy(spawn_blocking(move || {
                        let res = unsafe { buf.read_from(&mut &*std, max_buf_size) };
                        (Operation::Read(res), buf)
                    }));
                }
                State::Busy(ref mut rx) => {
                    // rx 此时就是JoinHandle，执行JoinHandle的poll，读取阻塞任务的执行状态
                    let (op, mut buf) = ready!(Pin::new(rx).poll(cx))?;

                    match op {
                        Operation::Read(Ok(_)) => {
                            buf.copy_to(dst);
                            inner.state = State::Idle(Some(buf));
                            return Poll::Ready(Ok(()));
                        }
                    }
                }
            }
        }
    }
```

阻塞线程的执行
```
阻塞线程: Task::run()                        pool.rs:160
   └─ UnownedTask::run() → task 系统
        └─ Harness::poll()                  harness.rs:153
             └─ poll_inner()
                  ├─ 1. poll future → BlockingTask::poll ，阻塞式执行 func()， 返回 Ready(func())，
                  ├─ 2. 把 func() 的返回值写入任务体
                  └─ 3. 状态机发现 future 已完成 → 调用 complete()
                          └─ Harness::complete()         harness.rs:331
                               └─ trailer().wake_join()  harness.rs:349
                                    └─ waker.wake_by_ref()  core.rs:560  // 唤醒 JoinHandle的poll
```