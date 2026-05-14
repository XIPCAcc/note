# 唤醒位置

### 方案一

在选择下一个协程的lopp中唤醒，理论上应该是延迟最低的。

更新方案一
cargo build --release --features uintr-core

```rust
tokio/tokio/src/runtime/scheduler/multi_thread/worker.rs
impl Context {
    fn run(&self, mut core: Box<Core>) -> RunResult {

        while !core.is_shutdown {
            #[cfg(all(target_os = "linux", feature = "uintr-core"))]
            {
                uintr_core::process_global_uintr_wakers();
            }
        }
    }
}
```

# interval

