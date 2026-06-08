# 关于实现基于用户态中断运行时 tokio的想法

要打破原本的 必须经过额外系统调用将 uintr 转换成event的僵局，需要从根本上改造运行时，将其从基于epoll机制转换成基于用户态中断机制。

如果是单线程的，那么直接把 IO driver 的turn() 中调用的epoll_wait 替换成 uintr_wait，问题就结束。

但是对应多线程，由于实现不知道 哪个worker会动态获得运行 turn() 机会（本身也是动态的，哪个worker 闲下来就尝试获取IO driver的锁），那么用户态中断的发送方就不知道应该把用户态中断发送给谁。

现在的想法是给所有worker都注册用户态中断 handler，然后发送方采用 RR策略依次选中一个worker发送用户态中断。这样所有worker都能处理。


或者是设计一个基于用户态中断的parker，让用户态中断能够唤醒worker。

## 改动思路

1. 目前只能先改动backend，gateway依然需要依赖外部的event，才能接收http请求。
backend 的tokio依赖于本地tokio代码，并修改tokio代码适配 uintr_wait。

gateway则改为依赖标准的tokio crate

2. 将backend和gateway的 ui_handler 分开，不再共享一个。

3. backend 每个worker thread需要单独注册 ui_handler，因此 中断标志位也要为每个worker thread分配一个，使用数组统一存储，并且使用worker注册 ui_handler的vector作为index。

4. 实现一个新的 Parker，基于uintr_wait等待并唤醒协程。

5. 修改 gateway，使其能够使用RR策略轮流给每个worker thread 发送中断。

6. backend的每个worker thread生成一个协程负责异步等待用户态中断，并读取共享内存和处理请求。