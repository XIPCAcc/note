# 使用uintr_wait 方式实现异步

首先检查内核是否支持 uintr

修改了 ipc_bench 中uintr的等待方式，使用uintr_wait 系统调用直接实现等待（虽然测出来的时间很过于小）

但是至少说明这个系统调用是可以使用的。

其次查看的linux源码中这个系统调用的具体实现

```c
arch/x86/kernel/uintr.c

SYSCALL_DEFINE2(uintr_wait, u64, usec, unsigned int, flags)
{
	ktime_t expires;

	if (!cpu_feature_enabled(X86_FEATURE_UINTR))
		return -ENOSYS;

	if (!IS_ENABLED(CONFIG_X86_UINTR_BLOCKING))
		return -ENOSYS;

	if (flags)
		return -EINVAL;

	/* Check: Do we need an option for waiting indefinitely */
	if (usec > UINTR_WAIT_MAX_USEC)
		return -EINVAL;

	if (usec == 0)
		return 0;

	expires = usec * NSEC_PER_USEC;
	return uintr_receiver_wait(&expires);
}
```

这样就说明 ipc-bench 中的系统调用定义不对，这里应该接收两个参数，第一个参数是最大等待时间。

所以要修改

```c
source/uintrfd/uintrfd-bi.c

#define uintr_wait(usec, flags)		syscall(__NR_uintr_wait, usec, flags)
#define UINTR_WAIT_MAX_USEC			10000 // 10000us

uintr_wait(UINTR_WAIT_MAX_USEC, 0);

```

虽然能够正常从uintr_wait返回，但是怎么每次都好像是超时返回的time[997] = 10013.952 us，似乎用户态中断不会使得uintr_wait 立刻返回，只有超时才会返回。

如果使用linux 6.0.0+ old 那么，执行uintr_wait 会返回-1，不会等待超时。

返回-1是因为C库的syscall()函数进行了封装处理

```c
		ret = uintr_wait(UINTR_WAIT_MAX_USEC, 0);
		printf("uintr_wait ret = %d\n", ret);

		if (ret == -1) {
			perror("uintr_wait");
			switch (errno) {
				case ENOSYS:
					printf("错误: CPU 或内核不支持 UINTR\n");
					break;
				case EINVAL:
					printf("错误: 参数无效\n");
					break;
				case EOPNOTSUPP:
					printf("错误: 进程未注册为 UINTR 接收者\n");
					break;
				case EINTR:
					printf("错误: 被信号中断\n");
					break;
			}
		}
```

如果这样，相当于不休眠，一个loop一直在运行。（相当于直接把 epoll_wait 去掉）

```
============ RESULTS ================
Message size:       1
Message count:      1
Total duration:     0.085208 ms
Average duration:   81.018000 us
Minimum duration:   81.018000 us
Maximum duration:   81.018000 us
Standard deviation: 0.000000 us
Latency P50:        81.018000 us
Latency P90:        81.018000 us
Latency P99:        81.018000 us
Message rate:       11736 msg/s
Message rate:       0.011 MB/s
CPU usage:          100.98%
=====================================
```


如果使用支持的内核

```
wait 10us情况下，从40-200us都有

============ RESULTS ================
Message size:       1
Message count:      1
Total duration:     0.071453 ms
Average duration:   70.034000 us
Minimum duration:   70.034000 us
Maximum duration:   70.034000 us
Standard deviation: 0.000000 us
Latency P50:        70.034000 us
Latency P90:        70.034000 us
Latency P99:        70.034000 us
Message rate:       13995 msg/s
Message rate:       0.013 MB/s
CPU usage:          100.36%


============ RESULTS ================
Message size:       1
Message count:      10
Total duration:     0.674662 ms
Average duration:   66.776000 us
Minimum duration:   18.877000 us
Maximum duration:   438.204000 us
Standard deviation: 124.218117 us
Latency P50:        21.146000 us
Latency P90:        438.204000 us
Latency P99:        438.204000 us
Message rate:       14822 msg/s
Message rate:       0.014 MB/s
CPU usage:          100.09%
=====================================

============ RESULTS ================
Message size:       1
Message count:      100
Total duration:     4.297781 ms
Average duration:   42.843000 us
Minimum duration:   8.892000 us
Maximum duration:   1728.679000 us
Standard deviation: 170.089642 us
Latency P50:        24.706000 us
Latency P90:        26.816000 us
Latency P99:        1728.679000 us
Message rate:       23268 msg/s
Message rate:       0.022 MB/s
CPU usage:          100.01%
=====================================
```

