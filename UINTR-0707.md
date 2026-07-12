# 用户态中断的路由

关于用户态中断是路由给线程还是路由给核的问题。

原本的设想是哪个线程注册了用户态中断，就路由给哪个线程，即中断跟核走。

从tokio的实验现象来看，好像也是这样。（也有可能是因为线程一直都运行在了同一个核上）


==========

从《Extended User Interrupts (xUI): Fast and Flexible  Notification without Polling》这篇论文的描述，中断是跟着核走的。

论据就是注册的使用使用UPID数据结构中保存的是APICID，用来路由用户态中断到指定的核。

```
Routing interrupts at user level. The interrupt system is responsible for routing messages from sources (devices, cores, timers) to destinations. Destinations are cores (addressed by APICID).
By design, routing is quite static — APICIDs are typically assigned to cores at startup time, and rarely change.

limitations of the existing interrupt routing scheme to overcome: (1) there was no notion of addressing threads—which are dynamically created, destroyed, and migrated between cores.
```

上面的理解错了，原文中的描述指的是传统的中断体系，而不是用户态中断。

UPID和UITT就是为了解决中断跟核走的问题。
```
To address these limitations, UIPI creates its own orthogonal virtual namespace (thread IDs) and vector space (a 6-bit user vector, or UV). To deliver interrupts in this new namespace, it introduces two new types of data structures that are shared in memory between cores—the per-process UITTs and per-thread UPIDs previously mentioned.
```

结论还是用户态中断跟注册的线程走

============

## NV和UV

NV 即 Notification Vector，当接收线程 不在 CPU（被切出）​ 时，发送方（或内核代理）要向 NDST核发的 传统 x86 IPI 所使用的向量号

UV即 User Vector，每个进程可定义 最多 64 个用户中断，发送方调用senduipi的时候根据UITT Entry中的UV指定。

Notification Destination (NDST) — bits 63:32，含义：该线程当前被认为正在（或最近在）运行的物理核的 APIC ID，发送端 / 代理用此决定 IPI 发往哪个 LAPIC，当线程被 context_switch()迁出：内核更新 NDST = new_cpu_apicid。

```
Bits     Field                  Description
─────    ──────────────────────  ─────────────────────────────────────────
23:16    Notification Vector (NV)
         → 用于投递 User Interrupt 的 *通知中断向量号*

63:32    Notification Destination (NDST)
         → 当前线程所在核的 APIC ID

    127:64   Posted Interrupt Requests (PIR)
         → 已 post 给该线程的 User Interrupt 位图（每 bit = 一个 User Vector）
```

# 系统调用

### 1. uintr_register_handler

执行完uintr_register_handler以后，改变的几个数据结构。

1. 内核会将中断处理函数的地址写入 MSR_IA32_UINTR_HANDLER，使得 CPU 在收到用户态中断时能跳转到该函数。
2. 内核会申请并构造upid，并将其地址写入MSR_IA32_UINTR_PD，
3. 将UINTR_NOTIFICATION_VECTOR写入MSR_IA32_UINTR_MISC -> UINV


```c
int uintr_register_handler(u64 handler_address, unsigned int flags);
```
uintr_register_handler()为调用进程注册一个用户中断处理程序。对于多线程进程，用户中断处理程序仅为发起此系统调用的线程进行注册。

handler_address是当进程接收到用户中断时将被调用的函数地址。该函数应按以下方式定义：

void __attribute__ ((interrupt)) ui_handler(struct __uintr_frame *frame,
                                                   unsigned long long vector)

// 以下内容由GCC定义
```c
struct __uintr_frame
{
unsigned long long rip;
unsigned long long rflags;
unsigned long long rsp;
};
```

进程内的每个线程都拥有自己独立的64个向量的中断向量空间。当用户中断被递送时，​向量号会被压入栈中。由于向量空间是每个线程独有的，因此每个接收方最多可以接收64个唯一的中断事件。

中断处理程序仅当进程正在运行时才会被调用。如果进程被调度出CPU或在内核中被阻塞，中断将在进程被重新调度时递送。

实现原理：
1. 内核会将该函数的地址写入 MSR_IA32_UINTR_HANDLER，使得 CPU 在收到用户态中断时能跳转到该函数。
2. 内核会申请并构造upid，并将其地址写入MSR_IA32_UINTR_PD，
3. 将UINTR_NOTIFICATION_VECTOR写入MSR_IA32_UINTR_MISC -> UINV

