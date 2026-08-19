# tokio的异步pipe

tokio/src/net/unix/pipe.rs

```rust
pub fn pipe() -> io::Result<(Sender, Receiver)> {
    let (tx, rx) = mio_pipe::new()?;          // 底层就是 libc::pipe(2) + O_NONBLOCK
    Ok((Sender::from_mio(tx)?, Receiver::from_mio(rx)?))
}
```

调用pipe会通过mio库的mio_pipe 打开pipe，然后生成一个Sender和一个Receiver

```rust
pub struct Sender {
    io: PollEvented<mio_pipe::Sender>,
}

pub struct Receiver {
    io: PollEvented<mio_pipe::Receiver>,
}
```

Sender和Receiver的本质是一个PollEvented。

Sender会实现一个AsyncWrite trait，Receiver会实现一个AsyncRead trait。
```rust

impl AsyncWrite for Sender {
    fn poll_write(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
        buf: &[u8],
    ) -> Poll<io::Result<usize>> {
        self.io.poll_write(cx, buf)
    }
}

impl AsyncRead for Receiver {
    fn poll_read(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
        buf: &mut ReadBuf<'_>,
    ) -> Poll<io::Result<()>> {
        unsafe { self.io.poll_read(cx, buf) }
    }

```


AsyncWrite trait提供异步写接口和AsyncRead trait提供异步读接口

```rust
tx.write(&mut buf).await

rx.read(&mut buf).await
```

read 返回的是struct Read，Read 实现Future时候最终调用的是struct PollEvented poll_read()

```rust
Read {
        reader,
        buf,
        _pin: PhantomPinned,
    }

impl<R> Future for Read<'_, R>
{
    type Output = io::Result<usize>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<io::Result<usize>> {
        let me = self.project();
        let mut buf = ReadBuf::new(me.buf);
        ready!(Pin::new(me.reader).poll_read(cx, &mut buf))?;
        Poll::Ready(Ok(buf.filled().len()))
    }
}

```

struct PollEvented poll_read()调用的是struct Registration poll_read_ready() 确定就绪状态，如果就绪则调用mio::unix::pipe::Receiver 上的 impl io::Read read()方法读取数据。

```rust
tokio/src/io/poll_evented.rs

    impl<E: Source> PollEvented<E> {
        pub(crate) unsafe fn poll_read<'a>(
            &'a self,
            cx: &mut Context<'_>,
            buf: &mut ReadBuf<'_>,
        ) -> Poll<io::Result<()>>
        {

            loop {
                let evt = ready!(self.registration.poll_read_ready(cx))?;

                match self.io.as_ref().unwrap().read(b) { 
                    Ok(n) => {
                        if 0 < n && n < len {
                            self.registration.clear_readiness(evt);
                        }
                        unsafe { buf.assume_init(n) };
                        buf.advance(n);
                        return Poll::Ready(Ok(()));
                    },
                   
                }
            }
        }
```


poll_read_ready 最终调用的是 shared.poll_readiness()
```rust
    pub(crate) fn poll_read_ready(&self, cx: &mut Context<'_>) -> Poll<io::Result<ReadyEvent>> {
        self.poll_ready(cx, Direction::Read)
    }

    fn poll_ready(
        &self,
        cx: &mut Context<'_>,
        direction: Direction,
    ) -> Poll<io::Result<ReadyEvent>> {
        let ev = ready!(self.shared.poll_readiness(cx, direction));
        Poll::Ready(Ok(ev))
    }
```

shared.poll_readiness()的实现即 ScheduledIo poll_readiness()，本质上是读取readiness 的值用于判断是否就绪。
```rust
pub(super) fn poll_readiness(
        &self,
        cx: &mut Context<'_>,
        direction: Direction,
    ) -> Poll<ReadyEvent> {
    // 1) 乐观无锁读
    let curr = self.readiness.load(Acquire);
    let ready = direction.mask() & Ready::from_usize(READINESS.unpack(curr));
    if ready.is_empty() && !is_shutdown {
        // 2) 没就绪 上锁,把 cx.waker() 存到 ScheduledIo.waiters.reader 或 .writer
        let mut waiters = self.waiters.lock();
        let waker = match direction {
            Direction::Read  => &mut waiters.reader,     //  直接用 reader 槽
            Direction::Write => &mut waiters.writer,     //  直接用 writer 槽
        };
        match waker {
            Some(w) => w.clone_from(cx.waker()),         // 已有 waker,复用空间
            None    => *waker = Some(cx.waker().clone()),// 首次存
        }
        // 3) 锁内再读一次 readiness,避免错过 wake
        let curr = self.readiness.load(Acquire);
        if ready.is_empty() { Poll::Pending }
        else                  { Poll::Ready(...) }
    } else {
        Poll::Ready(ReadyEvent { ... })
    }
```

