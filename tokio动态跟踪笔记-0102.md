main函数中的tokio::spawn负责生成任务，并放入队列。
对于多线程来说，不同线程的运行从thread::builder.spawn()开始分道扬镳

main函数所在的线程继续由block_on()中的loop f.poll()驱动

其他的线程由rt.inner.blocking_spawner().inner.run( )驱动

build_threaded_runtime()->launch.launch()->spawn_blocking()->spawn_blocking_inner()->Spawner.spawn_task()->Spawner.spawn_thread()->thread::Builder::spawn()

====
2026.2.4 补充
rt.inner.blocking_spawner().inner.run( )驱动的是BlockingPool中的任务，BlockingPool中的人物统一存放在全局shared.queue
区别于core worker pool 放在 local run queue或者inject队列中的任务
====

```rust
    builder.spawn(move || {
        // Only the reference should be moved into the closure
        let _enter = rt.enter();
        rt.inner.blocking_spawner().inner.run(id);
        drop(shutdown_tx);
    })
```

```rust
impl Inner {
    fn run(&self, worker_thread_id: usize) {
        if let Some(f) = &self.after_start {
            f();  // 执行线程开始前的回调函数
        }

        let mut shared = self.shared.lock();
        let mut join_on_thread = None; // 准备可能的前一个线程的JoinHandle

        'main: loop {
            // BUSY
            while let Some(task) = shared.queue.pop_front() {
                self.metrics.dec_queue_depth();
                drop(shared);
                task.run(); // 执行任务（无锁状态）

                shared = self.shared.lock();
            }

            // IDLE
            self.metrics.inc_num_idle_threads();// 空闲线程数+1

            while !shared.shutdown {
                // 空闲超过 keep_alive时间的线程会退出
                let lock_result = self.condvar.wait_timeout(shared, self.keep_alive).unwrap();

                shared = lock_result.0;
                let timeout_result = lock_result.1;

                if shared.num_notify != 0 { // 是合法的唤醒
                    shared.num_notify -= 1; // 确认唤醒
                    break;
                }

                if !shared.shutdown && timeout_result.timed_out() {
                    // 从线程映射中移除自己
                    let my_handle = shared.worker_threads.remove(&worker_thread_id);
                    // 前一个退出的线程的JoinHandle，由本线程来join 退出前保存自己的 JoinHandle到共享数据
                    join_on_thread = std::mem::replace(&mut shared.last_exiting_thread, my_handle);

                    break 'main;
                }
            }

        }

        if shared.shutdown && self.metrics.num_threads() == 0 {
            self.condvar.notify_one();
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
tokio\src\runtime\scheduler\multi_thread\worker.rs

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