```
UPID格式
23:16 Notification vector. Used by agents sending user-interrupt notifications (including SENDUIPI).
63:32 Notification destination Target physical APIC ID – used by SENDUIPI.
127:64   Posted Interrupt Requests (PIR)
```

向量编号 23:16​
​向量号（Notification Vector Number）​当目标线程未运行时，发送方用此 x86 中断向量号向 Notification Destination（bits 63:32，APIC ID）发 IPI 唤醒内核代理。Linux采用 UINTR_KERNEL_VECTOR，表示接收线程处于休眠等待态时由内核接受用户态中断，此时用户态中断就是传统中断处理路径，

NDST 63:32​ 表示目标物理APIC ID，用于路由用户态中断时找到目标处理器。

vector space = 6-bit user vector (UV / UID)，用户中断编号 0–63（共 64 个）独立于 x86 APIC vector（0–255）

用于置PIR对应的bit，PIR[uv]


### 2. uintr_create_fd

执行完uintr_create_fd，改变的数据结构
1. 接收方的 task stuct -> uintr_receiver -> uintr_upid_ctx 存储upid指针
2. task stuct -> uintr_receiver -> uvec_mask 64-bit 掩码，每一位对应一个 User Vector (UID 0–63)，标记该线程"允许/已激活"哪些用户中断
3. 用户态中断fd，其私有数据能够指向uintr_receiver，方便发送方根据fd获取接收方的 upid

```c
int uintr_create_fd(u64 vector, unsigned int flags);
```

uintr_create_fd()为调用进程分配一个新的用户中断文件描述符（uintr_fd）​。uintr_fd可被共享给其他进程或内核，允许通过uintr_fd创建正确的UITT 表项以发送用户态中断。

每个线程拥有独立的64个向量（0-63）​，优先级从高到低（63最高，0最低）。应用可通过选择向量号实现中断优先级控制。​中断触发时，​向量号被压入栈，供处理程序识别来源。

一个线程可以多次调用 uintr_create_fd()，创建多个 fd，分别接收不同 UID（类似多路复用）。

vector—— User Vector（UID）就是 UID / User Vector / 6-bit vector

接收方可将同一 uintr_fd共享给多个发送方。此时由于所有发送方使用同一向量号，接收方需额外机制（参数）​精确识别中断来源。

实现原理：
1. 创建 uintrfd_ctx
2. 从struct task 拿到uintr_receiver 将其与uintrfd_ctx绑定ui_recv = t->thread.ui_recv; 
```c
struct uintr_receiver {
	struct uintr_upid_ctx *upid_ctx;
	u64 uvec_mask;	/* track active vector per bit */
};
```
vector 记录在uvec_mask
ui_recv->uvec_mask |= BIT_ULL(r_info->uvec);
3. 创建匿名inode的文件描述符，将uintrfd_ctx与 file->private_data绑定，方便 通过在文件操作中访问私有数据，

### 3. uintr_register_sender


执行完uintr_register_sender，改变的数据结构
1. 发送方的UITT（每个进程共享一个UITT），创建UITTE，写入发送方的UPID地址

```c
int uintr_register_sender(int uintr_fd, unsigned int flags);
```

uintr_register_sender()允许发送方进程通过 uintr_fd与接收方建立连接，并返回一个 ​UIPI索引（uipi_index，也是UITT表项的下标）​。发送方可通过该索引配合 SENDUIPI指令直接生成用户中断​，而无需内核介入。

实现原理：
1. 初始化uitt，存在task->ui_send->uitt_ctx->uitt
2. 根据 uintrfd 拿到receiver端使用uintr_create_fd 创建的文件，在拿到其中private_data 所指向的uintrfd_ctx，其中有uintr_upid等信息
3. 创建uitte 并根据接收方的uintrfd_ctx 中的中断向量信息填充内容
4. 返回uitte 下标


## senduipi

SENDUIPI imm8

接收一个寄存器，UISEL（User Interrupt Selector / UITTE Selector）是硬件用来从当前进程的 UITT 中选出具体 UITTE 的索引寄存器/上下文。此时默认 imm8 = 0


接收一个立即数，手册里面说是指定 User Interrupt Vector (UIV / UID)，即目标线程 UPID.PIR[bit]中置位的位，一般不写，默认使用UITTE.UV

