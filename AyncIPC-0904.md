# IPC的统一接口格式

## 基本形式

需要实现三个API，open，poll_write()和poll_read()。

```rust
(rx, tx) = open();

tx.poll_write(&chunk).await;

rx.poll_read(&mut buf).await;
```

### open 语义

open作用是打开文件（如果有管道文件的话），创建发送端和接收端，注册事件至IO driver。

严格意义上说，Rust没有提供标准的open接口，每种IPC有自己的打开和创建方式。

tokio提供的异步IPC目前有匿名pipe，命名管道FIFO，UDS。

#### 匿名pipe

匿名pipe 通过同名函数创建pipe文件句柄、返回自定义的(Sender, Receiver)。

```rust
tokio\src\net\unix\pipe.rs

pub fn pipe() -> io::Result<(Sender, Receiver)> {
    let (tx, rx) = mio_pipe::new()?;
    Ok((Sender::from_mio(tx)?, Receiver::from_mio(rx)?))
}

```

因为匿名 pipe 的 Sender/Receiver 不能直接跨进程传递，一般是配合Command::new用于父子进程通信，此处不做过多讨论。


#### 命名管道FIFO

命名管道则通过OpenOptions封装的API打开读端或者写端，命名管道文件需要提前使用mkfifo创建;

```rust
tokio\src\net\unix\pipe.rs

const FIFO_NAME: &str = "path/to/a/fifo";
mkfifo(FIFO_NAME, Mode::S_IRWXU)?;
let rx = pipe::OpenOptions::new().open_receiver(FIFO_NAME)?;
let tx = pipe::OpenOptions::new().open_sender(FIFO_NAME)?;
```

open_receiver() 调用的pipe::OpenOptions::open() 本质上是对std::fs::OpenOptions()的封装。std::fs::OpenOptions()是Rust 标准库里对应 POSIX open(2) 的封装实现。

```rust
pub fn open_receiver<P: AsRef<Path>>(&self, path: P) -> io::Result<Receiver> {
    let file = self.open(path.as_ref(), PipeEnd::Receiver)?;
    // 构造Receiver 结构，和匿名管道共用同一类型的结构
    Receiver::from_file_unchecked(file)
}

fn open(&self, path: &Path, pipe_end: PipeEnd) -> io::Result<File> {
        let mut options = std::fs::OpenOptions::new();
        options
            .read(pipe_end == PipeEnd::Receiver) // 通过参数决定设置读写标志
            .write(pipe_end == PipeEnd::Sender)
            .custom_flags(libc::O_NONBLOCK);    // 构造读写，非阻塞打开文件标志

        let file = options.open(path)?; // std::fs::File

        Ok(file)
    }
```

open返回的file是std::fs::File类型，传给Receiver::from_file_unchecked(file)生成Receiver。 from_file_unchecked最终调用from_mio用于构造Receiver。期间调用PollEvented::new_with_interest 本质上是通过mio Source::register 将pipe 句柄注册到epoll，开始监听read事件。返回一个tokio PollEvented<T> 句柄用于管理事件。

```rust
pub fn from_owned_fd_unchecked(owned_fd: OwnedFd) -> io::Result<Receiver> {
    let mio_rx = unsafe { mio_pipe::Receiver::from_raw_fd(owned_fd.into_raw_fd()) };
    Receiver::from_mio(mio_rx)
}

impl Receiver {
    fn from_mio(mio_rx: mio_pipe::Receiver) -> io::Result<Receiver> {
        let io = PollEvented::new_with_interest(mio_rx, Interest::READABLE)?;
        Ok(Receiver { io })
    }
}
```

#### UDS

UDS创建相关的API是bind()，accept()，connect()。其实现本质是还是委托mio，并且通过PollEvented调用mio注册epoll事件。

