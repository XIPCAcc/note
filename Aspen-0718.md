# Runtime

一个runtime就是一个程序，里面可以有多个kthread内核线程和多个uthread用户线程。

runtime编译生成libruntime.a，应用程序需要静态链接libruntime.a，然后在main中调用runtime_init。


```c
runtime/init.c           # runtime 初始化
runtime/sched.c          # 用户态线程调度
runtime/kthread.c        # kthread 管理（绑定到 CPU）
runtime/net/tcp.c        # TCP 协议栈
runtime/preempt.c        # 抢占机制
runtime/timer.c          # 定时器
```

##  runtime_init

runtime_init 接收真正的应用程序main函数运行。

解析配置文件，maxks代表runtime最大可创建的kthread数量。根据配置文件，创建kthread。

在timer_init_thread() 创建 timer_softirq uthread

在net_init_thread() 创建 iokernel_softirq

在storage_init_thread() 创建 storage_softirq

调用pthread_create 创建多个kthead线程，入口函数为pthread_entry，调用进入sched_start

调用ioqueues_register_iokernel 通过UDS 连接io kernel。

调用thread_spawn_main 创建main uthread，负责执行真正的main函数，并设置为主thread。

最后late_init_handlers 调用uintr_init_late 创建 uintr_timer pthread，启动抢占计时。

最后会调用sched_start。

所有的 kthread最终都会进入sched_start

