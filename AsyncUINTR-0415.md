通过调试发现一个问题，就是无限等待的时候会falling back to epoll_wait

[ 2176.044378] epoll: no timeout, falling back to regular epoll_wait
[ 2176.069918] epoll: do_epoll_wait_uintr called, epfd=3, maxevents=1024, timeout=0000000000000000
[ 2176.078708] epoll: epoll instance 00000000bb3b3179
[ 2176.083562] epoll: no timeout, falling back to regular epoll_wait

修改fs/eventpoll.c do_epoll_wait_uintr()的实现

在uintr-bi 中调试 epoll_wait

sudo mount -t debugfs none /sys/kernel/debug
sudo sh -c 'echo -n "file fs/eventpoll.c +p" > /sys/kernel/debug/dynamic_debug/control'
ORIGINAL_LOGLEVEL=$(cat /proc/sys/kernel/printk | awk '{print $1}')
sudo sh -c 'echo 8 > /proc/sys/kernel/printk'
sudo dmesg -c
sudo dmesg | grep -i epoll


Linux 内核原本就支持用户态中断唤醒各种原本可以被信号唤醒的系统调用。（因为只要config UINTR_BLCOKING + 注册中断的时候使用UINTR_HANDLER_FLAG_WAITING_ANY，那么内核把用户态中断当成普通中断处理的时候，会在中断处理程序中把中断信号TIF_NOTIFY_SIGNAL 设置到对应的程序）
```c
DEFINE_IDTENTRY_SYSVEC(sysvec_uintr_kernel_notification)
{
	/* TODO: Add entry-exit tracepoints */
	ack_APIC_irq();
	inc_irq_stat(uintr_kernel_notifications);

	pr_debug_ratelimited("uintr: Kernel notification interrupt on %d\n",
			     smp_processor_id());

	if (IS_ENABLED(CONFIG_X86_UINTR_BLOCKING))
		uintr_wake_up_process();
}

void uintr_wake_up_process(void)
{
	struct uintr_upid_ctx *upid_ctx, *tmp;
	unsigned long flags;

	/* Fix: 'BUG: Invalid wait context' due to use of spin lock here */
	spin_lock_irqsave(&uintr_wait_lock, flags);
	list_for_each_entry_safe(upid_ctx, tmp, &uintr_wait_list, node) {
		if (test_bit(UINTR_UPID_STATUS_ON, (unsigned long *)&upid_ctx->upid->nc.status)) {
			pr_debug_ratelimited("uintr: Waking up task %d\n",
					     upid_ctx->task->pid);
			set_bit(UINTR_UPID_STATUS_SN, (unsigned long *)&upid_ctx->upid->nc.status);
			/* Check if a locked access is needed for NV and NDST bits of the UPID */
			upid_ctx->upid->nc.nv = UINTR_NOTIFICATION_VECTOR;
			upid_ctx->waiting = false;

            // 在这里设置TIF_NOTIFY_SIGNAL
			set_tsk_thread_flag(upid_ctx->task, TIF_NOTIFY_SIGNAL);
			wake_up_process(upid_ctx->task);
			list_del(&upid_ctx->node);
		}
	}
	spin_unlock_irqrestore(&uintr_wait_lock, flags);
}

```

# 接收方是多线程的情况如何处理用户态中断

多线程情况下，其中一个线程获取锁后进入park_driver。其他的都进入进入park_condvar，

进入park_driver 调用epoll_wait 系统调用等待，进入park_condvar 则调用 futex系统调用等待。

传统的同步机制（如 System V 信号量）在竞争时需要频繁进入内核，开销大。

futex的思路是：无竞争时在用户空间用原子操作完成同步，竞争时才进入内核，挂起或唤醒线程，从而减少系统调用开销。

