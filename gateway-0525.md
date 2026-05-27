# 关于长时间运行的死锁问题的分析

死锁问题主要集中在运行时间大于30s，小批量数据高并发测试场景。比如 矩阵2x2,8并发，每秒吞吐在10万。

目前没有好的调试方法，单步运行不现实，抓取log内容太多，而且打印log本身也会对测试产生影响。

目前只能靠分析代码并且猜想验证的方式进行排查。

## 猜想1

过于频繁的中断导致中断风暴，backend来不及处理导致中断覆盖问题。

170036 responses 说明gateway向backend 发送了170036个中断，每个中断应该会触发一次intr_callback，

```
Status code distribution:
  [200] 170036 responses

bpftrace results:
Attaching 4 probes...
=== 36s report ===
intr_callback: 100939
uintr_wait:     50076
wakers:         50075
```

验证方法：中断合并

原本的方法是gateway把一个req写入共享内存或者backend把一个resp写入共享内存后，都会向对方发送一个用户态中断。
现在gateway只有当共享内存为空，或者累计收到若干个req的时候才发送一个共享内存。
backend也是批量读取共享内存的req，等待该批次的req都处理完后，统一向gateway发送一次中断。

这样会导致单词请求的延迟增加，但是没有解决死锁问题。

甚至没有解决中断覆盖问题。

而且不知道是我实现的方式有问题还是这个设计方案本身的问题，中断合并后，更早的出现了死锁问题。


### 关于中断嵌套的问题

进入 ui_handler 时， UIF（User Interrupt Flag）被硬件自动清零 。

UINTR 中断门和普通中断一样，进入时自动关中断。

所以 不存在用户态中断嵌套 。handler 执行期间，后续 senduipi 能发到、ON 位会置位，但处理器不会触发新的 handler——直到 uiret 恢复 UIF=1 后，如果 ON 仍为 1，handler 立即再次触发。


场景 1：硬件自动合并中断

```
tempUPID.PIR[UITTE.UV] := 1;
IF tempUPID.SN = 0 AND tempUPID.ON = 0
    THEN
    tempUPID.ON := 1;
    sendNotify := 1;       ← 只发一次通知 IPI
ELSE
    sendNotify := 0;       ← ON 已经是 1 时，后续 senduipi 不会产生新的通知 IPI 。
FI;
```

## 猜想2

### 第一个版本

代码实现问题，只有执行了uintr_wait 等待期间返回才会唤醒协程。

可以出现的死锁情况是：协程1（异步等待用户态中断的协程）已经开始等待用户中断，但是阻塞协程还没有进入uintr_wait()就已经收到了所有gateway的中断请求，导致后面uintr_wait的返回都是超时返回，从而无法唤醒协程1。

这也解释了为什么中断合并后，更容易出现这种死锁情况了。

```rust
    match uintr::syscall::uintr_wait(uintr::UINTR_WAIT_MAX_USEC, 0) {
        Ok(true) => {
-                        let woken = process_global_uintr_wakers();
-                        let cur_wake = get_wake_count();
-                        info!(
-                            "UINTR: process_global_uintr_wakers returned {} (wake_cnt: {} -> {}, +{})",
-                            woken, last_wake, cur_wake, cur_wake - last_wake
-                        );
-                        last_wake = cur_wake;
-                    }
-                    Ok(false) => {
-                        debug!("UINTR blocking wait: timeout (no interrupt in window)");
                     }
-                    Err(e) => {
-                        warn!("UINTR blocking wait error: {}, exiting", e);
+                    _ => {
            break;
        }
    }
```

### 第二个版本的代码实现

确实不会导致死锁了。但是对于版本1遇到的死锁问题，依然需要进入uintr_wait等待超时返回，那么就浪费了中间等待的10秒钟（UINTR_WAIT_MAX_USEC），导致性能下降。