```rust
tokio\src\net\unix\listener.rs

pub struct UnixListener {
    io: PollEvented<mio::net::UnixListener>,
}

pub fn bind<P>(path: P) -> io::Result<UnixListener>
{
    let listener = mio::net::UnixListener::bind_addr(&addr)?;
    let io = PollEvented::new(listener)?;
    Ok(UnixListener { io })
}

impl UnixListener {
pub(crate) fn new(listener: mio::net::UnixListener) -> io::Result<UnixListener> {
    let io = PollEvented::new(listener)?;
    Ok(UnixListener { io })
}
```

```rust
pub async fn accept(&self) -> io::Result<(UnixStream, SocketAddr)> {
    let (mio, addr) = self
        .io
        .registration()
        .async_io(Interest::READABLE, || self.io.accept())
        .await?;

    let addr = SocketAddr(addr);
    let stream = UnixStream::new(mio)?;
    Ok((stream, addr))
}

pub async fn connect(self, path: impl AsRef<Path>) -> io::Result<UnixStream> {
        let addr = socket2::SockAddr::unix(path)?;
        if let Err(err) = self.inner.connect(&addr) {
            if err.raw_os_error() != Some(libc::EINPROGRESS) {
                return Err(err);
            }
        }
        let mio = {
            use std::os::unix::io::{FromRawFd, IntoRawFd};

            let raw_fd = self.inner.into_raw_fd();
            unsafe { mio::net::UnixStream::from_raw_fd(raw_fd) }
        };

        UnixStream::connect_mio(mio).await
    }
```

### poll_write 语义

poll_write的语义是非阻塞地尝试写一次数据，返回实际写入字节数；如果 fd 不可写，返回 Pending 并注册 waker，等 fd 就绪后由由IO driver唤醒再尝试写数据。

tokio的异步pipe只需要实现trait AsyncWrite的poll_write 函数，就可以使用trait AsyncWriteExt 提供的write和write_all 方法。write本质上就是poll_write。

```rust
impl AsyncWrite for Sender {
    fn poll_write(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
        buf: &[u8],
    ) -> Poll<io::Result<usize>> {
        // 实际调用的是PollEvented的poll_write
        self.io.poll_write(cx, buf)
    }

}

tokio\src\io\poll_evented.rs
pub(crate) fn poll_write<'a>(&'a self, cx: &mut Context<'_>, buf: &[u8]) -> Poll<io::Result<usize>>
{
    use std::io::Write;

    loop {
        let evt = ready!(self.registration.poll_write_ready(cx))?;   // ① 等待可写事件

        match self.io.as_ref().unwrap().write(buf) {                 // ② 有就绪事件执行sys_write系统调用
            Ok(n) => {
                // 部分写入时清除就绪状态（unix 边沿语义下表示缓冲区满）
                if n > 0 && (!cfg!(windows) && !cfg!(mio_unsupported_force_poll_poll) && n < buf.len()) {
                    self.registration.clear_readiness(evt);
                }
                return Poll::Ready(Ok(n));
            },
            Err(e) if e.kind() == io::ErrorKind::WouldBlock => {
                self.registration.clear_readiness(evt);              // ③ 假唤醒：清就绪态后继续循环等待
            }
            Err(e) => return Poll::Ready(Err(e)),
        }
    }
}
```

AsyncWriteExt 提供的async write和write_all方法，底层都依赖 AsyncWrite::poll_write，

write只调用一次 poll_write，write_all 循环调用 poll_write，用 split_at(n) 推进 buf，直到写完。

