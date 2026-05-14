关于waker.wake() 无法在调试时跳转的问题

直接全局搜索 fn wake() 逐一查看


Waker 使用的使用标准库的Waker。

精简版源码
```rust
std::task::Waker

// 来自 Rust 标准库：library/core/src/task/wake.rs
pub struct Waker {
    waker: RawWaker,
}

// std::task 模块
pub mod task {
    pub struct Waker {
        // 私有字段，无法直接访问
        waker: RawWaker,
    }
    
    impl Waker {
        /// 创建一个新的 Waker
        pub unsafe fn from_raw(waker: RawWaker) -> Waker {
            Waker { waker }
        }
        
        /// 唤醒关联的任务
        pub fn wake(self) {
            // 调用虚函数表中的 wake 函数
            let wake = self.waker.vtable.wake;
            let data = self.waker.data;
            
            // 确保在调用前不提前释放
            mem::forget(self);
            
            // 调用具体的实现
            unsafe { (wake)(data) };
        }
        
        /// 不消耗 Waker 的唤醒
        pub fn wake_by_ref(&self) {
            let wake = self.waker.vtable.wake_by_ref;
            let data = self.waker.data;
            
            unsafe { (wake)(data) };
        }
        
        /// 检查两个 Waker 是否"唤醒同一个任务"
        pub fn will_wake(&self, other: &Waker) -> bool {
            self.waker.data == other.waker.data && 
            ptr::eq(self.waker.vtable, other.waker.vtable)
        }
    }
}
```

Waker中就包含了一个RawWaker

```rust
// 原始唤醒器
pub struct RawWaker {
    /// 指向任务特定数据的指针
    pub data: *const (),
    
    /// 虚函数表
    pub vtable: &'static RawWakerVTable,
}

/// 虚函数表
pub struct RawWakerVTable {
    /// 克隆 RawWaker
    pub clone: unsafe fn(*const ()) -> RawWaker,
    
    /// 唤醒任务（消耗 Waker）
    pub wake: unsafe fn(*const ()),
    
    /// 唤醒任务（不消耗 Waker）
    pub wake_by_ref: unsafe fn(*const ()),
    
    /// 释放资源
    pub drop: unsafe fn(*const ()),
}
```

对于一般平台如果要创建RawWaker
```rust
// 创建 Waker
let task = Arc::new(Task::new());
let raw_waker = RawWaker::new(
    Arc::into_raw(task) as *const (),   // 重要的是这里，把对应的数据结构转换成指针方便指向该数据结构
    &VTABLE,          // 自定义和实现waker()函数等，创建出虚函数表
);
let waker = unsafe { Waker::from_raw(raw_waker) };
```

tokio代码对Waker的唯一改动是为Waker实现了外部自定义的WakerRef trait

```rust

trait WakerRef {
    fn wake(self);
    fn into_waker(self) -> Waker;
}

impl WakerRef for Waker {
    fn wake(self) {
        self.wake();
    }

    fn into_waker(self) -> Waker {
        self
    }
}
```

可以通过 tokio-master\tokio-test\src\task.rs 学习tokio如何使用标准库的Waker的

## AtomicWaker

tokio\src\sync\task\atomic_waker.rs 中定义的AtomicWaker是一个用于任务唤醒的同步原语
AtomicWaker会协调并发的唤醒操作，消费者可能会"唤醒"底层任务。
消费者应在检查计算结果之前调用 register，生产者应在产生计算结果后调用wake（这与通常的 thread::park模式不同）。


```rust
pub(crate) struct AtomicWaker {
    state: AtomicUsize,
    waker: UnsafeCell<Option<Waker>>,
}
```

可以查看其注释获取使用方法，以下是简要描述
AtomicWaker是一个多消费者、单生产者的传输单元。这个单元存储由 register调用产生的 Waker值，多个线程可以通过调用 wake来竞争获取这个 waker。
实现机制

实现使用单个 AtomicUsize值来协调对 Waker单元的访问。有两个独立操作的位，分别用 REGISTERING和 WAKING表示。

REGISTERING位：当生产者进入临界区时设置

WAKING位：当消费者进入临界区时设置

WAITING：表示两个位都没有设置

线程通过将状态从 WAITING转换为 REGISTERING或 WAKING来获得对 waker 单元的独占锁，具体取决于线程希望执行的操作。当进行此转换时，保证没有其他线程会访问 waker 单元。