```rust
    match uintr::syscall::uintr_wait(uintr::UINTR_WAIT_MAX_USEC, 0) {
        Ok(true) => {
-                        let woken = process_global_uintr_wakers();
-                        let cur_wake = get_wake_count();
-                        info!(
-                            "UINTR: process_global_uintr_wakers returned {} (wake_cnt: {} -> {}, +{})",
-                            woken, last_wake, cur_wake, cur_wake - last_wake
-                        );
-                        last_wake = cur_wake;
-                    }
-                    Ok(false) => {
                          process_global_uintr_wakers();
-                        debug!("UINTR blocking wait: timeout (no interrupt in window)");
                     }
-                    Err(e) => {
-                        warn!("UINTR blocking wait error: {}, exiting", e);
+                    _ => {
            break;
        }
    }
```

### 第三个版本

试一下如果阻塞协程完全调用uintr_wait进入阻塞状态，而是一直循环process_global_uintr_wakers会发生什么事情。

```rust
    while running_clone2.load(Ordering::SeqCst) {
        process_global_uintr_wakers();
    }
```

process_global_uintr_wakers 调用了300 0000 次。

但是QPS只有几千，感觉这一次是因为 token.waker 的锁竞争问题。


### 第四个版本

在进入uintr_wait 之前再次检查是否有中断发生。只能说进一步降低了前面所述情况发生的概率。大部分情况1min都能跑到后面40多秒，甚至能全程不死锁。

```rust
    while running_clone.load(Ordering::SeqCst) {
        if unsafe { uintr_received > 0 } {
            unsafe { uintr_received = 0; }
            process_global_uintr_wakers();
            continue;
        }
        match uintr::syscall::uintr_wait(uintr::UINTR_WAIT_MAX_USEC, 0) {
            Ok(true) => {
                process_global_uintr_wakers();
                unsafe { uintr_received = 0; }
            }
            _ => {
                continue;
            }
        }
    }
    warn!("UINTR blocking thread: exited");
```

从Log可以看到第一次出现UINTR blocking thread: uintr_wait timeout 返回后继续执行是因为 进入uintr_wait 前gateway的所有中断都发送了，超时后uintr_wait 返回被uintr_received 捕捉。但是最后两次UINTR blocking thread: uintr_wait timeout 返回后都没有继续执行，可能的原因，gateway发送的中断被覆盖了，确实没被backend接收并执行。
从bpftracede gateway执行send_uipi 343653的次数和 ui_handler 207367的执行次数来看，确实有10万的 用户态中断被覆盖。（此时backend依然使用了中断合并）