```rust
tokio\src\io\util\write.rs
impl<W> Future for Write<'_, W>
where
    W: AsyncWrite + Unpin + ?Sized,
{
    type Output = io::Result<usize>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<io::Result<usize>> {
        let me = self.project();
        Pin::new(&mut *me.writer).poll_write(cx, me.buf)
    }
}


tokio\src\io\util\write_all.rs
impl<W> Future for WriteAll<'_, W>
where
    W: AsyncWrite + Unpin + ?Sized,
{
    type Output = io::Result<()>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<io::Result<()>> {
        let me = self.project();
        while !me.buf.is_empty() {
            let n = ready!(Pin::new(&mut *me.writer).poll_write(cx, me.buf))?;
            {
                let (_, rest) = mem::take(&mut *me.buf).split_at(n);
                *me.buf = rest;
            }
            if n == 0 {
                return Poll::Ready(Err(io::ErrorKind::WriteZero.into()));
            }
        }

        Poll::Ready(Ok(()))
    }
}
``` 

### try_write 语义

try_write尝试将缓冲区数据写入管道，并返回实际写入的字节数。
该函数会尝试写入 buf 的全部内容，但也可能只写入其中一部分。若 buf 的长度不超过 PIPE_BUF（Linux 下为 4096），则写入操作保证是原子的(要么完整写入 buf 的全部内容，要么返回 WouldBlock 错误)。若 buf 超过 PIPE_BUF，则不提供此保证。
因为try_write不会阻塞，如果不可写立即返回 WouldBlock，所以一般配合 writable().await 使用。


```rust
/// #[tokio::main]
/// async fn main() -> io::Result<()> {
///     let tx = pipe::OpenOptions::new().open_sender("path/to/a/fifo")?;
///
///     loop {
///         tx.writable().await?;
///
///         match tx.try_write(b"hello world") {
///             Ok(n) => {
///                 break;
///             }
///             Err(e) if e.kind() == io::ErrorKind::WouldBlock => {
///                 continue;
///             }
///             Err(e) => {
///                 return Err(e.into());
///             }
///         }
///     }
///
///     Ok(())
/// }

pub async fn ready(&self, interest: Interest) -> io::Result<Ready> {
    let event = self.io.registration().readiness(interest).await?;
    Ok(event.ready)
}
pub async fn writable(&self) -> io::Result<()> {
    self.ready(Interest::WRITABLE).await?;
    Ok(())
}

pub fn try_write(&self, buf: &[u8]) -> io::Result<usize> {
    self.io
        .registration()
        .try_io(Interest::WRITABLE, || (&*self.io).write(buf))
        // .try_io(Interest::WRITABLE, || ...) 如果当前对 WRITABLE 是就绪的，就执行这个闭包；否则直接返回 WouldBlock，不执行闭包。后面的闭包是调用mio_pipe::Sender write，本质上是调用 std::io::Write::write，底层是 libc::write(fd, buf, len)
}
```
```rust
tokio\src\runtime\io\registration.rs

    pub(crate) fn try_io<R>(
        &self,
        interest: Interest,
        f: impl FnOnce() -> io::Result<R>,
    ) -> io::Result<R> {
        let ev = self.shared.ready_event(interest);

        // Don't attempt the operation if the resource is not ready.
        if ev.ready.is_empty() {
            return Err(io::ErrorKind::WouldBlock.into());
        }

        match f() {
            Err(ref e) if e.kind() == io::ErrorKind::WouldBlock => {
                self.clear_readiness(ev);
                Err(io::ErrorKind::WouldBlock.into())
            }
            res => res,
        }
    }
```

### poll_read 语义

poll_read的语义是从fd非阻塞地读一次数据到buf里。如果fd当前不可读，返回 Poll::Pending，runtime 会在 fd 可读时重新调用。