```
IF reg > UITTSZ THEN #GP(0); FI;

tempUITTE := MEM[IA32_UINTR_TT + (reg << 4)];
IF tempUITTE.V = 0 OR tempUITTE reserves bits set THEN #GP(0); FI;

tempUPID := MEM[tempUITTE.UPIDADDR];   // 以 superviser 权限，原子锁
IF tempUPID 保留位非法 THEN #GP(0); 释放锁; FI;

// 永远先 post 用户中断向量
tempUPID.PIR[tempUITTE.UV] := 1;

// 判断是否需要发 Notification IPI
//    注意：只有 SN=0 AND ON=0 时才置 ON 并发 IPI
//    若 ON=1（线程在跑）→ 不发 ON 但发 IPI（由前一次 context-switch 已置 ON=1）
//    若 SN=1（被 suppress，如正被调试/暂停）→ 既不置 ON 也不发 IPI
IF (tempUPID.SN = 0 AND tempUPID.ON = 0) THEN
    tempUPID.ON := 1;
    sendNotify := 1;
ELSE IF (tempUPID.SN = 0 AND tempUPID.ON = 1) THEN
    sendNotify := 1;       // PIR 已置，ON=1 → 只发 IPI，不改动 ON
ELSE                          // SN=1
    sendNotify := 0;       // suppress，不 IPI
FI;

MEM[tempUITTE.UPIDADDR] := tempUPID;   // 写回 UPID（含 PIR 和可能的 ON）
                                        // 释放锁

// 发普通物理 IPI（如有需要）
IF sendNotify = 1 THEN
    IF x2APIC mode
        send ordinary IPI with vector = tempUPID.NV
                         to 32-bit physical APIC ID = tempUPID.NDST;
    ELSE (xAPIC)
        send ordinary IPI with vector = tempUPID.NV
                         to 8-bit physical APIC ID = tempUPID.NDST[15:8];
FI;
```


## 编码速查表

| 汇编 | ModRM | imm8 | 机器码（64-bit REX.W） |
|---|---|---|---|
| `senduipi %eax` | C0 | 00 | `48 0F 01 C0 00` |
| `senduipi %ecx` | C1 | 00 | `48 0F 01 C1 00` |
| `senduipi %edx` | C2 | 00 | `48 0F 01 C2 00` |
| `senduipi %ecx` UID=1 | C1 | 01 | `48 0F 01 C1 01` |


# 理解senduipi

UPID格式
```
Offset  Size   Field        Description
────── ───── ──────────── ─────────────────────────────────────
00h    1 B    Status       bit0 = ON (Outstanding Notification)
                         bit1 = SN (Suppress Notification)
                         bit2‑7 reserved
01h    1 B    Reserved1
02h    1 B    NV           Notification Vector (UPID.NV)
03h    1 B    Reserved2
04h    4 B    NDST         Destination APIC ID (x2APIC format)
08h    8 B    PIR          Posted Interrupt Request (64‑bit bitmask)
10h~3Fh 56 B  Reserved     (may be used for software tags/debug)
```

UITT entry格式

```
位域 名称 说明

bit 0 V（Valid） 1=该条目有效；bits 7:1 保留且须为 0

bits 15:8 UV（User Vector） 用户中断向量，取值范围 0–63（故 bits 15:14 必须为 0）

bits 63:16 保留（必须为 0）

bits 127:64 UPIDADDR 目标线程的 User Posted Interrupt Descriptor（UPID）的线性地址（64 字节对齐，低 6 位必为 0）
```

senduipi的动作，设置接收方的UPID.PIR[UITTE.UV] = 1，置UPID.ON 为1。

（1）如果接收方处于运行态而且CPL=3，此时一般SN = 0，那么senduipi 会发送一个IPI到 UPID.NDST 对应的核，中断号为UPID.NV，然后接收方执行 UPID.PIR -> UIRR 寄存器。

接收方核的APIC 根据当前的IA32_UINTR_MISC.UINV 是否等于UPID.NV，决定这个IPI当成普通中断，还是用户态中断。

当CPU检测到 CPL == 3 && UIF == 1 && UIRR != 0 时候，跳转到用户态 ui handler。执行 ui handler 时候， 硬件置 UIF = 0。执行完后，硬件置 UPID.ON = 0，UPID.PIR[UITTE.UV] = 0，APIC硬件执行EOI。

（2）如果接收方处于运行态而且CPL=0，此时一般SN = 0，那么senduipi还会发送一个IPI到 UPID.NDST 对应的核，，然后接收方执行 UPID.PIR -> UIRR 寄存器。

但是 CPU检测到 CPL == 0，把用户态中断当成普通中断执行，跳转到IDT DEFINE_IDTENTRY_SYSVEC(sysvec_uintr_notification)，里面只做EOI。

等到从内核态返回用户态，iret后 CPL变为3，CPU检测到 CPL == 3 && UIF == 1 && UIRR != 0 时候，触发（1）的流程

