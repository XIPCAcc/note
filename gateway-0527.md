# busy loop 和 uintr_wait的比较

如果阻塞等待进行全程busy loop，读取中断标志，或者是内存观测睡眠等待中断标志。

是否会比uintr_wait 更好

## 非批量 父进程等待 uintr_wait syscall

```rust
            while running_clone2.load(Ordering::SeqCst) {
                if unsafe { uintr_received > 0 } {
                    unsafe { uintr_received = 0; }
                    token.set_pending();
                    process_global_uintr_wakers();
                    continue;
                }
                match uintr::syscall::uintr_wait(uintr::UINTR_WAIT_MAX_USEC, 0) {
                    Ok(true) => {
                        unsafe { uintr_received = 0; }
                        token.set_pending();
                        process_global_uintr_wakers();
                    }
                    Ok(false) => {
                        if unsafe { uintr_received > 0 } {
                            unsafe { uintr_received = 0; }
                            token.set_pending();
                        }
                        process_global_uintr_wakers();
                    }
                    Err(_) => {
                        warn!("UINTR blocking thread: uintr_wait error, exiting");
                        break;
                    }
                }
            }
```

### 2x2 8c

```
Summary:
  Success rate: 100.00%
  Total:        60003.6120 ms
  Slowest:      101.0161 ms
  Fastest:      0.0187 ms
  Average:      0.1100 ms
  Requests/sec: 71686.1178

Response time distribution:
  10.00% in 0.0616 ms
  25.00% in 0.0726 ms
  50.00% in 0.0889 ms
  75.00% in 0.1117 ms
  90.00% in 0.1464 ms
  95.00% in 0.1943 ms
  99.00% in 0.4481 ms
  99.90% in 0.6764 ms
  99.99% in 2.0507 ms
```

### 4x4 32c

```
Summary:
  Success rate: 100.00%
  Total:        60001.0169 ms
  Slowest:      101.0827 ms
  Fastest:      0.0227 ms
  Average:      0.2689 ms
  Requests/sec: 118437.6260

Response time distribution:
  10.00% in 0.1500 ms
  25.00% in 0.1895 ms
  50.00% in 0.2356 ms
  75.00% in 0.2925 ms
  90.00% in 0.3679 ms
  95.00% in 0.4379 ms
  99.00% in 0.6546 ms
  99.90% in 1.3637 ms
  99.99% in 100.5526 ms
```

## busy loop

busy loop 性能不好，但是目前分析不出原因。

```rust
            while running_clone2.load(Ordering::SeqCst) {
                if unsafe { uintr_received > 0 } {
                    unsafe { uintr_received = 0; }
                    token.set_pending();
                    process_global_uintr_wakers();
                    continue;
                }
            }
```
```
==========================================
Testing shm-uintr | Matrix: 2x2 | Concurrency: 8 (r1)
==========================================
Starting gateway (logging to log/matrix_latency_20260526-032120/shm-uintr_m2_c8_r1_gateway.log)...
Running oha (matrix 2x2, 8 connections)...
Summary:
  Success rate: 100.00%
  Total:        60001.7113 ms
  Slowest:      40.1255 ms
  Fastest:      0.6015 ms
  Average:      25.8717 ms
  Requests/sec: 309.1745

Response time distribution:
  10.00% in 25.1145 ms
  25.00% in 25.9031 ms
  50.00% in 25.9897 ms
  75.00% in 26.0636 ms
  90.00% in 26.1648 ms
  95.00% in 27.9414 ms
  99.00% in 33.0629 ms
  99.90% in 36.9934 ms
  99.99% in 40.0883 ms

Testing shm-uintr | Matrix: 4x4 | Concurrency: 32 (r1)
==========================================
Starting gateway (logging to log/matrix_latency_20260526-031856/shm-uintr_m4_c32_r1_gateway.log)...
Running oha (matrix 4x4, 32 connections)...
Summary:
  Success rate: 100.00%
  Total:        60002.6767 ms
  Slowest:      38.1436 ms
  Fastest:      15.6401 ms
  Average:      25.9860 ms
  Requests/sec: 1231.4117

Response time distribution:
  10.00% in 25.7190 ms
  25.00% in 25.8903 ms
  50.00% in 25.9947 ms
  75.00% in 26.0874 ms
  90.00% in 26.1954 ms
  95.00% in 26.3293 ms
  99.00% in 33.9753 ms
  99.90% in 37.1869 ms
  99.99% in 38.1167 ms
```

## umonitor 和 umwait

umonitor监控中断标志位，umwait超时等待。


```rust
use std::arch::asm;

/// 监控 uintr_received，睡眠直到它变成非零（最迟 timeout 微秒）
fn wait_for_uintr_received(timeout_us: u32) {
    unsafe {
        let ptr = &uintr_received as *const _ as usize;
        asm!(
            // UMONITOR: 设置监控地址
            "umonitor {ptr_reg}",
            // 设置 EAX/ECX/EDX 作为 UMWAIT 参数
            "mov eax, {hints:e}",
            "xor ecx, ecx",   // edx:ecx = timeout (TSC ticks)
            "mov edx, {timeout:e}",
            // UMWAIT: 睡眠直到 WOKEN 或超时
            "umwait 0, edx, eax",
            ptr_reg = in(reg) ptr,
            hints = in(reg) 0u32,   // C0.1 state (轻睡眠)
            timeout = in(reg) timeout_us,  // 最大等待 TSC ticks
            out("eax") _, out("ecx") _, out("edx") _,
        );
    }
}

// 在循环中使用:
while running.load(Ordering::Relaxed) {
    if read_volatile(uintr_received) > 0 {
        write_volatile(uintr_received, 0);
        token.set_pending();
        process_global_uintr_wakers();
        continue;
    }
    // 用 UMWAIT 替代 uintr_wait：零开销睡眠，handler 写入时立即唤醒
    wait_for_uintr_received(5000);  // 最多等 5 毫秒（兜底）
}
```

```
Summary:
  Success rate: 100.00%
  Total:        60001.9759 ms
  Slowest:      41.0508 ms
  Fastest:      4.3008 ms
  Average:      25.9723 ms
  Requests/sec: 307.9899

  Total data:   1.55 MiB
  Size/request: 88 B
  Size/sec:     26.51 KiB

Response time distribution:
  10.00% in 25.7891 ms
  25.00% in 25.9229 ms
  50.00% in 25.9919 ms
  75.00% in 26.0575 ms
  90.00% in 26.1507 ms
  95.00% in 26.3731 ms
  99.00% in 33.2540 ms
  99.90% in 37.0922 ms
  99.99% in 41.0276 ms
```

这里面有个死锁问题，即监控期间 用户态中断无法处理，就无法写变量，所以无法从监控返回。而且现在看起来就算超时返回，后面的也都没有正常运行了

后面可以尝试在真正的busy loop + 共享变量方式加上这个机制，并且进行对比