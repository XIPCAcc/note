# IO uring 结合 用户态中断的尝试

如何在异步运行时中使用IO uring。

从 [compio](https://github.com/compio-rs/compio)到[tokio-iouring](https://github.com/tokio-rs/io-uring)，都是使用IO uring替代原本的 MIO epoll。
Reactor 改成使用 io-uring 替换 epoll。原理：Tokio 的 Poll 机制封装 SQE 提交，Waker 绑定 CQ 通知，实现无缝 async/await。

除非类似于compio一样中Runtime级别使用io uring替换到epoll，不然还是无法同时兼顾io uring和异步运行。

 [compio跟踪笔记](./ipc-bench笔记-0120.md)

 [tokio和io_uring阅读笔记](https://github.com/rosy233333/weekly-progress/blob/dev/25.9.25~25.10.8/tokio%E5%92%8Cio_uring%E9%98%85%E8%AF%BB%E7%AC%94%E8%AE%B0.md)


## gRPC 如何结合IO uring？

 第一个方向，把所有微服务请求直接作为 SQ entry的内容写入 IO uring？backend直接用IO uring 的CQ异步等待，这样好像就不需要用户态中断了。

 第二个方向，使用IO uring解决用户态中断风暴的问题，即如何批量处理中断的问题。IO uring能系统调用合并问题，但是解决不了用户态中断风暴的问题。

 用户态中断风暴问题，可以采用中断合并的方式。