```
2026-05-25T14:49:36.981183Z  WARN gateway::shm_transport_uintr: SHM: 281us
2026-05-25T14:49:47.118171Z  WARN backend::shm_server_uintr: UINTR blocking thread: uintr_wait timeout
2026-05-25T14:49:48.636693Z  WARN gateway::shm_transport_uintr: SHM: 63us
2026-05-25T14:49:50.214333Z  WARN gateway::shm_transport_uintr: SHM: 35us
2026-05-25T14:50:01.537108Z  WARN backend::shm_server_uintr: UINTR blocking thread: uintr_wait timeout
2026-05-25T14:50:11.635881Z  WARN backend::shm_server_uintr: UINTR blocking thread: uintr_wait timeout
2026-05-25T14:50:11.877611Z  WARN gateway::shm_transport_uintr: SHM: 57us
2026-05-25T14:50:21.909872Z  WARN backend::shm_server_uintr: UINTR blocking thread: uintr_wait timeout
2026-05-25T14:50:31.946700Z  WARN backend::shm_server_uintr: UINTR blocking thread: uintr_wait timeout
2026-05-25T14:50:36.980581Z  WARN gateway: Error serving connection from 127.0.0.1:38858: connection closed before message completed
2026-05-25T14:50:36.980610Z  WARN gateway: Error serving connection from 127.0.0.1:38880: connection closed before message completed
2026-05-25T14:50:36.980628Z  WARN gateway: Error serving connection from 127.0.0.1:38870: connection closed before message completed
2026-05-25T14:50:36.980640Z  WARN gateway: Error serving connection from 127.0.0.1:38846: connection closed before message completed
2026-05-25T14:50:36.980651Z  WARN gateway: Error serving connection from 127.0.0.1:38850: connection closed before message completed
2026-05-25T14:50:36.980664Z  WARN gateway: Error serving connection from 127.0.0.1:38844: connection closed before message completed
2026-05-25T14:50:36.980681Z  WARN gateway: Error serving connection from 127.0.0.1:38890: connection closed before message completed
2026-05-25T14:50:36.980693Z  WARN gateway: Error serving connection from 127.0.0.1:38830: connection closed before message completed
Summary:
  Success rate: 100.00%
  Total:        6.0005 10 sec
  Slowest:      1.0001 10 sec
  Fastest:      0.0000 10 sec
  Average:      0.0001 10 sec
  Requests/sec: 5858.4642

  Total data:   29.54 MiB
  Size/request: 88 B
  Size/sec:     504.12 KiB

Response time histogram:
  0.000 10 sec [1]      |
  0.100 10 sec [351491] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.200 10 sec [0]      |
  0.300 10 sec [0]      |
  0.400 10 sec [0]      |
  0.500 10 sec [0]      |
  0.600 10 sec [0]      |
  0.700 10 sec [0]      |
  0.800 10 sec [0]      |
  0.900 10 sec [0]      |
  1.000 10 sec [40]     |

Response time distribution:
  10.00% in 0.0000 10 sec
  25.00% in 0.0000 10 sec
  50.00% in 0.0000 10 sec
  75.00% in 0.0000 10 sec
  90.00% in 0.0000 10 sec
  95.00% in 0.0000 10 sec
  99.00% in 0.0001 10 sec
  99.90% in 0.0001 10 sec
  99.99% in 1.0001 10 sec


Details (average, fastest, slowest):
  DNS+dialup:   0.0000 10 sec, 0.0000 10 sec, 0.0000 10 sec
  DNS-lookup:   0.0000 10 sec, 0.0000 10 sec, 0.0000 10 sec

Status code distribution:
  [200] 351532 responses

Error distribution:
  [8] aborted due to deadline

==========================================
All Matrix Multiplication Tests Complete!
==========================================

Results saved to: log/matrix_latency_20260525-224926/

bpftrace results:
Attaching 11 probes...
=== 80s report ===
--- Backend ---
intr_callback: 207367
ui_handler: 207367
uintr_wait:     101343
wakers:         101533
send_uipi: 90854
--- Gateway ---
intr_callback: 86414
ui_handler: 86414
uintr_wait:     83092
wakers:         83087
send_uipi: 343653


@irq_backend: 207367
@irq_gateway: 86414
@send_uipi_backend: 90854
@send_uipi_gateway: 343653
@ui_handler_backend: 207367
@ui_handler_gateway: 86414
@wait_backend: 101343
@wait_gateway: 83092
@wakers_backend: 101533
@wakers_gateway: 83087
```

