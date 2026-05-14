一个简单的rust示例程序
```rust
use tokio;
use std::time::Duration;

// worker_threads = 2，除了主线程，还会创建两个工作线程，一个三个线程
#[tokio::main(flavor = "multi_thread", worker_threads = 2)]
async fn main() {
    println!("Hello World!");
    
    // 任务1 - 在可能的 worker 线程1 上执行
    let task1 = tokio::spawn(async {
        println!("[任务1] 在 {:?} 上启动", std::thread::current().id());
        
        for i in 1..=2 {
            println!("[任务1] 步骤 {}", i);
            tokio::time::sleep(Duration::from_millis(100)).await;
        }
    });
    
    // 任务2 - 在可能的 worker 线程2 上执行
    let task2 = tokio::spawn(async {
        println!("[任务2] 在 {:?} 上启动", std::thread::current().id());
        tokio::time::sleep(Duration::from_millis(200)).await;
    });
    
    // 同时等待两个任务
    let (result1, result2) = tokio::join!(task1, task2);

}
```
代码展开后为
```rust
// 1. main 函数被重命名为 __tokio_main
async fn __tokio_main() {
    println!("Hello World!");
    
    let task1 = tokio::spawn(async {
        println!("[任务1] 在 {:?} 上启动", std::thread::current().id());
        
        for i in 1..=2 {
            println!("[任务1] 步骤 {}", i);
            tokio::time::sleep(Duration::from_millis(100)).await;
        }
    });
    
    let task2 = tokio::spawn(async {
        println!("[任务2] 在 {:?} 上启动", std::thread::current().id());
        tokio::time::sleep(Duration::from_millis(200)).await;
    });
    
    let (result1, result2) = tokio::join!(task1, task2);
}

// 2. 实际的 main 函数
fn main() -> Result<(), Box<dyn std::error::Error>> {
    use tokio::runtime::{Builder, Runtime};
    use std::io;
    use std::error::Error;
    
    // 创建运行时
    let rt = Builder::new_multi_thread()
        .worker_threads(2)  // ← 来自 flavor = "multi_thread", worker_threads = 2
        .enable_all()       // 启用所有特性（IO、时间等）
        .build()?;          // 构建运行时
    
    // 在运行时中执行异步主函数
    rt.block_on(__tokio_main());
    
    Ok(())
}
```

更进一步
```rust
// 最终的展开代码
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::Duration;
use std::io;
use std::error::Error;

// 主函数被重命名
async fn __tokio_main_713a2b9c() {
    // 你的代码开始
    {
        std::io::_print(std::fmt::Arguments::new_v1(
            &["Hello World!\n"],
            &[],
        ));
    };
    
    // task1 的完整展开
    let task1 = {
        // 1. 创建 async 块对应的 Future
        let future = {
            // async 块被转换为一个结构体
            struct __async_block_1 {
                __state: u8,
                __i: i32,
                __sleep_future_1: Option<tokio::time::Sleep>,
                __sleep_future_2: Option<tokio::time::Sleep>,
            }
            
            impl Future for __async_block_1 {
                type Output = ();
                
                fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
                    loop {
                        match self.__state {
                            0 => {
                                // println!("[任务1] 在 {:?} 上启动", ...)
                                {
                                    let current_thread = std::thread::current().id();
                                    std::io::_print(std::fmt::Arguments::new_v1_formatted(
                                        &["[任务1] 在 ", " 上启动\n"],
                                        &[
                                            std::fmt::ArgumentV1::new_display(&current_thread),
                                        ],
                                        &[
                                            std::fmt::rt::v1::Argument { position: 0, format: std::fmt::rt::v1::FormatSpec { fill: ' ', align: std::fmt::rt::v1::Alignment::Unknown, flags: 0, precision: std::fmt::rt::v1::Count::Implied, width: std::fmt::rt::v1::Count::Implied } },
                                        ],
                                    ));
                                }
                                self.__state = 1;
                                self.__i = 1;
                                continue;
                            }
                            1 => {
                                if self.__i > 2 {
                                    self.__state = 255; // Done
                                    continue;
                                }
                                
                                // println!("[任务1] 步骤 {}", i)
                                {
                                    std::io::_print(std::fmt::Arguments::new_v1_formatted(
                                        &["[任务1] 步骤 ", "\n"],
                                        &[
                                            std::fmt::ArgumentV1::new_display(&self.__i),
                                        ],
                                        &[
                                            std::fmt::rt::v1::Argument { position: 0, format: std::fmt::rt::v1::FormatSpec { fill: ' ', align: std::fmt::rt::v1::Alignment::Unknown, flags: 0, precision: std::fmt::rt::v1::Count::Implied, width: std::fmt::rt::v1::Count::Implied } },
                                        ],
                                    ));
                                }
                                
                                // tokio::time::sleep(Duration::from_millis(100)).await
                                if self.__sleep_future_1.is_none() {
                                    self.__sleep_future_1 = Some(tokio::time::sleep(
                                        Duration::from_millis(100)
                                    ));
                                }
                                
                                match Pin::new(self.__sleep_future_1.as_mut().unwrap()).poll(cx) {
                                    Poll::Ready(()) => {
                                        self.__sleep_future_1 = None;
                                        self.__i += 1;
                                        self.__state = 1; // 回到循环检查
                                        continue;
                                    }
                                    Poll::Pending => return Poll::Pending,
                                }
                            }
                            255 => return Poll::Ready(()),
                            _ => unreachable!(),
                        }
                    }
                }
            }
            
            __async_block_1 {
                __state: 0,
                __i: 0,
                __sleep_future_1: None,
                __sleep_future_2: None,
            }
        };
        
        // 2. 获取当前运行时句柄
        let handle = tokio::runtime::Handle::current();
        
        // 3. 将 future 提交到运行时
        handle.spawn(future)
    };
    
    // task2 类似展开（简化）
    let task2 = {
        // 类似的 Future 结构体
        struct __async_block_2 { /* ... */ }
        
        let future = __async_block_2 { /* ... */ };
        let handle = tokio::runtime::Handle::current();
        handle.spawn(future)
    };
    
    // tokio::join! 宏展开
    let (result1, result2) = {
        // join! 宏会创建一个组合的 Future
        struct JoinFuture<F1, F2> {
            f1: tokio::macros::support::MaybeDone<F1>,
            f2: tokio::macros::support::MaybeDone<F2>,
        }
        
        impl<F1: Future, F2: Future> Future for JoinFuture<F1, F2> {
            type Output = (F1::Output, F2::Output);
            
            fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
                let this = unsafe { self.get_unchecked_mut() };
                
                let mut all_ready = true;
                
                // poll task1
                if !this.f1.is_done() {
                    if let Poll::Ready(()) = unsafe { Pin::new_unchecked(&mut this.f1) }.poll(cx) {
                        // f1 完成了
                    }
                    all_ready = false;
                }
                
                // poll task2
                if !this.f2.is_done() {
                    if let Poll::Ready(()) = unsafe { Pin::new_unchecked(&mut this.f2) }.poll(cx) {
                        // f2 完成了
                    }
                    all_ready = false;
                }
                
                if all_ready {
                    Poll::Ready((
                        this.f1.take_output().unwrap(),
                        this.f2.take_output().unwrap(),
                    ))
                } else {
                    Poll::Pending
                }
            }
        }
        
        // 等待这个组合 Future
        JoinFuture {
            f1: tokio::macros::support::MaybeDone::Future(task1),
            f2: tokio::macros::support::MaybeDone::Future(task2),
        }.await
    };
}

// 实际的 main 函数
fn main() -> std::result::Result<(), Box<dyn std::error::Error>> {
    // 创建多线程运行时
    let rt = tokio::runtime::Builder::new_multi_thread()
        .worker_threads(2)
        .enable_all()
        .build()?;
    
    // 在运行时中执行异步主函数
    rt.block_on(__tokio_main_713a2b9c());
    
    Ok(())
}
```


