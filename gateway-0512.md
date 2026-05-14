
# wrk 测试负载

wrk -t8 -c16 -d30s

-c 16 代表并发度为16，即同时建立16个连接。
-t 8 代表wrk运行的线程数，代表8个线程共同负担 16个连接。
-d 30s 代表持续测试30s。

QPS = 1/avg_time * Concurrreny


# gateway 协程分析

0号协程 main，运行run_http_server() loop。run_htttp_server() 会调用 Tcplistener.accept()?，并spawn创建处理http请求的协程。对于wrk负载，处理http请求的协程数量等同的并发度。

1号协程 ShmTransportUintr::new() 创建的用于异步等待用户态中断的协程，并且从共享内存中读取resp。会循环等待用户态中断，阻塞点事uintr_wait()?

2-n号协程：处理http请求的协程。执行handle_request() 解析请求，写入共享内存并向backend发送用户态中断。阻塞点是serve_connection()? 和 rx.await()? 其中serve_connection()用于读取http请求，rx用于接收1号协程发送的resp。

# backend 协程分析

0号协程 main，运行server.run()，负责初始化中断，初始化中断，spawn() 1号协程，然后调用ctrl_c.await() 阻塞直到程序被终止。

1号协程 while loop()，执行uintr_wait()? 异步等待用户态中断，读取共享内存request，对于每个request，spawn一个计算协程进行处理。

2-n号协程：计算协程。计算矩阵乘法，写resp到共享内存，发送用户态中断。计算协程数量等于进行中的request数量，即并发度（wrk的每个并发连接在上一个请求未返回来之前不会发送下一个请求）。

# 矩阵大小对于性能的影响

用户态中断的又是在于能够更快的接收的通知，如果1号协程始终处理运行状态（单单是工作线程处于工作状态也是不行的，因为一旦发生调度，唤醒和排队的开销降抹掉通知开销的优势），然后不断地处理新的中断请求。

和epoll比较，epoll如果需要更新事件，必须等待空闲的working thread或者是运行完一个批次的协程后重新进入epoll_wait 读取外部事件才能继续处理请求。

1号协程如何才能够保持运行状态，即backend在1号协程的loop解析request和生成计算协程期间能够至少收到一个中断请求。

backend 1号协程的 解析requst和spawn生成计算协程的时间 t1

backend计算协程的运行时间加上调度时间，gateway解析，路由，返回resp的时间 t2。

t1 > t2/C (C是并发度，考虑到多个并发的计算协程是近似于流水的运行方式)

# 并发度对性能的影响

并发度度高，生成的协程数量越多，调度的等待时间就越长。对于gateway来说，几乎没有影响因为gateway的所有协程基本都是在异步阻塞等待resonse。

但是对于backend，所有的协程都需要排队等待计算。排队的时间取决于协程数量和运行时的工作线程数量。如果矩阵计算量太大并且并发度很高，那么每个计算协程都将长期占用工作线程，显然此时的 t2/C 大于 t1，1号协程必须进入异步等待状态，即使此时中断来到，也得和 epoll一样等待进入turn被唤醒。唤醒后还需要改等到同一批次的request 计算协程都运行完，才会被重新调度处理下一批request，相当于流水线排空，失去流水处理的优势。


排队时间 t_sche = t_cal * C / num_thread

# 中断的劣势

频繁的用户态中断打断0号 working thread的运行，导致运行在0号working thread的计算协程更慢。

对于tokio运行时，即使一直有任务运行，也需要间隔一段时间轮询一次外部事件，此时并不能节省epoll_wait 系统调用的开销。

此外，为了解决唤醒死锁问题，还需要采用self_pipe的方式，在中断处理程序中通过写入pipe的方式创造一个event。此时会带来一个sys_write开销。