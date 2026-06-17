          
##  `waker.wake()` 到唤醒 working thread 的分情况说明

---

### 情况总览

```
waker.wake()
  └─ raw.wake_by_ref()
       └─ scheduler.schedule(task)
            └─ Handle::schedule()
                 ├─ [同一runtime]  schedule_local(task)  → 情况1
                 └─ [不同runtime]  push_remote(task) + notify_parked_remote()
                                                          └─ 情况2/3/4/5
```

---

### 情况1：同一 runtime，当前 worker 仍在运行

**不需要唤醒线程，任务由当前 worker 自然消耗。**

[worker.rs#L1375-L1389](tokio/src/runtime/scheduler/multi_thread/worker.rs#L1375-L1389)：

```rust
fn schedule_task(&self, task: Notified, _is_yield: bool) {
    with_current(|maybe_cx| {
        if let Some(cx) = maybe_cx {
             if let Some(core) = cx.core.borrow_mut().as_mut() {
                self.schedule_local(core, task, is_yield);
                return;
        }
        self.push_remote_task(task);
        self.notify_parked_remote();               // 只有跨 runtime 才到这里
    });
}
```

---

### 情况2：跨 runtime 唤醒，worker 正通过 Driver（epoll）休眠

**链路：eventfd → epoll_wait 返回。**

[worker.rs#L1462-L1463](tokio/src/runtime/scheduler/multi_thread/worker.rs#L1462-L1463) → [park.rs#L272](tokio/src/runtime/scheduler/multi_thread/park.rs#L272) → [driver.rs#L258](tokio/src/runtime/io/driver.rs#L258)：

```
notify_parked_remote()
  └─ idle.worker_to_notify()              // 从 sleepers 弹出目标 worker
       └─ remotes[index].unpark.unpark()
            └─ state = PARKED_DRIVER → driver.unpark()
                 └─ self.waker.wake()     // mio::Waker 写入 eventfd
                      └─ epoll_wait() 被唤醒
```

当 worker 是最后一个 unparked 线程，获取到 driver 锁并在 `park_internal` 中阻塞于 `driver.park()` → `epoll_wait` 时，外部通过 eventfd 写入将其唤醒。

---

### 情况3：跨 runtime 唤醒，worker 通过 Condvar 休眠

**链路：Mutex + Condvar → notify_one() → futex_wake。**

[park.rs#L262-L268](tokio/src/runtime/scheduler/multi_thread/park.rs#L262-L268)：

```
notify_parked_remote()
  └─ unpark()
       └─ state = PARKED_CONDVAR → self.unpark_condvar()
            └─ drop(self.mutex.lock())    // 先释放锁
            └─ self.condvar.notify_one()  // → pthread_cond_signal → futex_wake
```

当 driver 被其他线程持有时（多 worker 场景），当前 worker 回退到 condvar 阻塞，外部通过 `notify_one` 唤醒。

---

### 情况4：有 worker 正在 searching，不需要唤醒

[idle.rs#L49-L62](tokio/src/runtime/scheduler/multi_thread/idle.rs#L49-L62)：

```rust
pub(super) fn worker_to_notify(&self, shared: &Shared) -> Option<usize> {
    if !self.notify_should_wakeup() {
        return None;  // 有 worker 在 searching，它会找到任务
    }
    // ...
    let ret = lock.idle.sleepers.pop();
    Some(ret)
}

fn notify_should_wakeup(&self) -> bool {
    let state = State(self.state.fetch_add(0, SeqCst));
    state.num_searching() == 0 && state.num_unparked() < self.num_workers
}
```

如果有 worker 处于 searching 状态（正在尝试 steal work），`worker_to_notify` 直接返回 `None`，不会唤醒任何线程。searching worker 会通过 steal 或 inject queue 取到新任务。

---

### 情况5：worker 尚未真正进入休眠（race）

[park.rs#L119-L131](tokio/src/runtime/scheduler/multi_thread/park.rs#L119-L131)：

```rust
fn park(&self, handle: &driver::Handle) {
    // 先 CAS 检查是否已被通知
    if self.state.compare_exchange(NOTIFIED, EMPTY, SeqCst, SeqCst).is_ok() {
        return;  // 已被通知过，直接返回
    }
    if let Some(mut driver) = self.shared.driver.try_lock() {
        self.park_driver(&mut driver, handle, None);
    } else {
        self.park_condvar(None);
    }
}
```

如果外部在 worker 调用 `transition_to_parked`（写 sleepers）之后、但在实际调用 `condvar.wait()` 或 `driver.park()` 之前发出了 wake：

- unpark 端将 state 设为 `NOTIFIED`，发现是 `EMPTY`，什么都不做
- park 端 CAS 发现 `NOTIFIED`，直接交换为 `EMPTY` 并返回，**避免了失醒**

---

### 情况6：wake 发生在 park_internal 期间（core 为 None）

当 worker 进入 `park_internal` 后，core 被取出置于 None。此时同一 runtime 上的 task 调用 waker 会走 defer 路径：

[worker.rs#L970-L975](tokio/src/runtime/scheduler/multi_thread/worker.rs#L970-L975)：

```rust
pub(crate) fn defer(&self, waker: &Waker) {
    if unsafe { (*self.core.get()).is_none() } {
        waker.wake_by_ref();       // core 为 None → 直接重新唤醒 task
    } else {
        self.defer.defer(waker);   // 否则放入 defer 队列
    }
}
```

worker 从 park 醒来后，在 [worker.rs#L960](tokio/src/runtime/scheduler/multi_thread/worker.rs#L960) 调用 `self.defer.wake()` 批量唤醒 defer 队列中积累的 waker，这些 task 会被重新调度。

---

### 情况7：worker 自己醒来，有任务时清理 sleepers

[worker.rs#L1279-L1298](tokio/src/runtime/scheduler/multi_thread/worker.rs#L1279-L1298)：

```rust
fn transition_from_parked(&mut self, worker: &Worker) -> bool {
    if self.has_tasks() {
        self.is_searching = !worker.handle.shared.idle
            .unpark_worker_by_id(&worker.handle.shared, worker.index);
        return true;
    }
    if ...is_parked(...) {
        return false;  // 仍在 sleepers 中且无任务，继续睡
    }
    self.is_searching = true;
    true
}
```

- 如果 worker 醒来后 local queue 有任务，通过 `unpark_worker_by_id` 把自己从 sleepers 移除
- 如果没任务但仍在 sleepers 中，说明是被 spurious wake 唤醒或其他 worker 已经接管了任务，继续睡
- 如果不在 sleepers 中，说明被外部主动唤醒来 steal，进入 searching 状态

---

### 完整状态流转图

```
                        ┌──────────────────────────┐
                        │  Worker 正在运行 (EMPTY) │
                        └──────┬──────────┬────────┘
                               │          │
           所有 queue 为空      │          │  同一 runtime 内 wake
                               ▼          ▼
                     ┌──────────────┐   push_signal（不唤醒）
                     │transition_to_│
                     │  parked()    │
                     │加入 sleepers │
                     └──────┬───────┘
                            │
               ┌────────────┴────────────┐
               ▼                         ▼
      获取到 driver 锁             未获取到 driver 锁
      PARKED_DRIVER                PARKED_CONDVAR
         │                             │
         │ epoll_wait                  │ condvar.wait
         │                             │
         ▼                             ▼
    ┌─────────────────────────────────────────┐
    │  外部 wake 到达                          │
    │  ├─ 有 searching worker → 不唤醒 (情况4) │
    │  ├─ PARKED_DRIVER → eventfd 写 (情况2)   │
    │  ├─ PARKED_CONDVAR → notify_one (情况3)  │
    │  └─ EMPTY → 仅设 NOTIFIED (情况5)        │
    └─────────────────────────────────────────┘
```