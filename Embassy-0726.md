# 如何在中断上下文能够唤醒协程

唤醒协程的本质是能够访问任务队列，能够将协程重新放回任务队列。

如何在中断上下文能够访问任务队列，需要保证入队和出队的操作是原子的。

如果不支持原子指令，那么就必须通过临界区（关中断）方式操作任务队列。

Embassy的任务队列采用的是cordyceps::TransferStack（lock-free 的 Treiber Stack （无锁并发栈））

x86/x86-64 用 lock cmpxchg{q}（push 的 CAS 循环）+ lock xchg/q（take_all 的整批摘取）就能正确实现 cordyceps::TransferStack。

问题：性能比不过连续内存的队列，拓展到多核任务窃取性能会更差。

# 空闲时如何休眠和唤醒

尽管线程可以调用任意一个阻塞式系统调用休眠，用户态中断也能够唤醒任意一个阻塞系统调用。但是中断的丢失问题仍然是一个比较棘手的问题。

通过 CLUI和STUI指令实现临界区。


```rust
        pub fn run(&'static mut self, init: impl FnOnce(Spawner)) -> ! {
            init(self.inner.spawner());

            loop {
                unsafe {
                    CLUI();  // Read Copy Update 解决每次需要进入临界区的问题
                    if queue.empty() {
                        // 需要x86架构保证了"stui + 下一条指令"的原子性。执行后，要等 下一条指令 执行完毕之后，才会响应中断
                        STUI();
                        sleep();
                    } else {
                        STUI();
                        self.inner.poll();
                    }
                }
            }
        }
```

# 多核

Embassy 的多核模型是thread-per-core（每核一线程）架构

thread-per-core（也叫 shared-nothing）是一种并发架构：

每个核心跑一个专用执行线程，每线程持有自己独立的状态（队列、缓存、分配器），线程之间不共享可变状态，只通过显式的消息传递/队列通信。

特征：

无锁争用（状态不共享，不需要全局锁）
亲和性（数据/外设固定在某个核上）
显式通信（跨核走 channel / IPC）
无 work-stealing（不做任务窃取）