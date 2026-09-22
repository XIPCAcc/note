# 多核运行时的分析

## 王志文的用户态线程库

定义了两层M:N任务模型。实现basic_rt的用户态线程库和无栈协程库。

一个进程之上可以运行多个用户态线程（或者理解为有栈协程），一个用户态线程之上可以运行多个无栈协程。

用户态线程的调度器是Scheduler，无栈协程的调度器是Executor。两个调度器都是全局唯一，需要使用锁竞争的。

一个进程创建一个idle线程和一个线程池（初始化时只有一个线程），idle线程执行idle_main 作为Scheduler的调用者，以及创建新的线程。所有线程的入口为thread_main_ex，获取Executor，poll协程。

从设计上看，idle线程需要在当前所有线程都处于忙碌或者阻塞状态时，判断是否仍有协程需要被运行，如果是则创建出新的线程负责运行。

但是从实现上来看，第一线程执行的thread_main_ex 会一直轮询协程队列，知道协程队列为空时才会退出（看起来也不是真正的退出，没有找到线程被回收的代码）。即使是回到 idle中，如果是tick时间耗尽，也还是会继续被调度执行。

所以永远不会进入线程列表为空，但任务队列不空，创建一个线程的情况，那么就变成永远只有一个线程在运行协程了。


```rust
pub struct ProcessorInner {
    pub pool: Box<ThreadPool>,
    // idle就是主线程
    idle: Box<Thread>,
    current: Option<(Tid, Box<Thread>)>,
}

pub fn init_cpu_test() {
    let scheduler = RRScheduler::new(50);
    let thread_pool = Box::new(ThreadPool::new(10, scheduler));
    
    // 新建idle ，其入口为 Processor::idle_main
    let idle = Thread::new_box_thread(Processor::idle_main as usize, &CPU as *const Processor as usize);
    
    // 初始化线程池时先创建一个线程, 启动线程执行器之后可以直接使用
    CPU.add_thread(
    {
        let thread = Thread::new_box_thread(thread_main_ex as usize, 1);
        thread
    }
    );
}


    pub fn idle_main(&self) {
        let inner = self.inner();
        loop {
            // 如果从线程池中获取到一个可运行线程
            if let Some(thread) = inner.pool.acquire() {

                // 将自身的正在运行线程设置为刚刚获取到的线程
                inner.current = Some(thread);

                // 从正在运行的线程 idle 切换到刚刚获取到的线程
                println!("\n>>>> will switch_to thread {} in idle_main!", inner.current.as_mut().unwrap().0);

                // 保存正在运行idle_main函数的这个线程上下文到idle
                inner.idle.switch_to(
                    &mut *inner.current.as_mut().unwrap().1
                );

                // 上个线程时间耗尽，切换回调度线程 idle
                println!("<<<< switch_back to idle in idle_main!");

                // 此时 current 还保存着上个线程
                let (tid, thread) = inner.current.take().unwrap();
                
                // 通知线程池这个线程需要将资源交还出去
                inner.pool.retrieve(tid, thread);
            }
            // 如果现在并无任何可运行线程.则检查协程队列是否为空
            else {

                //let mut queue = USER_TASK_QUEUE.lock();

                if EXCUTOR.lock().is_empty() {
                    println!("finish task exit");
                    drop(EXCUTOR.lock());
                    break;
                } else {
                    println!("[thread pool] coroutine not empty, creat thread");
                    //如果线程列表为空，但任务队列不空，创建一个线程
                    self.add_thread(        
                        {
                            let thread = Thread::new_box_thread(crate::task::thread_main_ex as usize, 1);
                            thread
                        }
                    )
                }
            }
        }
    }
```

## 异步IPC

