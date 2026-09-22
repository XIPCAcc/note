# 关于用户态中断运行时竞态的讨论

## 竞态问题的定义

```c
时间1

if (! queue.empty()) {
    continue
}

时间2

syscall uintr_wait() {

时间3

set TASK_INTERRUPTBLE

时间4

__switch_to() {

	switch_uintr_prepare(prev_p) {
	  -> uintr_switch_to_kernel_interrupt() {
时间5
		upid_ctx->upid->nc.nv = UINTR_KERNEL_VECTOR;
		list_add(&upid_ctx->node, &uintr_wait_list);
	  }
	}
}

时间6
}

```

在判断完queue.empty()后，__switch_to前发生的用户态中断都会导致竞态问题，即时间2之后，时间5之前发生的用户态中断。

竞态问题即原本用于唤醒进程的用户中断提前发生并且被处理消耗了，导致进程无法按期被唤醒的问题。

UINTR 硬件本身其实也没丢中断，中断请求被记在 UPID -> puir/ON 里，丢的是内核态睡眠任务的唤醒。

根据用户态中断到达的时间分几种情况讨论竞态问题

(1) 时间2和时间3之间

此阶段视为runtime已经判断完队列为空，准备调用syscall切换到内核态。由于这个过程不可能是原子的，有发生用户态中断的可能。

一旦发生了用户态中断，由于还没有完全切到内核态，runtime接收到用户态中断后，按照正常的用户态中断处理流程，执行用户态的用户态中断处理程序（注，简称中断处理程序1），将等到的协程唤醒。虽然此时队列不再为空，但是已经过了queue.empty()的时间，退出中断处理程序1后，仍会继续进入syscall yield()，导致竞态问题。

这种竞态问题的解决方法是通过CriticalSection关中断的方式，让用户态中断在完全切换到内核态之前关闭用户态中断。

(2)时间3

时间3过程中如果收到用户态中断，内核会执行内核态用户态中断处理程序sysvec_uintr_spurious_interrupt（注，简称中断处理程序2，[内核态中断处理程序](./UINTR-0707.md)），执行到最后会uintr_wake_up_process，但是此时由于进程还没有设置为TASK_INTERRUPTBLE，更没有挂到uintr_wait_list。如果原本是依赖这次中断唤醒的话，就会导致竞态。


```c
#define UINTR_NOTIFICATION_VECTOR       0xec
DECLARE_IDTENTRY_SYSVEC(UINTR_NOTIFICATION_VECTOR,	sysvec_uintr_spurious_interrupt);
INTG(UINTR_NOTIFICATION_VECTOR, asm_sysvec_uintr_spurious_interrupt),

DEFINE_IDTENTRY_SYSVEC(sysvec_uintr_spurious_interrupt)
{
	/* TODO: Add entry-exit tracepoints */
	ack_APIC_irq();
	inc_irq_stat(uintr_spurious_count);

	/*
	 * Typically, we only expect wake-ups to happen using the kernel
	 * notification. However, there might be a possibility that a process
	 * blocked while a notification with UINTR_NOTIFICATION_VECTOR was
	 * in-progress. This could result in a spurious interrupt that needs to
	 * wake up the process to avoid missing a notification.
	 *
	 * There might be an option to detect this wake notification earlier by
	 * checking the ON bit right before letting the task block. That needs
	 * further investigation. For now, leave it here for paranoid reasons.
	 */
	 // 上面的注释解释了为什么要执行这个唤醒操作
	//  存在一个竞态窗口：进程 正要阻塞（或正在阻塞的过程中），而一条带 0xec 的通知已经在路上了 。此时 nv 还没来得及被改成 0xeb（或通知在改写前已被 APIC 接收），这条中断就不会走 0xeb 的唤醒路径，而是以 0xec "spurious" 的形式进入内核
	// 如果 spurious 处理程序只做 EOI 不唤醒，任务就可能带着一条已到达的 pending 中断 永久睡下去 ——中断丢了。所以这里防御性地也调一次唤醒，补上这个窗口。
	// 作者提出一个理论上更"干净"的替代方案：在任务真正入睡之前检查 UPID 的 ON 位 （status bit 0，表示已有中断 pending）。若发现 ON=1 就不睡了，直接返回用户态处理，从源头避免漏通知，而不必靠 spurious 兜底。但这需要仔细论证与调度流程的同步，所以当时没做。
	// 因此暂时把唤醒留在这里——属于偏执式防御性编程：不是主路径，只为堵住难以完全证明的竞态。
	if (IS_ENABLED(CONFIG_X86_UINTR_BLOCKING))
		uintr_wake_up_process();

}

(3) 时间4到时间5

时间4到时间5期间，__schedule() 关中断之前，由于进程还没挂到等待队列上，所以导致即使收到用户态中断，也执行了中断处理程序2，但是依然无法唤醒。

(6) 时间6

时间6之后，用户态中断切换为UINTR_KERNEL_VECTOR，此时收到用户态中断执行的是内核态用户态中断处理程序sysvec_uintr_kernel_notification（注，简称中断处理程序3，功能看起来和中断处理程序2没差别），这个时候就能正常唤醒进程了。

#define UINTR_KERNEL_VECTOR		        0xeb
DECLARE_IDTENTRY_SYSVEC(UINTR_KERNEL_VECTOR,		sysvec_uintr_kernel_notification);
INTG(UINTR_KERNEL_VECTOR,		asm_sysvec_uintr_kernel_notification),
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
			set_tsk_thread_flag(upid_ctx->task, TIF_NOTIFY_SIGNAL);
			wake_up_process(upid_ctx->task);
			list_del(&upid_ctx->node);
		}
	}
	spin_unlock_irqrestore(&uintr_wait_lock, flags);
}
```

