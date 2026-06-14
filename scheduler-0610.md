# vschdule

https://github.com/rosy233333/vsched2

vschduler 共享调度器会被编译成为一个vdso lib

## Scheduler

```rust
pub(crate) struct Scheduler {
    /// 事件源数组
    /// 事件源的指针为用户空间中的地址，因此在内核中访问时需要经过地址转换。
    sources: RwLock<Vec<(*const (), EventSourceVtable), EVENT_SORCE_NUM>>,

    /// 就绪队列
    ready_queue: ReadyQueue,
    /// trap等待队列。
    trap_wait_queue: TrapWaitQueue,
}


pub(crate) fn init(self_ref: Pin<&LazyInit<Self>>, global_index: usize) {
        let ready_queue = ReadyQueue::new();
        let trap_wait_queue = TrapWaitQueue::new();
```

### TrapWaitQueue
```rust

pub(crate) struct TrapWaitQueue {
    /// 当前核心收到的trap的数量 per-cpu的队列
    queues: [Mutex<Deque<(&'static TrapInfoVirtImpl, Option<&'static TaskVirtImpl>), TRAP_WAIT_QUEUE_SIZE>,> CPU_NUM],
    /// 每个核心上的trap处理任务
    handlers: [LazyInit<&'static TaskVirtImpl>; CPU_NUM],
}

/// 初始化trap处理任务
pub(crate) fn init(self: Pin<&Self>) {
    for cpuid in 0..CPU_NUM {
        ////////////***** 可以看到 handler 是通过调用vdso_helper提供的内核TrapInfo::new_handler函数创建出来的一个任务，根据实现可知vdso_helper中调用的trap_handler
            
        let handler = unsafe {
            TaskVirtImpl::from_ptr(TrapInfoVirtImpl::new_handler(
                &self.as_ref().queues[cpuid] as *const _ as *const (),
            ))
        };
        self.handlers[cpuid].init_once(handler);
    }
}


/// 在trap处理任务中运行的函数。
///
/// OS需在`TrapInfo::new_handler`的实现中，用这个函数创建trap处理任务。
/// 该函数的参数即为`new_handler`接口中传入的参数，即指向trap等待队列中某个核心的队列的指针。
#[unsafe(no_mangle)]
pub extern "C" fn trap_handler(queue: *const ()) {
    crate::schedule::trap_wait_queue::trap_handler(queue);
}

/// 在trap处理任务中运行的函数。
///
/// OS需在`TrapInfo::new_handler`的实现中，用这个函数创建trap处理任务。
/// 该函数的参数即为`new_handler`接口中传入的参数，即指向trap等待队列中某个核心的队列的指针。
///
/// 该函数只能通过api调用，不能直接调用。

/////////////// ************* 内核创建的任务一直在循环运行这个trap_handler，如果TrapWaitQueue 有任务，那么就去除任务并唤醒
#[inline]
pub(crate) fn trap_handler(queue: *const ()) {
    loop {
        if let Some((trap_info, task)) = queue.lock().pop_front() {
            trap_info.handle(task.map(|t| t.to_ptr()));
            if let Some(task) = &task {
                // 唤醒被trap的任务
                task.set_state(TaskState::Ready);
                //////// *** 设置完任务状态后，将其放回 ready queue
                /////// trap wait queue 只是负责暂存阻塞的任务，最终还是要放回ready queue等待调度
                (*scheduler).push_task(task).unwrap();
            }
        } else {
            // 没有trap，等待
            // 不需要存储Waker，因为总是可以从`TrapWaitQueue`中获取该任务。
            let task = get_current_task();
            task.set_state(TaskState::Blocked);
            ///////// ****** 如果没有任务可以被唤醒，那么其本身就进入睡眠状态
            task.resched();
        }
    }
}


impl EventSource for TrapWaitQueue {
    fn hightest_priority(&self, cpu_id: usize) -> isize {
        // 只要队列非空就返回ACTIVE_PRIORITY 最高优先级
        // 否则返回INACTIVE_PRIORITY
        if self.queues[cpu_id].lock().is_empty() {
            INACTIVE_PRIORITY
        } else {
            ACTIVE_PRIORITY
        }
    }

    fn take_task(&self, cpu_id: usize) -> (*const (), isize) {
        if self.queues[cpu_id].lock().is_empty() {
            (core::ptr::null(), INACTIVE_PRIORITY)
        } else {
            //// ******* 队列非空，那么就返回一个handler 任务
            /// take_task 不是直接从 queue 中拿任务的
            let handler = self.handlers[cpu_id].get().unwrap();
            (handler.to_ptr(), INACTIVE_PRIORITY)
        }
    }
}
```

### ReadyQueue

```rust
src\schedule\ready_queue.rs

pub(crate) struct ReadyQueue {
    /// index = priority - HIGHEST_PRIORITY
    //// 优先级队列，每个优先级有一个队列，同一优先级就FIFO
    queues: [Mutex<Deque<&'static TaskVirtImpl, READY_QUEUE_SIZE>> PRIORITY_LEVELS],
    /// 有效值：[2^(63-PRIORITY_LEVELS), 2^64-2^(63-PRIORITY_LEVELS)]
    prio_bitmap: AtomicU64,
}
```

## register_event_source

init的时候已经把ReadyQueue和ReadyQueue 放到schduler的source事件源中了，如果后续需要其他事件源（比如 Timer 事件源、特定 I/O 事件源），就调用 register_event_source 插入到指定位置。

pop_task 则会从事件源中获取最高优先级任务。

