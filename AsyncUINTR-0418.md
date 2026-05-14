关于目前的三条路径。

对于work thread 1，执行的是是路径1，main函数所在的协程都运行在work thread1，最终会park 阻塞等待 futex。
对于其他 work thread，会争用路径2，等待外部事件，争抢成功的会park_driver 阻塞等待在 epoll_wait。否则就会park_convar阻塞等待在 futex。

一般来说，阻塞的协程都需要依赖其他work thread执行wake来唤醒。但是接收到用户态中断的work thread往往都处于睡眠状态，外部又没有一个能够一直检测用户态中断的waker。问题就在中断没有办法像外部事件一样，绑定在等待epoll_wait 的work thread上。（除非弄成单线程模式）

park()
  ↓
condvar_wait
  
waker.wake()
  ↓
unparker.unpark()
  ↓
condvar.notify_one()
  ↓
condvar_wait返回

# 关于tokio多线程runtime park状态下无法正确路由用户态中断的解决办法

多线程runtime环境中，working thread在没有任务运行的情况下会进入park等待。

park等待也分成两种状态，一种是等待外部事件唤醒(park_driver)，另一种是等待条件唤醒(park_condvar)。

多线程下只有一个working thread能够经过竞争锁进入等待外部事件唤醒状态，其余working thread则需要等待条件唤醒。

用户态中断会固定的路由给注册该中断的working thread负责处理，如果该working thread没有成功竞争到锁进入等待外部事件唤醒状态，那么该working thread就无法仅仅通过用户态中断从条件等待中返回。

解决的办法是注册用户态中断的working thread在进入park之前，注册一个事件，确保用户态中断处理程序能够触发该事件，从而让负责等待外部事件的working thread唤醒等待用户态中断的working thread，并且在working thread被唤醒后，取消用户态中断绑定的事件。

# 关于中断处理程序中使用锁导致死锁的问题

因为中断嵌套导致前一个中断处理程序还没释放锁，下一个中断处理程序就嵌套执行导致导致死锁的问题。

应该避免在中断处理程序中使用锁，转成使用原子变量


# 编译问题 一定要加上 --features uintr-core

cargo clean
cargo build --features uintr-core

最好还是将uintr-core feature 去除

要检查tokio是否使用的是本地版本，build是否lock住了


# 关于各个分支之间的差异

 git log --oneline --graph uintr uintr-2 --
* 136de1a (uintr-2) add uintr as submodule
* 902a5db use uintr_wait syscall which will cause a coroutinue occupies the thread individually. another corotine who is responsible for senduipi to gateway is hungry to death.
| * 41b742d (uintr) 1st uintr implment
|/  
* 076cf19 (origin/main, main) Update mixed workload test and performance analysis report
* 3678595 Update log path and performance analysis report
* d9368da 1st edition.

git log --oneline --graph uintr-3 uintr-2 --
* d6689b0 (HEAD -> uintr-3) trace: add shm latency tracing and perf runner
* 4bc8b65 using atomic and seq count
* 93a00df Add shm-intr test into scripts
* b4c6e03 Add tokio and uintr-core as submodules and refactor dependencies
* ee0e2ac Add tokio and uintr-core as submodules
* fbb8a89 Update doc and uintr
* ea68fd7 Async uintr wait share memory success.
* 136de1a (uintr-2) add uintr as submodule
* 902a5db use uintr_wait syscall which will cause a coroutinue occupies the thread individually. another corotine who is responsible for senduipi to gateway is hungry to death.
* 076cf19 (origin/main, main) Update mixed workload test and performance analysis report
* 3678595 Update log path and performance analysis report
* d9368da 1st edition.

git log --oneline --graph uintr-3 uintr-4-nolog --
* f2efb1e (uintr-4-nolog) add some println and change wake up
* 4c72e6c (origin/uintr-4-nolog) wake uintr corortine
* e8518df Modify compare_shm_transports
| * d6689b0 (HEAD -> uintr-3) trace: add shm latency tracing and perf runner
| * 4bc8b65 using atomic and seq count
|/  
* 93a00df Add shm-intr test into scripts
* b4c6e03 Add tokio and uintr-core as submodules and refactor dependencies
* ee0e2ac Add tokio and uintr-core as submodules
* fbb8a89 Update doc and uintr
* ea68fd7 Async uintr wait share memory success.
* 136de1a (uintr-2) add uintr as submodule
* 902a5db use uintr_wait syscall which will cause a coroutinue occupies the thread individually. another corotine who is responsible for senduipi to gateway is hungry to death.
* 076cf19 (origin/main, main) Update mixed workload test and performance analysis report
* 3678595 Update log path and performance analysis report
* d9368da 1st edition.

git log --oneline --graph uintr-2 uintr-4-nolog --
* f2efb1e (uintr-4-nolog) add some println and change wake up
* 4c72e6c (origin/uintr-4-nolog) wake uintr corortine
* e8518df Modify compare_shm_transports
* 93a00df Add shm-intr test into scripts
* b4c6e03 Add tokio and uintr-core as submodules and refactor dependencies
* ee0e2ac Add tokio and uintr-core as submodules
* fbb8a89 Update doc and uintr
* ea68fd7 Async uintr wait share memory success.
* 136de1a (uintr-2) add uintr as submodule
* 902a5db use uintr_wait syscall which will cause a coroutinue occupies the thread individually. another corotine who is responsible for senduipi to gateway is hungry to death.
* 076cf19 (origin/main, main) Update mixed workload test and performance analysis report
* 3678595 Update log path and performance analysis report
* d9368da 1st edition.

最终决定从uintr-3 分支出来uintr-5，并且退回到4bc8b65 using atomic and seq count commit
同时要将submodule都退回去。重命名所有submodule的branch名字为 uintr-5