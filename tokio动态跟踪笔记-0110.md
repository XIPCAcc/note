epoll 的底层实现是如何监听到事件完成的? 和信号有关系吗？

1. **epoll 底层是通过“设备回调 + 等待队列 + 就绪链表”来感知 I/O 事件完成的**，不是靠信号。
2. **epoll 自身和传统 POSIX 信号（`SIGINT` 那类）没有直接关系**；  

## epoll 底层是如何监听到事件完成的？

从内核视角来看，关键有三样东西：

- `struct eventpoll`：一个 epoll 实例，对应你调用 `epoll_create` 得到的 `epfd`
  - 里面有：
    - 一棵 **红黑树 `rbr`**：存储所有注册的 `<fd, 关注事件>`（监视列表）
    - 一个 **就绪队列 `rdlist`**：双向链表，存储“已经就绪的 epitem”
    - 一个 **等待队列 `wq`**：阻塞在 `epoll_wait` 上的进程/线程会挂在这上面
- `struct epitem`：每个被 epoll 监视的 fd 对应一个 epitem，既挂在红黑树也可挂在就绪链表
- 一个 **回调函数 `ep_poll_callback`**：当底层 fd 有 I/O 事件发生时，由内核调用

### 注册阶段：`epoll_ctl` 做了什么？

当你调用：

```c
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);
```

内核大致做几件事：

1. 为这个 `fd` 创建一个 `epitem`，挂到 `eventpoll->rbr` 这棵红黑树里。
2. **把 epoll 的回调挂到这个 fd 的等待队列上**：
   - 对应的关键函数是 `ep_ptable_queue_proc`：
     - 它会为这个 fd 分配一个 `eppoll_entry`，里面包含一个 `wait_queue_t` 节点；
     - 调用 `init_waitqueue_func_entry(&pwq->wait, ep_poll_callback)`，把 `ep_poll_callback` 作为这个等待队列项的回调；
     - 再把这个等待队列项 `add_wait_queue(whead, &pwq->wait)` 挂到 fd 自己的等待队列上（例如 socket 的 `sk_wq`）。

简单理解：

> 以后这个 fd 上有“收/发数据就绪”之类的事件时，  
> 它会唤醒自己的等待队列，而这个等待队列里有一项的回调函数就是 `ep_poll_callback`，  
> 从而把事件“推”给 epoll。

### 事件发生时：`ep_poll_callback` 做了什么？

当底层设备（比如网卡）收到数据、造成 socket 变为可读时，内核的网络协议栈会：

1. 把数据放入 socket 的接收缓冲区；
2. **唤醒挂在这个 socket 等待队列上的各个等待者**；
3. 其中就包括我们刚才添加的那个 `wait_queue_t`（回调为 `ep_poll_callback`）。

`ep_poll_callback` 的核心逻辑是：

1. 根据 `wait_queue_t` 找到对应的 `epitem` 和 `eventpoll`：
   ```c
   struct epitem *epi = ep_item_from_wait(wait);
   struct eventpoll *ep = epi->ep;
   ```
2. 判断这次事件是不是用户关心的（比如你注册的是 `EPOLLIN`，那就只关心可读事件）；
3. 如果关心，且这个 `epitem` 还不在就绪队列中，就把它加到 `eventpoll->rdlist` 这个双向链表末尾：
   ```c
   list_add_tail(&epi->rdllink, &ep->rdlist);
   ```
4. 如果有线程正阻塞在 `epoll_wait` 上（`ep->wq` 非空），就把它们唤醒：
   ```c
   if (waitqueue_active(&ep->wq))
       wake_up_locked(&ep->wq);
   ```

**这是 epoll“知道某个 fd 就绪”的关键：通过设备/文件自身的等待队列回调，把事件放进 epoll 的就绪链表，并唤醒等待中的 `epoll_wait`。**

### 等待和返回：`epoll_wait` 怎么拿到就绪事件？

用户空间调用：

```c
int n = epoll_wait(epfd, events, maxevents, timeout);
```

内核侧主要逻辑在 `ep_poll`：

1. 先看 `eventpoll->rdlist`（就绪链表）是不是空：
   - **不空**：直接把链表里的若干 `epitem` 拷贝到用户传入的 `events[]` 数组里，然后返回数量；
   - **为空**：
     - 把当前进程/线程挂到 `eventpoll->wq` 等待队列中；
     - 调度出去（`schedule()`），阻塞等待；
     - 当有 fd 事件发生、`ep_poll_callback` 往 `rdlist` 里塞东西、并 `wake_up` 时，epoll_wait 才会被唤醒继续执行。
2. 被唤醒后再次检查 `rdlist`，把里面的就绪 epitem 拷贝出去，返回。

所以，从宏观上看：

- **epoll_wait 并不主动轮询所有 fd**；
- 它只是：
  - 阻塞在自己的等待队列 `wq` 上；
  - 等其它 fd 的事件通过 `ep_poll_callback` 把“就绪 epitem”塞进 `rdlist`，再把它唤醒；
  - 醒来后一次性从 `rdlist` 里把就绪事件“收割”出来返回。

这就是 epoll“监听到事件完成”的全过程。

### pipe + epoll 举例