```rust


#[no_mangle]
pub fn main() -> i32 {

    println!("[user1 satp: {:#x}] main: Hello world from user mode program!", satp_read());

    // 调用 init_environment -> init_cpu_test
    init_coroutine_interface();

    test_for_user();

    println!("[user1 satp: {:#x}] main: end, time", satp_read());

    0
}

#[allow(unused_mut)]
pub fn test_for_user(){

    unsafe{
        
        let mut pipe_fd = [0usize; 2];
        pipe(&mut pipe_fd);

        async fn work1(fd: usize) {
            let mut buffer = [0u8; BUFFER_SIZE];
            let ac = AsyncCall::new(ASYNC_SYSCALL_READ, fd, buffer.as_ptr() as usize, buffer.len(), 0, 1, 33);
            ac.await;
            println!("[user] read {:#?}", buffer);
        }

        // 创建协程 work1 
        add_coroutine_with_prio(Box::pin(work1(pipe_fd[0])), 0);
        

        for i in 0..1 {
            
            async fn work2(fd: usize, id: usize) {
                //let mut buffer = [0u8; 32];
                let str = DATA;
                let ac = AsyncCall::new(ASYNC_SYSCALL_WRITE, fd, str.as_bytes().as_ptr() as usize, str.len(), id, 1, 33);
                ac.await;
                //close(fd)
                println!("[user] write {} ok", id);
            }
            add_coroutine_with_prio(Box::pin(work2(pipe_fd[1], i + 1)), 0);
        }
        
        // 调用  cpu_run -> thread_main_ex()
}
        coroutine_run();
    }

}


#[no_mangle]
pub fn thread_main_ex() {
    //println!(" > > > > > > > thread_main < < < < < < < ");
    // 计时, 每次取出开始协程时修改start, 取出结束协程tid == TEST_NUM 时修改end
    let mut end = 0;
    let mut cnt = 0;

    let mut cbq = unsafe { &mut *(CBQ_VA as *mut CBQueue) };

    loop {
        if !cbq.is_empty() { 
            let mut tids = cbq.pop();
            wakeup_all(&mut tids); 
        }
        //println!("cbq is empty");

        let tid;
        let task;
        let waker;
        // get EXCUTOR lock
        {
            let mut ex = EXCUTOR.lock();
            if ex.is_empty() { break; }

            let tid_wrap = ex.pop();
            if tid_wrap.is_none() { continue; }
            tid = tid_wrap.unwrap();

            let top = ex.get_task(&tid);
            if top.is_none() { continue; }
            task = top.unwrap().clone();

            waker = ex.get_waker(tid, task.prio);
        }
                                        
        // creat Context
        let mut context = Context::from_waker(&*waker);
        match task.future.lock().as_mut().poll(&mut context) {
            Poll::Pending => {  }
            Poll::Ready(()) => {
                // remove task
                EXCUTOR.lock().del_task(&tid);
                
            }
        }; 
    }

}
```

## 内核调度


理论上支持多核，但是后期好像把多核代码的删除掉了。

```rust
pub fn rust_main(hart_id: usize) -> ! {
    
    if hart_id == 0{
        
        trap::init();
        trap::enable_timer_interrupt();
       
        debug!("trying to add user test");
        task::add_user_test();
        
        send_ipi();

        basic_rt::thread::init_cpu_test();

    }else{
        init_other_cpu();
    }

    println_hart!("Hello", hart_id);
    
    if hart_id == 0 {
        println_hart!("run user task", hart_id);
        task::run_tasks();
    } else {
        //cpu_run();
    }
    
    panic!("Unreachable in rust_main!");
}

pub fn run_tasks() {
    debug!("run_tasks");
    PROCESSORS[hart_id()].run();
}

// 内核的调度器主循环
#[no_mangle]
pub fn run(&self) {
    static CNT: Mutex<usize> = Mutex::new(0);
    loop {
        let task = fetch_task();
            
            match task {
                Some(task) => {
                    unsafe { riscv::asm::sfence_vma_all()}
                    self.run_next(task);
                    // println_hart!("idel----", hart_id());
                    self.suspend_current();

                }
                None => {
                    //info!("all user process finished!");
                    
                    /* let c = *CNT.lock();
                    if c == 0 {
                        *CNT.lock() += 1;
                        super::add_initproc();
                    } */
                }
            }
    }
}
```