完全不使用中断合并
```
Running oha (matrix 2x2, 8 connections)...
2026-05-25T15:33:53.344757Z  WARN gateway::shm_transport_uintr: SHM: 360us
2026-05-25T15:34:03.770892Z  WARN backend::shm_server_uintr: UINTR blocking thread: uintr_wait timeout
2026-05-25T15:34:13.770995Z  WARN backend::shm_server_uintr: UINTR blocking thread: uintr_wait timeout
2026-05-25T15:34:23.771068Z  WARN backend::shm_server_uintr: UINTR blocking thread: uintr_wait timeout
2026-05-25T15:34:33.771142Z  WARN backend::shm_server_uintr: UINTR blocking thread: uintr_wait timeout
2026-05-25T15:34:43.771212Z  WARN backend::shm_server_uintr: UINTR blocking thread: uintr_wait timeout
2026-05-25T15:34:53.344559Z  WARN gateway: Error serving connection from 127.0.0.1:47856: connection closed before message completed
2026-05-25T15:34:53.344592Z  WARN gateway: Error serving connection from 127.0.0.1:47836: connection closed before message completed
2026-05-25T15:34:53.344605Z  WARN gateway: Error serving connection from 127.0.0.1:47858: connection closed before message completed
2026-05-25T15:34:53.344617Z  WARN gateway: Error serving connection from 127.0.0.1:47840: connection closed before message completed
2026-05-25T15:34:53.344630Z  WARN gateway: Error serving connection from 127.0.0.1:47868: connection closed before message completed
2026-05-25T15:34:53.344643Z  WARN gateway: Error serving connection from 127.0.0.1:47872: connection closed before message completed
2026-05-25T15:34:53.344645Z  WARN gateway: Error serving connection from 127.0.0.1:47854: connection closed before message completed
2026-05-25T15:34:53.344656Z  WARN gateway: Error serving connection from 127.0.0.1:47852: connection closed before message completed
Summary:
  Success rate: 100.00%
  Total:        60002.1146 ms
  Slowest:      1.3605 ms
  Fastest:      0.0442 ms
  Average:      0.1534 ms
  Requests/sec: 365.6038

  Total data:   1.85 MiB
  Size/request: 88 B
  Size/sec:     31.65 KiB

Response time histogram:
  0.044 ms [1]     |
  0.176 ms [17483] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.307 ms [2696]  |■■■■
  0.439 ms [1359]  |■■
  0.571 ms [323]   |
  0.702 ms [37]    |
  0.834 ms [9]     |
  0.966 ms [4]     |
  1.097 ms [8]     |
  1.229 ms [7]     |
  1.360 ms [2]     |

Response time distribution:
  10.00% in 0.0859 ms
  25.00% in 0.1034 ms
  50.00% in 0.1280 ms
  75.00% in 0.1634 ms
  90.00% in 0.2660 ms
  95.00% in 0.3604 ms
  99.00% in 0.4784 ms
  99.90% in 0.8166 ms
  99.99% in 1.1790 ms


Details (average, fastest, slowest):
  DNS+dialup:   0.2535 ms, 0.0824 ms, 0.6027 ms
  DNS-lookup:   0.0727 ms, 0.0122 ms, 0.4009 ms

Status code distribution:
  [200] 21929 responses

Error distribution:
  [8] aborted due to deadline

==========================================
All Matrix Multiplication Tests Complete!
==========================================

Results saved to: log/matrix_latency_20260525-233343/

bpftrace results:
Attaching 11 probes...
=== 80s report ===
--- Backend ---
intr_callback: 11774
ui_handler: 11774
uintr_wait:     5205
wakers:         5200
send_uipi: 20787
--- Gateway ---
intr_callback: 8189
ui_handler: 8189
uintr_wait:     4466
wakers:         4460
send_uipi: 21486


@irq_backend: 11774
@irq_gateway: 8189
@send_uipi_backend: 20787
@send_uipi_gateway: 21486
@ui_handler_backend: 11774
@ui_handler_gateway: 8189
@wait_backend: 5205
@wait_gateway: 4466
@wakers_backend: 5200
@wakers_gateway: 4460
```

### 第五个版本

增加一个中断关闭区域，但是没有什么用，性能变得很差。

```rust
unsafe { clui(); }
let has = unsafe { uintr_received > 0 };
if has { unsafe { uintr_received = 0; } }
unsafe { stui(); }
if has {
    process_global_uintr_wakers();
    continue;
}
```
### 1. CLUI — 清 UIF，阻断投递
来源：Intel SDM Volume 2，指令 CLUI（可在 felixcloutier.com/x86/clui 查看）：
 CLUI clears the user interrupt flag (UIF). Its effect takes place immediately: a user interrupt cannot be delivered on the instruction boundary following CLUI.

效果 ：UIF=0 后，用户态中断 不可投递 。但注意，"不可投递"不等于"不能发送"——这是两个独立动作。

