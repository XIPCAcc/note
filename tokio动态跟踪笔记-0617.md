# tokio的主线程

tokio的主线程执行 block_on。如果没有任务会会进入 park condv。不会走争用 park driver的路径。
thread id 为1号。

其余的thread id 为2-n的 working thread才会争用。

在 tokio 多线程 runtime 中，主线程和 worker 线程是分离的：

| 线程 | 职责 |
|---|---|
| **主线程** | 执行 `block_on` 中的顶层 future，阻塞等待其完成 |
| **Worker 线程** | 从任务队列中取 `spawn` 出来的任务执行 |

`tokio::spawn` 的任务只会被放入 worker 线程的本地队列或全局队列，由 worker 线程（ID 2~N）调度执行。主线程只负责驱动顶层 future，不参与 worker 任务调度。

所以用户态中断不会打断spawn的计算任务。