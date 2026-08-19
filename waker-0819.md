# Waker的工作模式

任务或者协程的唤醒分成两个阶段，第一个阶段是修改任务的状态，第二个阶段是将任务放回就绪队列。

在大部分的异步运行时中，两个阶段都在waker.wake()完成。

## Embassy

Embassy的Waker最终调用的是wake_task()
```rust
embassy-executor/src/raw/waker.rs

static VTABLE: RawWakerVTable = RawWakerVTable::new(clone, wake, wake, drop);

pub fn wake(self) {
    let data = self.waker.data();        //  从 self 取出 data（data是指向任务的指针）
    let vtable = self.waker.vtable();
    mem::forget(self);
    (vtable.wake)(data);                 //  把 data 传给 vtable 的 wake 函数
}
unsafe fn wake(p: *const ()) {
    wake_task(TaskRef::from_ptr(p as *const TaskHeader))

```

wake_task 获取任务的header状态，然后修改state并将任务放回就绪队列。

```rust
embassy-executor/src/raw/mod.rs

pub fn wake_task(task: TaskRef) {
    let header = task.header();
    header.state.run_enqueue(|l| {          
        unsafe {
            let executor = header.executor.load(Ordering::Relaxed).as_ref().unwrap_unchecked();
            executor.enqueue(task, l);      // 第二阶段：放入就绪队列
        }
    });
}

pub fn run_enqueue(&self, f: impl FnOnce(Token)) {
    let prev = self.state.fetch_or(STATE_RUN_QUEUED, Ordering::AcqRel);  // run_enqueue先原子修改任务的状态
    if prev & STATE_RUN_QUEUED == 0 {
        locked(f);                                // 执行闭包enqueue
    }
}

pub(crate) unsafe fn enqueue(&self, task: TaskRef, _tok: super::state::Token) -> bool {
    self.stack.push_was_empty(         // 原子入队保证了能够能够在中断上下文修改队列 TransferStack（lock-free 的 Treiber Stack （无锁并发栈））
        task,
        #[cfg(not(target_has_atomic = "ptr"))]
        _tok,
    )
}
```

## tokio

tokio 里每个任务的Header包含状态信息。Waker 的数据指针就是这个任务Header的首地址,即 &Header。

```rust
#[repr(C)]
pub(crate) struct Header {
    /// Task state.
    pub(super) state: State,

    /// Pointer to next task, used with the injection queue.
    pub(super) queue_next: UnsafeCell<Option<NonNull<Header>>>,

    /// Table of function pointers for executing actions on the task.
    pub(super) vtable: &'static Vtable,

}

static WAKER_VTABLE: RawWakerVTable =
    RawWakerVTable::new(clone_waker, wake_by_val, wake_by_ref, drop_waker);
```

waker.wake()最终调用wake_by_ref()

```rust
    pub(super) fn wake_by_ref(&self) {
        use super::state::TransitionToNotifiedByRef;

        match self.state().transition_to_notified_by_ref() { // transition_to_notified_by_ref转换任务状态
            TransitionToNotifiedByRef::Submit => {           // 确认要转换则调用schedule 景任务放回就绪队列
                self.schedule();
            }
            TransitionToNotifiedByRef::DoNothing => {}
        }

```

waker创建和使用

```rust
let header_ptr = self.header_ptr();                  // 任务内存块首地址 = &Header
let waker_ref = waker_ref::<S>(&header_ptr);         // ← 用 Header 构造 waker
let cx = Context::from_waker(&waker_ref);            // ← 构造 Context
let res = poll_future(self.core(), cx);              // 把 cx 传给用户 future


// 使用waker的例子
impl Future for MyWait {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        let mut shared = self.shared.lock().unwrap();

        if shared.done {
            return Poll::Ready(());
        }

        // 对waker引用计数 +1,得到一个能跨 poll 存活的独立 Waker
        shared.waker = Some(cx.waker().clone());

        Poll::Pending                         
    }
}

tokio/src/runtime/io/scheduled_io.rs
pub(super) fn wake(&self, ready: Ready) {
    let mut wakers = WakeList::new();
    let mut waiters = self.waiters.lock();

    if ready.is_readable() {
        if let Some(waker) = waiters.reader.take() {
            wakers.push(waker);
        }
    }
    wakers.wake_all();
}
```

## Waker的调用时机

抽象来说，Waker的调用时机都应该位于Reactor循环之中。对于Embassy而言，就是每个中断的中断上下文；对于tokio就是Reactor读取外部epoll事件的turn()函数。