```rust
impl AsyncRead for Receiver {
    fn poll_read(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
        buf: &mut ReadBuf<'_>,
    ) -> Poll<io::Result<()>> {
        unsafe { self.io.poll_read(cx, buf) }
    }
}

// poll_read的实现为PollEvented的poll_read
    impl<E: Source> PollEvented<E> {
        pub(crate) unsafe fn poll_read<'a>(
            &'a self,
            cx: &mut Context<'_>,
            buf: &mut ReadBuf<'_>,
        ) -> Poll<io::Result<()>>
        {
            use std::io::Read;

            loop {
                let evt = ready!(self.registration.poll_read_ready(cx))?;

                let b = unsafe { &mut *(buf.unfilled_mut() as *mut [std::mem::MaybeUninit<u8>] as *mut [u8]) };

                let len = b.len();

                match self.io.as_ref().unwrap().read(b) {
                    Ok(n) => {
                        // When mio is using the epoll or kqueue selector, reading a partially full
                        // buffer is sufficient to show that the socket buffer has been drained.
                        if 0 < n && n < len {
                            self.registration.clear_readiness(evt);
                        }

                        // Safety: We trust `TcpStream::read` to have filled up `n` bytes in the
                        // buffer.
                        unsafe { buf.assume_init(n) };
                        buf.advance(n);
                        return Poll::Ready(Ok(()));
                    },
                    Err(e) if e.kind() == io::ErrorKind::WouldBlock => {
                        self.registration.clear_readiness(evt);
                    }
                    Err(e) => return Poll::Ready(Err(e)),
                }
            }
        }
```

```rust
/// const FIFO_NAME: &str = "path/to/a/fifo";
///
/// # async fn dox() -> Result<(), Box<dyn std::error::Error>> {
/// let mut rx = pipe::OpenOptions::new().open_receiver(FIFO_NAME)?;
/// loop {
///     let mut msg = vec![0; 256];
///     match rx.read_exact(&mut msg).await {
///         Ok(_) => {
///             /* handle the message */
///         }
///         Err(e) if e.kind() == io::ErrorKind::UnexpectedEof => {
///             // Writing end has been closed, we should reopen the pipe.
///             rx = pipe::OpenOptions::new().open_receiver(FIFO_NAME)?;
///         }
///         Err(e) => return Err(e.into()),
///     }
/// }
```

同样tokio的异步pipe只需要实现trait AsyncRead的poll_rad 函数，就可以使用trait AsyncReadExt 提供的read和read_exact 方法。


read 读一次，有多少算多少；read_exact 读到buf满，不够就一直等。

```rust
impl<R> Future for Read<'_, R>
where
    R: AsyncRead + Unpin + ?Sized,
{
    type Output = io::Result<usize>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<io::Result<usize>> {
        let me = self.project();
        let mut buf = ReadBuf::new(me.buf);
        ready!(Pin::new(me.reader).poll_read(cx, &mut buf))?;
        Poll::Ready(Ok(buf.filled().len()))
    }
}


impl<A> Future for ReadExact<'_, A>
where
    A: AsyncRead + Unpin + ?Sized,
{
    type Output = io::Result<usize>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<io::Result<usize>> {
        let me = self.project();

        loop {
            // if our buffer is empty, then we need to read some data to continue.
            let rem = me.buf.remaining();
            if rem != 0 {
                ready!(Pin::new(&mut *me.reader).poll_read(cx, me.buf))?;
                if me.buf.remaining() == rem {
                    return Err(eof()).into();
                }
            } else {
                return Poll::Ready(Ok(me.buf.capacity()));
            }
        }
    }
}
```
### try_read 语义

非阻塞地尝试读一次。能读就返回读了多少字节，不能读就立即返回 WouldBlock，不挂起、不等待。

```rust
    /// #[tokio::main]
    /// async fn main() -> io::Result<()> {
    ///     // Open a reading end of a fifo
    ///     let rx = pipe::OpenOptions::new().open_receiver("path/to/a/fifo")?;
    ///
    ///     let mut msg = vec![0; 1024];
    ///
    ///     loop {
    ///         // Wait for the pipe to be readable
    ///         rx.readable().await?;
    ///
    ///         // Try to read data, this may still fail with `WouldBlock`
    ///         // if the readiness event is a false positive.
    ///         match rx.try_read(&mut msg) {
    ///             Ok(n) => {
    ///                 msg.truncate(n);
    ///                 break;
    ///             }
    ///             Err(e) if e.kind() == io::ErrorKind::WouldBlock => {
    ///                 continue;
    ///             }
    ///             Err(e) => {
    ///                 return Err(e.into());
    ///             }
    ///         }
    ///     }
    ///
    ///     println!("GOT = {:?}", msg);
    ///     Ok(())
    /// }

    pub fn try_read(&self, buf: &mut [u8]) -> io::Result<usize> {
        self.io
            .registration()
            .try_io(Interest::READABLE, || (&*self.io).read(buf))
    }
```

