# 用户态中断通过 futex wake 主线程

```
UINTR → handler_wake_park
  ↓ futex_wake
main thread futex_wait 返回
  ↓ process_global_uintr_wakers
waker.wake()
  → schedule_task()
  → with_current → None  ← 主线程不是 worker
  → push_remote_task → 全局 inject queue
  → notify_parked_remote  ← 还要唤醒一个 worker
  ↓
worker 醒来 → 从 inject queue 偷取任务 → 执行
```

对比之前的
```
UINTR → self-pipe write
  ↓ (内核立刻唤醒 epoll_wait)
worker epoll_wait 返回
  ↓ (同一线程，同一 stack frame)
process_uintr_wakers → waker.wake()
  → schedule_task()
  → with_current → Some(ctx)  ← 当前就是 worker
  → schedule_local → LIFO slot 或 push_back  ← 本地队列，无锁/低竞争
  ↓ (turn() 返回后，worker 主循环)
run_task() → poll UintrFuture → Ready
  ↓
读响应 → tx.send() → main thread unpark
```

```
Matrix Multiplication Performance Comparison
===========================================
Date: Sat Jun 20 09:03:42 AM CST 2026

Matrix Sizes: 2 4 8 16 32 64 128 256 512
Concurrencies: 8 16 32 64 128 256 512 1024 2048
Test Duration: 15s each

Transport    Matrix   Concurrency  QPS(req/s)     AvgLat(ms)     P99Lat(ms)    
---------    ------   ----------   ----------     ----------     ----------    


shm-uintr    2x2      8            116130.86      82.29us        516.00us      
shm-uintr    2x2      16           149716.18      121.12us       626.00us      
shm-uintr    2x2      32           153803.00      218.65us       812.00us      
shm-uintr    2x2      64           165279.66      392.50us       1.12ms        
shm-uintr    2x2      128          189569.61      677.39us       1.55ms        
shm-uintr    2x2      256          212156.35      1.20ms         2.33ms        
shm-uintr    2x2      512          204579.04      2.44ms         4.53ms        
shm-uintr    2x2      1024         191129.15      5.50ms         14.77ms       
shm-uintr    2x2      2048         185469.62      21.81ms        440.88ms      
shm-uintr    4x4      8            106627.98      105.43us       614.00us      
shm-uintr    4x4      16           138757.90      127.81us       619.00us      
shm-uintr    4x4      32           144200.92      232.87us       0.86ms        
shm-uintr    4x4      64           152057.01      426.37us       1.19ms        
shm-uintr    4x4      128          176206.13      727.00us       1.67ms        
shm-uintr    4x4      256          199234.33      1.28ms         2.40ms        
shm-uintr    4x4      512          62184.73       2.59ms         4.91ms        
shm-uintr    4x4      1024         174293.77      5.77ms         14.01ms       
shm-uintr    4x4      2048         153050.91      16.44ms        41.22ms       
shm-uintr    8x8      8            95543.78       99.17us        570.00us      
shm-uintr    8x8      16           122676.32      144.13us       665.00us      
shm-uintr    8x8      32           130314.44      255.70us       0.87ms        
shm-uintr    8x8      64           138757.51      468.69us       1.27ms        
shm-uintr    8x8      128          156966.26      816.10us       1.78ms        
shm-uintr    8x8      256          165741.46      1.55ms         3.21ms        
shm-uintr    8x8      512          160618.75      3.13ms         5.88ms        
shm-uintr    8x8      1024         160456.91      6.28ms         16.94ms       
shm-uintr    8x8      2048         126186.51      25.76ms        414.73ms      
shm-uintr    16x16    8            67985.87       131.79us       619.00us      
shm-uintr    16x16    16           85990.79       200.91us       781.00us      
shm-uintr    16x16    32           95536.13       346.72us       1.09ms        
shm-uintr    16x16    64           98327.71       659.17us       1.65ms        
shm-uintr    16x16    128          98687.51       1.30ms         2.81ms        
shm-uintr    16x16    256          101397.69      2.51ms         5.39ms        
shm-uintr    16x16    512          92303.26       5.44ms         11.37ms       
shm-uintr    16x16    1024         77447.58       12.90ms        26.05ms       
shm-uintr    16x16    2048         76217.20       37.94ms        549.53ms      
shm-uintr    32x32    8            37169.84       227.77us       0.86ms        
shm-uintr    32x32    16           41869.09       401.58us       1.35ms        
shm-uintr    32x32    32           49815.81       671.28us       2.13ms        
shm-uintr    32x32    64           51703.22       1.29ms         4.24ms        
shm-uintr    32x32    128          50915.17       2.54ms         6.46ms        
shm-uintr    32x32    256          53069.07       4.78ms         9.74ms        
shm-uintr    32x32    512          43714.99       11.47ms        21.91ms       
shm-uintr    32x32    1024         39556.46       24.93ms        44.14ms       
shm-uintr    32x32    2048         38901.41       51.39ms        88.24ms       
shm-uintr    64x64    8            14634.90       601.98us       2.41ms        
shm-uintr    64x64    16           20477.38       806.66us       2.54ms        
shm-uintr    64x64    32           26579.14       1.22ms         3.17ms        
shm-uintr    64x64    64           26742.37       2.41ms         5.78ms        
shm-uintr    64x64    128          25491.78       5.09ms         12.51ms       
shm-uintr    64x64    256          22688.57       11.21ms        24.87ms       
shm-uintr    64x64    512          19100.91       26.40ms        55.76ms       
shm-uintr    64x64    1024         17575.00       56.08ms        118.17ms      
shm-uintr    64x64    2048         16848.17       118.81ms       432.29ms      
shm-uintr    128x128  8            3030.57        2.75ms         8.09ms        
shm-uintr    128x128  16           4493.69        3.63ms         9.05ms        
shm-uintr    128x128  32           5367.91        6.00ms         12.71ms       
shm-uintr    128x128  64           5368.82        11.93ms        25.73ms       
shm-uintr    128x128  128          5269.08        24.49ms        55.24ms       
shm-uintr    128x128  256          5140.80        50.40ms        127.39ms      
shm-uintr    128x128  512          4882.93        104.16ms       263.00ms      
shm-uintr    128x128  1024         4604.17        220.80ms       712.22ms      
shm-uintr    128x128  2048         4307.22        377.88ms       934.58ms      
shm-uintr    256x256  8            407.45         20.47ms        60.48ms       
shm-uintr    256x256  16           682.32         24.08ms        64.74ms       
shm-uintr    256x256  32           689.65         47.67ms        135.26ms      
shm-uintr    256x256  64           683.04         97.05ms        291.99ms      
shm-uintr    256x256  128          672.59         206.66ms       760.45ms      
shm-uintr    256x256  256          655.32         399.74ms       1.23s         
shm-uintr    512x512  8            41.60          198.06ms       712.49ms      
shm-uintr    512x512  16           63.03          269.98ms       976.05ms      
shm-uintr    512x512  32           61.87          519.58ms       1.48s         
shm-uintr    512x512  64           60.27          808.24ms       1.92s         
shm-uintr    512x512  128          58.96          904.82ms       1.96s         
shm-uintr    512x512  256          59.00          1.01s          1.98s         


```