注册流程（Registering）

在调用 register时，尝试将状态从 WAITING转换为 REGISTERING。如果成功，调用者获得对 waker 单元的锁。

如果获得锁，线程将 waker 单元设置为参数提供的 waker。然后尝试将状态从 REGISTERING转换回 WAITING。

如果此转换成功，则注册过程完成，下一次 wake调用将看到这个 waker。

如果转换失败，则表示有一个并发的 wake调用无法访问 waker 单元（因为注册线程持有锁）。为处理这种情况，注册线程从单元中移除刚刚设置的 waker 并调用其 wake方法。这个 wake 调用代表另一个线程（设置了 WAKING位）的唤醒尝试。然后将状态从 REGISTERING | WAKING转换回 WAITING。此转换必须成功，因为此时状态不能被其他线程转换。

唤醒流程（Waking）

在调用 wake时，尝试将状态从 WAITING转换为 WAKING。如果成功，调用者获得对 waker 单元的锁。

如果获得锁，线程取得 waker 单元中当前值的所有权，并调用其 wake方法。然后将状态转换回 WAITING。此转换必须成功，因为此时状态不能被其他线程转换。

如果线程无法获得锁，WAKING位仍然被设置。这是因为要么当前线程设置了它但先前值包含 REGISTERING位，要么并发线程在 WAKING临界区内。无论哪种情况，都必须不采取任何操作。

如果当前线程是唯一的并发 wake调用，而另一个线程在 register临界区内，当另一个线程退出​ register临界区时，它将观察到 WAKING位并自己处理 waker。

如果另一个线程在 waker临界区内，那么它将处理唤醒调用者任务。

// take_waker() 尝试获取旧值，并将旧值和WAKING 做或运算存回
self.state.fetch_or(WAKING, AcqRel)

# tokio 中实现了Wake trait的数据结构

但是大多数数据结构都是通过自定义的Wake trait 实现wake()和wake_by_ref()
```rust
/// Simplified waking interface based on Arcs.
pub(crate) trait Wake: Send + Sync + Sized + 'static {
    /// Wake by value.
    fn wake(arc_self: Arc<Self>);

    /// Wake by reference.
    fn wake_by_ref(arc_self: &Arc<Self>);
}
```
# Handle

```rust
tokio\src\runtime\scheduler\current_thread\mod.rs

impl Wake for Handle {
    fn wake(arc_self: Arc<Self>) {
        Wake::wake_by_ref(&arc_self);
    }

    /// Wake by reference
    fn wake_by_ref(arc_self: &Arc<Self>) {
        arc_self.shared.woken.store(true, Release);
        arc_self.driver.unpark();
    }
}
```

可以看到Handle的wake()其实是调用了driver.unpark()，即runtime::driver::Handle::unpark()

即
```rust
tokio\src\runtime\driver.rs

impl Handle {
    pub(crate) fn unpark(&self) {
        #[cfg(feature = "time")]
        if let Some(handle) = &self.time {
            handle.unpark();
        }

        self.io.unpark();
    }
```

## ListEntry

```rust
tokio\src\util\idle_notified_set.rs

struct ListEntry<T> {
    /// Pointer to the shared `Lists` struct.
    parent: Arc<Lists<T>>,
}

impl<T: 'static> Wake for ListEntry<T> {
    fn wake_by_ref(me: &Arc<Self>) {
        let mut lock = me.parent.lock();

        // Safety: We are holding the lock and we will update the lists to
        // maintain invariants.
        let old_my_list = me.my_list.with_mut(|ptr| unsafe {
            let old_my_list = *ptr;
            if old_my_list == List::Idle {
                *ptr = List::Notified;
            }
            old_my_list
        });

        if old_my_list == List::Idle {
            // We move ourself to the notified list.
            let me = unsafe {
                // Safety: We just checked that we are in this particular list.
                lock.idle.remove(ListEntry::as_raw(me)).unwrap()
            };
            lock.notified.push_front(me);

            if let Some(waker) = lock.waker.take() {
                drop(lock);
                waker.wake();
            }
        }
    }

    fn wake(me: Arc<Self>) {
        Self::wake_by_ref(&me);
    }
}
```