```rust
use tokio::task;

#[tokio::main]
async fn main() {
    task::spawn(async {
        println!("Hello, Tokio!");
    }).await.unwrap();
}
```

代码会被编译成为下面的main函数
    /// let rt = runtime::Builder::new_multi_thread()
    ///     .enable_all()
    ///     .build()
    ///     .unwrap();


因此首先会进入到new_multi_thread() 创建一个新的Builder
```rust
tokio-1.48.0/src/runtime/builder.rs

    #[cfg(feature = "rt-multi-thread")]
    #[cfg_attr(docsrs, doc(cfg(feature = "rt-multi-thread")))]
    pub fn new_multi_thread() -> Builder {
        // The number `61` is fairly arbitrary. I believe this value was copied from golang.
        Builder::new(Kind::MultiThread, 61)
    }


    pub(crate) fn new(kind: Kind, event_interval: u32) -> Builder {
            Builder {
                kind,

                // I/O defaults to "off"
                enable_io: false,
                nevents: 1024,
```

然后进入enable_all() Enables both I/O and time drivers. (Doing this is a shorthand for calling `enable_io` and `enable_time`)

```rust
pub fn enable_all(&mut self) -> &mut Self {
        #[cfg(any(
            feature = "net",
            all(unix, feature = "process"),
            all(unix, feature = "signal")
        ))]
        self.enable_io();

        #[cfg(all(
            tokio_unstable,
            feature = "io-uring",
            feature = "rt",
            feature = "fs",
            target_os = "linux",
        ))]
        self.enable_io_uring();

        #[cfg(feature = "time")]
        self.enable_time();

        self
    }
```

调用build() 随后进入build_threaded_runtime() 创建多线程运行时
```rust
    pub fn build(&mut self) -> io::Result<Runtime> {
        match &self.kind {
            Kind::CurrentThread => self.build_current_thread_runtime(),  //所有任务都在当前线程上执行
            #[cfg(feature = "rt-multi-thread")]
            Kind::MultiThread => self.build_threaded_runtime(), // 多线程运行时 默认创建与 CPU 核心数相等的线程 工作窃取（work-stealing）调度器
        }
    }
```

build_threaded_runtime() 首先会创建Driver
```rust
 impl Builder {
        fn build_threaded_runtime(&mut self) -> io::Result<Runtime> {
          
            let (driver, driver_handle) = driver::Driver::new(self.get_cfg())?;
```


Driver是 Tokio 运行时的事件循环引擎，主要负责：
•管理 I/O 事件（epoll/kqueue/IOCP）
•调度定时器
•协调任务唤醒
•处理内部信号

包含很多种Driver
I/O Driver
时间 Driver
信号 Driver​
进程 Driver​

```
运行时 Driver (组合驱动)
    ↑
时间驱动栈 (TimeStack)
    ├─ CurrentThread(TimeDriver)  # 当前线程时间驱动
    ├─ Driver(TimeDriver)         # 基于驱动的定时器
    └─ Noop                       # 空实现
        ↑
    I/O 驱动栈 (IoStack)
        ├─ Enabled(ProcessDriver)  # 启用的驱动栈
        │       ↑
        │   ProcessDriver          # 进程驱动
        │       ↑
        │   SignalDriver          # 信号驱动
        │       ↑
        │   IoDriver              # I/O 驱动
        │       ├─ Linux: epoll
        │       ├─ macOS: kqueue
        │       └─ Windows: IOCP
        │
        └─ Disabled(ParkThread)   # 禁用的驱动栈
                ↑
            ParkThread           # 线程挂起实现
```