### 用户态中断的优点

用户态中断和信号比起来的优点是，用户态中断在进程运行的期间可以打断进程执行流。但是信号不行，必须等到进程切换进内核然后再返回用户态时才能处理。

如果单纯看这个benchmark二者几乎没有差距，但是如果有一个特定的场景，进程的优先级足够高，分配的时间片足够长，那么用户态中断的优势才能够显现出来。


### UINTR_HANDLER_FLAG_WAITING_ANY

找到了uintr_wait 无法被用户态中断唤醒的原因，注册时候必须使用 UINTR_HANDLER_FLAG_WAITING_ANY flag

const UINTR_HANDLER_FLAG_WAITING_ANY: c_int = 0x3000;
uintr_register_handler(server_ui_handler, UINTR_HANDLER_FLAG_WAITING_ANY)

不传 WAITING 标志时

```
uintr_register_handler(handler, 0)
        ↓
waiting_cost = UPID_WAITING_COST_NONE
        ↓
uintr_wait() → schedule()
        ↓
switch_uintr_prepare() 检查 is_uintr_waiting_enabled() → false
        ↓
不调用 uintr_switch_to_kernel_interrupt()
        ↓
UPID->nv 仍然是 UINTR_NOTIFICATION_VECTOR
        ↓
发送者 SENDUIPI → 发送 UINTR_NOTIFICATION_VECTOR 中断
        ↓
接收者在睡眠，中断无法被处理
        ↓
进程不会被唤醒！
```

传入 WAITING 标志时：

```
uintr_register_handler(handler, UINTR_HANDLER_FLAG_WAITING_ANY)
        ↓
waiting_cost = UPID_WAITING_COST_RECEIVER 或 SENDER
        ↓
uintr_wait() → schedule()
        ↓
switch_uintr_prepare() 检查 is_uintr_waiting_enabled() → true
        ↓
调用 uintr_switch_to_kernel_interrupt()
        ↓
UPID->nv = UINTR_KERNEL_VECTOR，加入等待列表
        ↓
发送者 SENDUIPI → 发送 UINTR_KERNEL_VECTOR 中断
        ↓
sysvec_uintr_kernel_notification() → uintr_wake_up_process()
        ↓
进程被唤醒
```
UINTR_HANDLER_FLAG_WAITING_NONE 0x0 不启用中断唤醒（默认） 
UINTR_HANDLER_FLAG_WAITING_RECEIVER 0x1000 接收者承担等待成本 
UINTR_HANDLER_FLAG_WAITING_SENDER 0x2000 发送者承担等待成本 
UINTR_HANDLER_FLAG_WAITING_ANY 0x3000 启用中断唤醒（两者之一）

收到用户态中断时， uintr_wait 返回 -EINTR （用户态看到 -1 ， errno = EINTR ）。

- 返回 -EINTR 表示等待被中断
- 返回用户态后，硬件会检测到待处理的用户中断并执行注册的处理函数


## 使用uintr_wait 以后的结果

```rust
	let ret = uintr_wait(10000, 0);
	// println!("turn: false, ret: {}", ret);
	if check_uintr_pending() {
		let waker_count = process_uintr_wakers();
	}
```