## ScheduledIo

由此可见，ScheduledIo是连接异步事件等待任务和Reactor的桥梁。一般而言，一个fd对应一个ScheduledIo。

readiness字段保存fd事件的就绪状态，waiters保存等待任务的waker。

```rust
tokio/src/runtime/io/scheduled_io.rs

pub(crate) struct ScheduledIo {
    readiness: AtomicUsize,
    waiters: Mutex<Waiters>,
}


```

### readiness字段

```rust
// scheduled_io.rs L164-L174
// | shutdown | driver tick | readiness |
// |----------+-------------+-----------|
// |   1 bit  |   15 bits   |  16 bits  |
// READINESS(16 bit):实际就绪位 READABLE | WRITABLE | READ_CLOSED | WRITE_CLOSED,这是 reactor 把 epoll 事件翻译过来的"任务可读/可写"语义。
const READINESS: bit::Pack = bit::Pack::least_significant(16);
const TICK: bit::Pack = READINESS.then(15);
const SHUTDOWN: bit::Pack = TICK.then(1);
```

### waiters

```rust
struct Waiters {
    /// List of all current waiters.
    list: WaitList,

    /// Waker used for `AsyncRead`.
    reader: Option<Waker>,

    /// Waker used for `AsyncWrite`.
    writer: Option<Waker>,
}
```

同一个 ScheduledIo 同时支持:

路径 A(AsyncRead::poll_read / AsyncWrite::poll_write 走的):waker 存 reader/writer 单槽,简单高效但同方向只能一个 task。
路径 B(readable().await / writable().await 走的):Waiter 节点挂 list 链表,支持多 task 并发等待 + cancel safe。

总之，阻塞等待的任务会景其waker保存到ScheduledIo之中。

## Reactor与ScheduledIo

Reactor接收到就绪事件，如何反查出ScheduledIo并唤醒任务。

ScheduledIo 的堆地址就是 epoll event 里的 data 字段。

创建ScheduledIo的时候，通过token()将ScheduledIo 地址转换为token，注册到 epoll 的时候携带该token作为标识。

```rust
// scheduled_io.rs L189-L191
pub(crate) fn token(&self) -> mio::Token {
    mio::Token(super::EXPOSE_IO.expose_provenance(self))
}

let scheduled_io = self.registrations.allocate(&mut self.synced.lock())?;
let token = scheduled_io.token();          // = Token(usize::from_raw(&ScheduledIo))
self.registry.register(source, token, interest.to_mio())?;   // epoll_ctl ADD
```

Reactor的主要功能在turn() 中实现，通过epoll_wait读取外部事件，返回一系列的event。event 携带返回的token数据反向拿到ScheduledIo，从而得到对应的等待任务的waker。

```rust
fn turn(&mut self, handle: &Handle, max_wait: Option<Duration>) {

    match self.poll.poll(events, max_wait) {        // 1. epoll_wait(阻塞)
        Ok(()) => {}
        Err(e) => panic!("unexpected error when polling the I/O driver: {e:?}"),
    }

    for event in events.iter() {                    // 2. 逐个事件分发，这里将epoll event 里的 data 字段转换为token
        let token = event.token();

        if token == TOKEN_WAKEUP {                  
        } else if token == TOKEN_SIGNAL {           
            self.signal_ready = true;
        } else {
            let ready = Ready::from_mio(event);
            let io: &ScheduledIo = unsafe { &*ptr };            // 将token 反向转换为ScheduledIo

            io.set_readiness(Tick::Set, |curr| curr | ready);   // 3. ScheduledIo readiness CAS 更新原子就绪位
            io.wake(ready);                                      // 4. 唤醒链表上的 waker

            ready_count += 1;
        }
    }
}
```


总而言之，压缩中间过程，可以理解为等待任务的waker 地址被当做标志注册到epoll，Reactor读取外部事件返回event， event的数据存储了waker的地址，从而能够唤醒对应的协程。

这个是基于epoll实现tokio 设计的Reactor比较巧妙的部分。

对于一般的基于epoll的异步运行时而言，Reactor还可以设计通过 fd → HashMap<waker> 间接查找waker。

以上是对epoll Reactor中事件与waker映射模式的分析。