```rust
// Driver的New
pub(crate) fn new(cfg: Cfg) -> io::Result<(Self, Handle)> {
    // 1. 创建 I/O 栈
    let (io_stack, io_handle, signal_handle) = create_io_stack(cfg.enable_io, cfg.nevents)?;
    
    // 2. 创建时钟
    let clock = create_clock(cfg.enable_pause_time, cfg.start_paused);
    
    // 3. 创建时间驱动
    let (time_driver, time_handle) = 
        create_time_driver(cfg.enable_time, cfg.timer_flavor, io_stack, &clock);
    
    // 4. 返回 Driver 和 Handle
    Ok((
        Self { inner: time_driver },
        Handle {
            io: io_handle,
            signal: signal_handle,
            time: time_handle,
            clock,
        },
    ))
```

Driver调用create_io_stack()创建各种类型的Driver（create_io_stack() 并不是为IO设备创建内存中的stack结构）
```rust
fn create_io_stack(enabled: bool, nevents: usize) -> io::Result<(IoStack, IoHandle, SignalHandle)> {
        #[cfg(loom)]
        assert!(!enabled);

        let ret = if enabled {
            // 1. 创建 I/O Driver
            let (io_driver, io_handle) = crate::runtime::io::Driver::new(nevents)?;
            // 2. 创建信号 Driver（基于 I/O Driver）
            let (signal_driver, signal_handle) = create_signal_driver(io_driver, &io_handle)?;
            // 3. 创建进程 Driver（基于信号 Driver）
            let process_driver = create_process_driver(signal_driver);

            (IoStack::Enabled(process_driver), IoHandle::Enabled(io_handle), signal_handle)
```

IO driver创建的是位于 tokio\src\runtime\driver.rs
```rust
pub(crate) struct Driver {
    inner: TimeDriver,
}


```rust
/// I/O driver, backed by Mio.
pub(crate) struct Driver {
    /// True when an event with the signal token is received
    signal_ready: bool,

    /// Reuse the `mio::Events` value across calls to poll.
    events: mio::Events,

    /// The system event queue.
    poll: mio::Poll,
}

IO driver的new函数
```rust
impl Driver {
    /// Creates a new event loop, returning any error that happened during the
    /// creation.
    pub(crate) fn new(nevents: usize) -> io::Result<(Driver, Handle)> {
        // 创建系统级的 I/O 多路复用器 Linux: epoll, macOS: kqueue, Windows: IOCP
        let poll = mio::Poll::new()?;
        #[cfg(not(target_os = "wasi"))]
        let waker = mio::Waker::new(poll.registry(), TOKEN_WAKEUP)?;
        let registry = poll.registry().try_clone()?;

        let driver = Driver {
            signal_ready: false,
            events: mio::Events::with_capacity(nevents),
            poll,
        };

        let (registrations, synced) = RegistrationSet::new();

        let handle = Handle {
            registry,
            registrations,
            synced: Mutex::new(synced),
            #[cfg(not(target_os = "wasi"))]
            waker,
            metrics: IoDriverMetrics::default(),
            #[cfg(all(
                tokio_unstable,
                feature = "io-uring",
                feature = "rt",
                feature = "fs",
                target_os = "linux",
            ))]
            uring_context: Mutex::new(UringContext::new()),
            #[cfg(all(
                tokio_unstable,
                feature = "io-uring",
                feature = "rt",
                feature = "fs",
                target_os = "linux",
            ))]
            uring_state: AtomicUsize::new(0),
        };

        Ok((driver, handle))
    }
```

mio Poll位于 mio-1.1.1/src/poll.rs
```rust
pub fn new() -> io::Result<Poll> {
            sys::Selector::new().map(|selector| Poll {
                registry: Registry {
                    selector,
                    #[cfg(all(debug_assertions, not(target_os = "wasi")))]
                    has_waker: Arc::new(AtomicBool::new(false)),
                },
            })
        }
```

Selector位于 mio-1.1.1/src/sys/unix/selector/epoll.rs
```rust
impl Selector {
    pub fn new() -> io::Result<Selector> {
        // 1. 创建 epoll 文件描述符
        let ep = unsafe { OwnedFd::from_raw_fd(syscall!(epoll_create1(libc::EPOLL_CLOEXEC))?) };
        
        // 2. 构造 Selector
        Ok(Selector {
            #[cfg(debug_assertions)] // 调试时给每个 Selector 唯一 ID便于跟踪和调试
            id: NEXT_ID.fetch_add(1, Ordering::Relaxed),
            ep,
        })
    }
}
```

epoll create的系统调用实现方式
```asm
; Symbol: epoll_create1
; Source: sysdeps/unix/syscall-template.S:120
7FFFF7D2ACF0: F3 0F 1E FA                   endbr64 
7FFFF7D2ACF4: B8 23 01 00 00                movl   $0x123, %eax  ; imm = 0x123 
7FFFF7D2ACF9: 0F 05                         syscall 
7FFFF7D2ACFB: 48 3D 01 F0 FF FF             cmpq   $-0xfff, %rax  ; imm = 0xF001 
7FFFF7D2AD01: 73 01                         jae    0x7ffff7d2ad04  ; <+20>
7FFFF7D2AD03: C3                            retq   
7FFFF7D2AD04: 48 8B 0D ED 80 0D 00          movq   0xd80ed(%rip), %rcx  ; _GLOBAL_OFFSET_TABLE_ + 632
7FFFF7D2AD0B: F7 D8                         negl   %eax
7FFFF7D2AD0D: 64 89 01                      movl   %eax, %fs:(%rcx)
7FFFF7D2AD10: 48 83 C8 FF                   orq    $-0x1, %rax
7FFFF7D2AD14: C3                            retq   
```

mio入门
https://hexilee.me/2018/12/17/rust-async-io/