```
============ RESULTS ================
Message size:       1
Message count:      100
Total duration:     1.530792 ms
Average duration:   15.190000 us
Minimum duration:   11.899000 us
Maximum duration:   62.225000 us
Standard deviation: 5.506747 us
Latency P50:        13.905000 us
Latency P90:        17.991000 us
Latency P99:        62.225000 us
Message rate:       65326 msg/s
Message rate:       0.062 MB/s
CPU usage:          100.03%
=====================================
Server: Test completed
Server: Communication complete

============ RESULTS ================
Message size:       1
Message count:      1
Total duration:     0.096642 ms
Average duration:   93.211000 us
Minimum duration:   93.211000 us
Maximum duration:   93.211000 us
Standard deviation: 0.000000 us
Latency P50:        93.211000 us
Latency P90:        93.211000 us
Latency P99:        93.211000 us
Message rate:       10347 msg/s
Message rate:       0.010 MB/s
CPU usage:          100.19%
=====================================

============ RESULTS ================
Message size:       1
Message count:      1
Total duration:     0.060618 ms
Average duration:   57.484000 us
Minimum duration:   57.484000 us
Maximum duration:   57.484000 us
Standard deviation: 0.000000 us
Latency P50:        57.484000 us
Latency P90:        57.484000 us
Latency P99:        57.484000 us
Message rate:       16497 msg/s
Message rate:       0.016 MB/s
CPU usage:          100.64%
=====================================

============ RESULTS ================
Message size:       1
Message count:      10
Total duration:     0.381023 ms
Average duration:   37.292000 us
Minimum duration:   15.343000 us
Maximum duration:   89.710000 us
Standard deviation: 21.743440 us
Latency P50:        40.248000 us
Latency P90:        89.710000 us
Latency P99:        89.710000 us
Message rate:       26245 msg/s
Message rate:       0.025 MB/s
CPU usage:          100.03%
=====================================

============ RESULTS ================
Message size:       1
Message count:      10
Total duration:     0.964641 ms
Average duration:   95.885000 us
Minimum duration:   12.781000 us
Maximum duration:   198.429000 us
Standard deviation: 64.535748 us
Latency P50:        95.564000 us
Latency P90:        198.429000 us
Latency P99:        198.429000 us
Message rate:       10367 msg/s
Message rate:       0.010 MB/s
CPU usage:          100.05%
=====================================

============ RESULTS ================
Message size:       1
Message count:      10
Total duration:     0.368585 ms
Average duration:   36.424000 us
Minimum duration:   10.056000 us
Maximum duration:   144.228000 us
Standard deviation: 47.568417 us
Latency P50:        10.950000 us
Latency P90:        144.228000 us
Latency P99:        144.228000 us
Message rate:       27131 msg/s
Message rate:       0.026 MB/s
CPU usage:          100.08%
=====================================

============ RESULTS ================
Message size:       1
Message count:      100
Total duration:     2.511233 ms
Average duration:   24.920000 us
Minimum duration:   13.192000 us
Maximum duration:   177.167000 us
Standard deviation: 19.376391 us
Latency P50:        20.511000 us
Latency P90:        33.142000 us
Latency P99:        177.167000 us
Message rate:       39821 msg/s
Message rate:       0.038 MB/s
CPU usage:          100.02%
=====================================

============ RESULTS ================
Message size:       1
Message count:      100
Total duration:     3.897635 ms
Average duration:   38.536000 us
Minimum duration:   23.504000 us
Maximum duration:   180.517000 us
Standard deviation: 20.848000 us
Latency P50:        35.748000 us
Latency P90:        46.991000 us
Latency P99:        180.517000 us
Message rate:       25657 msg/s
Message rate:       0.024 MB/s
CPU usage:          100.02%
=====================================

============ RESULTS ================
Message size:       1
Message count:      200
Total duration:     2.767041 ms
Average duration:   13.721000 us
Minimum duration:   10.204000 us
Maximum duration:   245.304000 us
Standard deviation: 20.446596 us
Latency P50:        11.070000 us
Latency P90:        11.985000 us
Latency P99:        143.376000 us
Message rate:       72279 msg/s
Message rate:       0.069 MB/s
CPU usage:          100.01%
=====================================
```