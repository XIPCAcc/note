compio:
completion-based IO（基于完成的IO）
thread-per-core 模型
使用 signalfd (Linux) 或 ctrl handlers (Windows)
信号作为异步任务处理
IO 操作和信号处理都在同一个完成队列中

tokio:
poll-based IO（基于轮询的IO）
reactor 模型
使用独立的 signal 线程和 channel
signal 处理相对简单，但可能不够高效