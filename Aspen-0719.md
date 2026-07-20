# Ksched内核模块

ksched 内核模块的作用是通过注册一个字符设备/dev/ksched驱动，为系统提供线程的抢占调度和唤醒功能。

```c
ksched/ksched.c          # /dev/ksched 设备驱动实现
```

最主要是ksched_ioctl函数和ksched_idle函数。

## ksched_ioctl

```c
ksched_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	switch (cmd) {
	case KSCHED_IOC_START:
		return ksched_start();
	case KSCHED_IOC_PARK:
		return ksched_park();
	case KSCHED_IOC_INTR:
		return ksched_intr((void __user *)arg);
	}

}
```

### ksched_start

所有的kthread启动后，会调用ioctl(ksched_fd, KSCHED_IOC_START, 0) 向/dev/sched 发送KSCHED_IOC_START请求。

ksched_ioctl对于KSCHED_IOC_START请求，则调用ksched_start，设置线程状态为TASK_INTERRUPTIBLE，并schedule。

```c
static long ksched_start(void)
{
	/* put this task to sleep and reschedule so the next task can run */
	__set_current_state(TASK_INTERRUPTIBLE);
	mark_task_parked(current);
	schedule();
	__set_current_state(TASK_RUNNING);
	return get_granted_core_id();
}
```

为什么刚创建的kthread要马上睡眠？

因为刚创建出来额kthread 会被内核根据调度策略随机分配CPU执行。

睡眠后，等待IO kernel分配专属的核，schedule() 返回后，线程已经跑在 IOKernel 指定的核心上，之后永远只在分配的核上跑，方便IO kernel管理和回收CPU资源。


### ksched_park

runtime kthread通过 ioctl(KSCHED_IOC_PARK) 触发，主动让出 CPU 并等待新任务。

```c
static long ksched_park(void)
{

    // 获取当前kthread运行在哪个核心，并禁用抢占
	cpu = get_cpu();
    // 获取当前核心的 ksched_percpu per-CPU 数据结构
    // struct ksched_percpu {
    //     unsigned int		last_gen; last_gen 就像一个确认信号：让ksched告诉IO kernel"上次的指令执行完毕"。
    //     local_t			busy; 此核心是否有 kthread 在跑
    //     u64			last_sel; 上次读取性能计数器的选择寄存器值
    //     struct task_struct	*running_task; 当前在此核心上跑的 kthread
    // };
	p = this_cpu_ptr(&kp);
    // 每个CPU有一个 ksched_shm_cpu per-CPU 数据结构
    // struct ksched_shm_cpu {
    //     /* written by userspace */
    //     unsigned int		gen; 充当版本号 每次 IOKernel 决定改变某个核心上跑的 kthread 时， gen++。gen 的单调递增保证了每次切换都有唯一标识。共享内存 shm[core].gen IOKernel 写，内核读（决定是否切换）
    //     pid_t			tid; 在该核心上运行的线程 TID
    //     unsigned int		mwait_hint; mwait 的 hint 值（控制 C-state 深度）
    //     unsigned int		sig; 信号代际号，和gen一样，记录发出信号请求那一刻的调度代数，而不是signum是否有变化，其值还是gen，用来表明发送信号新网发送给哪个正在运行的线程，如果这个线程被调度走了，那么gen就会发生变化，sig就会不相等，这个sig就作废。为什么不复用gen？两者恰好都用 ksched_gens[core] 做 单调递增计数器 （共享同一个命名空间），但它们的递增时机不同、触发条件不同。IOKernel 可以只发信号而不改 tid，也可以只改 tid 而不发信号，复用同一个 gen 会让这两种操作无法区分。
    //     unsigned int		signum; 信号编号（SIGUSR1=cede, SIGUSR2=yield）
    //     unsigned int		pmc; 性能计数器请求代际号
    //     __u64			pmcsel; 性能计数器选择寄存器值

    //     /* written by kernelspace */
    //     unsigned int		busy; 核心是否忙碌（有 kthread 在跑）
    //     unsigned int		last_gen; 当前在跑的 kthread 属于哪个调度代数"
    //     __u64			pmcval;
    //     __u64			pmctsc;

    //     /* extra space for future features (and cache alignment) */
    //     unsigned long		rsv[1];
    // };
    // 该内存由 ksched 申请，IO kernel通过调用ksched_mmap驱动映射到用户态空间
	s = &shm[cpu];

    // 因为即将park，所以先标记核心处于不忙状态
	local_set(&p->busy, false);

    // 在入睡前检查是否有信号等着处理
	if (unlikely(signal_pending(current))) {
		local_set(&p->busy, true);
		put_cpu();
		return -ERESTARTSYS;
	}

	/* check if a new request is available yet */
	gen = smp_load_acquire(&s->gen);
	if (gen == p->last_gen) {
        // 写入共享内存，告诉IO kernel 处理器核不忙
		WRITE_ONCE(s->busy, false);
        // 传入 0 表示"没有找到下一个任务"——因为没有新的调度请求，自然没有下一个任务可运行
		ksched_next_tid(p, cpu, 0);
		put_cpu();
		goto park;
	}

	/* determine the next task to run */
	tid = READ_ONCE(s->tid);
	p->last_gen = gen;

	// 从共享内存读出的 s->tid 等于当前 kthread 的 PID。这意味着 IOKernel 希望 当前线程继续运行 ，不需要切换到其他 kthread。
	if (tid == task_pid_vnr(current)) {
		WRITE_ONCE(s->busy, true); // 向共享内存写入 busy = true ，告知 IOKernel 该 CPU 忙碌 。这样 IOKernel 的快速路径（fast pass）在看到 busy 为 true 时不会重复唤醒该核心。
		local_set(&p->busy, true);
		smp_store_release(&s->last_gen, gen);
		put_cpu();
		return smp_processor_id();
	}

    // ksched_next_tid找到目标线程，调用wake_up_process "标记可运行 + 入队"
	ksched_next_tid(p, cpu, tid);
	WRITE_ONCE(s->busy, p->running_task != NULL);
	local_set(&p->busy, p->running_task != NULL);
	smp_store_release(&s->last_gen, gen);
	put_cpu();

park:
	/* put this task to sleep and reschedule so the next task can run */
	__set_current_state(TASK_INTERRUPTIBLE);
	mark_task_parked(current);

    // 切换上下文，切到新的线程运行
    // 虽然schedule() 不知道要切到谁，但是Aspen只让该 CPU 上有一个可运行线程，调度器自然就只能选它。靠"CPU 专用化 + 亲和性 + 状态切换"三者配合保证的，这是一种典型的"用约束代替通知"的设计模式。
	schedule();
	__set_current_state(TASK_RUNNING);
	return get_granted_core_id();
}
```