```c
void sched_start(void)
{
	preempt_disable();
    // 切换到 runtime 给 kthread 分配的栈
    // 执行这行之前，kthread 跑在 pthread 创建时 GLIBC 分配的栈 上。执行之后切换到runtime_stack
    // kthread的栈完全给 调度器使用
    // 调度器做上下文切换时只需要保存/恢复 uthread 侧的寄存器，不需要担心调度器自己的栈帧被破坏
	jmp_runtime_nosave(schedule_start);
}


static __noreturn void schedule_start(void)
{
    // 获取struct kthread per thread 结构
	struct kthread *k = myk();
    // 调用ioctl阻塞当前thread线程
	kthread_wait_to_attach();
    // 被唤醒执行
    k->parked = false;
    // 进入调度器
	schedule();
}

struct kthread {
    // kthread_idx 是 Aspen 内部的数组下标
	uint32_t		kthread_idx;
    // Linux 内核的 PID
	pid_t			tid;
    // kthread 是否已 park
	bool			parked;

	// 就绪队列
	thread_t		*rq[RUNTIME_RQ_SIZE];
    // 队头队尾
	uint32_t		rq_head;
	uint32_t		rq_tail;

#ifdef PREEMPTED_RQ
    // 被抢占线程的专用就绪队列，优先级低于rq
    // 写 rq 的是线程自己，自己把自己 enqueue——这是个同步操作，不需要锁。写 preempted_rq 的是 softirq handler，而 softirq 在 schedule() 锁内执行——所以 preempted_rq 的写入是 在 kthread lock 保护下 完成的，写入者和读者是同一个执行路径的不同阶段。
    // 普通 rq 无额外状态。 preempted_rq 有配套的 is_preempted 字段
	thread_t        *preempted_rq[RUNTIME_RQ_SIZE];
#endif 

	thread_t		*iokernel_softirq; // 处理 IOKernel 消息的uthread
	thread_t		*timer_softirq;    // 处理定时器的uthread
	thread_t		*storage_softirq;  // // 处理存储 IO 完成的 uthead
    // 除了以上的三个一直都存在的uthread 需要三个指针直接定位，其他处于阻塞状态的uthread都是挂载在对应的等待对象上，比如等锁 mutex.m_write_waiters 链表，定时器 k->timers[]，等待 IO 完成 关联到存储完成队列


	// 每个 kthread 绑定一个 NVMe 硬件队列（queue pair）。uthread 发起的存储 IO 请求通过这个队列提交到 NVMe 设备，设备完成后通过 hq 返回完成事件。
	struct storage_q	storage_q;


};


static __noreturn __noinline void schedule(void)
{
	struct kthread *r = NULL, *l = myk();
	
	thread_t *th = NULL;
	ACCESS_ONCE(l->is_preempted) = -1; // 在调度器中， 没有 uthread 在跑 0代表有 uthread 在跑，且是 普通 的（未被抢占过），1代表有 uthread 在跑，但它是从 preempted_rq 取出的—— 之前被抢占过，现在恢复

	// 确保 preempt_disable 正确
	BUG_ON((perthread_read(preempt_cnt) & ~PREEMPT_NOT_PENDING) != 1);

	// DEFINE_PERTHREAD(thread_t *, __self); __self是一个 per-thread 变量，指向当前运行的 uthread struct thread
    // 把 thread_running 置为false
	store_release(&perthread_get_stable(__self)->thread_running, false);

	/* check for pending preemption */
    // IOKernel 在需要收回核心时，会在共享内存中设 cede_gen = curr_grant_gen 。相等时意味着 "IOKernel 要求你立刻交出核心" 
    // 
	if (unlikely(preempt_cede_needed(l))) {
		l->parked = true;
        // 调用 ioctl(ksched_fd, KSCHED_IOC_PARK, 0) 睡眠当前线程
		kthread_park_now();
		l->parked = false;
	}


	// rq队列非空
	if (l->rq_head != l->rq_tail)
		goto done;

    // preempted_rq 队列非空
	if (l->preempted_rq_head != l->preempted_rq_tail)
		goto done;

again:
	 // 本地 rq 和 preempted_rq 都空了，说明没有 uthread 要跑。此时 找软中断
	 // 分别检查IOKernel 软中断 IOKernel 往共享内存里的 LRPC 通道写入消息 → kthread 的 rxq 非空 → 激活
     // 定时器软中断
     // 存储软中断 uthread 发起 NVMe IO → 硬件完成后 DMA 写入完成队列 → hq 非空 → 激活。
     // 直接路径收包
     // 如果发生了通过thread_ready_head_locked 把对应的 uthread 推到 rq 队头，让 schedule() 下一次选到它
	if (softirq_run_locked(l)) {
		STAT(SOFTIRQS_LOCAL)++;
		goto done;
	}

	/* then try to steal from a sibling kthread */
	sibling = cpu_map[l->curr_cpu].sibling_core;
	r = cpu_map[sibling].recent_kthread;
	if (r && r != l && steal_work(l, r))
		goto done;

	/* try to steal from every kthread */
	start_idx = rand_crc32c((uintptr_t)l);
	for (i = 0; i < maxks; i++) {
		int idx = (start_idx + i) % maxks;
		if (ks[idx] != l && steal_work(l, ks[idx]))
			goto done;
	}

	}

	l->parked = true;

    // 没有任务运行，park
	kthread_park();
	l->parked = false;
	goto again;

done:
	if (l->rq_head != l->rq_tail) {
	    th = l->rq[l->rq_tail++ % RUNTIME_RQ_SIZE];
    } else {
        th = l->preempted_rq[l->preempted_rq_tail++ % RUNTIME_RQ_SIZE];
	}

jmp_thread(th);
}
```

## uintr timer

每个 runtime 进程都有自己独立的 uintr_timer 线程

用户态抢占（preemption）的三种机制：UINTR 硬件中断、Concord 标志、Signal。

核心是一个独立的 timer pthread，定期检查每个kthread 是否需要被抢占。

定时器线程 uintr_timer 是一个 独立的 Linux pthread，绑定到专用的 timer_core CPU。

```
timer_core 上的 pthread 主循环:
   │
   ├─ 遍历所有 kthread (i = 0..maxks-1)
   │     │
   │     ├─ 当前 uthread 运行时间未超 TIMESLICE → 跳过
   │     │
   │     └─ 超时 → 发送抢占信号:
   │           ├─ UINTR_PREEMPT:  _senduipi() → 硬件用户中断
   │           ├─ CONCORD_PREEMPT: 写标记位
   │           └─ SIGNAL_PREEMPT:  pthread_kill(SIGUSR1)
   │
   └─ loop 继续

              timer_core (独立 pthread)
              ┌─────────────────────────┐
              │  uintr_timer() 主循环    │
              │  rdtsc() 读取时间        │
              │  遍历 maxks 个 kthread  │
              │  超 TIMESLICE?           │
              │    ├─ UINTR: _senduipi() │
              │    ├─ Concord: 写标志    │
              │    └─ Signal: pthread_kill│
              └──────┬──────────────────┘
                     │
    ┌────────────────┼────────────────┐
    ▼                ▼                ▼
 kthread 0      kthread 1      kthread N-1
    │
    ├─ UINTR → ui_handler → thread_yield()
    ├─ Signal → signal_handler → thread_yield()
    └─ Concord → 检查标志 → thread_yield()
```