总结起来，create_io_stack 就是创建Waker和Driver。Waker和Driver的本质是创建epoll和eventfd。

不同的driver有不同的token，Waker的token是TOKEN_WAKEUP，TOKEN_SIGNAL。

然后创建BlockingPool
// BlockingPool 是一个线程池，专门用于：
// 1. 运行阻塞的 CPU 密集型计算
// 2. 执行同步的 IO 操作
// 3. 调用阻塞的系统调用
// 4. 运行不兼容 async 的库

```rust
let (shutdown_tx, shutdown_rx) = shutdown::channel();

BlockingPool {
            // 任务生成器
            spawner: Spawner {
                inner: Arc::new(Inner {
                    shared: Mutex::new(Shared {
                        // 任务队列
                        queue: VecDeque::new(),
                        // 通知计数
                        num_notify: 0,
                        // 关闭标志
                        shutdown: false,
                        // 关闭通道发送端
                        shutdown_tx: Some(shutdown_tx),
                        last_exiting_thread: None,
                         // 工作线程映射
                        worker_threads: HashMap::new(),
                        // 工作线程索引
                        worker_thread_index: 0,
                    }),
                    condvar: Condvar::new(),
                    // 线程名前缀
                    thread_name: builder.thread_name.clone(),
                    // 线程栈大小
                    stack_size: builder.thread_stack_size,
                    // 线程启动后回调
                    after_start: builder.after_start.clone(),
                    // 线程停止前回调
                    before_stop: builder.before_stop.clone(),
                    // 最大线程数限制
                    thread_cap,
                    // 线程空闲保活时间
                    keep_alive,
                    // 性能指标收集
                    metrics: SpawnerMetrics::default(),
                }),
            },
            // 关闭接收器
            shutdown_rx,
        }
```

然后创建MultiThread 结构体
表示多线程运行时的实例
是 Tokio 默认的异步运行时
使用工作窃取算法在多个线程间平衡任务负载

```rust
pub(crate) struct MultiThread;  // 没有字段！实际的数据（线程池、任务队列等）都存储在返回的 Arc<Handle>中，MultiThread实例只是一个类型标记。

pub(crate) fn new(
        size: usize,
        driver: Driver,
        driver_handle: driver::Handle,
        blocking_spawner: blocking::Spawner,
        seed_generator: RngSeedGenerator,
        config: Config,
    ) 

size: usize,                    // 线程池大小（线程数量）
driver: Driver,                 // I/O 事件驱动（处理网络、文件等I/O事件）
driver_handle: driver::Handle,  // 驱动器的控制句柄
blocking_spawner: blocking::Spawner, // 阻塞任务调度器
seed_generator: RngSeedGenerator,    // 随机数种子生成器
config: Config,                 // 运行时配置
```

MultiThread new()创建parker和worker
```rust
    let parker = Parker::new(driver);
    let (handle, launch) = worker::create(
        size,
        parker,
        driver_handle,
        blocking_spawner,
        seed_generator,
        config,
    );
    (MultiThread, handle, launch)
```

Parker负责管理工作线程的休眠和唤醒：
当线程没有任务可执行时，让线程休眠以节省 CPU，每个线程都有自己的 Parker
当有新任务到来时，唤醒休眠的线程
同时集成 I/O 事件的通知

worker create 创建工作窃取线程池
```rust
pub(super) fn create(
    size: usize, // 线程数量
    park: Parker,// 线程调度器
    driver_handle: driver::Handle,// I/O 事件驱动
    blocking_spawner: blocking::Spawner,// 阻塞任务调度器
    seed_generator: RngSeedGenerator, // 随机数生成器
    config: Config,
) -> (Arc<Handle>, Launch) {
    let mut cores = Vec::with_capacity(size);// 每个线程的核心数据结构
    let mut remotes = Vec::with_capacity(size);// 远程访问接口
    let mut worker_metrics = Vec::with_capacity(size);// 指标收集

    // Create the local queues，每个工作线程有一个任务队列，一般来说一个线程运行在一个处理器上
    for _ in 0..size {
        // steal    -> 其他线程窃取任务的接口
        // run_queue -> 本地任务队列（双端队列）
        let (steal, run_queue) = queue::local();

        // park unpark 都是park clone出来一份，只是为了代码的易读性
        // park()：让当前线程休眠
        // unpark()：唤醒休眠的线程
        let park = park.clone();
        let unpark = park.unpark();
        let metrics = WorkerMetrics::from_config(&config);
        let stats = Stats::new(&metrics);

        cores.push(Box::new(Core {
            tick: 0,    // 任务执行计数器 记录工作线程执行了多少个任务 
            lifo_slot: None, // LIFO 后进先出 任务槽 普通任务入队到本地队列（run_queue） 如果是当前线程自己 spawn 的任务，放入 lifo_slot 提高缓存局部性，减少上下文切换
            lifo_enabled: !config.disable_lifo_slot,
            run_queue, // 本地任务队列
            is_searching: false, // 是否正在搜索任务
            is_shutdown: false, // 是否已关闭
            is_traced: false, // 是否启用追踪
            park: Some(park),
            global_queue_interval: stats.tuned_global_queue_interval(&config),
            stats,
            rand: FastRand::from_seed(config.seed_generator.next_seed()),
        }));

        remotes.push(Remote { steal, unpark });
        worker_metrics.push(metrics);
    }

    // 创建两个 idle 实例
    let (idle, idle_synced) = Idle::new(size);
    let (inject, inject_synced) = inject::Shared::new();

    let remotes_len = remotes.len();
    let handle = Arc::new(Handle {
        task_hooks: TaskHooks::from_config(&config),
        shared: Shared {
            remotes: remotes.into_boxed_slice(),
            inject,
            idle,
            owned: OwnedTasks::new(size),
            synced: Mutex::new(Synced {
                idle: idle_synced,
                inject: inject_synced,
            }),
            shutdown_cores: Mutex::new(vec![]),
            trace_status: TraceStatus::new(remotes_len),
            config,
            scheduler_metrics: SchedulerMetrics::new(),
            worker_metrics: worker_metrics.into_boxed_slice(),
            _counters: Counters,
        },
        driver: driver_handle,
        blocking_spawner,
        seed_generator,
    });

    let mut launch = Launch(vec![]);

    for (index, core) in cores.drain(..).enumerate() {
        launch.0.push(Arc::new(Worker {
            handle: handle.clone(),
            index,
            core: AtomicCell::new(Some(core)),
        }));
    }

    (handle, launch)
}
```