## ScheduledIo
ScheduledIo是 Tokio 中表示可等待 I/O 资源的中心数据结构：
跟踪 I/O 资源（如 TCP 流、UDP socket）的就绪状态
```rust
tokio\src\runtime\io\scheduled_io.rs

struct Waiters {
    /// List of all current waiters.
    list: WaitList,

    /// Waker used for `AsyncRead`.
    reader: Option<Waker>,

    /// Waker used for `AsyncWrite`.
    writer: Option<Waker>,
}

pub(crate) struct ScheduledIo {
    pub(super) linked_list_pointers: UnsafeCell<linked_list::Pointers<Self>>,

    /// Packs the resource's readiness and I/O driver latest tick.
    readiness: AtomicUsize,

    waiters: Mutex<Waiters>,
}

type WaitList = LinkedList<Waiter, <Waiter as linked_list::Link>::Target>;


pub(super) fn wake(&self, ready: Ready) {
        let mut wakers = WakeList::new();

        let mut waiters = self.waiters.lock();

        // check for AsyncRead slot
        if ready.is_readable() {
            if let Some(waker) = waiters.reader.take() {
                wakers.push(waker);
            }
        }

        // check for AsyncWrite slot
        if ready.is_writable() {
            if let Some(waker) = waiters.writer.take() {
                wakers.push(waker);
            }
        }

        'outer: loop {
            let mut iter = waiters.list.drain_filter(|w| ready.satisfies(w.interest));

            while wakers.can_push() {
                match iter.next() {
                    Some(waiter) => {
                        let waiter = unsafe { &mut *waiter.as_ptr() };

                        if let Some(waker) = waiter.waker.take() {
                            waiter.is_ready = true;
                            wakers.push(waker);
                        }
                    }
                    None => {
                        break 'outer;
                    }
                }
            }

            drop(waiters);

            wakers.wake_all();

            // Acquire the lock again.
            waiters = self.waiters.lock();
        }

        // Release the lock before notifying
        drop(waiters);

        wakers.wake_all();
    }
```

## Defer

Defer 没有严格意义上实现Wake trait，只是单独实现了一个wake()

延迟唤醒器管理器
作用：存储需要在稍后唤醒的 Waker
// 在某些情况下，不能立即调用 waker.wake()：
// 1. 持有锁时（可能死锁）
// 2. 在信号处理程序中
// 3. 在中断上下文中
// 4. 在某些关键代码段中

// 解决方案：先"记住"要唤醒谁，稍后再唤醒

// 多个事件可能同时需要唤醒同一个任务
// 使用 Defer 可以：
// 1. 收集所有需要唤醒的 waker
// 2. 一次性唤醒
// 避免重复唤醒同一任务

```rust
tokio\src\runtime\scheduler\defer.rs

pub(crate) struct Defer {
    deferred: RefCell<Vec<Waker>>,
}

impl Defer {
    pub(crate) fn defer(&self, waker: &Waker) {
        let mut deferred = self.deferred.borrow_mut();

        // If the same task adds itself a bunch of times, then only add it once.
        if let Some(last) = deferred.last() {
            if last.will_wake(waker) {
                return;
            }
        }

        deferred.push(waker.clone());
    }

    pub(crate) fn is_empty(&self) -> bool {
        self.deferred.borrow().is_empty()
    }

    pub(crate) fn wake(&self) {
        while let Some(waker) = self.deferred.borrow_mut().pop() {
            waker.wake();
        }
    }
}
```

## runtime::time_lt::Handle

计时器条目的句柄，用于管理计时器的生命周期和状态
register_waker()- 注册新唤醒器
```rust
pub(crate) struct Entry {  
    ......
    state: Mutex<State>,
}

struct State {
    ......
    woken_up: bool,
    waker: Option<Waker>,
}

pub(crate) struct Handle {
    ......
    pub(crate) entry: Arc<Entry>,
}

impl Handle {
    /// Wake the entry if it is already in the pending queue of the timer wheel.
    pub(crate) fn wake(&self) {
        let mut lock = self.entry.state.lock();

        if !lock.cancelled {
            lock.woken_up = true;
            if let Some(waker) = lock.waker.take() {
                // unlock before calling waker
                drop(lock);
                waker.wake();
            }
        }
    }
```

# tokio中实现了RawWaker转换的数据结构

## UnparkThread

