# 多核运行时分析

## Rust多核用户态线程库

用户态线程分成有栈绿色线程（stackful coroutine）​ 和 无栈协程。

无栈协程 - tokio

有栈协程 - may https://github.com/Xudong-Huang/may

## C语言用户态线程库

https://github.com/Yuandong-Chen/coroutine/tree/ezco.v.0.0.1 

https://github.com/jesHrz/mThread/tree/main

https://www.cnblogs.com/github-Yuandong-Chen/p/6849168.html

https://blog.csdn.net/qq_42659989/article/details/119345832?spm=1001.2014.3001.5502

有栈用户态线程的实现方式就是用户自己定义任务描述符，分配任务堆栈，任务队列，然后在用户态实现一个switch_to切换任务。一般不支持抢占，都是主动让权。

对于如何使用多核没有什么启发。

从tokio到may运行时，如果要使用多核，必须通过 标准库的 thread::spawn 创建出多线程，然后再在多线程上面实现用户态的调度。

至于要创建多少个线程，需要初始化runtime时期就指定，然后每个线程再单独执行自己的用户态调度代码。

## Embassy和tokio的多核运行时对比

tokio的多核运行时初始化runtime时候通过 std:thread::spawn()生成多个worker thread。一般来说，默认有多少个核，就生成多少个worker thread，这样可以确保每个worker thread单独占用一个核。每个worker thread有局部任务队列，所有worker thread共享全局任务队列，IO driver（epoll）和Scheduler，Executor。任务可以在不同worker thread之间迁移运行（通过任务窃取）。

Embassy的多核运行时，每个核单独生成一个runtime，每个runtime单独运行自己的Executor和任务队列，各自调度，任务不能够的不同运行时之间迁移。

## 用户态中断和tokio多核运行时有什么冲突

tokio负责等待外部事件的IO driver是共享的，如果执行IO driver的协程会被随机调度到任意一个worker thread上运行（或者说哪个 worker thread空闲了就会尝试获取IO driver锁，然后执行IO driver协程，阻塞在epoll_wait()，没有获取到IO driver锁的thread空闲了执行futex_wait）。

正确的唤醒语义是用户态中断路由给IO driver协程，通过用户态中断打断epoll_wait。

用户态中断与线程是绑定的，只能够路由给注册用户态中断的线程。

因此如果想要避免这个问题，就要保证IO driver协程绑定在中断注册的线程上。