queue local 创建了一个vector buffer，对内增加队列头尾成为Local，对外成为Steal
```rust
pub(crate) fn local<T: 'static>() -> (Steal<T>, Local<T>) {
    let mut buffer = Vec::with_capacity(LOCAL_QUEUE_CAPACITY);

    for _ in 0..LOCAL_QUEUE_CAPACITY {
        buffer.push(UnsafeCell::new(MaybeUninit::uninit()));
    }

    let inner = Arc::new(Inner {
        head: AtomicUnsignedLong::new(0),
        tail: AtomicUnsignedShort::new(0),
        buffer: make_fixed_size(buffer.into_boxed_slice()),
    });

    let local = Local {
        inner: inner.clone(),
    };

    let remote = Steal(inner);

    (remote, local)
}

// 线程A：入队任务
local.push(task1).unwrap();
local.push(task2).unwrap();

// 线程A：出队任务（LIFO）
let task = local.pop();  // 得到 task2（后进先出）

// 线程B：窃取任务
if let Steal::Success(tasks) = steal.steal() {
    // 得到一批任务
    for task in tasks {
        task.run();
    }
}
```


统计性能的指标
```rust
// 每个工作线程的性能指标
pub(crate) struct WorkerMetrics {
    tasks_processed: AtomicU64,      // 处理的任务数
    steals_initiated: AtomicU64,     // 发起的窃取次数
    steals_received: AtomicU64,      // 被窃取的次数
    park_count: AtomicU64,           // 休眠次数
    noop_count: AtomicU64,           // 空转次数
    // ...
}

// 可以获取运行时统计
fn print_stats(handle: &Handle) {
    for (i, metrics) in handle.shared.worker_metrics.iter().enumerate() {
        println!("Worker {}: {} tasks processed", 
            i, metrics.tasks_processed.load(Relaxed));
    }
}

// 根据指标调整行为
fn adjust_behavior(worker: &mut Worker) {
    let metrics = &worker.handle.shared.worker_metrics[worker.index];
    
    // 如果这个线程经常被窃取，可能负载过重
    let steal_ratio = metrics.steals_received.load(Relaxed) as f64 
                    / metrics.tasks_processed.load(Relaxed) as f64;
    
    if steal_ratio > 0.3 {
        // 调整策略，比如增加批量窃取大小
    }
}

// 诊断性能问题
fn diagnose_performance(metrics: &[WorkerMetrics]) {
    let total_parked: u64 = metrics.iter()
        .map(|m| m.park_count.load(Relaxed))
        .sum();
    
    if total_parked > 1000 {
        println!("Warning: workers are parking too frequently");
    }
}
```

worker create在为每个线程创建一个工作队列后，创建两个 idle 实例（所有工作线程共享的）
Idle负责跟踪和管理哪些工作线程空闲以及哪些正在搜索任务
idle: 无锁原子操作（快速路径）
idle_synced: 有锁保护（慢速路径，用于复杂操作）
```rust
impl Idle {
    pub(super) fn new(num_workers: usize) -> (Idle, Synced) {
        let init = State::new(num_workers);

        let idle = Idle {
            state: AtomicUsize::new(init.into()),
            num_workers,
        };

        // 慢速路径只有一个Vec，存work id，返回后有Share synced: Mutex<Synced> 封装出一个锁
        let synced = Synced {
            sleepers: Vec::with_capacity(num_workers),
        };

        (idle, synced)
    }
```
ilde的state字段是一个打包的 32 位或 64 位整数
```rust
// 状态位的典型布局（32位示例）：
// 高 16 位            低 16 位
// ┌─────────────────┬─────────────────┐
// │  searching      │   unparked      │
// └─────────────────┴─────────────────┘
// searching: 正在搜索任务的线程数
// unparked:  被唤醒（非休眠）的线程数

// 工作线程状态机：
运行中 → 空闲搜索 → 休眠
   ↑         ↓         ↑
   └─────────┴── 唤醒 ←─┘


为什么要跟踪 searching 状态
// 避免"惊群效应"
// 如果没有 searching 跟踪：
// 1. 新任务到来
// 2. 所有空闲线程都被唤醒
// 3. 只有一个线程能拿到任务
// 4. 其他线程白唤醒 → 浪费CPU

// 有了 searching 跟踪：
// 1. 新任务到来
// 2. 只唤醒非searching的线程
// 3. 或者只唤醒一个searching线程
// 4. 减少不必要的唤醒

关于快慢路径

// 快速路径只知道"有线程在搜索"
// 但不知道"是哪个线程"

// 当需要：
// 1. 选择特定线程唤醒
// 2. 处理饥饿问题
// 3. 复杂的负载均衡
// → 需要慢速路径
或者
CAS Compare-And-Swap 失败 → 慢速路径
// 快速路径：基于随机数随机选择一个
// 尝试标记这个线程
// 快速路径失败：需要精确选择
```

worker create 创建两个 idle 实例后，创建两个全局任务队列的共享结构

inject队列是 Tokio 调度器的全局任务队列
// 1. 从外部（非工作线程）提交的任务进入 inject 队列
// 2. 工作线程本地队列满时的溢出，如果本地队列满，进入 inject
// 3. 当工作线程窃取不到任务时，检查 inject 队列

