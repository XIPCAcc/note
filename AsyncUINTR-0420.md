# process_global_uintr_wakers 调用时间节点的探究

## current process

### 在turn()中调用

进入turn 分两种情况
1. 整个任务队列都没有任务可以运行
2. 运行event_interval个任务后强制查询外部的事件

对于情况1，turn 会阻塞等待在epoll_wait直到事件发生
对于情况2，turn 执行epoll_wait 后里面返回

如果在turn 返回前调用process_global_uintr_wakers 就可以唤醒协程，方式比较直接简单。
上下文环境都有park负责设置处理好。

而且好处是不用频繁查询中断标志，剩下一部分开销，

但是问题就是，任务的调度时效性难以得到保证，而且对于情况1有可能发生死锁(epoll_wait 等待期间既没有事件也没有用户态中断)。

### 在run()中调用

也分成两种情况

1. 每执行完一个任务查询一次标志，然后唤醒等待协程
2. 等到队列没有就绪任务，即将进入park再查询标志

如果考虑延迟，情况1更好。如果考虑吞吐量，情况2更好，

其次，在turn中调用process_global_uintr_wakers 需要自己处理上下文。

具体而言，在run() 中 cx.core的所有权已经被转移到'outer: loop  中的局部变量core

```rust
'outer: loop {
                let handle = &context.handle;

                if handle.reset_woken() {
                    let (c, res) = context.enter(core, || {
                        crate::task::coop::budget(|| future.as_mut().poll(&mut cx))
                    });

                    core = c;
```

而在schedule中，需要使用cx.core 景任务放回队列之中。

```rust
fn schedule(&self, task: task::Notified<Self>) {
        use scheduler::Context::CurrentThread;

        context::with_scheduler(|maybe_cx| match maybe_cx {
            Some(CurrentThread(cx)) if Arc::ptr_eq(self, &cx.handle) => {
                let mut core = cx.core.borrow_mut();

                // If `None`, the runtime is shutting down, so there is no need
                // to schedule the task.
                // 这里得到的core 为None就是在 run() 中调用process_global_uintr_wakers需要处理的事情
                if let Some(core) = core.as_mut() {
                    core.push_task(self, task);
                }
            }
            _ => {
                // Track that a task was scheduled from **outside** of the runtime.
                self.shared.scheduler_metrics.inc_remote_schedule_count();

                // Schedule the task
                self.shared.inject.push(task);
                self.driver.unpark();
            }
        });
    }

```

turn的做法是将core作为参数传递给 park，park返回后再将所有权返回给 outer loop

park负责调用context将core 设置到cx.core

```rust
park() 
  → park_internal() 
    → Context::enter(core, || driver.park())

fn enter<R>(&self, core: Box<Core>, f: impl FnOnce() -> R) -> (Box<Core>, R) {
    // 先把 core 放入 thread-local
    *self.core.borrow_mut() = Some(core);

    // 执行闭包 f()，此时 core 在 thread-local 中是有值的
    let ret = f();

    // Step 3: 闭包结束后，取回 core
    let core = self.core.borrow_mut().take().expect("core missing");
    (core, ret)
}

fn park_internal(
        &self,
        core: Box<Core>,
        handle: &Handle,
        driver: &mut Driver,
        duration: Option<Duration>,
    ) -> Box<Core> {
        let (core, ()) = self.enter(core, || {
            // ....
        });

        core
    }
```


```
RUST_LOG=info ./target/debug/backend --transport shm-uintr --shm-name gateway_shm_uintr_test_manual
RUST_LOG=info ./target/debug/gateway --transport shm-uintr --shm-name gateway_shm_uintr_test_manual
```

### 每次调度下一个任务前
冷启动3次

wrk -t8 -c256 -d5s --latency -s "scripts/wrk.lua" "http://127.0.0.1:8080/api/echo"
Running 5s test @ http://127.0.0.1:8080/api/echo
  8 threads and 256 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   202.67ms   68.46ms 386.15ms   64.50%
    Req/Sec   155.35     79.06   434.00     68.39%
  Latency Distribution
     50%  197.92ms
     75%  250.75ms
     90%  294.59ms
     99%  371.83ms
  6034 requests in 5.02s, 1.05MB read
Requests/sec:   1201.73
Transfer/sec:    214.80KB

### 队列没有任务运行时
冷启动3次

