# 多核爬虫方案

## 需求

在用户态写多核的爬虫程序
在启动时确定有多少个核可以用，然后用尽可能多的核；
爬虫任务是动态变化的：固定时间间隔启动任务；
在任务不足时，释放多余的CPU核（线程释放退出 exit ）；


## 多核方案

主程序是一个守护进程，通过TCP监听外部请求（比如爬虫请求）。

守护进程每收到爬虫请求后，创建爬虫协程任务。

当全局就绪任务队列中的协程任务超过一定数量时，runtime创建额外的工作线程执行协程任务。

当全局就绪任务队列为空，工作线程无法获取就绪协程任务执行时，工作线程主动退出。

## 守护任务

守护任务会启动一个TCP监听，等待外部的计算或者爬虫请求。

守护任务池收到爬虫请求连接后，创建新的协程任务读取后续的爬虫目标地址以及创建爬虫任务。

每个请求会携带若干个爬虫目标地址，每个地址相应生成一个爬虫任务。

如果外部爬虫请求发起方断开TCP连接，等到所有爬虫任务结束后退出，释放所有资源，最后只剩下一个work thread和一个守护任务。

## 系统调用

### 初始化阶段系统调用（一次性）

基于epoll的Reactor初始化 - epoll_create1、epoll_ctl(ADD eventfd)
信号注册 - rt_sigaction
TCP初始化 - socket、setsockopt、bind、listen

### 运行期

基于epoll的Reactor等待外部事件 - epoll_wait

基于epoll的Reactor注册或修改或删除外部事件 - epoll_ctl (ADD/MOD/DEL)

TCP相关 - connect、accept、read、write、fcntl、close（设置fd为非阻塞）

线程相关 - clone3、munmap

锁相关 - futex

一次请求，32条数据（32个爬虫任务）涉及的系统调用统计。

| 分类 | 系统调用 | 次数 | 占比 | 性质 |
|------|----------|-----:|-----:|------|
| 同步原语 | futex | 696 | 47.2% | 队列锁竞争 |
|*reactor | epoll_ctl 101 + epoll_wait 10 + eventfd 写 96 + epoll_create1/eventfd2 各 1 | 209 | 14.2% | 稳态为主 |
| 网络 I/O | read 111 + socket 33 + connect 32 + getsockopt 32 + close 42 + accept 3 + write→socket 64（ACK 32 + GET 32） | 317 | 21.5% | 稳态 |
| fd 配置 | fcntl | 134 | 9.1% | 稳态（每 fd 设非阻塞等） |
| 线程生命周* | clone3 2 + set_robust_list 3 + sigaltstack 9 + munmap 7 + rt_sigprocmask 9 + rseq 3 | 33 | 2.2% | glibc NPTL 在线程自举阶段自动调用的 |
| 初始化（一次性） | mmap 19 + mprotect 11 + openat 9 + rt_sigaction 8 + statx 5 + fstat/lseek/brk 各 4 + bind/listen/setsockopt/execve 等 | ~86 | 5.8% | 启动一次性 |