## 吞出量测试

假设共享缓冲区为64KB，发送方每次发送64KB，接收方每次读取64KB。测试发送256MB数据量所需要的时间。

所有测试都运行在单核上，通过taskset将发送方和接收方各自绑定在一个固定的核上。

```rust
const CHUNK: usize = 64 * 1024; // 单次写入 64 KB，因为内核pipe的默认环形缓冲区大小为 64KB
const TOTAL: usize = 256 * 1024 * 1024; // 总传输量 256 MB

fn sender()
{
    let (rx, tx) = open();

    let chunk = vec![0x5Au8; CHUNK];
    let mut sent = 0usize;

    let start = Instant::now();
    while sent < TOTAL {
        tx.write_all(&chunk).await.expect("write 失败");
        sent += CHUNK;
    }

    // to do: 等待receiver 退出
    let elapsed = start.elapsed();

    let secs = elapsed.as_secs_f64();
    let mib = TOTAL as f64 / (1024.0 * 1024.0);
    println!(
        "{name:<20} {mib:7.1} MiB / {secs:6.3} s = {:8.1} MiB/s",
        mib / secs
    );
}

fn receiver()
{
    let (rx, tx) = open();

    let mut buf = vec![0u8; CHUNK];
    let mut got = 0usize;
    while got < TOTAL {
        match rx.read(&mut buf).await {
            Ok(0) => break,
            Ok(n) => got += n,
            Err(e) => return Err(e),
        }
    }
}
```

## 延迟测试

发送方每次发送8 Bytes，接收方每次读取8 Bytes，然后将数据重新发送给发送方。测试来回发送的时间。

```rust
const WARMUP: usize = 2_000; // 延迟预热轮数（不计入统计）
const ROUNDS: usize = 100_000; // 延迟统计轮数
const MSG: [u8; 8] = [0xA5; 8];


/// 延迟测试：两条 pipe 全双工，服务端原样回显，统计 RTT。

/// 单方向 n 轮 ping-pong：写 8 字节 -> 读 8 字节回包。
fn ping_pong()
{
    let mut pong = [0u8; 8];
    for _ in 0..n {
        tx.write_all(&MSG).await?;
        rx.read_exact(&mut pong).await?;
    }
    Ok(())
}

fn client()
{
    let (server_rx, client_tx) = open();
    let (client_rx, server_tx) = open();

    let total = WARMUP + ROUNDS;

    // 预热（摊平首次注册/缺页等一次性开销）
    ping_pong(&mut client_tx, &mut client_rx, WARMUP).await?;

    let start = Instant::now();
    ping_pong(&mut client_tx, &mut client_rx, ROUNDS).await?;
    // to do: 等待receiver 退出
    let elapsed = start.elapsed();


    let rtt_us = elapsed.as_micros() as f64 / ROUNDS as f64;
    let msgs_per_s = ROUNDS as f64 / elapsed.as_secs_f64();
    println!(
        "{name:<20} RTT = {rtt_us:7.2} µs/轮  ({:9.0} msg/s)",
        msgs_per_s
    );
    Ok(())
}



fn server()
{
    let (server_rx, client_tx) = open();
    let (client_rx, server_tx) = open();

    for _ in 0..total {
        server_rx.read_exact(&mut buf).await?;
        server_tx.write_all(&buf).await?;
    }
}
```