uintr_timer 是一个 独立的 Linux pthread ，绑在 timer_core 上， 持续运行不死循环 ：
```c
void* uintr_timer(void*) {
    while (uintr_timer_flag != -1) {
        current = rdtsc();                        // 不停读 TSC
        for (i = 0; i < maxks; ++i) {
            // - 检测 uthread 是否切换了
            // - 通过比较 last_check[i] 和 ks[i]->uthread_start_ts 来感知 uthread 切换（调度器在切换 uthread 时会更新 uthread_start_ts ）
            // - 当运行时间超过 TIMESLICE （由 uthread_quantum_us 配置），触发抢占
            long long start_ts = ks[i]->uthread_start_ts;
            if (last_check[i] < start_ts)
                last_check[i] = start_ts;         // 新 uthread → 重置计时

            if (current - last_check[i] < TIMESLICE)
                continue;                          // 未超时 → 跳过

            // 超时发送抢占信号
            _senduipi(uipi_index[i]);              // UINTR 硬件中断
        }
    }
}
```

```c
// UINTR 的处理函数
ui_handler(...) {
    if (preempt_enabled()) {
        thread_yield();        // 如果让抢占，直接yied，让出当前 uthread
    } else {
        set_upreempt_needed(); // 如果不让抢占， 设置set_upreempt_needed标志，等到运行抢占再yield，避免临界区。preempt_enable开启抢占，检测标志并决定当前uthread是非让出
    }
}
```

## uthread

### 创建与启动

| API | 说明 |
|-----|------|
| `thread_create(fn, arg)` | 创建 uthread，分配栈空间，初始化 trap frame，手动加入 runqueue |
| `thread_create_with_buf(fn, &buf, len)` | 同上，但栈顶预留一段 buffer 供 uthread 使用 |
| `thread_spawn(fn, arg)` | 创建并自动将 uthread 加入 runqueue（`thread_create` + `thread_ready`） |

`thread_spawn` 是最常用的高层接口，相当于创建并立即调度。

### 调度控制

| API | 说明 |
|-----|------|
| `thread_ready(th)` | 将 uthread 加入当前 kthread 的 rq 队尾，按 FIFO 被调度 |
| `thread_ready_head(th)` | 加入 rq 队头，优先被调度（用于软中断等紧急 uthread） |
| `thread_yield()` | 当前 uthread 主动让出 CPU，将自身加入 rq 队尾，跳回调度器 |
| `thread_preempt_yield()` | 同上，但加入的是 `preempted_rq`（被抢占队列），而非普通 rq |

### 阻塞与退出

| API | 说明 |
|-----|------|
| `thread_park_and_unlock_np(l)` | 释放自旋锁，阻塞当前 uthread（不加入任何队列），跳回调度器 |
| `thread_park_and_preempt_enable()` | 开启抢占并阻塞当前 uthread |
| `thread_exit()` | 终止当前 uthread，释放栈空间，跳回调度器，永不返回 |

### softirq

#### iokernel_softirq

iokernel_softirq_poll 是 IOKernel 软中断 uthread 的核心轮询函数 。

它从 IOKernel → Runtime 的 LRPC 通道（ k->rxq ）消费消息，处理网络数据包。

当 IOKernel 有数据到达时，调度器通过 softirq_run_locked 将其放进 rq 队头，它被调度执行后调用此函数。


