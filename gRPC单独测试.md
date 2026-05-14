
rm /dev/shm/gateway_shm_uintr_* 
# 如果需要停止进程，可以使用以下命令
pkill -f "backend --transport shm-uintr"
pkill -f "gateway --transport shm-uintr"

# 使用 info 日志级别运行 backend
    RUST_LOG=info ./target/debug/backend --transport shm-uintr --shm-name gateway_shm_uintr_test_manual

# 使用 info 日志级别运行 gateway
RUST_LOG=info ./target/debug/gateway --transport shm-uintr --shm-name gateway_shm_uintr_test_manual

# 发送测试请求
curl -s "http://127.0.0.1:8080/echo?message=test_uintr"

wrk -t8 -c256 -d30s --latency -s "scripts/wrk.lua" "http://127.0.0.1:8080/api/echo"


# 查看log

sudo mount -t debugfs none /sys/kernel/debug
sudo sh -c 'echo -n "file fs/eventpoll.c +p" > /sys/kernel/debug/dynamic_debug/control'
ORIGINAL_LOGLEVEL=$(cat /proc/sys/kernel/printk | awk '{print $1}')
sudo sh -c 'echo 8 > /proc/sys/kernel/printk'
sudo dmesg -c
sudo dmesg | grep -i epoll


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