```rust
// 用于快速入队/出队（无锁）
let (inject, inject_synced) = inject::Shared::new();
```

worker create 然后把remote，idle，inject等等封装到handle中

worker create 然后创建Launch，为每个 Core 创建一个 Worker
```rust
let mut launch = Launch(vec![]);

// drain是 Rust 中集合类型（如 Vec、String、HashMap等）的一个方法，用于消耗性地移除并返回元素
for (index, core) in cores.drain(..).enumerate() {
        // 将之前创建的 Core结构包装成 Worker，并放入 launch启动器中
        launch.0.push(Arc::new(Worker {
            handle: handle.clone(),
            index,
            core: AtomicCell::new(Some(core)),
        }));
    }
```

至此，worker create()执行完毕，回到MultiThread::new()得到handle（里面有任务队列，Driver，提交任务、查询状态）和Launch（启动线程、管理线程）

回到build_threaded_runtime()执行enter() 创建一个 运行时上下文守卫
```rust
let _enter = handle.enter();

将handle放置到线程局部变量CONTEXT存储，主要防止嵌套运行时，即EnterGuard 是 RAII (Resource Acquisition Is Initialization 对象的构造和析构来管理资源的生命周期)守卫
pub fn enter(&self) -> EnterGuard<'_> {
    // 将当前运行时句柄设置到线程局部存储
    context::try_set_current(&self.inner)
}

pub(crate) fn try_set_current(handle: &scheduler::Handle) -> Option<SetCurrentGuard> {
    CONTEXT.try_with(|ctx| ctx.set_current(handle)).ok()
}
```

build_threaded_runtime() 调用launch.launch();启动所有工作线程，为每个工作线程创建一个阻塞线程，本质上最后是调用rust标准库thread::Builder.spawn 为每个核创建一个线程。

```rust
struct Launch(Vec<Arc<Worker>>); // 元组结构体；只是类型别名：创建一个新类型，或者简单包装：只需要一个字段，不需要给字段命名可以用元祖结构体，使用.0访问第一个字段

impl Launch {
    pub(crate) fn launch(mut self) {
        for worker in self.0.drain(..) {
            runtime::spawn_blocking(move || run(worker));
        }
    }
}

worker
pub(super) struct Worker {
    /// Reference to scheduler's handle
    handle: Arc<Handle>,

    /// Index holding this worker's remote state
    index: usize,

    /// Used to hand-off a worker's core to another thread.
    core: AtomicCell<Core>,
}
```

这里的spawn_blocking会进入blocking/pool.rs的spawn_blocking
```rust
tokio-1.48.0/src/runtime/blocking/pool.rs

pub(crate) fn spawn_blocking<F, R>(func: F) -> JoinHandle<R>
where
    F: FnOnce() -> R + Send + 'static,
    R: Send + 'static,
{
    let rt = Handle::current();
    rt.spawn_blocking(func)
}

这里的func是|| run(worker)


tokio-1.48.0/src/runtime/handle.rs
pub fn spawn_blocking<F, R>(&self, func: F) -> JoinHandle<R>
where
    F: FnOnce() -> R + Send + 'static,
    R: Send + 'static,
{
    self.inner.blocking_spawner().spawn_blocking(self, func)
}

self.inner.blocking_spawner() 返回的是对应类型的
tokio-1.48.0/src/runtime/scheduler/mod.rs
pub(crate) fn blocking_spawner(&self) -> &blocking::Spawner {
    match_flavor!(self, Handle(h) => &h.blocking_spawner)
}

// match_flavor! 是编译时分发
// match_flavor!是 Tokio 中的一个内部宏，用于处理不同运行时类型的模式匹配
// 方便处理不同运行时
match self {
    Handle::MultiThread(h) => &h.blocking_spawner,
    Handle::CurrentThread(h) => &h.blocking_spawner,
}

在找到对应blocking_spawner的spawn_blocking
tokio-1.48.0/src/runtime/blocking/pool.rs

impl Spawner {
pub(crate) fn spawn_blocking<F, R>(&self, rt: &Handle, func: F) -> JoinHandle<R>
    where
        F: FnOnce() -> R + Send + 'static,
        R: Send + 'static,
{
    // 计算函数大小
    let fn_size = std::mem::size_of::<F>();

    // 1. 创建可连接的 future
    let (join_handle, spawn_result) = if fn_size > BOX_FUTURE_THRESHOLD {
        // 大函数在堆上分配
        self.spawn_blocking_inner(
            Box::new(func),
            Mandatory::NonMandatory,
            SpawnMeta::new_unnamed(fn_size),
            rt,
        )
    } else {
        // 小函数在栈上分配
        self.spawn_blocking_inner(
            func,
            Mandatory::NonMandatory,
            SpawnMeta::new_unnamed(fn_size),
            rt,
        )
    };

    match spawn_result {
        Ok(()) => join_handle,
        // Compat: do not panic here, return the join_handle even though it will never resolve
        Err(SpawnError::ShuttingDown) => join_handle,
        Err(SpawnError::NoThreads(e)) => {
            panic!("OS can't spawn worker thread: {e}")
        }
    }
}

pub(crate) fn spawn_blocking_inner<F, R>(
        &self,
        func: F,
        is_mandatory: Mandatory,
        spawn_meta: SpawnMeta<'_>,
        rt: &Handle,
    ) -> (JoinHandle<R>, Result<(), SpawnError>)
    {
        // 生成唯一任务 ID
        let id = task::Id::next();
        // 使用func生成一个BlockingTask
        let fut = blocking_task::<F, BlockingTask<F>>(BlockingTask::new(func), spawn_meta, id.as_u64());

        // 用于创建一个"无所有权"的任务
        let (task, handle) = task::unowned(
            fut, // 执行的异步 Future
            BlockingSchedule::new(rt),  // 创建一个调度器，指定任务在哪里运行
            id,
            task::SpawnLocation::capture(), // 捕获任务生成的位置（用于调试）
        );

        let spawned = self.spawn_task(Task::new(task, is_mandatory), rt);
        (handle, spawned)
    }


impl<T> BlockingTask<T> {
    /// Initializes a new blocking task from the given function.
    pub(crate) fn new(func: T) -> BlockingTask<T> {
        BlockingTask { func: Some(func) }
    }
}

blocking_task::<F, BlockingTask<F>>(BlockingTask::new(func), spawn_meta, id.as_u64())
是泛型函数调用
pub(crate) fn blocking_task<Fn, Fut>(task: Fut, spawn_meta: SpawnMeta<'_>, id: u64)

::<F, BlockingTask<F>> 指定泛型类型
```

