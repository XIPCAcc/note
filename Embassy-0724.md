

```rust
#[cortex_m_rt::entry]
fn main() -> ! {
    unsafe fn __make_static<T>(t: &mut T) -> &'static mut T {
        unsafe { ::core::mem::transmute(t) }
    }

    let mut executor = ::embassy_executor::Executor::new();
    let executor = unsafe { __make_static(&mut executor) };
    executor.run(|spawner| {
        let main_task = __embassy_main(spawner).unwrap();
        spawner.spawn(main_task);
    })
}
```

```rust
// #[::embassy_executor::task()] 宏展开会调用TaskPool::_spawn_async_fn遍历池里所有 TaskStorage ，用 AvailableTask::claim 找第一个空闲槽(原子抢占一个槽位)，initialize_impl — 把__embassy_main Future 写进去，最后用 TaskRef::new 包装成一个不透明指针，返回 SpawnToken(对TaskRef的封装，表明此时已经占用了TaskPool的一个槽位，编译器保证必须传给spawn()放进队列)
#[::embassy_executor::task()] 
#[allow(clippy::future_not_send)]
async fn __embassy_main(_spawner: Spawner) {
    let p = embassy_stm32::init(Default::default());
    let mut led = Output::new(p.PB14, Level::Low, Speed::VeryHigh);
    let mut button = ExtiInput::new(p.PC13, p.EXTI13, Pull::Up, Irqs);

    loop {
        button.wait_for_any_edge().await;
        if button.is_low() {
            led.set_high();
        } else {
            led.set_low();
        }
    }
}

#[derive(Copy, Clone)]
pub struct Irqs;

#[allow(non_snake_case)]
#[unsafe(no_mangle)]
unsafe extern "C" fn EXTI15_10() {
    unsafe {
        <embassy_stm32::exti::InterruptHandler<
            embassy_stm32::interrupt::typelevel::EXTI15_10
        > as embassy_stm32::interrupt::typelevel::Handler<
            embassy_stm32::interrupt::typelevel::EXTI15_10
        >>::on_interrupt();
    }
}

unsafe impl
    embassy_stm32::interrupt::typelevel::Binding<
        embassy_stm32::interrupt::typelevel::EXTI15_10,
        embassy_stm32::exti::InterruptHandler<
            embassy_stm32::interrupt::typelevel::EXTI15_10
        >,
    > for Irqs
{
}
```

## run

Executor 的run函数是整个Embassy的核心循环，因为run的实现也是和平台紧密关联的。

### RISCV
```rust
pub fn run(&'static mut self, init: impl FnOnce(Spawner)) -> ! {
            init(self.inner.spawner());

            loop {
                unsafe {
                    self.inner.poll();
                    // 为了避免丢失唤醒，这里必须重用临界区方式，执行wfi之前关中断。
                    critical_section::with(|_| {
                        // if there is work to do, loop back to polling
                        // TODO can we relax this?
                        if SIGNAL_WORK_THREAD_MODE.load(Ordering::SeqCst) {
                            SIGNAL_WORK_THREAD_MODE.store(false, Ordering::SeqCst);
                        }
                        // if not, wait for interrupt
                        else {
                            // RISC-V 的 wfi 在中断被
                            // 屏蔽时也能唤醒（wfi 不受 mstatus.MIE 影响）
                            // 或者：执行 wfi 时即使全局中断关闭，
                            // 只要有 pending 中断，wfi 也会立即返回
                            core::arch::asm!("wfi");
                        }
                    });
                    // if an interrupt occurred while waiting, it will be serviced here
                }
            }
        }
```

### Cortex-m

```rust
        pub fn run(&'static mut self, init: impl FnOnce(Spawner)) -> ! {
            init(self.inner.spawner());

            loop {
                unsafe {
                    self.inner.poll();
                    // SEV 的效果是"锁存"的 。即使当前 CPU 没在 wfe， SEV 也会把事件位置 1，等下次执行 wfe 时会立刻看到 1 → 清 0 → 直接返回，不会睡
                    asm!("wfe");
                };
            }
        }
```

