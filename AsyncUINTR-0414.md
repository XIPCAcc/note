# 1. GRUB配置（输出日志到串口）

sudo vim /etc/default/grub
```bash 
GRUB_CMDLINE_LINUX="console=ttyS0,115200n8"
```

sudo update-grub

cat /proc/cmdline

# 动态调试

```bash
## 挂载 debugfs
if ! mountpoint -q /sys/kernel/debug; then
    echo "挂载 debugfs..."
    sudo mount -t debugfs none /sys/kernel/debug
fi

## 启用 pr_debug
echo "启用 eventpoll pr_debug 日志..."
sudo sh -c 'echo -n "file fs/eventpoll.c +p" > /sys/kernel/debug/dynamic_debug/control'
sudo sh -c 'echo 8 > /proc/sys/kernel/printk'
```

sudo dmesg | grep -i epoll 可以查看

串口母头转USB连接到windows上位机

所有使用到epoll_wait 的服务都将打印log

[ 1256.083305] epoll: X86_FEATURE_UINTR is enabled
[ 1256.083305] epoll: CONFIG_X86_UINTR_BLOCKING is enabled
[ 1256.083305] epoll: current task is not a user interrupt receiver
[ 1256.083306] epoll: using regular epoll_wait implementation
[ 1256.083309] epoll: epoll_wait called, epfd=4, maxevents=40, timeout=-1
[ 1256.083309] epoll: X86_FEATURE_UINTR is enabled
[ 1256.083310] epoll: CONFIG_X86_UINTR_BLOCKING is enabled
[ 1256.083310] epoll: current task is not a user interrupt receiver

[  816.474631] epoll: epoll_wait called, epfd=7, maxevents=70, timeout=-1

主要看epfd 是否和测试程序对得上


# 内核如何处理CPL=0时的用户态中断
- 用户态处理 （Ring 3）：当处理器在用户态执行时，用户态中断会直接使用 MSR（Model Specific Registers）中定义的用户态处理程序
- 内核态处理 （Ring 0）：当处理器在内核态执行时，用户态中断会通过 IDT（Interrupt Descriptor Table）表进行处理，确保内核的安全性和控制

虽然从代码里面读起来是这样，但是253668-091-sdm-vol-3a.pdf Chapter 9 并没有明确定义
```c
static void uintr_switch_to_kernel_interrupt(struct uintr_upid_ctx *upid_ctx)
{
	unsigned long flags;

    // 这里切换到内核态的时候，会把nv从原本的注册时候的nv，变成内核定义的用户态中断处理中断号
	upid_ctx->upid->nc.nv = UINTR_KERNEL_VECTOR;
	upid_ctx->waiting = true;
	spin_lock_irqsave(&uintr_wait_lock, flags);
	list_add(&upid_ctx->node, &uintr_wait_list);
	spin_unlock_irqrestore(&uintr_wait_lock, flags);
}

#ifdef CONFIG_X86_USER_INTERRUPTS
/*
 * Handler for UINTR_NOTIFICATION_VECTOR.
 *
 * The notification vector is used by the cpu to detect a User Interrupt. In
 * the typical usage, the cpu would handle this interrupt and clear the local
 * apic.
 *
 * However, it is possible that the kernel might receive this vector. This can
 * happen if the receiver thread was running when the interrupt was sent but it
 * got scheduled out before the interrupt was delivered. The kernel doesn't
 * need to do anything other than clearing the local APIC. A pending user
 * interrupt is always saved in the receiver's UPID which can be referenced
 * when the receiver gets scheduled back.
 *
 * If the kernel receives a storm of these, it could mean an issue with the
 * kernel's saving and restoring of the User Interrupt MSR state; Specifically,
 * the notification vector bits in the IA32_UINTR_MISC_MSR.
 */
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
	if (IS_ENABLED(CONFIG_X86_UINTR_BLOCKING))
		uintr_wake_up_process();

}

/*
 * Handler for UINTR_KERNEL_VECTOR.
 */
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
#endif
```

9.1 introduction
Such user interrupts will
be delivered after receipt of an ordinary interrupt (identified in the UPID) called a user-interrupt notification. 1
System software can define operations to post user interrupts and to send user-interrupt notifications. 

UINV: user-interrupt notification vector.This is the vector of the ordinary interrupts that are treated as user-interrupt notifications (Section 9.5.1).When the logical processor receives user-interrupt notification, it processes the user interrupts in the user posted-interrupt descriptor (UPID) referenced by UPIDADDR (see below and Section 9.5.2).

UPID 23:16 Notification vector Used by agents sending user-interrupt notifications (including SENDUIPI).

The local APIC is acknowledged; this provides the processor core with an interrupt vector, V.2. If V = UINV, the logical processor continues to the next step. Otherwise, an interrupt with vector V is delivered normally; the remainder of this algorithm does not apply and user-interrupt notification processing does not occur.

如果 V != UINV：那么这个中断就是一个普通中断，按正常流程通过IDT处理。“用户态中断通知处理不会发生”。

如果 V == UINV：那么CPU核心认出这是一个“用户态中断通知”。此时，它不会去执行IDT中对应的中断服务例程，而是跳过普通中断处理，转去处理UPID中记录的用户态中断。