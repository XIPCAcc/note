runtime初始化后，调用block on执行异步代码
```rust
rt.block_on(async {
        // 你的异步代码在这里
        tokio::task::spawn(async {
            println!("Hello, Tokio!");
        }).await.unwrap();
    });
```
一个async代码块会生成一个匿名的struct，并为这个struct实现future，async代码块的功能基本都在poll函数中实现。
```rust
// 内部async块也会生成一个Future
struct __InnerAsyncBlock$ {
    __state: u32,
    // 没有局部变量需要保存
}

impl Future for __InnerAsyncBlock$ {
    type Output = ();
    
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        match self.__state {
            0 => {
                println!("Hello, Tokio!");
                self.__state = 1;
                Poll::Ready(())
            }
            1 => Poll::Ready(()),  // 已完成状态
            _ => unreachable!(),
        }
    }
}
```
block_on()接受实现了future的struct结构
```rust
pub fn block_on<F: Future>(&self, future: F) -> F::Output {
        let fut_size = mem::size_of::<F>();
        // 通过future的大小决定是否要放到堆上
        if fut_size > BOX_FUTURE_THRESHOLD {
            self.block_on_inner(Box::pin(future), SpawnMeta::new_unnamed(fut_size))
        } else {
            self.block_on_inner(future, SpawnMeta::new_unnamed(fut_size))
        }
    }
```

```rust
tokio-1.48.0/src/runtime/runtime.rs

fn block_on_inner<F: Future>(&self, future: F, _meta: SpawnMeta<'_>) -> F::Output {
       
    let future = super::task::trace::Trace::root(future); // 启用任务调用栈跟踪

    let _enter = self.enter(); // 进入Tokio运行时上下文，确保异步操作（如spawn、sleep）能正确找到运行时

    match &self.scheduler {
        Scheduler::CurrentThread(exec) => exec.block_on(&self.handle.inner, future),
        #[cfg(feature = "rt-multi-thread")]
        Scheduler::MultiThread(exec) => exec.block_on(&self.handle.inner, future),
    }
}
```

```rust
tokio-1.48.0/src/runtime/scheduler/multi_thread/mod.rs
    /// Blocks the current thread waiting for the future to complete.
    /// The future will execute on the current thread, but all spawned tasks
    /// will be executed on the thread pool.
    pub(crate) fn block_on<F>(&self, handle: &scheduler::Handle, future: F) -> F::Output
    {
        crate::runtime::context::enter_runtime(handle, true, |blocking| {
            blocking.block_on(future).expect("failed to park thread")
        })
    }
```

enter_runtime()
```rust
tokio-1.48.0/src/runtime/context/runtime.rs

pub(crate) fn enter_runtime<F, R>(
    handle: &scheduler::Handle,  // 调度器句柄
    allow_block_in_place: bool,  // 是否允许阻塞操作
    f: F                         // 要执行的闭包函数
) -> R

    // CONTEXT: 线程局部变量，跟踪当前线程的运行时状态
    let maybe_guard = CONTEXT.with(|c| {
        if c.runtime.get().is_entered() { // 检查当前线程是否已经在运行时中，只要enum EnterRuntime不是 NotEntered说明正在运行，和allow_block_in_place的值没关系
        None // 如果已进入，返回 None（避免嵌套）
    } else {
        // Set the entered flag
        c.runtime.set(EnterRuntime::Entered {
            allow_block_in_place,
        });

        // Generate a new seed
        // 为什么要生成随机种子? 要确保任务调度的随机性,当多个工作线程同时尝试获取任务时，如果使用确定性算法产生羊群效应（Herd Behavior）
        // 同时检查同一个任务队列, 同时被同一个热点任务吸引, 导致某些线程忙死，某些线程闲死
        let rng_seed = handle.seed_generator().next_seed();

        // RAII (Resource Acquisition Is Initialization)​ 守卫对象
        Some(EnterRuntimeGuard {
                blocking: BlockingRegionGuard::new(),  // 阻塞区域管理 跟踪当前是否在阻塞操作中，防止死锁。
                handle: c.set_current(handle), // 调度器句柄管理   设置当前线程的调度器句柄，退出时自动恢复。
                old_seed, // 旧的随机种子 保存进入运行时前的随机种子，退出时恢复，确保随机性隔离。
            })

        // 如果生成maybe_guard成功，则调用传进来的闭包fun，并以guard.blocking作为参数
        if let Some(mut guard) = maybe_guard {
            return f(&mut guard.blocking);
        }
    }
```

enter_runtime()中最后调用闭包 f(&mut guard.blocking)
```rust
|blocking| {
    blocking.block_on(future).expect("failed to park thread")
}
```

这里进入闭包后会调用BlockingRegionGuard的block_on，future是aync代码块
```rust
BlockingRegionGuard {
    pub(crate) fn block_on<F>(&mut self, f: F) -> Result<F::Output, AccessError>
    {
        let mut park = CachedParkThread::new();
        park.block_on(f)
    }
```

然后调用CachedParkThread的block_on()，执行async main生成的future代码f.as_mut().poll(&mut cx)
```rust
pub(crate) struct CachedParkThread { // 缓存优化：重用 waker 和 parker，减少分配
    pub(crate) fn block_on<F: Future>(&mut self, f: F) -> Result<F::Output, AccessError> {
         // 1. 创建唤醒器 - 让异步操作能唤醒这个线程
        let waker = self.waker()?;
        // 2. 创建执行上下文 - 包装waker供future使用
        let mut cx = Context::from_waker(&waker);
        // 3. Pin住future - 防止future在内存中移动
        pin!(f);

        loop {
            // 在预算限制下poll future
            if let Ready(v) = crate::task::coop::budget(|| f.as_mut().poll(&mut cx)) {
                return Ok(v);
            }
            // future未就绪，挂起线程等待事件
            self.park();
        }
    }
```