得到BlockingTask后，调用unowned创建"无所有权"任务（不由运行时任务列表（OwnedTasks）管理的任务）

```rust
    /// Creates a new task with an associated join handle. This method is used
    /// only when the task is not going to be stored in an `OwnedTasks` list.
    ///
    /// Currently only blocking tasks use this method.
    pub(crate) fn unowned<T, S>(
        task: T,
        scheduler: S,
        id: Id,
        spawned_at: SpawnLocation,
    ) -> (UnownedTask<S>, JoinHandle<T::Output>)
    {
        // 1. 创建新任务 RawTask封装成为Task
        // Task       - 任务的所有权句柄（用于存储和传递所有权）
        // Notified   - 任务的就绪通知句柄（用于调度执行）
        // JoinHandle - 任务的输出句柄（用于获取结果）
        let (task, notified, join) = new_task(
            task,
            scheduler,
            id,
            spawned_at,
        );

        // This transfers the ref-count of task and notified into an UnownedTask.
        // This is valid because an UnownedTask holds two ref-counts.
        // 2. 转移所有权到 UnownedTask
        let unowned = UnownedTask {
            raw: task.raw,
            _p: PhantomData,
        };
        // 防止task 的析构器（drop）被调用
        std::mem::forget(task);
        std::mem::forget(notified);

        (unowned, join)
    }
}

spawn_blocking_inner()->Spawner.spawn_task()->Spawner.spawn_thread()->thread::Builder::spawn()

fn spawn_task(&self, task: Task, rt: &Handle) -> Result<(), SpawnError> {
        let mut shared = self.inner.shared.lock();
        // 把生成的task放到队列之中
        shared.queue.push_back(task);
        self.inner.metrics.inc_queue_depth();

        // 判断是否还有空闲的worker线程
        if self.inner.metrics.num_idle_threads() == 0 {
            // 如果没有空闲worker线程就要尝试创建线程

            if self.inner.metrics.num_threads() == self.inner.thread_cap {
                // 前提是没达到最大工作线程
            } else {
                assert!(shared.shutdown_tx.is_some());
                let shutdown_tx = shared.shutdown_tx.clone();

                if let Some(shutdown_tx) = shutdown_tx {
                    let id = shared.worker_thread_index;

                    match self.spawn_thread(shutdown_tx, rt, id) {
                        Ok(handle) => {
                            self.inner.metrics.inc_num_threads();
                            shared.worker_thread_index += 1;
                            shared.worker_threads.insert(id, handle);
                        }
                    }
                }
            }
        } else {
            // Notify an idle worker thread. The notification counter
            // is used to count the needed amount of notifications
            // exactly. Thread libraries may generate spurious
            // wakeups, this counter is used to keep us in a
            // consistent state.
            self.inner.metrics.dec_num_idle_threads();
            shared.num_notify += 1;
            // 如何有空闲的线程，则通知一个
            self.inner.condvar.notify_one();
        }

        Ok(())
    }

tokio-1.48.0/src/runtime/blocking/pool.rs
fn spawn_thread(
        &self,
        shutdown_tx: shutdown::Sender,
        rt: &Handle,
        id: usize,
    ) -> io::Result<thread::JoinHandle<()>> {
        let mut builder = thread::Builder::new().name((self.inner.thread_name)());

        if let Some(stack_size) = self.inner.stack_size {
            builder = builder.stack_size(stack_size);
        }

        let rt = rt.clone();

        // 调用rust 标准库的spawn创建出线程
        builder.spawn(move || {
            // Only the reference should be moved into the closure
            let _enter = rt.enter();
            rt.inner.blocking_spawner().inner.run(id);
            drop(shutdown_tx);
        })
    }
```


