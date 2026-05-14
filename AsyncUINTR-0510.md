
```
                    ┌─────────────┐
   tokio::spawn ──▶ │ Injection   │ (全局注入队列)
                    │   Queue     │
                    └──────┬──────┘
                           │ work-stealing
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
     ┌──────────┐    ┌──────────┐    ┌──────────┐
     │ Worker 0 │    │ Worker 1 │... │ Worker15 │
     │ local_q  │    │ local_q  │    │ local_q  │
     │  (LIFO)  │    │  (LIFO)  │    │  (LIFO)  │
     └──────────┘    └──────────┘    └──────────┘
          │               │               │
          ▼               ▼               ▼
      park/unpark    park/unpark     park/unpark
      (epoll wait)   (epoll wait)    (epoll wait)
```


size=16, 35K req/s
```
Gateway 调度器状态
SCHEDULER[2s] workers=16 local_q_total=0 inj_q=0 parks=3 noops=2 | idle_park (no work, 3 parks)
  W0: local_q=0 park+0 noop+0 busy=57.4%
  W1: local_q=0 park+0 noop+0 busy=57.4%
  ...
  W12:local_q=0 park+3 noop+2 busy=57.4%   ← 只有 W12 在 park
  W13:local_q=0 park+0 noop+0 busy=57.4%

Backend 调度器状态
SCHEDULER[2s] workers=16 local_q_total=1 inj_q=0 parks=83426 noops=23473 |  PARK_WITH_WORK!
  W0: local_q=0 park+15906 noop+4537 busy=14.9%
  W10:local_q=0 park+20872 noop+4649 busy=14.9%
  W12:local_q=0 park+17739 noop+5462 busy=14.9%
  W13:local_q=1 park+16190 noop+4981 busy=14.9%
  W14:local_q=0 park+12719 noop+3844 busy=14.9%
```

1. 队列深度：几乎永远为 0
指标 Gateway Backend local_q_total 0 0~1 injection_q 0 0 

任务一到就被执行，从不排队 偶尔有 1 个任务。

调度延迟（spawn → task_start）：3~8us

Gateway（正常）：

- 57% busy，43% 空闲
- Park 只发生在真正没有工作时（ idle_park ）
- 每 2 秒只有 3-4 次 park，说明 worker 大部分时间在忙或刚醒
- 没有 "有任务却 park" 的情况
Backend（异常）：

- 15% busy，85% 空闲
- 出现 PARK_WITH_WORK ：有任务在队列中但 worker 仍在 park
- Park 频率极高（~40K 次/秒/worker）

Gateway：不存在。 Worker 只在队列为空时才 park，这是正确的节能行为。

Backend：存在！ 任务在 injection queue 中，但 worker 在 park。这是因为 backend 使用 event_interval=64 ，worker 每 64 次 poll 才检查一次全局队列。

750us "等待时间"是 backend 进程被内核调度出去 （deschedule）导致的，不是 Tokio 协程调度的问题。
