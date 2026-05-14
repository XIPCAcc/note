# Linux 修改epoll_wait 使得 用户态中断能够唤醒

首先开启了epoll_wait 支持的内核收到用户态中断好像就是会打断系统调用的睡眠的，比如read write
所以到底有没有必要专门去改，反正现在确实能够返回了。
linux/fs/eventpoll.c
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
		   int maxevents, struct timespec64 *timeout)
{
    if (signal_pending(current) || test_thread_flag(TIF_NOTIFY_SIGNAL)) {
			if (test_thread_flag(TIF_NOTIFY_SIGNAL))
				pr_debug("epoll: woken up by user interrupt (TIF_NOTIFY_SIGNAL)\n");
			return -EINTR;
		}
}

用户态中断导致了 "epoll_wait failed: Interrupted system call (os error 4)
当 epoll_wait 被用户态中断唤醒时，它会返回 -EINTR （系统调用被中断），这与 uintr_wait 的行为一致。

## 原因分析
1. EINTR 错误 ： os error 4 就是 EINTR ，表示系统调用被外部事件中断。
2. 用户态中断处理 ：当用户态中断发生时：
   
   - uintr_wake_up_process() 被调用
   - 设置 TIF_NOTIFY_SIGNAL 标志
   - 唤醒 epoll_wait 中的进程
   - 我们的修改检测到 TIF_NOTIFY_SIGNAL 标志，返回 -EINTR
   - 返回用户态后，用户中断处理函数被执行

## 一些问题

   但是使用 dmesg | grep "epoll: woken up by user interrupt" 没有输出


# 异步运行结果

现在能够运行小批量的http请求，curl -s "http://127.0.0.1:8080/echo?message=test_uintr"

但是大批次会出现卡死，而且性能也不太好（虽然有一部分是debug 输出的原因，还有采用协程方式 wake 异步用户态中断协程也会有延迟）
 wrk -t8 -c256 -d30s --latency -s ./scripts/wrk.lua "http://127.0.0.1:8080/echo?message=test_uintr"
Running 30s test @ http://127.0.0.1:8080/echo?message=test_uintr
  8 threads and 256 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.04s   142.18ms   1.33s    90.46%
    Req/Sec    54.62     42.96   313.00     64.83%
  Latency Distribution
     50%    1.06s 
     75%    1.09s 
     90%    1.14s 
     99%    1.28s 
  7231 requests in 30.05s, 1.26MB read
Requests/sec:    240.66

最后卡死的debug信息
2026-03-30T11:04:06.139147Z  WARN gateway: Error serving connection from 127.0.0.1:45422: connection closed before message completed
2026-03-30T11:04:06.138738Z  WARN gateway: Error serving connection from 127.0.0.1:45368: connection closed before message completed

2026-03-30T11:06:45.959876Z  INFO backend::shm_server_uintr: wait_for_uintr: 收到 UINTR 中断
2026-03-30T11:06:45.959943Z  WARN backend::shm_server_uintr: Failed to read from shared memory: Buffer empty
2026-03-30T11:06:45.959967Z  INFO backend::shm_server_uintr: wait_for_uintr: 开始异步等待 UINTR 中断...


现在怀疑还是因为backend用户态中断协程阻塞在 epoll_wait了。但是换完内核为什么还是不行呢

# 单步调试一些backend，看看最后死锁在哪里

目前已经解决死锁的问题，而且通过5ms的定时任务唤醒可以跑通使用uintr作为通知的共性内存

5ms 性能如下


可以看出性能比不过UDS和EPOLL


# 抽象uintr—core

 改成在tokio 的turn中使用process_uintr_waker 唤醒协程，反而导致性能进一步下降

 性能如下


# 尝试在 run()中使用process_uintr_waker


 # 下一次

 1. 分析每种方式的延迟分布，查找性能瓶颈

 2. 看看批量处理的可能性