async main的代码poll返回not ready后，block_on()调用park()进入到阻塞状态
```rust
tokio-1.48.0/src/runtime/park.rs

    pub(crate) fn park(&mut self) {
        self.with_current(|park_thread| park_thread.inner.park())
            .unwrap();
    }

const EMPTY: usize = 0;    // 初始/空闲/可通知状态
const PARKED: usize = 1;   // 线程正在休眠等待
const NOTIFIED: usize = 2; // 有通知待处理

compare_exchange是原子比较并交换操作，也称为 CAS（Compare-And-Swap）

初始:    EMPTY
park():  EMPTY → PARKED    (获取锁后设置)
unpark(): PARKED → NOTIFIED (唤醒)
被唤醒:  NOTIFIED → EMPTY   (清除通知)

impl Inner {
    fn park(&self) {
        // 这里是处理竞态条件，如果设置为PARKED的过程中出现别的线程又把当前设置为NOTIFIED，就清空
        if self
            .state
            .compare_exchange(NOTIFIED, EMPTY, SeqCst, SeqCst) // 如果当前是 NOTIFIED，改为 EMPTY
            .is_ok()
        {
            return;
        }

        // Otherwise we need to coordinate going to sleep
        let mut m = self.mutex.lock();

        // 将当前状态设置成PARKED
        match self.state.compare_exchange(EMPTY, PARKED, SeqCst, SeqCst) {
            Ok(_) => {}  // 这里结束后进入loop处理竞态
            Err(NOTIFIED) => {
                // We must read here, even though we know it will be `NOTIFIED`.
                // This is because `unpark` may have been called again since we read
                // `NOTIFIED` in the `compare_exchange` above. We must perform an
                // acquire operation that synchronizes with that `unpark` to observe
                // any writes it made before the call to unpark. To do that we must
                // read from the write it made to `state`.
                // 处理竞态，如果返回NOTIFIED，设置成EMPTY
                let old = self.state.swap(EMPTY, SeqCst);
                debug_assert_eq!(old, NOTIFIED, "park state changed unexpectedly");

                return;
            }
            Err(actual) => panic!("inconsistent park state; actual = {actual}"),
        }

        loop {
            // 进入阻塞等待
            m = self.condvar.wait(m).unwrap();

            // 被唤醒后清空状态
            if self
                .state
                .compare_exchange(NOTIFIED, EMPTY, SeqCst, SeqCst)
                .is_ok()
            {
                // got a notification
                return;
            }

            // spurious wakeup, go back to sleep
        }
    }
```

关于阻塞等待
```rust
tokio-1.48.0/src/loom/std/parking_lot.rs

   pub(crate) fn wait<'a, T>(
        &self,
        mut guard: MutexGuard<'a, T>,
    ) -> LockResult<MutexGuard<'a, T>> {
        self.1.wait(&mut guard.1);
        Ok(guard)
    }
```

最终调用 parking_lot_core-0.9.12/src/thread_parker/linux.rs futex_wait()
```rust
impl ThreadParker {
    #[inline]
    fn futex_wait(&self, ts: Option<libc::timespec>) {
        let ts_ptr = ts
            .as_ref()
            .map(|ts_ref| ts_ref as *const _)
            .unwrap_or(ptr::null());
        let r = unsafe {
            libc::syscall(
                libc::SYS_futex,
                &self.futex,
                libc::FUTEX_WAIT | libc::FUTEX_PRIVATE_FLAG,
                1,
                ts_ptr,
            )
        };
```

使用 Linux futex 系统调用进行线程等待
通过 Linux 的 futex（快速用户空间互斥锁）系统调用实现的一个等待操作
```c
#include <stdio.h>
#include <unistd.h>
#include <sys/syscall.h>
#include <linux/futex.h>

int val = 1;  // futex变量

int main() {
    // 等待线程
    if (fork() == 0) {  // 子进程
        printf("子进程: 开始等待\n");
        
        // 等待 val 变成 0
        syscall(SYS_futex, &val, FUTEX_WAIT, 1, 0, 0, 0);
        
        printf("子进程: 被唤醒\n");
        return 0;
    }
    
    // 父进程
    sleep(1);
    printf("父进程: 修改值并唤醒\n");
    
    val = 0;  // 修改值
    
    // 唤醒子进程
    syscall(SYS_futex, &val, FUTEX_WAKE, 1, 0, 0, 0);
    
    wait(NULL);  // 等待子进程结束
    return 0;
}
```

# tokio::join!(task1, task2)

```rust
tokio\src\macros\join.rs

macro_rules! join {
    ($($t:tt)*) => {{
        // 将每个 Future 包装为 MaybeDone，然后通过poll_fn 轮询等待结果
        let mut futures = ( $( $crate::macros::support::maybe_done($e), )* );
        $crate::macros::support::poll_fn(move |cx| {
    }}
}
```


allow_block_in_place允许在异步上下文中执行阻塞的同步代码，而不会阻塞整个运行时。
allow_block_in_place=true时允许在block on中调用allow_block_in_place()函数，否则不允许
当 allow_block_in_place = true时：允许阻塞任务在工作线程上运行，阻塞任务代码会被移动到专门的阻塞线程
当 allow_block_in_place = false时：阻塞任务在工作线程(block on)上运行会panic