wrk -t8 -c256 -d5s --latency -s "scripts/wrk.lua" "http://127.0.0.1:8080/api/echo"
Running 5s test @ http://127.0.0.1:8080/api/echo
  8 threads and 256 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   194.34ms   73.26ms 472.39ms   68.47%
    Req/Sec   180.71    100.22   380.00     56.57%
  Latency Distribution
     50%  190.82ms
     75%  244.81ms
     90%  282.23ms
     99%  398.67ms
  6428 requests in 5.02s, 1.12MB read
Requests/sec:   1280.86
Transfer/sec:    229.02KB


### turn 返回时

wrk -t8 -c256 -d5s --latency -s "scripts/wrk.lua" "http://127.0.0.1:8080/api/echo"
Running 5s test @ http://127.0.0.1:8080/api/echo
  8 threads and 256 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   238.64ms   64.32ms 378.13ms   65.59%
    Req/Sec   141.63     86.86   323.00     59.52%
  Latency Distribution
     50%  235.11ms
     75%  292.98ms
     90%  326.04ms
     99%  370.57ms
  5242 requests in 5.03s, 0.91MB read
Requests/sec:   1042.95
Transfer/sec:    186.40KB


## sleep DELAY_US=1000us

### 吞吐量对比 (Requests/sec)

| 并发数 | shm-uds | shm-eventfd | shm-uintr | 最优方案 |
|--------|--------:|------------:|----------:|----------|
| 16 | 7,040 | 6,791 | 6,456 | **uds** |
| 32 | 13,140 | 13,410 | 13,316 | eventfd |
| 64 | 20,339 | **25,107** | 23,641 | eventfd |
| 256 | **89,427** | 71,922 | 47,628 | uds |
| 1024 | 111,477 | 141,816 | **172,024** | **uintr** |
| 4096 | **100,907** | 30,163 | 80,428 | uds |

### 平均延迟对比 (Avg Latency)

| 并发数 | shm-uds | shm-eventfd | shm-uintr | 最优方案 |
|--------|--------:|------------:|----------:|----------|
| 16 | **2.26ms** | 2.34ms | 2.47ms | uds |
| 32 | 2.42ms | **2.37ms** | 2.39ms | eventfd |
| 64 | 3.13ms | **2.53ms** | 2.69ms | eventfd |
| 256 | **2.85ms** | 3.64ms | 5.37ms | uds |
| 1024 | 9.15ms | 7.19ms | **5.90ms** | **uintr** |
| 4096 | **39.70ms** | 131.94ms | 51.70ms | uds |

### P99 延迟对比

| 并发数 | shm-uds | shm-eventfd | shm-uintr | 最优方案 |
|--------|--------:|------------:|----------:|----------|
| 16 | 2.81ms | 2.97ms | 3.68ms | **uds** |
| 32 | 3.20ms | 3.19ms | 3.18ms | uintr |
| 64 | 5.16ms | **3.56ms** | 3.98ms | eventfd |
| 256 | 3.88ms | 10.85ms | **6.45ms** | uds |
| 1024 | 14.78ms | **9.56ms** | 8.48ms | uintr |
| 4096 | **71.11ms** | 171.16ms | 64.50ms | **uintr** |

---
          
## Sleep 300μs 性能测试报告

### 吞吐量对比 (Requests/sec)

| 并发数 | shm-uds | shm-eventfd | shm-uintr | 最佳选择 |
|--------|--------:|------------:|----------:|----------|
| 16 | 9,884 | 10,545 | **10,410** | eventfd |
| 32 | 18,826 | **19,444** | 17,839 | eventfd |
| 64 | 29,958 | **33,656** | 32,603 | eventfd |
| 256 | **132,083** | 72,035 | 114,435 | **uds** |
| 1024 | 114,759 | 115,958 | **131,718** | uintr |
| 4096 | 76,678 | 72,614 | **98,700** | uintr |

### 平均延迟对比 (Avg Latency)

| 并发数 | shm-uds | shm-eventfd | shm-uintr | 最佳选择 |
|--------|--------:|------------:|----------:|----------|
| 16 | 1.62ms | **1.51ms** | 1.53ms | eventfd |
| 32 | 1.69ms | **1.64ms** | 1.78ms | eventfd |
| 64 | 2.12ms | **1.89ms** | 1.95ms | eventfd |
| 256 | **1.93ms** | 3.62ms | 2.23ms | **uds** |
| 1024 | 8.83ms | 8.79ms | **7.74ms** | **uintr** |
| 4096 | 55.29ms | 55.85ms | **40.63ms** | **uintr** |