### ksched_intr

ksched_intr 由 IOKernel 调用，用于 向一组CPU 同步注入 IPI，让每个目标 CPU 执行 ksched_ipi 。

IOKernel 无法直接从用户态向 kthread 发送信号。它走迂回路线：在共享内存中设置 sig （要投递的信号 gen）和 signum （信号编号），然后通过 IPI 让目标 CPU 上的 ksched_ipi 调用 ksched_deliver_signal 代为投递。相当于目标CPU收到ipi后再自己给自己发送信号。

SIGUSR1 KSCHED_INTR_CEDE 抢占 ：IOKernel 要求该核心上的 runtime 线程立即交出 CPU，让给更高优先级的 kthread
SIGUSR2 KSCHED_INTR_YIELD 让出 ：IOKernel 建议该核心上的 runtime 线程主动让出 CPU

为什么不能直接发送信号？因为IPI 机制让信号投递和调度状态（busy、gen） 在同一个同步点（目标 CPU 上）一起判断 ，保证了调度一致性。

```c
static long ksched_intr(struct ksched_intr_req __user *ureq)
{
    // struct ksched_intr_req { 从用户态（IOKernel）向内核态（ksched）传递 批量 CPU 中断 (IPI) 请求 的数据结构。 IOKernel 在共享内存中更新完调度信息（gen、tid）后，将需要通知的 CPU 号累积到 ksched_set 位图中，最后通过一次 ioctl(KSCHED_IOC_INTR) 批量通知所有目标 CPU
	//     size_t			len;  // CPU 掩码的字节长度
	//     const void __user	*mask; // 指向用户态 CPU 位图的指针
    // };
	struct ksched_intr_req req;

	// 只有 IOKernel 进程 （以 root 或具备 CAP_SYS_ADMIN 运行）才能调用
	if (unlikely(!capable(CAP_SYS_ADMIN)))
		return -EACCES;

    // 目标 CPU 收到 IPI 后执行 ksched_ipi() 函数
	smp_call_function_many(mask, ksched_ipi, NULL, false);
}

static void ksched_ipi(void *unused)
{
	cpu = get_cpu();
	p = this_cpu_ptr(&kp);
	s = &shm[cpu];

	/* check if a signal has been requested */
	tmp = smp_load_acquire(&s->sig);
    // 注意这里比较的是sig 和 last_gen，必须是确保sig发送给原本期望发送给的正在运行线程，sig有IO kernel发送的时候是等于gen的，这个线程没有发生过调度，last_gen才会等于sig。
    // 如果不等，这个信号作废，因为没有必要抢占了。
	if (tmp == p->last_gen) {
		ksched_deliver_signal(p, READ_ONCE(s->signum));
		smp_store_release(&s->sig, 0);
	}

	put_cpu();
}
```
