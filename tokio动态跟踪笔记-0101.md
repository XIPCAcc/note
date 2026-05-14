```rust
 tokio::spawn(async {
        println!("[任务1] 在 {:?} 上启动", std::thread::current().id());
        
        for i in 1..=2 {
            println!("[任务1] 步骤 {}", i);
            tokio::time::sleep(Duration::from_millis(100)).await;
        }
    });
```

```rust
tokio-1.48.0/src/task/spawn.rs

    pub fn spawn<F>(future: F) -> JoinHandle<F::Output>
    {
        let fut_size = std::mem::size_of::<F>();
        if fut_size > BOX_FUTURE_THRESHOLD {
            spawn_inner(Box::pin(future), SpawnMeta::new_unnamed(fut_size))
        } else {
            spawn_inner(future, SpawnMeta::new_unnamed(fut_size))
        }
    }


    pub(super) fn spawn_inner<T>(future: T, meta: SpawnMeta<'_>) -> JoinHandle<T::Output>
    {
        let future = task::trace::Trace::root(future);
        let id = task::Id::next();
        // tokio\src\util\trace.rs 中的task函数，用 tracing::Instrument包装的 Future，为任务添加 tracing span，但不改变任务执行逻辑
        // tracing span是 tracing库的核心概念，表示一个具有开始和结束的代码执行区间。它是分布式追踪和结构化日志记录的基础单元
        let task = crate::util::trace::task(future, "task", meta, id.as_u64());

        match context::with_current(|handle| handle.spawn(task, id, meta.spawned_at)) {
            Ok(join_handle) => join_handle,
            Err(e) => panic!("{}", e),
        }
    }
```

```rust
tokio-1.48.0/src/runtime/context/current.rs
    pub(crate) fn with_current<F, R>(f: F) -> Result<R, TryCurrentError>
    {
        // try_with通过标准库 thread_local!宏生成的 LocalKey的 try_with方法获取线程局部静态变量CONTEXT
        // 如果获取成功，访问CONTEXT.current.handle。如果handle获取成功，将其作为参数调用函数f
        // map(f)是 Rust 中 Option类型的方法调用，它会对 Option值进行转换
        match CONTEXT.try_with(|ctx| ctx.current.handle.borrow().as_ref().map(f)) {
            Ok(Some(ret)) => Ok(ret),
            Ok(None) => Err(TryCurrentError::new_no_context()),
            Err(_access_error) => Err(TryCurrentError::new_thread_local_destroyed()),
        }
    }
```
    其中执行的f是下面的闭包
    |handle| handle.spawn(task, id, meta.spawned_at)
```rust
tokio-1.48.0/src/runtime/scheduler/mod.rs
    pub(crate) fn spawn<F>(&self, future: F, id: Id, spawned_at: SpawnLocation) -> JoinHandle<F::Output>
    {
        match self {
            Handle::CurrentThread(h) => current_thread::Handle::spawn(h, future, id, spawned_at),

            #[cfg(feature = "rt-multi-thread")]
            Handle::MultiThread(h) => multi_thread::Handle::spawn(h, future, id, spawned_at),
        }
    }
```

```rust
tokio-1.48.0/src/runtime/scheduler/multi_thread/handle.rs

impl Handle {
    /// Spawns a future onto the thread pool
    pub(crate) fn spawn<F>( ) -> JoinHandle<F::Output>
    {
        Self::bind_new_task(me, future, id, spawned_at)
    }

    pub(super) fn bind_new_task<T>( ) -> JoinHandle<T::Output>
    {
        // 生成任务，绑定运行时，返回handle
        // notified 等同于可运行的Task
        let (handle, notified) = me.shared.owned.bind(future, me.clone(), id, spawned_at);

        // 检查是否有设置回调函数，如果有则执行任务创建时的回调 task_spawn_callback
        // 回调函数的用法见TaskHooks 用法
        me.task_hooks.spawn(&TaskMeta {
            id,
            spawned_at,
            _phantom: Default::default(),
        });

        // 任务调度入口函数，专门用于将新创建的任务快速放入调度系统执行
        me.schedule_option_task_without_yield(notified);

        handle
    }
}
```

```rust
tokio-1.48.0/src/runtime/task/list.rs
    pub(crate) fn bind<T>( ) -> (JoinHandle<T::Output>, Option<Notified<S>>)
    {
        // 生成Task, Notified, JoinHandle
        // Task存储用户的 Future, 维护任务状态（运行、完成、取消等）, 保存执行结果
        // Notified 是任务的调度单元 进入调度器的就绪队列 轻量级，可在线程间传递
        // 定义在 tokio\src\runtime\task\mod.rs 区别于tokio\src\sync\notify.rs
        // JoinHandle - 任务的控制接口 获取任务执行结果 查询任务状态 等待任务完成
        let (task, notified, join) = super::new_task(task, scheduler, id, spawned_at);
        let notified = unsafe { self.bind_inner(task, notified) };
        (join, notified)
    }

    unsafe fn bind_inner(&self, task: Task<S>, notified: Notified<S>) -> Option<Notified<S>>
    {
        unsafe {
            // 通过绑定后：Notified任务"属于"这个运行时
            task.header().set_owner_id(self.id);
        }

        // 采用分片列表技术 分片列表（Sharded List）​ 是一种高性能并发数据结构，专门用于减少多线程环境下的锁竞争
        // 分片是将一个大集合分割成多个小部分（分片），每个分片可以独立加锁的技术
        let shard = self.list.lock_shard(&task);
        if self.closed.load(Ordering::Acquire) {
            drop(shard);
            task.shutdown();
            return None;
        }
        // 将任务放入分片列表
        shard.push(task);
        Some(notified)
    }
```