```rust
/// 从调度器中取出最高优先级的下一任务
///
/// 返回值：
///
/// - 就绪任务的指针，指向外部定义，实现`Task` trait的类型，若没有就绪任务则返回空指针；
/// - 取出就绪任务后事件源中就绪任务的最高优先级。
///     - 若没有事件源，则返回`isize::MAX`；
///     - 若有事件源但没有就绪任务，返回比最低优先级更低一级的优先级。
pub(crate) fn pop_task(&self) -> (Option<&TaskVirtImpl>, isize) {
    sources.iter()
        .map(|(ptr, vtable)| (vtable.hightest_priority)(*ptr, cpu_id))  // ①问每个事件源：你最优先的任务是什么级别？
        .enumerate()                                                      // ②附上索引
        .fold(..., |(first, second), current| {                          // ③找出最高的两个 priority
            if current.1 < first.1 {          // current 比 first 还高
                (current, first)              // 顶替 first，旧的 first 降为 second
            } else if current.1 < second.1 {  // current 比 second 高但不如 first
                (first, current)              // 只更新 second
            } else {
                (first, second)               // 不够格，忽略
            }
        })
         (Some(unsafe { TaskVirtImpl::from_ptr(task) }), prio)
```

## 入口函数

入口函数可以认为是 src\arch\x86.rs，raw_trap_entry（从内核态的异常处理程序跳转过来）

raw_trap_entry 调用 trap_entry，

```rust
vsched2\src\main_loop.rs

pub extern "C" fn trap_entry(trap_type: usize, privilege: usize) -> usize {
    match trap_type {
        // 外部中断，将当前任务重新放回就绪态后进入对应调度器。
        1 => {
          if privilege == 1 {
                // let new_stack_base = STACK_HANDLER.lock().alloc_stack().base;
                // sset_user_pre_stack!(new_stack_base);
                let current_task = get_current_task();
                current_task.set_state(TaskState::Ready);
                2
            } else {
                unreachable!("unknown privilege level: {privilege}")
            }
        }
}
```

回到汇编后根据返回值决定下一步跳转到哪里
```rust
        call trap_entry
        pop_2_arg
        cmp eax, 0
        je raw_trap_handle
        cmp eax, 1
        je raw_kschedule
        cmp eax, 2
        je raw_uschedule


    raw_uschedule:
        call uschedule
        pop_1_arg
        jmp raw_run_task

    raw_run_task:
        call run_task
        je raw_kschedule
        cmp edi, 1
        je raw_uschedule
```
### uschedule

```rust
pub extern "C" fn uschedule(stack_status: usize) {
    let scheduler = USER_SCHEDULER.get().unwrap();
    push_prev_task(scheduler);
    loop {
        let next_pid = process_schedule(scheduler);

        let res = utask_schedule(next_pid, stack_status);
        if res == 0 {
            break;
        }
    }
}
```

```rust
/// - 0：接下来调用run_task
/// - 1：未获取到任务，需要重新获取任务后重新调用utask_schedule。
fn utask_schedule(next_pid: usize, stack_status: usize) -> usize {
    let uscheduler = USER_SCHEDULER.get().unwrap();
    let current_pid = uscheduler.global_index();
    if next_pid == current_pid {
        // 从当前调度器获取下一任务并运行
        if let (Some(next_task), new_prio) = uscheduler.pop_task() {
            get_vvar_data!(PROCESS_INFO_TABLE).table[current_pid]
                .highest_prio
                .store(new_prio, Ordering::Release);
            // next_task.set_state(TaskState::Running);
            set_current_task(next_task);
            return 0;
        } else {
            return 1;
        }
    }
}
```

### run_task

run task 如果拿到了协程，最后通过跳板jump_to_trampoline跳转到 coroutine_trampoline，
coroutine_trampoline 最后跳转到run_coroutine
```rust
#[no_mangle]
pub extern "C" fn run_task(privilege: usize, stack_status: usize) -> usize {
    if get_current_task().is_coroutine() {
        // 切换或回收栈
        let new_sp = {
            let mut stack_handler = if privilege != 0 {
                STACK_HANDLER.lock()
            } else {
                get_vvar_data!(KERNEL_STACKS).lock()
            };
            stack_handler.get_empty_stack(stack_status)
        };
        jump_to_trampoline!(coroutine_trampoline, new_sp);
    } else {
        let thread_stack = { get_current_task().thread_stack_base() };
        {
            let mut stack_handler = if privilege != 0 {
                STACK_HANDLER.lock()
            } else {
                get_vvar_data!(KERNEL_STACKS).lock()
            };
            stack_handler.get_thread_stack(Some(thread_stack.into()), stack_status);
        };
        jump_to_trampoline!(thread_trampoline, thread_stack);
    }
    unreachable!();
}

```

### run_coroutine

```rust
#[no_mangle]
pub(crate) unsafe extern "C" fn run_coroutine() -> usize {
    get_current_task().set_state(TaskState::Running);
    let res = get_current_task().poll();
    // ************** 协程主动让权的入口 **************
    match res {
        Poll::Ready(val) => {
            get_current_task().set_return_value(val);
            get_current_task().set_state(TaskState::Exited);
        }
        Poll::Pending => {
            // 协程主动让权时，可能设置了任务状态也可能不设置。在不设置任务状态的情况，在此处设置为`Blocked`状态。
            if get_current_task().state() == TaskState::Running {
                get_current_task().set_state(TaskState::Blocked);
            }
        }
    }
    let in_kernel = {
        // get_current_task().save_thread_context();
        get_vvar_data!(IN_KERNEL)[SMPVirtImpl::cpu_id()].load(core::sync::atomic::Ordering::Acquire)
    };
    if in_kernel {
        0
    } else {
        1
    }
}
```