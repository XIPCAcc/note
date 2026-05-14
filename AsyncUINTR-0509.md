# 第一阶段

| Size | uintr Req/s | eventfd Req/s | UINTR 优势 |
|------|-------------|---------------|-----------|
| 4    | 155,504     | 131,025       | **+18.7%** |
| 8    | 140,441     | 121,749       | **+15.4%** |
| 12   | 124,048     | 106,626       | **+16.3%** |
| 13   | 117,055     | 104,378       | **+12.1%** |
| 14   | 111,745     | 94,183        | **+18.6%** |
| 15   | 104,776     | 94,445        | **+10.9%** |
| 16   | 97,292      | 89,170        | **+9.1%** |
| 17   | 92,687      | 85,086        | **+8.9%** |

# 第二阶段

### 绝对吞吐对比（c=64）

| Size | tcp | uds | shm-eventfd | shm-uintr | SHM vs TCP |
|------|-----|-----|-------------|-----------|------------|
| 4 | 34,403 | 38,480 | 130,731 | **152,822** | **4.4x** |
| 8 | 33,613 | 36,999 | 121,923 | **134,751** | **4.0x** |
| 16 | 32,087 | 34,327 | 88,926 | **94,208** | **2.9x** |
| 32 | 24,889 | 26,233 | 52,981 | **54,781** | **2.2x** |
| 64 | 12,444 | 12,370 | 26,197 | **26,781** | **2.2x** |
| 128 | 3,341 | 3,197 | **6,494** | 6,306 | 1.9x |

### 四、延迟对比（c=64，avg）

| Size | tcp | uds | shm-eventfd | shm-uintr |
|------|-----|-----|-------------|-----------|
| 4 | 1.86ms | 1.67ms | 490us | **421us** |
| 8 | 1.90ms | 1.74ms | 536us | **478us** |
| 16 | 2.01ms | 1.87ms | 727us | **694us** |
| 32 | 2.56ms | 2.44ms | 1.22ms | **1.19ms** |
| 64 | 5.15ms | 5.17ms | 2.52ms | **2.46ms** |
| 128 | 19.4ms | 20.4ms | **10.1ms** | 10.7ms |

---

#### 1. UINTR 最优场景：小矩阵 + 中高并发

```
最优参数：size=4~8, concurrency=64~256
最大优势：+17.3%（size=8, c=256）
```

此时每次请求的计算量极小（size=4 仅 128 次浮点运算），通知延迟占总延迟比例最高，内核旁路的收益最大。

#### 2. UINTR 优势随矩阵增大单调递减

```
size=4:  +16.9%  ← 通知延迟占比高，UINTR 价值大
size=8:  +10.5%
size=16:  +5.9%
size=32:  +3.4%  ← 计算开始主导
size=64:  +2.2%  ← 优势基本消失
size=128: -2.9%  ← UINTR 反而更慢
```

### 请求的完整生命周期（可拆为 ~15 个阶段）

```
Gateway 侧                              Backend 侧
─────────────                          ─────────────
1. pending lock                        10. UINTR wait (收通知)
2. bincode deserialize matrix_a/b      11. ring buffer read
3. flatten + f64→bytes                 12. bincode parse ShmRequest
4. data_pool.allocate (CAS)            13. data_pool read (memcpy)
5. data_pool.write ×2 (memcpy)         14. bytes→f64 + reconstruct ×2
6. bincode serialize ShmRequest        15. multiply_matrices O(n³)
7. ring buffer write (Mutex)           16. calculate_checksum O(n²)
8. senduipi (UINTR 通知)               17. bincode serialize ×2
9. rx.await (等响应) ─────────────────→ 18. ring buffer write
                                        19. senduipi (响应通知)
                                        ↓
                                    Gateway 响应监听器:
                                    20. UINTR wait
                                    21. ring buffer read
                                    22. bincode parse ShmResponse
                                    23. pending lock + tx.send
                                        ↓
                                    回到 9. rx.await 返回
```

### 各阶段随 size 的变化规律

| 阶段 | 复杂度 | size=4 | size=128 | 说明 |
|------|--------|--------|----------|------|
| deserialize matrix | O(n²) | ~128B | ~128KB | bincode 解包 |
| flatten + bytes | O(n²) | 128B | 128KB | 矩阵展平 |
| data_pool alloc | O(1) | ~50ns | ~50ns | CAS 无锁 |
| data_pool write | O(n²) | 256B | 256KB | memcpy |
| ring buffer write | O(1) | ~100B | ~100B | 只传 offset |
| senduipi | O(1) | ~200ns | ~200ns | 写 MSR |
| **multiply** | **O(n³)** | **128 FLOP** | **4M FLOP** | **主导因素** |
| checksum | O(n²) | 16 次 | 16384 次 | 可忽略 |
| UINTR wait | O(1) | ~2-5us | ~2-5us | 用户态唤醒 |
