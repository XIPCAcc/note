以PS2 键盘为例，分析ctrl-c如何终止当前进程

PS2键盘按下后，8042芯片会向CPU发送一个中断，中断好为IRQ1，中断处理函数为i8042_interrupt。

```c
error = request_irq(I8042_KBD_IRQ, i8042_interrupt, IRQF_SHARED,
                    "i8042", i8042_platform_device);   // I8042_KBD_IRQ = 1

```

中断处理函数为i8042_interrupt调用serio_interrupt交给 serio 总线层
```c
static irqreturn_t i8042_interrupt(int irq, void *dev_id) {
    str = i8042_read_status();          // 读端口 0x64
    data = i8042_read_data();           // 读端口 0x60，取出扫描码字节
    port_no = (str & I8042_STR_AUXDATA) ? I8042_AUX_PORT_NO   // 鼠标
                                        : I8042_KBD_PORT_NO;  // 键盘
        serio_interrupt(serio, data, dfl);   // 交给 serio 总线层

}
```

serio_interrupt调用atkbd_interrupt负责把 PS/2 键盘扫描码翻译成Linux按键事件

```c
static irqreturn_t atkbd_interrupt(struct serio *serio, unsigned char data,
				   unsigned int flags)

{
    keycode = atkbd->keycode[code];
    input_event(dev, EV_KEY, keycode, value);
}

```

input_event()是 整个输入子系统中，所有设备驱动上报事件的唯一统一入口。
最终调用的是input_pass_values将按键值传给具体的handler。
```c
static void input_pass_values(struct input_dev *dev,
			      struct input_value *vals, unsigned int count)
{
	handle = rcu_dereference(dev->grab);
	if (handle) {
		count = input_to_handler(handle, vals, count);
	} else {
		list_for_each_entry_rcu(handle, &dev->h_list, d_node)
			if (handle->open) {
				count = input_to_handler(handle, vals, count);
				if (!count)
					break;
			}
	}
}

static unsigned int input_to_handler(struct input_handle *handle,
			struct input_value *vals, unsigned int count)
	if (handler->events)
		handler->events(handle, vals, count);
	else if (handler->event)
		for (v = vals; v != vals + count; v++)
			handler->event(handle, v->type, v->code, v->value);
}

static struct input_handler kbd_handler = {
	.event		= kbd_event,
	.match		= kbd_match,
	.connect	= kbd_connect,
	.disconnect	= kbd_disconnect,
	.start		= kbd_start,
	.name		= "kbd",
	.id_table	= kbd_ids,
};
```

负责ctrl-C的是VT控制台的kbd_handler，所以input_to_handler调用kbd_handler的kbd_event。

| handler | 文件 | 作用 |
|---|---|---|
| `kbd_handler` | drivers/tty/vt/keyboard.c | `.event = kbd_event`—按键翻译成字符给 VT 控制台（Ctrl+C） |
| `evdev` | drivers/input/evdev.c | `.events = evdev_events`—把事件包写进每个打开者的环形缓冲，供用户态 `read()` |


```c
static void kbd_event(struct input_handle *handle, unsigned int event_type,
		      unsigned int event_code, int value)
{
	if (event_type == EV_KEY && event_code <= KEY_MAX)
		//  Ctrl+C 映射为 ASCII ETX
		kbd_keycode(event_code, value, kbd_is_hw_raw(handle->dev)); 

	tasklet_schedule(&keyboard_tasklet);
	do_poke_blanked_console = 1;
	schedule_console_callback();
}
```

kbd_event() 调用 kbd_keycode() 解析字符，然后kbd_keycode()调用put_queue()将字符写入TTY 输入缓冲，然后退出中断上下文，剩下的交给内核线程完成。
```c
static void put_queue(struct vc_data *vc, int ch)
{
    tty_insert_flip_char(&vc->port, ch, 0);
	// 将内容写入tty_buffer，将work struct 挂到queue_work，然后退出中断上下文
    tty_flip_buffer_push(&vc->port);
}
```

## kworker

内核线程kworker 获取到worklist 里面的work struct，执行flush_to_ldisc()。