### avr

```rust
        pub fn run(&'static mut self, init: impl FnOnce(Spawner)) -> ! {
            init(self.inner.spawner());

            loop {
                unsafe {
                    avr_device::interrupt::disable();
                    if !SIGNAL_WORK_THREAD_MODE.swap(false, Ordering::SeqCst) {
                        // AVR 架构保证了"sei + 下一条指令"的原子性sei。执行后，要等 下一条指令 执行完毕之后，才会响应中断
                        avr_device::interrupt::enable();
                        avr_device::asm::sleep();
                    } else {
                        avr_device::interrupt::enable();
                        self.inner.poll();
                    }
                }
            }
        }
```

### init

run函数接收一个闭包init作为参数。对于示例程序，该闭包为

```rust
|spawner| {
        let main_task = __embassy_main(spawner).unwrap(); // main_task 是一个SpawnToken
        spawner.spawn(main_task);
    }
```

run首先创建一个spawner并调用init闭包，在闭包中，调用了__embassy_main生成一个SpawnToken，相当于在TaskPool中占用了一个槽位，

调用spawner.spawn() 将SpawnToken 中的TaskRef放进runqueue。

```rust
pub fn spawn<S>(&self, token: SpawnToken<S>) {
    let task = token.raw_task;
    mem::forget(token);
    unsafe { self.executor.spawn(task) }
}
```

### spawn

```rust
pub(super) unsafe fn spawn(&'static self, task: TaskRef) {
    task.header()
        .executor // 把当前 executor 的指针存到 task header 的 executor 字段里。任务从此"知道"自己属于哪个 executor。后续任务被 wake 时
       // 唤醒方手里只有 TaskRef （一个指向 TaskStorage 的指针），它怎么知道把这个任务投递到哪个 executor 的 run queue？
       // 通过task header 里存着的 executor 指针 。这个绑定就是在 spawn 这一行建立的。
        .store((self as *const Self).cast_mut(), Ordering::Relaxed);

    // 对于没有原子指令的环境，需要设计State作为临界区，临界区同时保护了两件事 ：
    // 1. State 的标志位操作（在 run_enqueue 里）
    // 2. run_queue 的链表操作（在 enqueue → push_was_empty 里）
    // 无原子 ：State 用 Cell ，run_queue 用 UnsafeCell<Stack> ，必须靠临界区串行化所有访问
    // 有原子 ：State 用 CAS 保证唯一性，run_queue 用 lock-free 链表，整体不需要锁
    state::locked(|l| {
        self.enqueue(task, l);
    })
}
```


### enqueue

```rust
    unsafe fn enqueue(&self, task: TaskRef, l: state::Token) {

        if self.run_queue.enqueue(task, l) {
            self.pender.pend();
        }
    }
```

run_queue 是一个TransferStack，有原子指令的平台用 cordyceps::TransferStack（lock-free 的 Treiber Stack （无锁并发栈））

```rust
pub(crate) struct RunQueue {
    stack: TransferStack<TaskHeader>,
}

pub(crate) unsafe fn enqueue(&self, task: TaskRef, _tok: super::state::Token) -> bool {
    self.stack.push_was_empty(
        task,
        #[cfg(not(target_has_atomic = "ptr"))]
        _tok,
    )
}
```

没有指针原子的平台用 MutexTransferStack，用 critical_section::Mutex 把普通的 Stack 包起来，访问需要 Token （即临界区凭证）。本质就是 关中断 → push → 返回是否为空 ，完全串行化。
```rust
struct MutexTransferStack<T> {
    inner: critical_section::Mutex<core::cell::UnsafeCell<cordyceps::Stack<T>>>,
}

fn push_was_empty(&self, item: T::Handle, token: super::state::Token) -> bool {
    let inner = unsafe { &mut *self.inner.borrow(token).get() };
    let is_empty = inner.is_empty();
    inner.push(item);
    is_empty
}
```

### pend

```rust
impl Pender {
    pub(crate) fn pend(self) {
        unsafe extern "Rust" {
            fn __pender(context: *mut ());
        }
        unsafe { __pender(self.0) };
    }
}
```
各平台的 __pender


