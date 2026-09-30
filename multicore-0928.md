# 进程导向的用户态中断的可行性分析

## 为什么要实现进程导向

Linux中对用户态中断的注册和接收都是线程导向(thread directly)，即描述接收方接收能力的UPID数据结构struct uintr_upid_ctx直接挂载在线程描述符task_struct->thread->upid_ctx。

用户态中断只会路由给注册该中断的线程，线程退出后，用户态中断相应地无法被响应处理。

```c
struct uintr_upid_ctx {
    struct list_head node;
    struct task_struct *task;   /* 这个 UPID 对应的接收任务 */
    u64 uvec_mask;              /* 哪些向量已注册（按位） */
    struct uintr_upid *upid;    /* 指向上面的硬件 UPID */
    refcount_t refs;            /* 引用计数 */
    bool receiver_active;       /* 是否当前仍有接收者 */
    bool waiting;               /* 是否当前在阻塞等待 */
    unsigned int waiting_cost;  /* 阻塞开销由谁承担（sender/receiver） */
};

struct uintr_upid {
    struct {
        u8 status;     /* bit 0: ON, bit 1: SN, 其余保留/扩展 (含 BLKD) */
        u8 reserved1;
        u8 nv;         /* Notification vector */
        u8 reserved2;
        u32 ndst;      /* Notification destination (APIC ID) */
    } nc __packed;     /* Notification control */
    u64 puir;          /* Posted user interrupt requests (64-bit bitmap) */
}
```

在用户态中断作为外部事件源驱动的异步运行时中，线程导向的用户态中断面临两个问题。

(1)多核情况下，线程资源的动态分配与回收问题。用户态中断在只能由注册该中断的线程接收和处理，因此为了避免中断事件的丢失，注册了中断的线程空闲时也不能退出，不利于资源的动态分配与回收。

(2)协程作为运行的基本单位，其被调度运行后所处的线程是不固定的。负责注册用户态中断和协程和响应中断的协程若不运行在同一个线程会导致中断无法正确唤醒阻塞等待的线程。

## 实现方式

### 方案一 - 接收方退出由同组的其他线程接管用户态中断

接收方退出由同组的其他线程接管用户态中断，需要处理的问题是由谁负责接管。

如果有任务树的概念，那么应该由上一级任务负责接管。但是Linux中目前没有任务树的概念，其次，Linux线程组中的线程都是平级的概念（只有一个leader属性，但是不保证leader线程是最后退出的）。接管线程如果注册过用户态中断，那么还会涉及到两个中断向量空间的合并问题以及发送方的UITE所指向的UPID地址如何更新的问题。

### 方案二 - 线程组共享用户态中断

同一个线程组的所有线程共享中断向量空间和UPID数据结构uintr_upid_ctx，用户态中断处理函数。

组内任意一个线程注册后，所有成员都具备接收和处理用户态中断的能力（实际只选择一个线程负责响应）。

即便线程组内原本注册中断的线程被切出，只要其他线程处于运行态也能够即时响应用户态中断。




Note: uintr.c 的 TODO:UPID 分配器尚不兼容 KPTI