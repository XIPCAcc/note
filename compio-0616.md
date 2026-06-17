# ompio 在多核的支持
          
## Compio 多核支持方式

Compio 采用 **thread-per-core** 模型，需要用户手动管理多核：

### 1. 线程绑定 CPU 核心

```rust
use compio::runtime::RuntimeBuilder;
use std::collections::HashSet;

let mut cpus = HashSet::new();
cpus.insert(0);  // 绑定到 CPU 0
cpus.insert(1);  // 绑定到 CPU 1

let runtime = RuntimeBuilder::new()
    .thread_affinity(cpus)  // 绑定线程到指定核心
    .build()?;
```

### 2. 多核模式：每个核心一个 Runtime

```rust
use compio::runtime::RuntimeBuilder;

fn start_worker(cpu: usize) {
    let mut cpus = HashSet::new();
    cpus.insert(cpu);
    
    std::thread::spawn(move || {
        let runtime = RuntimeBuilder::new()
            .thread_affinity(cpus)
            .build()
            .unwrap();
        runtime.block_on(async {
            // 在这个核心上运行任务
        });
    });
}

fn main() {
    start_worker(0);
    start_worker(1);
    // ... 启动更多 worker
}
```


### 与 Tokio 的对比

| 特性 | Tokio | Compio |
|------|-------|--------|
| 线程模型 | 线程池 + work-stealing | Thread-per-core |
| 负载均衡 | 自动任务窃取 | 无，需手动管理 |
| 多核扩展 | 单 runtime 多线程 | 多 runtime 实例 |
| 阻塞处理 | `spawn_blocking` | `spawn_blocking` |

Compio 需要用户通过启动多个线程 + 每个线程创建独立 Runtime 来利用多核，没有自动负载均衡机制。这适合 CPU 绑定型任务，避免了 work-stealing 的开销，但需要应用层自己做任务分发。

```
Runtime A (线程 0, CPU 0)
  ├── spawn(task1) → 始终在 线程 0 执行
  ├── spawn(task2) → 始终在 线程 0 执行
  └── block_on(future) → 始终在 线程 0 执行

Runtime B (线程 1, CPU 1)
  ├── spawn(task3) → 始终在 线程 1 执行
  └── ...
  ```


  可以跨线程唤醒，但是不能跨线程运行。