#### cortex-m

根据 context 判断是线程模式还是中断模式，分别走 sev 或 NVIC::pend 。

```rust
#[cfg(any(feature = "executor-thread", feature = "executor-interrupt"))]
fn __pender(context: *mut ()) {
    
        #[cfg(feature = "executor-thread")]
        if !cfg!(feature = "executor-interrupt") || context == THREAD_PENDER {
            core::arch::asm!("sev");
            return;
        }

        #[cfg(feature = "executor-interrupt")]
        {
            NVIC::pend(irq);
        }
    }
}
```

#### avr和RISCV

真正唤醒 wfi 的是外设中断本身，不是 __pender 。 
__pender 只是在中断里"顺便"设个标志，让主循环醒来后知道"是有工作要做，不是误唤醒"。

```rust
    #[unsafe(export_name = "__pender")]
    fn __pender(_context: *mut ()) {
        SIGNAL_WORK_THREAD_MODE.store(true, Ordering::SeqCst);
    }
```

## ExtiInput

button的类型为ExtiInput是 GPIO 输入 + EXTI 中断驱动的组合。

通过调用wait_for_any_edge创建ExtiInputFuture Future


```
pub async fn wait_for_any_edge(&mut self) {
    ExtiInputFuture::new(&self.pin, TriggerEdge::Any, true).await
}
```

### ExtiInputFuture

```rust
impl<'a> Future for ExtiInputFuture<'a> {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        //  把当前任务的 waker 注册到全局 waker 数组，以Pin的编号作为索引
        EXTI_WAKERS[self.pin as usize].register(cx.waker());

        let imr = cpu_regs().imr(0).read();
        if !imr.line(self.pin as _) {
            Poll::Ready(())   // IMR bit13 = 0 → 中断已触发(被 ISR 关掉了) → 完成
        } else {
            Poll::Pending     // IMR bit13 = 1 → 还在等 → 挂起
        }
    }
}
```


### on_irq


main中写的
```rust
bind_interrupts!(
    pub struct Irqs {
        EXTI15_10 => exti::InterruptHandler<interrupt::typelevel::EXTI15_10>;
    }
);

// 展开后成为EXTI15_10，链接器 把 EXTI15_10 这个符号填到中断向量表的对应 slot
#[allow(non_snake_case)]
#[unsafe(no_mangle)]
unsafe extern "C" fn EXTI15_10() {        // ← 这个名字必须和向量表一致
    unsafe {
        <exti::InterruptHandler<interrupt::typelevel::EXTI15_10>
            as interrupt::typelevel::Handler<interrupt::typelevel::EXTI15_10>
        >::on_interrupt();                 // ← 调到绑定 1
    }
}

unsafe impl interrupt::typelevel::Binding<
    interrupt::typelevel::EXTI15_10,
    exti::InterruptHandler<interrupt::typelevel::EXTI15_10>,
> for Irqs {}
```

```
PC13 电平变化 (硬件)
    ↓
EXTI pending bit 13 置位 (硬件)
    ↓
NVIC 触发 EXTI15_10 中断 (硬件查向量表)
    ↓
向量表 slot 指向 EXTI15_10() 函数  ←  cortex-m-rt 启动时设置
    ↓
EXTI15_10()                          ←  bind_interrupts! 宏生成
    └─ <InterruptHandler<EXTI15_10>>::on_interrupt()
       ↓
       InterruptHandler::on_interrupt()    ←  exti/mod.rs#L468-472 (编译期绑定)
       └─ on_irq()                         ←  exti/mod.rs#L57 (编译期绑定)
          ├─ read pending
          ├─ mask off fired lines
          ├─ EXTI_WAKERS[pin].wake()
          └─ clear pending
```


```rust
unsafe fn on_irq() {
    // We don't handle or change any EXTI lines above 16.
    let bits = read_pending() & 0x0000FFFF;

    // Mask all the channels that fired.
    cpu_regs().imr(0).modify(|w| w.0 &= !bits);

    // 调用Waker，唤醒任务
    for pin in BitIter(bits) {
        EXTI_WAKERS[pin as usize].wake();
    }

    // Clear pending
    low_level::clear_exti_pending_mask(bits);
}
```