```rust
    pub(super) fn schedule_option_task_without_yield(&self, task: Option<Notified>) {
        if let Some(task) = task {
            // is_yield = false 说明只是把任务放置到就绪队列，不需要yield
            self.schedule_task(task, false);
        }
    }

tokio-1.48.0/src/runtime/scheduler/multi_thread/worker.rs
    impl Handle {
    // Tokio 多线程调度器的核心调度函数，负责决定任务如何分配到不同的队列和执行路径
    pub(super) fn schedule_task(&self, task: Notified, is_yield: bool) {
        with_current(|maybe_cx| {
            // 1. 检查是否有当前调度上下文
            if let Some(cx) = maybe_cx {
                // 2. 检查是否属于当前调度器
                // Make sure the task is part of the **current** scheduler.
                if self.ptr_eq(&cx.worker.handle) {
                    // And the current thread still holds a core
                    // 3. 检查当前线程是否持有core
                    if let Some(core) = cx.core.borrow_mut().as_mut() {
                        // 4. 本地调度
                        self.schedule_local(core, task, is_yield);
                        return;
                    }
                }
            }

            // Otherwise, use the inject queue.
            // 5. 否则，远程调度
            self.push_remote_task(task);
            self.notify_parked_remote();
        });
    }

    // 通过with_current 调用执行了上面的闭包
    fn with_current<R>(f: impl FnOnce(Option<&Context>) -> R) -> R {
        context::with_scheduler(|ctx| match ctx {
            Some(MultiThread(ctx)) => f(Some(ctx)),
            _ => f(None),
        })
    }

    pub(super) fn with_scheduler<R>(f: impl FnOnce(Option<&scheduler::Context>) -> R) -> R {
        // 封装为Some，避免处理空值的情况, 同时避免编译器无法静态分析f 作为FnOnce 在不同分支中只能单次调用的问题
        let mut f = Some(f);
        CONTEXT.try_with(|c| {
            // 再从Some中取出
            let f = f.take().unwrap();
            if matches!(c.runtime.get(), EnterRuntime::Entered { .. }) {
                c.scheduler.with(f)
            } else {
                f(None)
            }
        })
    }

    pub(super) fn with<F, R>(&self, f: F) -> R
    {
        let val = self.inner.get();

        if val.is_null() {
            f(None)
        } else {
            unsafe { f(Some(&*val)) }
        }
    }
```

```rust
tokio-1.48.0/src/runtime/scheduler/multi_thread/worker.rs

// Tokio 调度器的本地调度核心函数，实现了智能的任务排队策略
fn schedule_local(&self, core: &mut Core, task: Notified, is_yield: bool) {
        core.stats.inc_local_schedule_count();// 记录本地调度次数，用于性能监控

        // 情况A：yield 或 LIFO 禁用 → 普通队尾
        // 任务进入队列尾部，公平调度
        let should_notify = if is_yield || !core.lifo_enabled {
            core.run_queue
                .push_back_or_overflow(task, self, &mut core.stats);
            true
        } else {
            // Push to the LIFO slot
            let prev = core.lifo_slot.take(); // 取出旧任务
            let ret = prev.is_some();// 记录是否有旧任务

            if let Some(prev) = prev {
                // 旧任务放入普通队列
                core.run_queue
                    .push_back_or_overflow(prev, self, &mut core.stats);
            }

            // 所谓LIFO只有一个slot，其余任务都是按照FIFO
            core.lifo_slot = Some(task);// 新任务放入 LIFO 槽

            ret
        };

        // Only notify if not currently parked. If `park` is `None`, then the
        // scheduling is from a resource driver. As notifications often come in
        // batches, the notification is delayed until the park is complete.
        if should_notify && core.park.is_some() {
            self.notify_parked_local();
        }
    }
```

传统 FIFO 队列：
任务入队: A → B → C → D ：A 先入队，然后是 B，然后是 C，最后是 D
任务出队: A ← B ← C ← D ：先出队 A，然后 B，然后 C，最后 D
    ↓
可能有上下文切换开销

LIFO 优化：
最新任务放在特殊槽位，下次优先执行
任务入队: [LIFO槽: D] 队列: A → B → C
任务出队: D ← A ← B ← C
    ↓
提高缓存局部性，减少切换
