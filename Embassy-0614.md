# Embassy

Embassy 如何通过 waker获取到Task并修改器状态的

Waker 的 data 指针存的就是 *TaskHeader ，唤醒时直接 cast 回去。

Tokio 比 Embassy 复杂不少，因为它支持动态创建任务、多线程调度、取消、JoinHandle 等。


| | Embassy | Tokio |
|---|---|---|
| Waker data  | `*TaskHeader`（直接指针） | 指向堆分配的 task 结构的某种引用 |
| 任务存在哪 | 静态 `.bss` 段（编译期确定） | 堆（`Box`） |
| 需要引用计数 | 不需要 | 需要（`Arc`/`AtomicWaker`） |
| 反查方式 | 直接 cast | 通过自定义 vtable 间接调用 |

waker 唤醒：

```rust
// Waker::wake() 调用 vtable 里的函数
// 大致流程：
//   1. data → RawTask 指针
//   2. RawTask 的 vtable.schedule(self) 
//       → 把任务放回调度器队列
//       → 调度器线程被唤醒，poll 这个任务
```

Embassy 的假设条件比 Tokio 多：

1. 任务数量编译期已知 → 不需要 `Box`，`TaskStorage` 放 `.bss`
2. 单核（通常） → 不需要 `Arc` 引用计数，不需要 `AtomicWaker`
3. 没有 JoinHandle → 不需要考虑"有人拿着 handle 但任务已完成"的情况
4. 任务不能被随意取消 → 不需要复杂的生命周期管理

Tokio 要面对的场景是：任意线程可以 spawn 任务、任意线程可以持有 JoinHandle、任务可能被 abort、多线程同时竞争 waker……

能在中断里安全操作任务状态，靠的是三个设计。

1. 原子状态位（无锁）

TaskHeader::state的操作没有任何 mutex：

2. 无锁 RunQueue（链表栈）

RunQueue，中断里推入任务不需要等待：

如果用了 mutex，在中断上下文里等锁会死锁——因为中断可能抢占了持有锁的 Thread Mode 代码。无锁栈直接避免了这个问题。

Tokio 的 waker 依赖 Mutex / Condvar ，这些在中断上下文里可能导致死锁

3. 静态分配 = 所有指针永久有效

这是 Waker→Task 反查能成立的前提，也是中断里安全的关键。

```
Tokio（堆分配）:                  Embassy（静态分配）:

Task 在堆上                        TaskStorage 在 .bss
  │                                  │
  ├─ Arc::new(task)                   ├─ 地址编译期确定，永不变
  │  引用计数可能为 0 → drop             │  永不 drop
  │                                    │
  ├─ 中断里要读写 Ar？                  ├─ 中断里直接读 TaskHeader*
  │  需要原子的 refcount ops              │  没有 refcount，直接读
  │                                    │
  └─ task 可能已被释放                  └─ 不可能被释放
     要小心 UAF                           清除 SPAWNED 位 = "逻辑释放"
```

通过 CriticalSection 强制关中断后才访问Mutex 。
```
RTC1() 中断处理函数:
  │
  ├─ 清除硬件事件
  │
  ├─ critical_section::with(|cs| {
  │     //       ↑ 关中断（但其实已经在 Handler Mode，优先级低于它的已经进不来）
  │     //         关中断是为了防止更高优先级中断嵌套访问同一数据
  │     trigger_alarm(cs)
  │         → queue.borrow(cs) 
  │         → next_expiration()
  │         → waker.wake()
  │         → wake_task()
  │             → state::locked(|l| ...)
  │   })
  │
  └─ 中断返回

```
Async-Signal-Safe 要求信号处理程序中不能拿锁，

Mutex<RefCell<T>>会发生死锁。


thread 'main' (xxxx) panicked at tokio/tokio/src/runtime/scheduler/current_thread/mod.rs:662:40: RefCell already borrowed

比如刚好当前working thread 刚好拿到scheduler准备调度，然后来了一个用户态中断，进入wake 也准备借用scheduler就会出现这样的报错。
core.borrow_mut() [已借用]
RefCell 的 运行时借用冲突 ：中断处理函数打断了正在 borrow 的代码。

Embassy采用UnsafeCell
UnsafeCell 没有任何运行时检查 ， .get() 直接返回裸指针。这里靠的不是"借检查"，而是程序员+临界区来保证安全。

对于有原子指令的Embassy则直接采用原子指令。配置cordyceps crate，直接用里面实现的push_was_empty()
TransferStack（ Treiber栈无锁数据结构）

```
Embassy 的 RunQueue:
─────────────────────────────
  enqueue(): 
    head.fetch_and_set()     ← 原子指令，无借用状态
    不需要 borrow
    
  dequeue_all():
    head.swap(null)          ← 原子指令
    不需要 borrow

  → 无论谁、在什么时机调用 enqueue，都不会冲突


Tokio  的 RunQueue:
───────────────────────────────────
  schedule() → push:
    queue.borrow_mut()       ← 检查借用计数
      → 如果借用计数 > 0 → panic
    
  dequeue() → pop:
    queue.borrow_mut()       ← 检查借用计数
      → 如果借用计数 > 0 → panic

  → borrow_mut 和 borrow 不能并存
  ```

## 为什么 Tokio 不用原子链表

| | Embassy | Tokio current_thread |
|---|---|---|
| 环境 | 裸机，只有几个任务 | Linux，可能有数千任务 |
| 数据结构 | `AtomicPtr` 单链表 | `Vec<Arc<Task>>`（动态扩容） |
| 内存管理 | 静态分配，大小固定 | Arc 引用计数，可动态 spawn/drop |
| 遍历 | 单链表遍历 | Vec pop，连续内存，缓存友好 |

`Vec` 天然需要 `&mut self` 来 push/pop。Rust 的标准 `Mutex` 太重了，Tokio 选了 `RefCell` 做轻量保护——代价就是不能嵌套 borrow。