## Waker

### AtomicWaker

有原子指令的采用AtomicWaker::wake。

- 数据 ：原子状态位
- register ：CAS(WAITING→REGISTERING) 抢锁 → 写 waker → CAS 释放锁 + 检查握手
- wake ： fetch_or(WAKING) 无条件设位 → 如果 WAITING 就自己读 waker 并调 wake_by_ref ，否则留给持有者处理（握手）
- 临界区 ： 不需要 。纯靠原子 CAS/fetch_or 串行化


```rust
#[cfg(target_has_atomic = "32")]
pub struct AtomicWaker {
    state: AtomicUsize,
    waker: UnsafeCell<Option<Waker>>,
}
```

### wake

```rust
    /// [`CriticalSectionWaker`](super::CriticalSectionWaker).
    pub fn wake(&self) {
        // 设置WAKING 位，表明正在执行，持有 waker 所有权 
        match self.state.fetch_or(WAKING, AcqRel) {
            // WAITING 0b00 空闲
            WAITING => {
                unsafe {
                    if let Some(w) = &*self.waker.get() {
                        w.wake_by_ref();
                    }
                }
                self.state.swap(WAITING, Release); // 释放，回到空闲
            }
            _ => {
            // REGISTERING 0b01 register() 正在执行，持有 waker 所有权
            // WAKING 0b10 wake() 正在执行，持有 waker 所有权 
            // REGISTERING | WAKING 0b11 register() 中途 wake() 来了—— "握手"信号
            }
        }
    }
```

### register

一开始创建出来的静态数组EXTI_WAKERS，即创建了一系列空的AtomicWaker，通过调用register将新的waker存入EXTI_WAKERS数组（以外部中断号座位下标索引）
static EXTI_WAKERS: [AtomicWaker; EXTI_COUNT] = [const { AtomicWaker::new() }; EXTI_COUNT];


```rust
    pub fn register(&self, waker: &Waker) {
        match self
            .state
            .compare_exchange(WAITING, REGISTERING, Acquire, Acquire)
            .unwrap_or_else(|x| x)
        {
            WAITING => {
                let evicted: Option<Waker>;
                let pending_wake: Option<Waker>;
                unsafe {
                    evicted = match &*self.waker.get() {
                        Some(old) if old.will_wake(waker) => None,
                        _ => (*self.waker.get()).replace(waker.clone()),
                    };

                    let res = self.state.compare_exchange(REGISTERING, WAITING, AcqRel, Acquire);
                    pending_wake = match res {
                        Ok(_) => None,
                        Err(_) => {
                            let w = (*self.waker.get()).take().unwrap();
                            self.state.swap(WAITING, AcqRel);
                            Some(w)
                        }
                    };
                }

                if let Some(w) = pending_wake {
                    w.wake();
                }
                // 唤醒旧的waker
                if let Some(w) = evicted {
                    w.wake();
                }
            }
            WAKING => {
                // A wake is currently in flight. Schedule the supplied waker
                // directly so the task does not miss the wakeup.
                waker.wake_by_ref();
            }
            state => {
                debug_assert!(state == REGISTERING || state == REGISTERING | WAKING);
            }
        }
```

### CriticalSectionWaker

无原子指令则采用 CriticalSectionWaker。

- 数据 ： Mutex<Cell<Option<Waker>>>
- 锁 ： CriticalSectionRawMutex → critical_section::with(...) → 关中断
- register ：关中断 → 写 waker → 开中断
- wake ：关中断 → 读 waker → 调 wake_by_ref → 开中断
- 临界区 ： 每次都要关中断+开中断两条指令

