
# uintr-5

决定从uintr-3 分支出来uintr-5，并且退回到4bc8b65 using atomic and seq count commit
同时要将submodule都退回去。重命名所有submodule的branch名字为 uintr-5

using atomic and seq count commit commit 是采用单独的coroutine 定时唤醒协程的，现在要将这个协程去掉，在current thread，multithrea的 run 函数插入 process_global_uintr_wakers，同时在turn中插入process_global_uintr_wakers。

同时确保current thread的场景能够不依赖定时唤醒协程运行。为了避免中断嵌套导致的锁竞争死锁，需要修改中断处理程序中对于lock的使用。

为了避免后续编译忘记 uintr-core feature，让所有情况都依赖这个feature

process_uintr_wakers 应该要take waker，防止二次唤醒

process_global_uintr_wakers 放置的位置应该避免park回来又执行一次。

# 需要但需seq和consume的事情吗

目前分析下来不需要担心，因为只有一个协程会等待用户态中断，所有可以中断合并，直接一次性消费所有内存内的数据。

而且每次read也能保证只读取一条request

# bug

通过打印全局编号，发现gateway明明收到11个请求，也向内存写入并且发送了中断。

但是backend只能响应第一次的中断，后面的都无法通过waker唤醒，虽然确实执行了process_global_wakers

wait_for_uintr: 开始异步等待 UINTR 中断...
seq == consumed for server
seq != consumed for server
waker is Some
seq != consumed for server
waker is Some
seq != consumed for server
waker is Some
turn poll
polled ok
process_global_uintr_wakers
seq != consumed for server
waker is Some
seq != consumed for server
waker is Some
turn poll
polled interrupted
process_global_uintr_wakers
seq != consumed for server
waker is Some
seq != consumed for server
waker is Some
turn poll