（3）如果接收方处于阻塞态，也就是接收方schedule，context_switch了，此时接收方不在核上运行。内核在contex_switch的路径中把 UPID.SN = 1，senduipi依然设置接收方的UPID.PIR[UITTE.UV] = 1，置UPID.ON 为1。但是因为SN=1，所以不发送 IPI。等到接收方再次schedule回来，再次设置UPID.SN = 0，同时检查PIR，不为0，则发送一个self IPI，触发（2）的流程。

```c
/**
 * switch_uintr_prepare - called on context switch OUT of a UINTR receiver
 * @prev: task being switched out
 */
void switch_uintr_prepare(struct task_struct *prev)
{
  
    /*
     * Suppress Notifications while thread is not on any CPU.
     * SENDUIPI will still atomically set PIR.
     */
    set_bit(UINTR_UPID_STATUS_SN,      // bit 1
            (unsigned long *)&rcv->upid_ctx->upid->nc.status);

    /* Optional: clear ON to indicate no outstanding IPI */
    clear_bit(UINTR_UPID_STATUS_ON,
              (unsigned long *)&rcv->upid_ctx->upid->nc.status);
}


void switch_uintr_return(void)
{
    /*
     * Allow Notifications again — thread back on CPU.
     * This clears SN (bit1).
     */
    clear_bit(UINTR_UPID_STATUS_SN,
              (unsigned long *)&upid_ctx->upid->nc.status);  // ★ SN=0

    /* 恢复 NDST 为本核 */
    upid_ctx->upid->nc.ndst = cpu_to_ndst(smp_processor_id());

    /* 若切出期间有 post，补触发 PIR→UIRR */
    if (READ_ONCE(upid_ctx->upid->puir) != 0)
        apic->send_IPI_self(UINTR_NOTIFICATION_VECTOR);
}
```

（4）目前唯一解释不通的就是CONFIG_UINTR_BLOCKING开启用户态中断唤醒系统调用机制是如何做到进程阻塞被切出，但是senduipi能够唤醒阻塞的接收方进程的。

猜想是发送方发送了一个IPI，让接收方核收到后把它当成普通中断执行，然后唤醒的接收方。

比如 uintr_wait的实现里面，特意在切出之前把upid的NV改成 UINTR_KERNEL_VECTOR，接收方核的APIC 收到IPI后，发现和IA32_UINTR_MISC.UINV 设置的UINTR_NOTIFICATION_VECTOR不同，决定这个IPI当成普通中断，然后执行IDT DEFINE_IDTENTRY_SYSVEC(sysvec_uintr_kernel_notification)，里面会调用uintr_wake_up_process()。

但是矛盾点在于，接收方切出以后，SN=1，发送方是不会发送IPI的。而且从实验来看，普通的系统调用阻塞也能够被senduipi唤醒。但是普通系统调用没有设置UINTR_KERNEL_VECTOR。