spawn_thread中调用的rt.inner.blocking_spawner().inner.run(id); 就是对每个worker从阻塞线程池拿到任务运行，（注意，这里从shared.queue.pop_front()拿到的任务是 || run(worker)，虽然名字都叫做run，但是参数不一样了，调用的是）
```rust
tokio\src\runtime\blocking\pool.rs

impl Inner {
    fn run(&self, worker_thread_id: usize) {
        if let Some(f) = &self.after_start {
            // 如何有回调函数，执行线程启动后的回调函数
            f();
        }

        let mut shared = self.shared.lock();
        let mut join_on_thread = None;

        // 新的worker线程就是一个整个大循环，不断的从任务队列中获取任务
        'main: loop {
            // BUSY
            while let Some(task) = shared.queue.pop_front() {
                self.metrics.dec_queue_depth();
                drop(shared);
                // 每个task还有自己的run，其实就是UnownedTask的run，最终调用的是future的poll()
                task.run();

                shared = self.shared.lock();
            }

            // IDLE
            self.metrics.inc_num_idle_threads();

            while !shared.shutdown {
                // 如果实在没有任务运行了，就进入休眠状态等待条件变量唤醒
                let lock_result = self.condvar.wait_timeout(shared, self.keep_alive).unwrap();

                shared = lock_result.0;
                let timeout_result = lock_result.1;

                if shared.num_notify != 0 {
                    // We have received a legitimate wakeup,
                    // acknowledge it by decrementing the counter
                    // and transition to the BUSY state.
                    shared.num_notify -= 1; // 确认接收唤醒
                    break;// 跳出IDLE循环，进入BUSY状态
                }

                // Even if the condvar "timed out", if the pool is entering the
                // shutdown phase, we want to perform the cleanup logic.
                // 如果超时且无任务，线程退出
                if !shared.shutdown && timeout_result.timed_out() {
                    // We'll join the prior timed-out thread's JoinHandle after dropping the lock.
                    // This isn't done when shutting down, because the thread calling shutdown will
                    // handle joining everything.
                    let my_handle = shared.worker_threads.remove(&worker_thread_id);
                    join_on_thread = std::mem::replace(&mut shared.last_exiting_thread, my_handle);

                    break 'main;
                }

                // Spurious wakeup detected, go back to sleep.
            }

            if shared.shutdown {
                // Drain the queue
                while let Some(task) = shared.queue.pop_front() {
                    self.metrics.dec_queue_depth();
                    drop(shared);
                    // 关闭或执行强制任务
                    task.shutdown_or_run_if_mandatory();

                    shared = self.shared.lock();
                }

                // Work was produced, and we "took" it (by decrementing num_notify).
                // This means that num_idle was decremented once for our wakeup.
                // But, since we are exiting, we need to "undo" that, as we'll stay idle.
                self.metrics.inc_num_idle_threads();
                // NOTE: Technically we should also do num_notify++ and notify again,
                // but since we're shutting down anyway, that won't be necessary.
                break;
            }
        }

        // Thread exit
        self.metrics.dec_num_threads(); // 减少线程计数

        // num_idle should now be tracked exactly, panic
        // with a descriptive message if it is not the
        // case.
        let prev_idle = self.metrics.dec_num_idle_threads();
        assert!(
            prev_idle >= self.metrics.num_idle_threads(),
            "num_idle_threads underflowed on thread exit"
        );

        if shared.shutdown && self.metrics.num_threads() == 0 {
            self.condvar.notify_one();  // 通知关闭完成
        }

        drop(shared);

        if let Some(f) = &self.before_stop {
            f();  // 执行线程停止前的回调
        }

        if let Some(handle) = join_on_thread {
            let _ = handle.join(); // 执行线程停止前的回调
        }
    }
}
```

每个Task的run最终调用的是

```rust
tokio\src\runtime\scheduler\multi_thread\worker.rs

fn run(worker: Arc<Worker>) {

    // 1. 取出这个 worker 对应的 Core，如果已经被别的线程拿走，就直接返回
    let core = match worker.core.take() {
        Some(core) => core,
        None => return,
    };

    // 2. 记录当前 OS 线程 ID（用于 metrics）
    worker.handle.shared.worker_metrics[worker.index]
        .set_thread_id(thread::current().id());

    // 3. 构造一个 MultiThread 调度器的 Handle
    let handle = scheduler::Handle::MultiThread(worker.handle.clone());

    // 4. 进入 Tokio runtime 上下文，并执行调度循环
    crate::runtime::context::enter_runtime(&handle, true, |_| {
        let cx = scheduler::Context::MultiThread(Context {
            worker,
            core: RefCell::new(None),
            defer: Defer::new(),
        });

        context::set_scheduler(&cx, || {
            let cx = cx.expect_multi_thread();

            // 5. 核心调度循环：run(core)
            assert!(cx.run(core).is_err());

            // 6. 如果 core 被 block_in_place 换走了，执行延迟唤醒
            cx.defer.wake();
        });
    });
}

Context的run()
impl Context {
    fn run(&self, mut core: Box<Core>) -> RunResult {
        // Reset `lifo_enabled` here in case the core was previously stolen from
        // a task that had the LIFO slot disabled.
        self.reset_lifo_enabled(&mut core);

        // Start as "processing" tasks as polling tasks from the local queue
        // will be one of the first things we do.
        core.stats.start_processing_scheduled_tasks();

        while !core.is_shutdown {
            self.assert_lifo_enabled_is_correct(&core);

            if core.is_traced {
                core = self.worker.handle.trace_core(core);
            }

            // Increment the tick
            core.tick();

            // Run maintenance, if needed
            core = self.maintenance(core);

            // First, check work available to the current worker.
            if let Some(task) = core.next_task(&self.worker) {
                core = self.run_task(task, core)?;
                continue;
            }

            // We consumed all work in the queues and will start searching for work.
            core.stats.end_processing_scheduled_tasks();

            // There is no more **local** work to process, try to steal work
            // from other workers.
            if let Some(task) = core.steal_work(&self.worker) {
                // Found work, switch back to processing
                core.stats.start_processing_scheduled_tasks();
                core = self.run_task(task, core)?;
            } else {
                // Wait for work
                core = if !self.defer.is_empty() {
                    self.park_yield(core)
                } else {
                    self.park(core)
                };
                core.stats.start_processing_scheduled_tasks();
            }
        }

        #[cfg(all(tokio_unstable, feature = "time"))]
        {
            match self.worker.handle.timer_flavor {
                TimerFlavor::Traditional => {}
                TimerFlavor::Alternative => {
                    util::time_alt::shutdown_local_timers(
                        &mut core.time_context.wheel,
                        &mut core.time_context.canc_rx,
                        self.worker.handle.take_remote_timers(),
                        &self.worker.handle.driver,
                    );
                }
            }
        }

        core.pre_shutdown(&self.worker);
        // Signal shutdown
        self.worker.handle.shutdown_core(core);
        Err(())
    }
```