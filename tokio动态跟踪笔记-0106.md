由于Driver的wake_by_ref()最终都调用的是 arc_self.driver.unpark();

虽然名字叫做driver，但是其数据结构其实是Handle

tokio\src\runtime\scheduler\current_thread\mod.rs
```rust
pub(crate) struct Handle {
    ......
    /// Resource driver handles，
    pub(crate) driver: driver::Handle,
}
```

跟踪Handle的实现

```rust
tokio\src\runtime\driver.rs

runtime::driver::Driver

pub(crate) struct Handle {
    /// IO driver handle
    pub(crate) io: IoHandle,

    /// Signal driver handle
    #[cfg_attr(any(not(unix), loom), allow(dead_code))]
    pub(crate) signal: SignalHandle,

    /// Time driver handle
    pub(crate) time: TimeHandle,
}

impl Handle {
    pub(crate) fn unpark(&self) {
        #[cfg(feature = "time")]
        if let Some(handle) = &self.time {
            handle.unpark();
        }

        // runtime::io::Handle.unpark
        self.io.unpark();
    }
```
Handle的第一个结构IoHandle

```rust
// 每个组件都有启用和禁用两种状态
enum IoHandle {
    Enabled(crate::runtime::io::Handle),  // 启用状态
    Disabled(UnparkThread),               // 禁用状态
}

enum IoStack {
    Enabled(ProcessDriver),  // 启用时的完整堆栈
    Disabled(ParkThread),   // 禁用时的简单实现
}
```

下一层是 runtime::io::Handle
```rust
tokio\src\runtime\io\driver.rs
/// A reference to an I/O driver.
pub(crate) struct Handle {
    /// Registers I/O resources.
    registry: mio::Registry,

    /// Tracks all registrations
    registrations: RegistrationSet,

    /// Used to wake up the reactor from a call to `turn`.
    /// Not supported on `Wasi` due to lack of threading support.
    waker: mio::Waker,
}
    /// 强制唤醒在 `turn` 调用中被阻塞的反应器，或者使下一次 `turn` 调用立即返回。
    pub(crate) fn unpark(&self) {
        // 调用mio Waker的wake
        self.waker.wake().expect("failed to wake I/O driver");
    }
```

至于time的unpark只是一个测试环境用的
```rust
tokio\src\runtime\time\handle.rs

impl Handle {
    pub(crate) fn unpark(&self) {
        // 测试环境下的unpark
        #[cfg(feature = "test-util")]
        match self.inner {
            super::Inner::Traditional { ref did_wake, .. } => {
                did_wake.store(true, std::sync::atomic::Ordering::SeqCst);
            }
            #[cfg(all(tokio_unstable, feature = "rt-multi-thread"))]
            super::Inner::Alternative { ref did_wake, .. } => {
                did_wake.store(true, std::sync::atomic::Ordering::SeqCst);
            }
        }
    }
}

```

如果Io被禁用了，调用的unpark是

```rust
tokio\src\runtime\park.rs

impl UnparkThread {
    pub(crate) fn unpark(&self) {
        self.inner.unpark();
    }
}

fn unpark(&self) {
        match self.state.swap(NOTIFIED, SeqCst) {
            EMPTY => return,    // 没有线程在等待
            NOTIFIED => return, // 已经被唤醒过了
            PARKED => {}        // 需要去唤醒某个线程
            _ => panic!("inconsistent state in unpark"),
        }
        // 在被停放（parked）的线程将 `state` 设置为 `PARKED` 存在一个时间窗口。如果我们在这一时期通知，通知会被忽略，然后当被停放的线程去睡眠时，它将永远不会醒来。幸运的是，在此阶段它已经锁定了 `lock`，所以我们可以获取 `lock` 来等待，直到它准备好接收通知。

        // 在调用 `notify_one` 之前释放 `lock` 意味着当被停放的线程醒来时，它不会刚醒来就必须等待我们释放 `lock`。
        drop(self.mutex.lock());

        self.condvar.notify_one();
    }

```
condvar 是 loom::sync::Condvar
```rust

#[test]
fn notify_one() {
    loom::model(|| {
        let tx = Arc::new(Notify::new());
        let rx = tx.clone();

        let th = thread::spawn(move || {
            block_on(async {
                rx.notified().await;
            });
        });

        tx.notify_one();
        th.join().unwrap();
    });
}
```

```rust
tokio\src\sync\notify.rs

pub fn notify_one(&self) {
    self.notify_with_strategy(NotifyOneStrategy::Fifo);
}

fn notify_with_strategy(&self, strategy: NotifyOneStrategy) {
        // Load the current state
        let mut curr = self.state.load(SeqCst);

        // If the state is `EMPTY`, transition to `NOTIFIED` and return.
        while let EMPTY | NOTIFIED = get_state(curr) {
            // The compare-exchange from `NOTIFIED` -> `NOTIFIED` is intended. A
            // happens-before synchronization must happen between this atomic
            // operation and a task calling `notified().await`.
            let new = set_state(curr, NOTIFIED);
            let res = self.state.compare_exchange(curr, new, SeqCst, SeqCst);

            match res {
                // No waiters, no further work to do
                Ok(_) => return,
                Err(actual) => {
                    curr = actual;
                }
            }
        }

        // There are waiters, the lock must be acquired to notify.
        let mut waiters = self.waiters.lock();

        // The state must be reloaded while the lock is held. The state may only
        // transition out of WAITING while the lock is held.
        curr = self.state.load(SeqCst);

        if let Some(waker) = notify_locked(&mut waiters, &self.state, curr, strategy) {
            drop(waiters);
            waker.wake();
        }
}
```