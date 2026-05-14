## park

park里面有两种等待，一种是争抢driver锁，一种是在condvar上睡眠
```rust
fn park(&self, handle: &driver::Handle) {
        // 优先争抢 I/O driver 锁
        if let Some(mut driver) = self.shared.driver.try_lock() {
            self.park_driver(&mut driver, handle, None);
        } else {
            // 抢不到 driver 锁，就退而求其次在 condvar 上睡
            self.park_condvar(None);
        }
    }
```

park_driver 才会调用turn 等待外部事件，park_condvar 等待别的worker唤醒

## future唤醒

把被唤醒的 task 放进“当前这个执行 waker 的 worker”的本地队列

优先本地化

如果 waker 在同一个 runtime 的、同一个 worker 线程上被调用，并且这个 worker 还拿着 core，
那么任务会通过 schedule_local 回到这个 worker 的本地队列（lifo_slot 或 run_queue），
这就是“再次被放回原本 worker 的队列”。
退化为全局分发

如果 waker 在别的线程 / 没有 core 的上下文里调用，
任务就会进入全局 inject 队列，
之后由某个被唤醒或正在找活干的 worker 从 inject / steal 中取走，
这在结果上表现为“不保证是原 worker，有点像随机分配”。