```c
static void iokernel_softirq(void *arg)
{
	struct kthread *k = arg;

	while (true) {
        // 处理缓冲区的消息
		iokernel_softirq_poll(k);
		preempt_disable();
		k->iokernel_busy = false;
        // 将当前 uthread 阻塞并让出 CPU 的函
		thread_park_and_preempt_enable();
	}
}

static void iokernel_softirq_poll(struct kthread *k)
{
	while (true) {
		if (!lrpc_recv(&k->rxq, &cmd, &payload))
			break;

		switch (cmd) {
		case RX_NET_RECV:
			hdr = shmptr_to_ptr(&netcfg.rx_region,
					    (shmptr_t)payload,
					    MBUF_DEFAULT_LEN);
			m = net_rx_alloc_mbuf(hdr);
			if (unlikely(!m)) {
				STAT(DROPS)++;
				continue;
			}
			net_rx_one(m);
			break;

		case RX_NET_COMPLETE:
			mbuf_free((struct mbuf *)payload);
			break;

		case RX_REFILL_BUFS:
			BUG_ON(!net_ops.trigger_rx_refill);
			net_ops.trigger_rx_refill();
			break;

		default:
			panic("net: invalid RXQ cmd '%ld'", cmd);
		}
	}
}
```

runtime 通过轮询的方式读取共享缓冲区内容，轮询的时间点是每次schduler选择下一个uthread的循环，且优先级低于当前所有rq队列中的uthread。

如果当前runtime所有的kthread都处于阻塞态，iokernel_softirq如何触发?

IOKernel 有专门的唤醒机制，保证当所有 kthread 都 park 时，新数据到达能唤醒一个 kthread 来处理。

```
网卡收到数据包
  │
  ▼
dataplane_loop() [iokernel/main.c]
  │
  └─→ rx_burst() [iokernel/rx.c]
       │   rte_eth_rx_burst() 拉取数据包
       │
       └─→ rx_send_to_runtime() [iokernel/rx.c#L58]
            │
            ├─ sched_threads_active(p) > 0 ?
            │   │  flow_tbl 选一个活跃 kthread，直接 lrpc_send  + poll 通知
            │   │  → 目标 kthread 的 softirq_run_locked 检测到 → thread_ready_head(iokernel_softirq)
            │
            └─ sched_threads_active(p) == 0  ← 全部 park 了
                 │
                 ├─ sched_add_core(p)        ← 唤醒一个 kthread！
                 │     │
                 │     └─→ sched_ops->notify_core_needed()
                 │           │
                 │           └─→ [simple|ias]_add_kthread()
                 │                 │
                 │                 └─→ sched_run_on_core()      [iokernel/sched.c#L204]
                 │                       │
                 │                       ├─ sched_pick_kthread()        选最优 idle kthread
                 │                       │   (优先上次跑的核 → sibling核 → LRU)
                 │                       ├─ sched_enable_kthread()      标记 active
                 │                       └─ __sched_run() → ksched_run(core, tid)
                 │                            写共享内存: gen++, tid 
                 │
                 └─ 同时 lrpc_send(&th->rxq, ...) 写消息到 LRPC
IOkernel接收到数据会创建唤醒runtime

或者在slow pass 阶段
sched_poll() 被 main.c 的 dataplane_loop 每轮调用
  │
  ├─ [每 10us] slow pass
  │   ├─ sched_measure_delay(p)  ← 测量延迟 + 汇报拥塞
  │   └─ (触发 notify_congested → 可能 sched_add_core 唤醒 park 的 kthread)
  │
  ├─ [每轮] fast pass
  │   ├─ ksched_poll_run_done()      检查上下文切换是否完成
  │   ├─ ksched_poll_idle()          检测核心空闲
  │   └─ sched_try_fast_rewake()     尝试快速重唤醒
  │
  └─ [每轮] final pass
      └─ 调度策略决定 CPU 分配
- 调用周期 ： IOKERNEL_POLL_INTERVAL = 10 微秒 （ defs.h#L53 ），即每 10us 测量一次
- 调用范围 ：遍历所有 dataplane client（ dp.clients ），对每个 Runtime 进程调用一次
```

#### timer_softirq

```c
static void timer_softirq(void *arg)
{
	while (true) {
		preempt_disable();
		timer_softirq_one(k);
		k->timer_busy = false;
		thread_park_and_preempt_enable();
	}
}
```


#### storage_softirq

```c
void storage_softirq(void *arg)
{
	while (true) {
		preempt_disable();
		do {
			spin_lock(&q->lock);
			ret = storage_softirq_one(q);
			spin_unlock(&q->lock);
		} while (!preempt_needed() && ret > 0);
		k->storage_busy = false;
		thread_park_and_preempt_enable();
	}
}
```