### P99 延迟对比

| 并发数 | shm-uds | shm-eventfd | shm-uintr | 最佳选择 |
|--------|--------:|------------:|----------:|----------|
| 16 | 2.69ms | **2.54ms** | 2.51ms | uintr |
| 32 | 2.74ms | **2.74ms** | 2.75ms | eventfd |
| 64 | 3.35ms | **2.62ms** | 2.67ms | eventfd |
| 256 | **3.08ms** | 8.55ms | 3.52ms | **uds** |
| 1024 | 12.93ms | 14.42ms | **13.03ms** | uintr |
| 4096 | 167.09ms | 73.36ms | **64.60ms** | **uintr** |

---

### 与 500μs Spinloop 对比 (关键差异)

| 并发 | 指标 | 500μs | 300μs | 变化 |
|------|------|-------|-------|------|
| 256 | uintr 吞吐 | 46,561 | **114,435** | **+146%**  |
| 256 | uintr 延迟 | 5.60ms | **2.23ms** | **-60%**  |
| 1024 | uintr 吞吐 | 96,102 | **131,718** | **+37%**  |
| 4096 | uintr 吞吐 | 115,067 | 98,700 | -14%  |
| 4096 | uintr 延迟 | 35.38ms | 40.63ms | +15%  |

---

## Log 分析报告: `20260425-040112-spinloop300us`

### 测试配置
- **测试时长**: 30秒
- **线程数**: 8
- **连接数**: 16, 32, 64, 256, 1024, 4096
- **三种传输机制**: `eventfd`, `UDS`, `UINTR`
- **测试场景**: spinloop 300us 唤醒间隔

---

### 性能对比总览

| 连接数 | eventfd RPS | eventfd 延迟 | UDS RPS | UDS 延迟 | UINTR RPS | UINTR 延迟 |
|--------|-------------|--------------|---------|----------|-----------|------------|
| 16     | 3311.71    | 4.82ms       | 3297.28 | 4.84ms   | 3322.61  | 4.81ms     |
| 32     | 3311.11    | 9.65ms       | 3289.18 | 9.71ms   | 3319.95  | 9.63ms     |
| 64     | 3305.61    | 19.34ms      | 3300.07 | 19.37ms  | 3315.03  | 19.26ms    |
| 256    | 3307.70    | 77.13ms      | 3297.02 | 77.51ms  | 3315.34  | 77.03ms    |
| 1024   | 3303.46    | 307.84ms     | 3288.41 | 308.99ms | 3300.82  | 304.13ms   |
| 4096   | 3300.38    | 1.19s       | 3291.45 | 1.19s    | 3307.66  | 1.17s      |

---

### 关键发现

#### 1. **RPS (Requests Per Second) 基本持平**
所有配置下，三种传输机制的 RPS 都在 **3300 req/s** 左右，差异不超过 **1%**。这说明在当前负载下，传输层不是瓶颈。

#### 2. **UINTR 在高连接数时延迟略优**
- 在 **1024 连接**时，UINTR 延迟 **304.13ms** vs eventfd **307.84ms** vs UDS **308.99ms** - UINTR 领先约 **3-5ms**
- 在 **4096 连接**时，UINTR 延迟 **1.17s** vs eventfd/UDS 都是 **1.19s** - UINTR 领先约 **20ms**

#### 3. **Stdev (标准差) 角度 - UINTR 更稳定**
- 1024 连接时：UINTR stdev **27.37ms** vs eventfd **18.30ms** vs UDS **18.73ms**  
  （UINTR 的 stdev 更高，说明延迟分布更分散）
- 但 99th percentile：UINTR **312.02ms** vs eventfd **331.14ms** vs UDS **319.15ms**
  - UINTR 的 99th percentile 反而是最好的

#### 4. **Timeout 错误分析**
| 连接数 | eventfd timeouts | UDS timeouts | UINTR timeouts |
|--------|-------------------|--------------|----------------|
| 4096   | 17               | 227          | 631            |

**UINTR 的 timeout 数量明显更多**（631 vs UDS 227 vs eventfd 17），这是一个值得关注的问题。UINTR 可能在高连接数时存在某些唤醒或事件处理的问题。

---