```rust
/// Unblocks a thread that was blocked by `ParkThread`.
pub(crate) struct UnparkThread {
    inner: Arc<Inner>, // Rust 中线程安全的引用计数智能指针，和ParkThread共用一个Inner
}

// TODO: Is this really a unsafe function?
unsafe fn unparker_to_raw_waker(unparker: Arc<Inner>) -> RawWaker {
    RawWaker::new(
        Inner::into_raw(unparker),
        &RawWakerVTable::new(clone, wake, wake_by_ref, drop_waker),
    )
}
```

其实是将其中的Inner字段实现被转换成RawWaker的data

```rust
struct Inner {
    state: AtomicUsize,
    mutex: Mutex<()>,
    condvar: Condvar,
}

impl Inner {
    fn into_raw(this: Arc<Inner>) -> *const () {
        Arc::into_raw(this) as *const ()
    }
}
```

具体存在虚函数表中的wake()

```rust
unsafe fn wake(raw: *const ()) {
    let unparker = unsafe { Inner::from_raw(raw) };
    unparker.unpark();
}

调用了Inner的unpark()
tokio\src\runtime\park.rs

fn unpark(&self) {
        // To ensure the unparked thread will observe any writes we made before
        // this call, we must perform a release operation that `park` can
        // synchronize with. To do that we must write `NOTIFIED` even if `state`
        // is already `NOTIFIED`. That is why this must be a swap rather than a
        // compare-and-swap that returns if it reads `NOTIFIED` on failure.
        match self.state.swap(NOTIFIED, SeqCst) {
            EMPTY => return,    // no one was waiting
            NOTIFIED => return, // already unparked
            PARKED => {}        // gotta go wake someone up
            _ => panic!("inconsistent state in unpark"),
        }

        // There is a period between when the parked thread sets `state` to
        // `PARKED` (or last checked `state` in the case of a spurious wake
        // up) and when it actually waits on `cvar`. If we were to notify
        // during this period it would be ignored and then when the parked
        // thread went to sleep it would never wake up. Fortunately, it has
        // `lock` locked at this stage so we can acquire `lock` to wait until
        // it is ready to receive the notification.
        //
        // Releasing `lock` before the call to `notify_one` means that when the
        // parked thread wakes it doesn't get woken only to have to wait for us
        // to release `lock`.
        drop(self.mutex.lock());

        self.condvar.notify_one();
    }
```

## CachedParkThread

每个线程第一次访问 CURRENT_PARKER时，会创建并初始化一个 ParkThread实例，存在TLS中。之后的所有访问都会返回同一个实例。

一个线程只需要一个Park和unPark，所以就一直用这个

```rust
tokio_thread_local! {
    static CURRENT_PARKER: ParkThread = ParkThread::new();
}
```

CachedParkThread 就是专门设计用来访问TLS中的CURRENT_PARKER
```rust
// CachedParkThread 是访问 TLS 单例的门面
struct CachedParkThread {
    // 不存储数据，只是标记类型
    _anchor: PhantomData<Rc<()>>,
}

    fn unpark(&self) -> Result<UnparkThread, AccessError> {
        self.with_current(ParkThread::unpark)
    }

    pub(crate) fn park(&mut self) {
        self.with_current(|park_thread| park_thread.inner.park())
            .unwrap();
    }

    
    pub(crate) fn waker(&self) -> Result<Waker, AccessError> {
        self.unpark().map(UnparkThread::into_waker)
    }
```

关于ParkThread和UnparkThread的关系

ParkThread和UnparkThread共享Inner

```rust
pub(crate) fn unpark(&self) -> UnparkThread {
        let inner = self.inner.clone();
        UnparkThread { inner }
    }
```

为什么从 park创建 unpark
// 从"阻塞的能力"创建"唤醒的能力"
// 因为：要唤醒，必须先知道要唤醒谁
// 而 ParkThread 知道"我是谁"

// 反之不成立：不能从 unpark 创建 park
// 因为：知道如何唤醒别人，不代表自己能被阻塞
// 通常流程：
// 1. 线程创建 ParkThread（拥有阻塞权）
// 2. 导出 UnparkThread（分享唤醒权）
// 3. 发送 UnparkThread 到其他地方
// 4. 其他地方可以唤醒这个线程

用法
```rust
// tokio/src/runtime/driver.rs
struct Driver {
    park: CachedParkThread,  // 包装 ParkThread
    unpark: UnparkThread,    // 用于唤醒
}

impl Driver {
    fn park(&mut self) {
        self.park.park();
    }
    
    fn unpark(&self) {
        self.unpark.unpark();
    }
}
```

