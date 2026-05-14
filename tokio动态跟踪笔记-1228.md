https://github.com/Hexilee/async-io-demo

对mio poll的理解
```rust
fn main() -> Result<(), Error> {
   
// Create a poll instance
    let poll = Poll::new()?;

// Start listening for incoming connections
    poll.register(&server, SERVER_ACCEPT, Ready::readable(),
                  PollOpt::edge())?;
// Register the client
    poll.register(&client, CLIENT, Ready::readable() | Ready::writable(),
                  PollOpt::edge())?;

// Create storage for events
    let mut events = Events::with_capacity(1024);

    'top: loop {
        poll.poll(&mut events, None)?;
        for event in events.iter() {
            if start.elapsed() >= timeout {
                break 'top
            }
            match event.token() {
                SERVER_ACCEPT => {
                    let (handler, addr) = server.accept()?;
                    println!("accept from addr: {}", &addr);
                    poll.register(&handler, SERVER, Ready::readable() | Ready::writable(), PollOpt::edge())?;
                    server_handler = Some(handler);
                }

            }
        }
    }
}

pub fn fs_async() -> (Fs, FsHandler) {
    let poll = Poll::new().unwrap();
    let (registration, set_readiness) = Registration::new2();

    poll.register(
        &registration,
        FS_TOKEN,
        Ready::readable(),
        PollOpt::oneshot(),
    ).unwrap();
    
    let io_worker = std::thread::spawn(move || {
        while let Ok(task) = task_receiver.recv() {
            match task {
                Task::Open(path, callback, fs) => {

                    set_readiness.set_readiness(Ready::readable())?;
                }
```

```rust
    // 方法1：使用 poll.registry() + register
    let registry = poll.registry();
    registry.register(&listener, Token(0), Interest::READABLE)?;
    
    // 方法2：使用 poll.register()（语法糖）
    poll.register(&listener, Token(1), Interest::READABLE)?;

    // 类型：
    let (registration, set_readiness): (Registration, SetReadiness) = Registration::new2();
    // 返回元组：
    // 1. Registration - 用于注册到事件循环
    // 2. SetReadiness - 用于设置就绪状态

```

对async的理解

async 修饰的函数会被编译生成一个GenFuture。

```rust
struct GenFuture<T: Generator<Yield = ()>>(T);
```

GenFuture 中包含一个类型为Generator的字段（因为不同的async参数和返回值都不一样，所以要向上再封装一层）

Generator 是 Rust 中的协程（coroutine）特性，它允许函数在执行过程中暂停（yield）和恢复（resume）。这是 async/await 的底层机制。

```rust
trait Generator<R = ()> {
    type Yield;     // yield 产生的类型
    type Return;    // 最终返回的类型
    
    fn resume(
        self: Pin<&mut Self>, 
        arg: R
    ) -> GeneratorState<Self::Yield, Self::Return>;
}
```

同时这个GenFuture需要实现!Unpin和Future两个trait
```rust

impl<T: Generator<Yield = ()>> !Unpin for GenFuture<T> {}

impl<T: Generator<Yield = ()>> Future for GenFuture<T> {
    type Output = T::Return;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let gen = unsafe { Pin::map_unchecked_mut(self, |s| &mut s.0) };
        set_task_context(cx, || match gen.resume() {
            GeneratorState::Yielded(()) => Poll::Pending,
            GeneratorState::Complete(x) => Poll::Ready(x),
        })
    }
}
```

协程并不意味着"非阻塞"，你可以在异步块或函数中调用阻塞API。
非阻塞IO的关键在于，当GenFuture即将阻塞时（例如，一个API返回io::ErrorKind::WouldBlock），通过本地唤醒器注册一个源任务并休眠（yield），底层的非阻塞调度器会在任务完成后唤醒这个GenFuture。

eventfd 是 Linux 提供的一种轻量级进程间通信机制，专门用于事件通知。它创建一个文件描述符，用于在进程或线程之间传递事件计数。eventfd 提供了一种比管道更高效、比信号更可控的事件通知机制。

```rust
int efd = eventfd(0, 0);

pid_t pid = fork();
if (pid == 0) {
    // 子进程
    write(efd, &val, sizeof(uint64_t));
} else {
    // 父进程
    read(efd, &u, sizeof(uint64_t));
}

// 创建 epoll 实例
int epoll_fd = epoll_create1(0);
struct epoll_event event;
event.events = EPOLLIN; // 监听可读事件
event.data.fd = efd;
epoll_ctl(epoll_fd, EPOLL_CTL_ADD, efd, &event);

// 等待事件
struct epoll_event events[10];
int n = epoll_wait(epoll_fd, events, 10, -1);
for (int i = 0; i < n; i++) {
    if (events[i].data.fd == efd) {
        uint64_t u;
        read(efd, &u, sizeof(uint64_t));
        // 处理事件
    }
}
```

mio中的waker本质就是创建eventfd

```rust
mio-1.1.1/src/sys/unix/waker/eventfd.rs

    pub(crate) fn new(selector: &Selector, token: Token) -> io::Result<Waker> {
        let waker = Waker::new_unregistered()?;
        selector.register(waker.fd.as_raw_fd(), token, Interest::READABLE)?;
        Ok(waker)
    }

    pub(crate) fn new_unregistered() -> io::Result<Waker> {
        #[cfg(not(target_os = "espidf"))]
        let flags = libc::EFD_CLOEXEC | libc::EFD_NONBLOCK;
        // ESP-IDF is EFD_NONBLOCK by default and errors if you try to pass this flag.
        #[cfg(target_os = "espidf")]
        let flags = 0;
        let fd = syscall!(eventfd(0, flags))?;
        let file = unsafe { File::from_raw_fd(fd) };
        Ok(Waker { fd: file })
    }

创建 eventfd → 创建 epoll → 配置事件 → 注册
    ↓
write(eventfd) → 计数器增加 → 变为可读
    ↓
epoll 检测到 → epoll_wait 返回 → 处理事件
    ↓
read(eventfd) → 计数器清零

epollfd = 监听器​ （多路复用器）
eventfd = 信号源​ （事件发生器）
```

selector.register则是创建epoll_event并注册eventfd到epollfd
```rust
// libc crate 中的定义
pub struct epoll_event {
    pub events: u32,        // 事件类型
    pub u64: u64,           // 用户数据（通常）
    #[cfg(target_os = "redox")]
    pub _pad: u32,          // Redox OS 的特殊填充
}

syscall!(epoll_ctl(ep, libc::EPOLL_CTL_ADD, fd, &mut event)).map(|_| ())
```

poll()调用了select()，select()其实就是调用epoll_wait在等待
```rust
    pub fn select(&self, events: &mut Events, timeout: Option<Duration>) -> io::Result<()> {
        let timeout = timeout
            .map(|to| {
                // `Duration::as_millis` truncates, so round up. This avoids
                // turning sub-millisecond timeouts into a zero timeout, unless
                // the caller explicitly requests that by specifying a zero
                // timeout.
                to.checked_add(Duration::from_nanos(999_999))
                    .unwrap_or(to)
                    .as_millis() as libc::c_int
            })
            .unwrap_or(-1);

        events.clear();
        syscall!(epoll_wait(
            self.ep.as_raw_fd(),
            events.as_mut_ptr(),
            events.capacity() as i32,
            timeout,
        ))
        .map(|n_events| {
            // This is safe because `epoll_wait` ensures that `n_events` are
            // assigned.
            unsafe { events.set_len(n_events as usize) };
        })
    }
```