## 为什么传统的中断不会出现类似的问题

（1）传统中断的处理程序永远是同一个handler，中断也永远路由给固定的CPU，不依赖任何用户线程的调度状态。

所以不会出现情况1


（2）内核通过锁和condition状态位的结合保证睡眠不丢唤醒。

标准 `wait_event` 的结构是，

```c
include/linux/wait.h#L300-L322

for (;;) {
    long __int = prepare_to_wait_event(&wq, &__wq_entry, state);
    if (condition)          /* 复查：解锁之后 */
        break;
    if (___wait_is_interruptible(state) && __int)   /* 信号分支 */
        return __int;       /* -ERESTARTSYS */
    schedule();
}
finish_wait(&wq, &__wq_entry);

```

唤醒方（中断 handler）：

```c
condition = true;                 /* 只要求 store 先于 wake_up */
wake_up_interruptible(&wq);       /* 内部自取 wq 锁 */
```


对照 UINTR 原始代码，这个范式被破坏了四次：

| 范式要求 | UINTR 原始实现 |
|---|---|
| 先入等待队列，再置睡眠状态 | 任务在 `schedule()` → `__switch_to` 里才 `list_add`，远晚于 `set_current_state()` |
| 入队后在同一把锁内复查条件 | 入队后不复查 ON 位 |
| 条件置位与唤醒在同一把锁内 | 发送方置 ON 位不持 `uintr_wait_lock`，与入队/扫队列没有串行关系 |
| 漏一次唤醒后事件还能再触发 | ON=1 后后续发送方 `test_and_set_bit` 失败就抑制 IPI，没有第二次机会 |

和传统驱动里错误使用 waitqueue 导致的 lost wakeup 是同一类 bug。

## 设计方案

采用 uintr_wait() 系统调用在系统没有任务时让权。 

(1) 用户态中断注册时候必须使用 UINTR_HANDLER_FLAG_WAITING_ANY flag

(2) 进入临界区（判断 queue.empty 前）通过 CLUI 关闭用户态中断，uintr_wait 返回后才能重新开启。（一些仍未澄清的问题：UIF=0，CPL=3时。UPID-ON 位是否会置1？进入内核态后，UIF=0，CPL=0时，用户态中断是否会正常触发进入内核用户态中断处理程序？内核态中断处理程序退出后，是否会清理 UPID->ON位? 目前的推断为无论UIF和CPL的状态是什么，UPID->ON 总会置1，因为senduipi 的指令定义中没有提到这两个状态对ON位的影响，内核态中断处理程序退出后，硬件不会自动清楚ON）

(3) 修改uintr_wait

首先要支持无限期等待，原本的uintr_wait支持最长1000s定时等待。

其次，要修改其符合wait_event范式。