```c
long syscall(SYS_futex, uint32_t *uaddr, int futex_op, uint32_t val,
             const struct timespec *timeout, uint32_t *uaddr2, uint32_t val3);
uaddr​

指向一个用户空间的 32 位整数，通常表示锁的状态（比如 0=未锁定，1=锁定，可能有等待者等）。

futex_op​

操作类型 + 标志。常见操作：

FUTEX_WAIT：如果 *uaddr == val，则休眠，直到被 FUTEX_WAKE唤醒或超时。

FUTEX_WAKE：唤醒最多 val个在 uaddr上等待的线程。

FUTEX_REQUEUE：将一些等待者从 uaddr移到 uaddr2（用于实现条件变量，避免“惊群”）。

FUTEX_CMP_REQUEUE：带比较的 requeue（比较 *uaddr是否等于 val3再 requeue，避免竞态）。

FUTEX_WAKE_OP：对两个 futex 变量执行原子操作并唤醒（用于实现移交锁的所有权）。

val​

操作相关参数，对不同 op 意义不同。比如在 FUTEX_WAIT中，它是线程期望的 *uaddr的值。
```

```rust
libc::syscall(
                        libc::SYS_futex,
                        futex as *const Atomic<u32>, // self.futex变量
                        libc::FUTEX_WAIT_BITSET | libc::FUTEX_PRIVATE_FLAG,
                        expected,
                        timespec.as_ref().map_or(null(), |t| t as *const libc::timespec),
                        null::<u32>(), // This argument is unused for FUTEX_WAIT_BITSET.
                        !0u32,         // A full bitmask, to make it behave like a regular FUTEX_WAIT.
                    )

- futex as *const Atomic<u32> - futex 变量的地址，通常是一个原子整数
- libc::FUTEX_WAIT_BITSET | libc::FUTEX_PRIVATE_FLAG - 操作类型：
- FUTEX_WAIT_BITSET - 等待 futex 值变化，支持位掩码
- FUTEX_PRIVATE_FLAG - 表示这是进程内私有 futex，性能更好
- expected - 期望值，只有当 futex 当前值等于此值时才会等待
- timespec - 超时时间，为 None 时表示无限期等待
```


多线程情况下， 等待uintr的线程id 是12

但是最后接收中断，执行中断处理程序的线程 id是1

所以首先要把异步等待用户态中断和注册用户态中断的代码放在同一个协程中，保证始终运行在同一个thread（这一点还存疑）

其次，backend的1号线程负责执行用户态中断处理程序，实际单步调试的时候，1号工作线程并不是通过run()->park()->park_internal()->park_condvar()

而是通过block_on()->park()->park()


```c
tokio/src/runtime/park.rs
loop {
            m = self.condvar.wait(m).unwrap();

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
```

wait 调用到最后是futex_wait，用户态中断能够唤醒SYS_futex 系统调用，但是因为futex.load(Relaxed)  没有设置为目标值，所以还是会一直阻塞在这个循环里面

```rust
/home/zwp/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/std/src/sys/pal/unix/futex.rs
pub fn futex_wait(futex: &Atomic<u32>, expected: u32, timeout: Option<Duration>) -> bool {
    loop {
        // No need to wait if the value already changed.
        if futex.load(Relaxed) != expected {
            return true;
        }

        let r = unsafe {
                any(target_os = "linux", target_os = "android") => {
                    // Use FUTEX_WAIT_BITSET rather than FUTEX_WAIT to be able to give an
                    // absolute time rather than a relative time.
                    libc::syscall(
                        libc::SYS_futex,
                        futex as *const Atomic<u32>,
                        libc::FUTEX_WAIT_BITSET | libc::FUTEX_PRIVATE_FLAG,
                        expected,
                        timespec.as_ref().map_or(null(), |t| t as *const libc::timespec),
                        null::<u32>(), // This argument is unused for FUTEX_WAIT_BITSET.
                        !0u32,         // A full bitmask, to make it behave like a regular FUTEX_WAIT.
                    )
                }
                _ => {
                    compile_error!("unknown target_os");
                }
            }
        };

        
    }
}
```

现在无法在标准库中调用process_wakeup，而且也不好修改标准库的代码。

需要使用nightly重新编译工具链。
```
rustup override set stable-x86_64-unknown-linux-gnu
rustup component add rust-src
cargo build -Z build-std -Z build-std-features=panic-unwind --features uintr-core
```

futex_wait需要能够返回才能调用process_wakeup，但是futex_wait 依赖于在process_wakeup执行unpark之类的唤醒操作才能返回，因此形成一个死锁。
其实最理想的办法是在中断处理函数中调用wakeup，但是这样会导致panic（变量的争用导致的）