## 只有用户态中断协程阻塞时才调用futex wake

(没差别)

```Matrix Multiplication Performance Comparison
===========================================
Date: Sat Jun 20 09:36:14 AM CST 2026

Matrix Sizes: 2 4 8 16 32 64 128 256 512
Concurrencies: 8 16 32 64 128 256 512 1024 2048
Test Duration: 15s each

Transport    Matrix   Concurrency  QPS(req/s)     AvgLat(ms)     P99Lat(ms)    
---------    ------   ----------   ----------     ----------     ----------    


shm-uintr    2x2      8            118457.70      81.32us        530.00us      
shm-uintr    2x2      16           146594.75      122.92us       637.00us      
shm-uintr    2x2      32           156496.78      215.22us       815.00us      
shm-uintr    2x2      64           167214.56      388.81us       1.13ms        
shm-uintr    2x2      128          191213.77      667.71us       1.54ms        
shm-uintr    2x2      256          212160.41      1.20ms         2.34ms        
shm-uintr    2x2      512          196315.92      2.56ms         4.63ms        
shm-uintr    2x2      1024         179887.73      5.97ms         12.31ms       
shm-uintr    2x2      2048         183607.80      19.01ms        280.97ms      
shm-uintr    4x4      8            109775.33      86.71us        86.71us       
shm-uintr    4x4      16           141288.59      128.14us       640.00us      
shm-uintr    4x4      32           143757.10      246.81us       0.91ms        
shm-uintr    4x4      64           153189.96      428.40us       1.23ms        
shm-uintr    4x4      128          178851.61      715.37us       1.65ms        
shm-uintr    4x4      256          197670.87      1.29ms         2.48ms        
shm-uintr    4x4      512          195751.73      2.56ms         4.55ms        
shm-uintr    4x4      1024         177307.46      5.96ms         12.47ms       
shm-uintr    4x4      2048         144714.76      22.41ms        334.18ms      
shm-uintr    8x8      8            93130.02       102.90us       582.00us      
shm-uintr    8x8      16           123689.10      144.54us       660.00us      
shm-uintr    8x8      32           129571.82      257.61us       0.90ms        
shm-uintr    8x8      64           138879.65      464.94us       1.22ms        
shm-uintr    8x8      128          155389.49      826.14us       1.85ms        
shm-uintr    8x8      256          165934.70      1.54ms         3.17ms        
shm-uintr    8x8      512          161127.81      3.16ms         6.15ms        
shm-uintr    8x8      1024         154898.39      6.62ms         16.48ms       
shm-uintr    8x8      2048         125834.92      16.05ms        30.68ms       
shm-uintr    16x16    8            67499.79       132.23us       620.00us      
shm-uintr    16x16    16           82227.07       207.69us       799.00us      
shm-uintr    16x16    32           97369.18       339.61us       1.08ms        
shm-uintr    16x16    64           97609.93       665.83us       1.70ms        
shm-uintr    16x16    128          97831.66       1.32ms         2.94ms        
shm-uintr    16x16    256          100575.12      2.58ms         6.23ms        
shm-uintr    16x16    512          96923.62       5.21ms         10.74ms       
shm-uintr    16x16    1024         81056.00       12.55ms        28.76ms       
shm-uintr    16x16    2048         73324.97       26.36ms        50.98ms       
shm-uintr    32x32    8            36647.69       232.41us       0.89ms        
shm-uintr    32x32    16           40534.13       417.22us       1.41ms        
shm-uintr    32x32    32           43023.51       0.89ms         5.26ms        
shm-uintr    32x32    64           51766.06       1.28ms         4.11ms        
shm-uintr    32x32    128          51391.82       2.53ms         6.85ms        
shm-uintr    32x32    256          53075.49       4.80ms         10.16ms       
shm-uintr    32x32    512          45487.65       11.07ms        22.43ms       
shm-uintr    32x32    1024         35595.75       27.70ms        53.83ms       
shm-uintr    32x32    2048         37312.49       47.71ms        78.93ms       
shm-uintr    64x64    8            15633.08       523.20us       1.51ms        
shm-uintr    64x64    16           20331.15       811.38us       2.50ms        
shm-uintr    64x64    32           25796.38       1.26ms         3.37ms        
shm-uintr    64x64    64           27060.45       2.39ms         5.93ms        
shm-uintr    64x64    128          24987.49       5.18ms         11.91ms       
shm-uintr    64x64    256          22432.78       11.58ms        27.95ms       
shm-uintr    64x64    512          19939.59       25.31ms        53.30ms       
shm-uintr    64x64    1024         18457.30       54.11ms        2.34k         
shm-uintr    64x64    2048         18277.56       104.77ms       190.09ms      
shm-uintr    128x128  8            3092.88        2.68ms         8.04ms        
shm-uintr    128x128  16           4502.56        3.62ms         8.95ms        
shm-uintr    128x128  32           5320.19        6.05ms         13.07ms       
shm-uintr    128x128  64           5382.18        11.93ms        24.88ms       
shm-uintr    128x128  128          5337.67        24.06ms        50.63ms       
shm-uintr    128x128  256          5248.63        48.38ms        103.59ms      
shm-uintr    128x128  512          5086.80        98.96ms        236.83ms      
shm-uintr    128x128  1024         4811.06        204.79ms       582.41ms      
shm-uintr    128x128  2048         4353.01        420.21ms       1.17s         
shm-uintr    256x256  8            422.13         19.78ms        62.30ms       
shm-uintr    256x256  16           677.57         24.25ms        65.29ms       
shm-uintr    256x256  32           687.72         47.72ms        129.75ms      
shm-uintr    256x256  64           693.73         95.05ms        270.39ms      
shm-uintr    256x256  128          680.64         197.12ms       620.52ms      
shm-uintr    256x256  256          677.13         387.96ms       1.18s         
shm-uintr    512x512  8            43.12          191.98ms       749.80ms      
shm-uintr    512x512  16           62.45          271.61ms       983.06ms      
shm-uintr    512x512  32           62.31          510.71ms       1.45s         
shm-uintr    512x512  64           61.50          807.79ms       1.95s         
shm-uintr    512x512  128          60.99          915.23ms       1.95s         
shm-uintr    512x512  256          58.53          1.07s          2.00s         


```