write 进程调用 write() 后进入内核；
内核在 pipe 的实现里，在写入数据时检查 pipe 的状态；
如果因为写入导致 pipe 从“不可读”变成“可读”，
内核会通过 pipe 的 等待队列（wait queue）+ 回调机制，唤醒相关的等待者（其中就包括 epoll）。
epoll 在注册时，把自己的回调挂到 pipe 的等待队列上
当 write 写入导致 pipe 可读时：

内核在 pipe 写操作里会：
把数据放进 pipe 缓冲区；
如果原来是空、现在有数据（状态从“不可读”变为“可读”），就唤醒 pipe 的等待队列；
被唤醒的等待队列项中，有一个就是 epoll 挂上的那个 wait_queue_t，其回调是 ep_poll_callback：
ep_poll_callback 会把这个 pipe 对应的 epitem 挂到 epoll 的 就绪链表 rdlist；
然后唤醒阻塞在 epoll_wait 上的进程（挂在 eventpoll->wq 上）。

更精确地说：
write 调用触发了内核对 pipe 状态的更新，
真正负责唤醒的，是内核的 wait queue / 回调机制，不是用户进程本身发送了信号或者内核给接收方设置了信号标志。

```
write()（用户态） 
   ↓
内核 pipe 代码：唤醒 pipe 的等待队列
   ↓
epoll 挂在 pipe 等待队列上的回调 ep_poll_callback 被调用
   ↓
把 epitem 加入 eventpoll->rdlist
   ↓
wake_up(eventpoll->wq) → 阻塞在 epoll_wait 的进程被唤醒
```
---

## epoll 和“信号”到底什么关系？

要分清两个层次：

1. **epoll 自身实现**：  
   - 它完全是基于「文件/设备等待队列 + 回调 + 就绪链表 + 进程等待队列」这一套机制；
   - **不会用 POSIX 信号来通知事件**，也不会给你发 `SIGIO` 之类的信号。
2. **但是从应用层来看， epoll 可以用来处理信号**：  
   - 常见有两种做法：
     - 用 `signalfd` 把信号“转成一个 fd”，然后把这个 fd 加到 epoll 中；
     - 用自管道（signal handler 里往 pipe 写字节），把 pipe 的读端加到 epoll 中。

### epoll 自身与信号机制：**没有直接关系**

- 信号的投递与处理由 **内核信号子系统** 管理，典型接口是 `kill`、`sigaction`、`sigprocmask` 之类；
- 两者走的是不同的内核子系统：  
  - epoll：走的是文件系统/设备驱动的 `poll`/`wait_queue` 路线；  
  - 信号：走的是 `task_struct` 里的信号队列、信号掩码等逻辑。

所以如果你什么特殊事都不做，**epoll 不会“自动帮你处理 SIGINT 之类的信号”**，也不会因为有信号就从 epoll_wait 返回（除非被中断，返回 `-1` 并设置 `errno = EINTR`，那是 syscall 被信号打断的通用规则，不是 epoll 特有机制）。

### 怎么把“信号”交给 epoll 处理？

**方式 1：signalfd + epoll**

1. 用 `sigprocmask` 把某些信号（如 `SIGINT`, `SIGTERM`）**阻塞**，让它们不再以传统异步方式送到线程；
2. 调用 `signalfd(mask, ...)` 得到一个 fd；
3. 把这个 signalfd 加到 epoll：
   ```c
   epoll_ctl(epfd, EPOLL_CTL_ADD, sfd, &ev);
   ```
4. epoll_wait 返回时，如果是 `sfd` 可读，你就 `read(sfd, &info, sizeof(info))`，从里面解析出具体哪个信号来了。

这里“信号 -> signalfd -> epoll 监听 fd 可读”，  
**epoll 仍然只是在监听“fd 上的 I/O 事件”**，信号只是被 signalfd 封装成了一种特殊的可读事件。

**方式 2：自管道（self-pipe trick）+ epoll**

很多库（包括 tokio 的 Unix 信号处理）会这样做：

1. 建一个 pipe 或本地 socketpair；
2. 注册一个信号处理函数 `handler(int sig)`，在里面往 pipe 写入一个字节（只干这点简单、安全的事）；
3. 把这个 pipe 的读端 fd 加入 epoll；
4. epoll_wait 返回时，发现读端可读，就读出一个字节，再自己根据上下文知道是什么信号。

**epoll 还是只管“这个 fd 可读了”**，真正的“信号 → 写管道”转换是在 signal handler 里做的。

---

## 总结

- **epoll 底层监听事件完成的方式**：  
  给每个被监控的 fd 在其等待队列上挂一个回调 `ep_poll_callback`；  
  当 fd 上有 I/O 事件就绪时，内核通过这个回调把对应 epitem 丢进 epoll 的就绪队列 `rdlist`，并唤醒阻塞在 `epoll_wait` 上的线程；`epoll_wait` 醒来后从 `rdlist` 把事件返回给用户。

- **和信号的关系**：  
  epoll 自身不依赖、不使用 POSIX 信号；  
  但可以通过 `signalfd` 或自管道等方式把“信号”转成“普通 fd 的可读事件”，再交给 epoll 来统一处理。