### 2. STUI — 设 UIF，触发投递
来源： felixcloutier.com/x86/stui ：
 STUI sets the user interrupt flag (UIF). Its effect takes place immediately; a user interrupt may be delivered on the instruction boundary following STUI.
```
UIF := 1;
```
STUI 后，如果 UPID.ON=1（表示有未处理的通知），硬件在 STUI 的下一条指令边界触发 handler。

### 3. UPID 的 ON 位—硬件级别的"暂存"
来自 SENDUIPI 的 Operation 文档 ：
 Bit 0 (ON) indicates an outstanding notification. If this bit is set, there is a notification outstanding for one or more user interrupts in PIR.
UPID.ON 就是硬件级别的 "pending" 标志。CLUI 期间 SENDUIPI 设置 ON=1 ，但硬件不投递；STUI 后 UIF 变为 1，硬件检测到 ON=1，触发 handler。

# 猜想三

在 uintr_received = 0 到 进入 match uintr::syscall::uintr_wait 期间刚好剩下额所有中断都发送了（8个连接的resp如果不返回，后面的都不发送），导致uintr_wait等待期间没有发送中断。

而且只要在 uintr_received = 0 到 进入 match uintr::syscall::uintr_wait 期间收到一个中断，那么由于中断处理路径比较长，在处理上一个中断的期间收到下一个中断的可能性也非常大，退出上一个中断处理的时候优惠马上进入下一个中断处理，一直连续。


```rust
            while running_clone.load(Ordering::SeqCst) {
                if unsafe { uintr_received > 0 } {
                    unsafe { uintr_received = 0; }
                    process_global_uintr_wakers();
                    continue;
                }
                match uintr::syscall::uintr_wait(uintr::UINTR_WAIT_MAX_USEC, 0) {
                    Ok(true) => {
                        warn!("UINTR blocking thread: uintr_wait success");
                    }
                    _ => {
                        warn!("UINTR blocking thread: uintr_wait timeout");
                    }
                }
            }
            warn!("UINTR blocking thread: exited");
        });
```

尝试的解决方案是缩短中断处理程序。

目前这个解决方案是最好的，经过实验证明，降低大幅度降低出现死锁的概率。

如果UINTR_WAIT_MAX_USEC 设置为100ms超时，那么死锁带来的性能开销也会减少。

数据非常清晰——P10=25.79ms, P50=25.99ms, P90=26.15ms，**几乎所有请求的延迟都集中在 ~26ms，离散度极低**。这指向一个确定性的瓶颈，不是随机争用。

## Rust 编译器在 Release 模式下缓存了 `uintr_received`

```
C 代码:      volatile unsigned long uintr_received;
Rust FFI:    extern "C" { static mut uintr_received: libc::c_ulong; }
                                                       ↑
                                            volatile 丢失了！
```

Rust 的 FFI `static mut` **不继承 C 的 `volatile` 语义**。在 Release + LTO 下，LLVM 看到只是一个普通的全局变量读，有权**将其缓存到寄存器**。

快速路径下的读取无法将其视为volatile，并不一定会更新。只有从uintr_wait返回才会更新。

`uintr_wait` 是一个 syscall。**syscall 本身就是最强的编译器屏障**——编译器保守地假设 syscall 可能修改任何内存，所以回来后必定重新从内存加载 `uintr_received`。

把 `uintr_received` 的访问从普通读/写改成 `read_volatile` / `write_volatile`：

```diff
- if unsafe { uintr_received > 0 } {
-     unsafe { uintr_received = 0; }
+ if unsafe { std::ptr::read_volatile(&raw const uintr_received) > 0 } {
+     unsafe { std::ptr::write_volatile(&raw mut uintr_received, 0); }
```

这会强制编译器每次循环迭代都从内存重新加载 `uintr_received`，不再使用寄存器缓存。

（这个volatile read依然不能解决busy loop 等待的性能问题）