```rust
pub struct CriticalSectionWaker {
    waker: GenericAtomicWaker<CriticalSectionRawMutex>,
}

impl CriticalSectionWaker {
    /// Create a new `CriticalSectionWaker`.
    pub const fn new() -> Self {
        Self {
            waker: GenericAtomicWaker::new(CriticalSectionRawMutex::new()),
        }
    }

    /// Register a waker. Overwrites the previous waker, if any.
    pub fn register(&self, w: &Waker) {
        self.waker.register(w);
    }

    /// Wake the registered waker, if any.
    pub fn wake(&self) {
        self.waker.wake();
    }
}

```

### wake_by_ref

```rust
pub fn wake_by_ref(&self) {
    // Rust Waker API，直接调 vtable 的 wake 函数实现，并且把data作为参数，对于Embassy，这里的data指针指向TaskHeader
    unsafe { (self.waker.vtable.wake)(self.waker.data) }
}
```

### wake_task

vtable的wake最终调用的都是wake_task，最终调用的[enqueue](#enqueue)。

```rust
unsafe fn wake(p: *const ()) {
    wake_task(TaskRef::from_ptr(p as *const TaskHeader))
}

unsafe fn wake(p: *const ()) {
    wake_task(TaskRef::from_ptr(p as *const TaskHeader))
}

pub fn wake_task(task: TaskRef) {
    let header = task.header();
    header.state.run_enqueue(|l| {        // 原子/临界区检查 STATE_RUN_QUEUED
        unsafe {
            let executor = header.executor.load(Ordering::Relaxed)
                              .as_ref().unwrap_unchecked();
            executor.enqueue(task, l);     // push 到 run_queue
        }
    });
}
```

## Timer

```rust
#[embassy_executor::task]
async fn blink(pin: Peri<'static, AnyPin>) {
    let mut led = Output::new(pin, Level::Low, OutputDrive::Standard);

    loop {
        // Timekeeping is globally available, no need to mess with hardware timers.
        led.set_high();
        Timer::after_millis(150).await;
        led.set_low();
        Timer::after_millis(150).await;
    }
}
```

```rust
pub struct Timer {
    expires_at: Instant,
    yielded_once: bool,
}

pub struct Instant {
    ticks: u64,
}

// 构造出一个Timer future，定时在expires_at = Instant::now() + duration.into(),
pub fn after_millis(millis: u64) -> Self {
    Self::after(Duration::from_millis(millis))
}

pub fn after(duration: impl Into<Duration>) -> Self {
    Self {
        expires_at: Instant::now() + duration.into(),
        yielded_once: false,
    }
}

impl Future for Timer {
    type Output = ();
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // 如果当前的时间大于expires_at了，就返回ready
        if self.yielded_once && self.expires_at <= Instant::now() {
            Poll::Ready(())
        } else {
            // 否则注册waker
            embassy_time_driver::schedule_wake(self.expires_at.as_ticks(), cx.waker());
            self.yielded_once = true;
            Poll::Pending
        }
    }
}
```

TimerDriver 是一个 全局静态变量 ，编译期就存在了。不同芯片用不同的 time driver：
```rust
time_driver_impl!(
    static TIME_DRIVER: TimerDriver = TimerDriver {
        periods: AtomicU32::new(0),              // 计数器溢出的 32bit 高位
        timekeeper: Mutex::new(RefCell::new(None)),  // 硬件 timekeeper 寄存器（初始化前是 None）
        alarm_timer: Mutex::new(RefCell::new(None)),  // 硬件 alarm 寄存器
        alarms: Mutex::new(AlarmState::new()),      // 最近闹钟时间戳
        queue: Mutex::new(RefCell::new(Queue::new())),  // waker 队列
    }
);
```


schedule_wake 是一个 函数指针，编译时根据芯片 HAL 的 time-driver feature 被启用，指向不同的实现。

```rust
impl Driver for TimerDriver {
    fn schedule_wake(&self, at: u64, waker: &core::task::Waker) {
        critical_section::with(|cs| {
            let mut queue = self.queue.borrow(cs).borrow_mut();
            if queue.schedule_wake(at, waker) {       // 把(waker, 到期时间)插入队列
                let mut next = queue.next_expiration(self.now());
                while !self.set_alarm(cs, next) {     // 编程硬件闹钟
                    next = queue.next_expiration(self.now());
                }
            }
        })
    }
```

TimerDriver的queue用于保存waker，按照类型分成Generic Queue和Integrated Queue，Generic Queue 是外挂 Vec<(at, Waker)> ，和 TaskHeader 没有直接关系； Integrated Queue 是把定时器队列的节点（ QueueItem ）直接嵌在TaskHeader.timer_queue_item 里。

Generic Queue类型是ConstGenericQueue。

```rust
pub struct ConstGenericQueue<const QUEUE_SIZE: usize> {
    queue: Vec<Timer, QUEUE_SIZE>,
}

    pub fn schedule_wake(&mut self, at: u64, waker: &Waker) -> bool {
        self.queue
            .iter_mut()
            .find(|timer| timer.waker.will_wake(waker)) // 先在队列里找：这个 waker 是否已经注册
            .map(|timer| {
                if timer.at > at {   // // 已存在，新到期时间更早 → 更新
                    timer.at = at;
                    true
                } else {
                    false
                }
            })
            .unwrap_or_else(|| { // // ② 没找到 → 这个 waker 不在队列里，新增
                let mut timer = Timer {
                    waker: waker.clone(),
                    at,
                };

                loop {
                    match self.queue.push(timer) {
                        Ok(()) => break,
                        Err(e) => timer = e,
                    }

                    self.queue.pop().unwrap().waker.wake();
                }

                true
            })
    }
```

```
中断触发
  │
  ▼
RTC1()                                    
  ▼
DRIVER.on_interrupt()                      
  │
  ▼
trigger_alarm(cs)                          
  │
  ▼
queue.next_expiration(now)                 
  │
  ▼
waker.wake()                               
  │
  ▼
wake_task(blink) → enqueue → pender.pend()
  │
  ▼
asm!("sev")                                
  │
  ▼
中断返回                                   
  │
  ▼
executor.poll() → dequeue_all → poll(blink) 
```

```rust
fn on_interrupt(&self) {
        let r = rtc();

        let n = TIME_DRIVER_CC_N;
        if r.events_compare(n).read() == 1 {
            r.events_compare(n).write_value(0);
            critical_section::with(|cs| {
                self.trigger_alarm(cs);
            });
        }
}

fn trigger_alarm(&self, cs: CriticalSection) {
    let n = TIME_DRIVER_CC_N;
    let r = rtc();

    let alarm = &self.alarms.borrow(cs);
    alarm.timestamp.set(u64::MAX);

    let mut next = self.queue.borrow(cs).borrow_mut().next_expiration(self.now());
    while !self.set_alarm(cs, next) {
        next = self.queue.borrow(cs).borrow_mut().next_expiration(self.now());
    }
}

    pub fn next_expiration(&mut self, now: u64) -> u64 {
        while i < self.queue.len() {
            let timer = &self.queue[i];
            if timer.at <= now {       // 遍历所有timer，对于到期的timer，获取waker，然后wake 任务
                let timer = self.queue.swap_remove(i);
                timer.waker.wake();
            } 
        }

        next_alarm
    }
```

## TaskStorage

TaskStorage 是存放一个任务的静态内存。TaskStorage存在TaskPool中，TaskPool在编译时就确定大小的TaskStorage数组。

```rust
pub struct TaskStorage<F: Future + 'static> {
    raw: TaskHeader,         // 元数据头（状态、队列节点、poll 函数指针...）
    future: UninitCell<F>,   // 实际的 Future（async fn 编译后的状态机）
}

pub(crate) struct TaskHeader {
    pub(crate) state: State,                          // 原子位: SPAWNED | RUN_QUEUED
    pub(crate) run_queue_item: RunQueueItem,          // 链表节点，用于挂到 RunQueue 上
    pub(crate) executor: AtomicPtr<SyncExecutor>,     // 指向所属 executor
    poll_fn: SyncUnsafeCell<Option<unsafe fn(TaskRef)>>, // 实际的 poll 函数指针
    pub(crate) timer_queue_item: TimerQueueItem,       // 集成定时器队列节点
    pub(crate) metadata: Metadata,                     // 任务名、优先级、deadline
}
```