```c
int uintr_receiver_wait(void)
{
    
    /*  临时把 UPID.NV 改成内核向量 */
    upid_ctx->upid->nc.nv = UINTR_KERNEL_VECTOR;

    set_current_state(TASK_INTERRUPTIBLE);
    schedule();   // → switch_uintr_prepare() 置 SN=1

    return 0;
}

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

解释这个疑问
```c
/* Suppress notifications since this task is being context switched out */
void switch_uintr_prepare(struct task_struct *prev)
{
	
	/*
	 * A task being interruptible is a dynamic state. Need synchronization
	 * in schedule() along with singal_pending_state() to avoid blocking if
	 * a UINTR is pending
	 */
	if (IS_ENABLED(CONFIG_X86_UINTR_BLOCKING) &&
	    is_uintr_waiting_enabled(prev) &&
	    task_is_interruptible(prev)) {
		if (!is_uintr_waiting_cost_sender(prev)) {
			uintr_switch_to_kernel_interrupt(upid_ctx);
			return;
		}

		uintr_set_blocked_upid_bit(upid_ctx);
	}

	set_bit(UINTR_UPID_STATUS_SN, (unsigned long *)&upid_ctx->upid->nc.status);
}
```

## switch_uintr_prepare() 中 SN 的设置逻辑

会设置 SN 的情况 （set_bit(UINTR_UPID_STATUS_SN, ...) ）：

以下 所有条件不满足 时，才走到设置 SN：

1. CONFIG_X86_UINTR_BLOCKING 未启用， 或者
2. waiting_cost == UPID_WAITING_COST_NONE （即等待未启用）， 或者
3. prev 任务 不可中断 （非 TASK_INTERRUPTIBLE 状态）

不会设置 SN 的情况 （提前 return）：

当三个条件 全部满足 时（ CONFIG_X86_UINTR_BLOCKING 已启用 && 等待已启用 && 任务可中断），再根据 waiting_cost 细分

| `waiting_cost` | 动作 | 是否设置 SN |
|---|---|---|
| `UPID_WAITING_COST_NONE` | 不进入 if 分支 | **设置 SN**（走第 1356 行） |
| 非 `UPID_WAITING_COST_SENDER`（即 receiver 侧等待） | 调用 `uintr_switch_to_kernel_interrupt()` | **不设置 SN**，直接 return |
| `UPID_WAITING_COST_SENDER` | 调用 `uintr_set_blocked_upid_bit()` | **不设置 SN**，直接 return |


uintr_switch_to_kernel_interrupt和uintr_set_blocked_upid_bit两者都用于任务可中断睡眠时的 UINTR 等待，但机制完全不同：

## `uintr_switch_to_kernel_interrupt()` — Receiver 侧等待

1. **修改通知向量**：把 UPID 的 `nv` 从用户态通知向量改为 `UINTR_KERNEL_VECTOR`，让硬件中断路由到**内核**
2. **加入等待链表**：挂到 `uintr_wait_list`，内核收到中断后可以据此唤醒任务
3. **不设 SN，不设 BLKD**：通知不被抑制，也不阻塞，而是**重定向到内核**处理

核心思想：中断到来时，硬件仍会产生通知，只是送到内核而非用户态，内核负责唤醒睡眠的 receiver。

## `uintr_set_blocked_upid_bit()` — Sender 侧等待

1. **设置 BLKD 位**：在 UPID status 中置 `UINTR_UPID_STATUS_BLKD`
2. **标记 waiting**：`upid_ctx->waiting = true`
3. **不加入链表，不改 nv**：不需要内核介入

核心思想：sender 发送 UIPI 时，如果目标被 BLKD，硬件会将发送者挂起（等待），直到目标 receiver 恢复后清除 BLKD 位。这是利用硬件本身的阻塞-等待机制。

## 关键区别

| | Receiver 侧等待 | Sender 侧等待 |
|---|---|---|
| 机制 | 中断重定向到内核 | 硬件 BLKD 位阻塞 |
| 谁等待 | **内核替 receiver 等**，收到中断后唤醒 receiver | **硬件让 sender 等**，receiver 恢复后 sender 自动继续 |
| 数据结构 | 需要内核链表追踪 | 无需内核追踪 |
| 开销 | 内核中断处理 + 唤醒 | 硬件自动挂起 sender |
| 适用场景 | receiver 睡眠时希望中断能唤醒它 | sender 频繁发送，不希望 receiver 被唤醒的开销 |

          
BLKD 是 UPID（User Interrupt Posted Descriptor）status 字段中的一个位：

| 位 | 名称 | 含义 |
|---|---|---|
| bit 0 | **ON** | Outstanding Notification — 有待处理的通知 |
| bit 1 | **SN** | Suppressed Notification — 抑制通知（任务被切换出时设置，避免无用中断） |
| bit 7 | **BLKD** | **Blocked** — 阻塞，表示 receiver 在等待内核处理 |

**BLKD 的作用**：当 receiver 任务可中断睡眠且 `waiting_cost == SENDER` 时，内核设置 BLKD 位。此时硬件的 UIPI 发送机制会：
- 不向 receiver 发送中断（receiver 在睡眠，发了也白发）
- **将 sender 挂起**（硬件自动阻塞 sender 的 UIPI 指令），直到 receiver 恢复后内核清除 BLKD 位

BLKD (bit 7) 是 Linux 内核自己占用了 status 字段中的保留位，用作软件标志。证据如下：

- 触发方式 ： SENDUIPI 指令遇到 BLKD 位时产生的是 #GP（General Protection Fault） ，然后内核在 #GP handler（ traps.c:533 ）中用软件方式处理——清除 BLKD、设置 ON、唤醒 receiver。这是纯软件模拟，不是硬件行为。

- 代码中的 TODO 也印证了这一点—— traps.c:523 注释 /* TODO: Confirm: Can we come here because of a GP not related to UPID Blocked? */ ，说明内核也不确定 #GP 是否全是因为 BLKD，因为这个位并非硬件规范的一部分。

- 注释说"Blocked waiting for kernel" ，而非引用任何 Intel 规范章节。