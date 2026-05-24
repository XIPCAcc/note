## `uintr_wait` 系统调用分析：用户态中断打断后，是否会执行用户态中断处理程序？

当 `uintr_wait` 被用户态中断打断后，用户态中断处理程序最终会被执行。但它不是在系统调用执行期间立即执行（因为 CPU 处于内核态，无法接收用户中断），而是在系统调用返回用户态时执行（返回时会重新给自己发送一个用户态中断，具体要看重新发送的用户态中断什么时候到达）。

---

### 完整流程

整个流程涉及内核中两条路径的协作：

#### 步骤 1：进入等待 (handler.c → kernel)

`uintr_wait` 系统调用进入 [uintr_receiver_wait](file:///home/zwp/uintr-linux-kernel/arch/x86/kernel/uintr.c#L1236-L1267)：

```c
int uintr_receiver_wait(ktime_t *expires)
{
    hrtimer_init_sleeper_on_stack(&t, CLOCK_MONOTONIC, HRTIMER_MODE_REL);
    hrtimer_set_expires_range_ns(&t.timer, *expires, 0);
    hrtimer_sleeper_start_expires(&t, HRTIMER_MODE_REL);

    set_current_state(TASK_INTERRUPTIBLE);   // 设置为可中断睡眠
    if (t.task)
        schedule();                          // 调度出去，进入睡眠

    // 醒来后：
    hrtimer_cancel(&t.timer);
    __set_current_state(TASK_RUNNING);

    return !t.task ? 0 : -EINTR;  // t.task为NULL=超时返回0；否则被中断返回-EINTR
}
```

关键细节：`t.task` 是 `hrtimer_sleeper` 结构中的字段。如果定时器到期自然唤醒，`t.task` 被置为 `NULL`，返回 0；如果被其他方式唤醒（如收到中断），`t.task` 非 `NULL`，返回 `-EINTR`。

#### 步骤 2：上下文切换时重定向中断 (switch_uintr_prepare)

当任务调度出去时，[switch_uintr_prepare](file:///home/zwp/uintr-linux-kernel/arch/x86/kernel/uintr.c#L1331-L1357) 被调用，关键操作在 [uintr_switch_to_kernel_interrupt](file:///home/zwp/uintr-linux-kernel/arch/x86/kernel/uintr.c#L1301-L1311)：

```c
static void uintr_switch_to_kernel_interrupt(struct uintr_upid_ctx *upid_ctx)
{
    upid_ctx->upid->nc.nv = UINTR_KERNEL_VECTOR;  // 把通知向量改成内核向量
    upid_ctx->waiting = true;
    list_add(&upid_ctx->node, &uintr_wait_list);   // 加入等待链表
}
```

**核心设计**：把 UPID 中的通知向量 (NV) 从 `UINTR_NOTIFICATION_VECTOR`（用户态中断向量）改为 `UINTR_KERNEL_VECTOR`（内核中断向量）。这样当中断到达时，硬件会将其投递为**内核中断**而非用户中断。

#### 步骤 3：中断到达时的内核处理

当发送方发送中断时，`uintr_notify_receiver` 看到 SN 位未设置、ON 位未设置，于是发送 IPI，但 NV 已被改为 `UINTR_KERNEL_VECTOR`，所以内核中断处理程序 [sysvec_uintr_kernel_notification](file:///home/zwp/uintr-linux-kernel/arch/x86/kernel/irq.c#L388-L399) 被触发，进而调用 [uintr_wake_up_process](file:///home/zwp/uintr-linux-kernel/arch/x86/kernel/uintr.c#L1536-L1556)：

```c
void uintr_wake_up_process(void)
{
    list_for_each_entry_safe(upid_ctx, tmp, &uintr_wait_list, node) {
        if (test_bit(UINTR_UPID_STATUS_ON, ...)) {
            set_bit(UINTR_UPID_STATUS_SN, ...);           // 设置SN，防止重复通知
            upid_ctx->upid->nc.nv = UINTR_NOTIFICATION_VECTOR; // 恢复用户态通知向量
            upid_ctx->waiting = false;
            set_tsk_thread_flag(upid_ctx->task, TIF_NOTIFY_SIGNAL);
            wake_up_process(upid_ctx->task);               // 唤醒任务
            list_del(&upid_ctx->node);
        }
    }
}
```

这里做了几件关键事：
1. 设置 **SN (Suppress Notification)** 位，抑制后续通知
2. 将 NV **恢复为** `UINTR_NOTIFICATION_VECTOR`
3. 唤醒任务

#### 步骤 4：系统调用返回 -EINTR

任务被唤醒后，从 `schedule()` 返回。由于 `t.task` 非 `NULL`（是 `wake_up_process` 唤醒的，不是定时器到期的），`uintr_receiver_wait` 返回 `-EINTR`。

在用户态代码中表现为：`uintr_wait` 返回 `false`（因为 result == -1，而 `0 == 0` 为 false）。

#### 步骤 5：返回用户态时触发用户态中断处理程序

当系统调用完成、CPU 准备返回用户态时，`switch_uintr_return` 被调用：

```c
void switch_uintr_return(void)
{
    // ...
    clear_bit(UINTR_UPID_STATUS_SN, ...);   // 清除SN位，重新允许通知

    // 如果UPID中有待处理的中断(PUIR非零)，发送自IPI
    if (READ_ONCE(upid->puir))
        apic->send_IPI_self(UINTR_NOTIFICATION_VECTOR);
}
```

此时：
- `SN` 位被清除，允许中断投递
- `puir` 中有发送方写入的中断向量
- 发送自 IPI，向量为 `UINTR_NOTIFICATION_VECTOR`（用户通知向量）
- CPU 进入用户态后，该 IPI 被投递 → **用户态中断处理程序（`server_ui_handler` / `client_ui_handler`）被执行！**

---

### 时序总结

```
用户态                   内核态                    硬件
  │                       │                        │
  ├─ uintr_wait() ──────► │                        │
  │                       ├─ schedule() 睡眠        │
  │                       ├─ NV=KERNEL_VECTOR ────► │ (重定向)
  │                       │                        │
  │                    ◄──├─ 内核通知中断 ◄───────── │ (中断到达)
  │                       ├─ wake_up_process()      │
  │                       ├─ NV=USER_VECTOR ──────► │ (恢复)
  │                       ├─ 返回 -EINTR            │
  │  ◄── 返回 false ──────┤                        │
  │                       ├─ switch_uintr_return()  │
  │                       ├─ clear SN               │
  │                       ├─ send_IPI_self() ─────► │
  │                       │                        │
  │  ◄── 用户中断 ────────┤ ◄─────────────────────── │
  │  ├─ ui_handler()      │                        │
```

### 结论

**用户态中断处理程序一定会执行。** 它不在 `uintr_wait` 系统调用**之中**执行（因为 CPU 在内核态无法接收用户中断），而是在系统调用**返回之后**、CPU 重新进入用户态时执行。`uintr_wait` 返回 `-EINTR` 只是告诉你"有中断来了"，实际的 handler 执行被推迟到了返回用户态的时刻。