```c

static void flush_to_ldisc(struct work_struct *work)
{
	struct tty_port *port = container_of(work, struct tty_port, buf.work);
	struct tty_bufhead *buf = &port->buf;

	mutex_lock(&buf->lock);

	while (1) {
	...
		rcvd = receive_buf(port, head, count);
		
	}

	mutex_unlock(&buf->lock);

}
```

flush_to_ldisc()调用 行规程的receive_buf()，receive_buf()调用VT 控制台回调tty_port_default_receive_buf()—>n_tty_receive_buf2()->n_tty_receive_buf_common()->__receive_buf()->n_tty_receive_buf_standard()->n_tty_receive_char_special()

```c

static void __receive_buf(struct tty_struct *tty, const unsigned char *cp,
			  const char *fp, int count)
{
...
		if (count > la_count)
			n_tty_receive_buf_standard(tty, cp, fp, count - la_count, false);


	if (read_cnt(ldata)) {
		// 唤醒tty->read_wait等待队列上的所有任务
		kill_fasync(&tty->fasync, SIGIO, POLL_IN);
		wake_up_interruptible_poll(&tty->read_wait, EPOLLIN | EPOLLRDNORM);
	}
}


static void n_tty_receive_buf_standard(struct tty_struct *tty,
		const unsigned char *cp, const char *fp, int count, bool lookahead_done)
{
		if (test_bit(c, ldata->char_map))
			// 特殊符号走这里
			n_tty_receive_char_special(tty, c, lookahead_done);
}


static void n_tty_receive_char_special(struct tty_struct *tty, unsigned char c,
				       bool lookahead_done)
{
	

	if (L_ISIG(tty)) {
		if (c == INTR_CHAR(tty)) {
			// 处理 ctrl-C
			n_tty_receive_signal_char(tty, SIGINT, c);
			return;
		} else if (c == QUIT_CHAR(tty)) {
			n_tty_receive_signal_char(tty, SIGQUIT, c);
			return;
		} else if (c == SUSP_CHAR(tty)) {
			n_tty_receive_signal_char(tty, SIGTSTP, c);
			return;
		}
	}
	put_tty_queue(c, ldata);
}

```

n_tty_receive_signal_char()调用isig()挂载信号
```c
static void
n_tty_receive_signal_char(struct tty_struct *tty, int signal, unsigned char c)
{
	isig(signal, tty);
	.....
}

static void __isig(int sig, struct tty_struct *tty)
{
	// 取该 tty 的 前台进程组id
	struct pid *tty_pgrp = tty_get_pgrp(tty);
	if (tty_pgrp) {
		kill_pgrp(tty_pgrp, sig, 1);
		put_pid(tty_pgrp);
	}
}
```

## kill_pgrp

