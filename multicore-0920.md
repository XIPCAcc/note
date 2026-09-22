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