```c
int kill_pgrp(struct pid *pid, int sig, int priv)
{
	int ret;

	read_lock(&tasklist_lock);
	ret = __kill_pgrp_info(sig, __si_special(priv), pid);
	read_unlock(&tasklist_lock);

	return ret;
}

int __kill_pgrp_info(int sig, struct kernel_siginfo *info, struct pid *pgrp)
{
	retval = -ESRCH;
	// 遍历进程组中的每个进程 （比如管道会串起来三个进程）
	do_each_pid_task(pgrp, PIDTYPE_PGID, p) {
		int err = group_send_sig_info(sig, info, p, PIDTYPE_PGID);
		success |= !err;
		retval = err;
	} while_each_pid_task(pgrp, PIDTYPE_PGID, p);
	return success ? 0 : retval;
}

int group_send_sig_info(int sig, struct kernel_siginfo *info,
			struct task_struct *p, enum pid_type type)
{

	rcu_read_lock();
	ret = check_kill_permission(sig, info, p);
	rcu_read_unlock();

	if (!ret && sig)
		ret = do_send_sig_info(sig, info, p, type);

	return ret;
}

int do_send_sig_info(int sig, struct kernel_siginfo *info, struct task_struct *p,
			enum pid_type type)
{
	unsigned long flags;
	int ret = -ESRCH;

	if (lock_task_sighand(p, &flags)) {
		ret = send_signal_locked(sig, info, p, type);
		unlock_task_sighand(p, &flags);
	}

	return ret;
}


static int __send_signal_locked(int sig, struct kernel_siginfo *info,
				struct task_struct *t, enum pid_type type, bool force)
{
...
	// 选 pending 队列, 进程定向信号（Ctrl+C 选 shared_pending 线程组共享队列)
	// 线程定向信号 pthread_kill 选 t->pending 私有队列
	pending = (type != PIDTYPE_PID) ? &t->signal->shared_pending : &t->pending;
	
	// 信号去重，已经有同类信号未处理直接退出
	if (legacy_queue(pending, sig))
		goto ret;

	// 跳过SIGKILL和内核线程，因为不需要设置位图操作
	// ctrl-c发送SIGINT
	if ((sig == SIGKILL) || (t->flags & PF_KTHREAD))
		goto out_set;

	// 分配一个sigqueue节点
	q = __sigqueue_alloc(sig, t, GFP_ATOMIC, override_rlimit, 0);

	if (q) {
		list_add_tail(&q->list, &pending->list);
		switch ((unsigned long) info) {
	...
		case (unsigned long) SEND_SIG_PRIV:
			clear_siginfo(&q->info);
			
			q->info.si_signo = sig;
			q->info.si_errno = 0;
			q->info.si_code = SI_KERNEL;
			q->info.si_pid = 0;
			q->info.si_uid = 0;
			break;
		
		}
	}

out_set:
	// 如何信号创建了signalfd并注册了epoll，那么这里就唤醒epoll等待方
	signalfd_notify(t, sig);

	// 设置信号位图，t->signal->shared_pending，sigset_t signal 设置SIGINT
	sigaddset(&pending->signal, sig);

...
	complete_signal(sig, t, type);
ret:
	trace_signal_generate(sig, info, t, type != PIDTYPE_PID, result);
	return ret;
}

```


```c

static void complete_signal(int sig, struct task_struct *p, enum pid_type type)
{
	// 判断主线程是否适合处理信号
	if (wants_signal(sig, p))
		t = p;
	else if ((type == PIDTYPE_PID) || thread_group_empty(p))
		/*
		 * There is just one thread and it does not need to be woken.
		 * It will dequeue unblocked signals before it runs again.
		 */
		return;
	else {
		/*
		 * Otherwise try to find a suitable thread.
		 curr_target 是线程组级别的"上次信号发给了谁"的记忆指针，全组共享一个
		 */
		t = signal->curr_target;
		while (!wants_signal(sig, t)) {
			t = next_thread(t);
			if (t == signal->curr_target)
				/*
				 * No thread needs to be woken.
				 * Any eligible threads will see
				 * the signal in the queue soon.
				 */
				return;
		}
		signal->curr_target = t;
	}

	/*
	 * Found a killable thread.  If the signal will be fatal,
	 * then start taking the whole group down immediately.
	 */
	//  如果是 SIGKILL 类型的信号，需要停掉所有进程
	if (sig_fatal(p, sig) &&
	    (signal->core_state || !(signal->flags & SIGNAL_GROUP_EXIT)) &&
	    !sigismember(&t->real_blocked, sig) &&
	    (sig == SIGKILL || !p->ptrace)) {
		/*
		 * This signal will be fatal to the whole group.
		 */
		if (!sig_kernel_coredump(sig)) {
			/*
			 * Start a group exit and wake everybody up.
			 * This way we don't have other threads
			 * running and doing things after a slower
			 * thread has the fatal signal pending.
			 */
			// 本进程组正在集体退出
			signal->flags = SIGNAL_GROUP_EXIT;
			signal->group_exit_code = sig;
			signal->group_stop_count = 0;
			t = p;
			do {
				task_clear_jobctl_pending(t, JOBCTL_PENDING_MASK); // 清各线程作业控制待办
				sigaddset(&t->pending.signal, SIGKILL);            // 塞私有 SIGKILL
				signal_wake_up(t, 1);                              // 踢醒（fatal=1，可破 TASK_KILLABLE）
			} while_each_thread(p, t);
			return;
		}
	}

	/*
	 * The signal is already in the shared-pending queue.
	 * Tell the chosen thread to wake up and dequeue it.
	 */
	//  给线程挂TIF_SIGPENDING，并把它从睡眠里拉起来
	signal_wake_up(t, sig == SIGKILL);
	return;
}

static inline bool wants_signal(int sig, struct task_struct *p)
{
	// 没屏蔽信号
	if (sigismember(&p->blocked, sig))
		return false;
	// 没退出 
	if (p->flags & PF_EXITING)
		return false;
	// SIGKILL 直通
	if (sig == SIGKILL)
		return true;
	// 没停住
	if (task_is_stopped_or_traced(p))
		return false;

	// 此线程 正在 CPU 上运行信号给它响应最快，或者当前没有私有待办信号
	return task_curr(p) || !task_sigpending(p);
}

static inline void signal_wake_up(struct task_struct *t, bool fatal)
{
	unsigned int state = 0;
	if (fatal && !(t->jobctl & JOBCTL_PTRACE_FROZEN)) {
		t->jobctl &= ~(JOBCTL_STOPPED | JOBCTL_TRACED);
		state = TASK_WAKEKILL | __TASK_TRACED;
	}
	signal_wake_up_state(t, state);
}

void signal_wake_up_state(struct task_struct *t, unsigned int state)
{
	lockdep_assert_held(&t->sighand->siglock);

	// 设置TIF_SIGPENDING
	set_tsk_thread_flag(t, TIF_SIGPENDING);

	/*
	 * TASK_WAKEKILL also means wake it up in the stopped/traced/killable
	 * case. We don't check t->state here because there is a race with it
	 * executing another processor and just now entering stopped state.
	 * By using wake_up_state, we ensure the process will wake up and
	 * handle its death signal.
	 */
	//  唤醒任务
	if (!wake_up_state(t, state | TASK_INTERRUPTIBLE))
		kick_process(t);
}
```

## 信号转换为用户态中断

ctrl C 转换成为用户态中断的方案

__isig()作为最终识别ctrl-C向进程发送信号的函数，可以在此处获取接收信号的的进程信息，然后调用内核用户态中断发送函数uintr_notify_receiver()	向目标进程组的每个已注册线程发送一个用户态中断。

```c
static void __isig(int sig, struct tty_struct *tty)
{
	struct pid *tty_pgrp = tty_get_pgrp(tty);
	if (tty_pgrp) {
		kill_pgrp(tty_pgrp, sig, 1);
		/* 额外向前台进程组每个线程发一个 UINTR */
		uintr_notify_pgrp(tty_pgrp);
		put_pid(tty_pgrp);
	}
}

int uintr_notify_pgrp(struct pid *pgrp)
{
	struct task_struct *p;
	int notified = 0;

	if (!READ_ONCE(uintr_notify_isig_enabled))
		return 0;

	read_lock(&tasklist_lock);
	do_each_pid_task(pgrp, PIDTYPE_PGID, p) {
		if (uintr_kern_notify(p) == 0)
			notified++;
	} while_each_pid_task(pgrp, PIDTYPE_PGID, p);
	read_unlock(&tasklist_lock);

	pr_debug("uintr: notified %d task(s) in pgrp\n", notified);
	return notified;
}

int uintr_kern_notify(struct task_struct *task)
{
	struct uintr_upid_ctx *upid_ctx;
	int bit;

	if (!is_uintr_receiver(task))
		return -EINVAL;

	upid_ctx = task->thread.upid_ctx;
	if (!upid_ctx || !upid_ctx->receiver_active)
		return -EINVAL;

	// 接收线程uvec_mask中最低已注册位
	bit = find_first_bit((unsigned long *)&upid_ctx->uvec_mask,
			     UINTR_MAX_UVEC_NR);
	if (bit >= UINTR_MAX_UVEC_NR)
		return -EINVAL;

	pr_debug("notify: kernel notifying task %d uvec %d\n",
		 task->pid, bit);

	return uintr_notify_receiver(bit, upid_ctx);
}
EXPORT_SYMBOL_GPL(uintr_kern_notify);
```

创建一个 /proc/sys/kernel/uintr_notify_isig sysctl节点控制该功能的开关

```bash
cat /proc/sys/kernel/uintr_notify_isig

echo 1 | sudo tee /proc/sys/kernel/uintr_notify_isig   # 启用
echo 0 | sudo tee /proc/sys/kernel/uintr_notify_isig   # 禁用
```

运行后查看日志
```bash
echo 'file arch/x86/kernel/uintr.c +p' | sudo tee /sys/kernel/debug/dynamic_debug/control
运行程序后即可查看日志
sudo dmesg | grep 'uintr: notify'
```

### 测试程序

```c

#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>
#include <unistd.h>
#include <sys/syscall.h>
#include <errno.h>
#include <x86gprintrin.h>	/* __attribute__((interrupt)), struct __uintr_frame, _stui */

/* UINTR syscall numbers (x86_64), from arch/x86/entry/syscalls/syscall_64.tbl */
#ifndef __NR_uintr_register_handler
#define __NR_uintr_register_handler	471
#define __NR_uintr_unregister_handler	472
#define __NR_uintr_vector_fd		473
#endif

#define uintr_register_handler(h, f)	syscall(__NR_uintr_register_handler, h, f)
#define uintr_unregister_handler(f)	syscall(__NR_uintr_unregister_handler, f)
#define uintr_vector_fd(v, f)		syscall(__NR_uintr_vector_fd, v, f)

#define UVEC		0

static volatile sig_atomic_t sigint_fired;
static volatile unsigned int uintr_received;

/* ---- SIGINT handler ---- */
static void sigint_handler(int sig)
{
	sigint_fired = 1;
}

/* ---- UINTR handler (GCC emits prologue/epilogue incl. uiret) ---- */
static void __attribute__((interrupt))
uintr_handler(struct __uintr_frame *ui_frame, unsigned long long vector)
{
	uintr_received = 1;
}

static inline void cpu_delay(void)
{
	long long dl = 1000;
	volatile long long cnt = dl << 10;
	while (cnt--)
		dl++;
}

int main(void)
{
	struct sigaction sa;
	int uvec_fd;
	int wait_for = 1000000;

	printf("[info] pid=%d  Run in foreground, press Ctrl-C.\n", getpid());
	printf("[info] Make sure /proc/sys/kernel/uintr_notify_isig == 1\n");

	/* 1) Register UINTR handler */
	if (uintr_register_handler(uintr_handler, 0)) {
		fprintf(stderr, "[FAIL] uintr_register_handler: %s\n", strerror(errno));
		return 1;
	}
	printf("[info] uintr_register_handler OK\n");

	/* 2) Register vector 0 (creates uvec_fd / UPID) */
	uvec_fd = uintr_vector_fd(UVEC, 0);
	if (uvec_fd < 0) {
		fprintf(stderr, "[FAIL] uintr_vector_fd(%d): %s\n", UVEC, strerror(errno));
		uintr_unregister_handler(0);
		return 1;
	}
	printf("[info] uintr_vector_fd(%d) OK -> fd=%d\n", UVEC, uvec_fd);

	/* 3) Install SIGINT handler */
	sigemptyset(&sa.sa_mask);
	sa.sa_handler = sigint_handler;
	sa.sa_flags = 0;
	if (sigaction(SIGINT, &sa, NULL) < 0) {
		perror("sigaction");
		close(uvec_fd);
		uintr_unregister_handler(0);
		return 1;
	}
	printf("[info] SIGINT handler installed\n");

	/* 4) Enable user interrupts */
	_stui();
	printf("[info] UIF enabled (stui). Waiting for Ctrl-C...\n");

	/*
	 * Spin in user space with UIF=1 so the kernel-posted UINTR is
	 * delivered immediately. SIGINT also interrupts this loop.
	 */
	while (wait_for-- && !sigint_fired)
		cpu_delay();

	/* Re-sync UIF in case the signal path cleared it */
	_stui();

	printf("[result] SIGINT=%d  UINTR_received=%d\n",
	       sigint_fired, uintr_received);

	if (sigint_fired && uintr_received)
		printf("[PASS] both SIGINT and UINTR delivered\n");
	else if (sigint_fired && !uintr_received)
		printf("[FAIL] SIGINT delivered but NO UINTR "
		       "(is /proc/sys/kernel/uintr_notify_isig == 1?)\n");
	else
		printf("[FAIL] SIGINT not delivered (did you press Ctrl-C?)\n");

	close(uvec_fd);
	uintr_unregister_handler(0);
	return (sigint_fired && uintr_received) ? 0 : 1;
}

```