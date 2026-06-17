
# 如果绑定了用户态中断的核才能获取park driver锁

那么每次进入driver才更新token，然后唤醒，这样就和原本的epoll机制一样，只是少了进入epoll_wait这一次系统调用的时间，但是却需要无数次切换中断上下文。

而且还牺牲了多核竞争 park driver 

```
==========================================
Generating Summary Report...
==========================================
Summary saved to: log/matrix_20260617-033255/summary.txt
Matrix Multiplication Performance Comparison
===========================================
Date: Wed Jun 17 03:56:28 AM CST 2026

Matrix Sizes: 2 4 8 16 32 64 128 256 512
Concurrencies: 16 32 64 128 256 512 1024
Test Duration: 15s each

Transport    Matrix   Concurrency  QPS(req/s)     AvgLat(ms)     P99Lat(ms)    
---------    ------   ----------   ----------     ----------     ----------    


shm-uintr    2x2      16           157782.33      100.11us       260.00us      
shm-uintr    2x2      32           170062.14      185.13us       435.00us      
shm-uintr    2x2      64           139932.18      451.99us       0.99ms        
shm-uintr    2x2      128          184427.03      684.38us       1.39ms        
shm-uintr    2x2      256          186899.25      1.35ms         2.73ms        
shm-uintr    2x2      512          199521.69      2.50ms         4.78ms        
shm-uintr    2x2      1024         210382.51      4.88ms         7.93ms        
shm-uintr    4x4      16           73175.32       118.01us       339.00us      
shm-uintr    4x4      32           6342.93        203.14us       457.00us      
shm-uintr    4x4      64           158801.67      398.32us       0.88ms        
shm-uintr    4x4      128          159525.04      794.77us       1.60ms        
shm-uintr    4x4      256          184170.91      1.38ms         2.72ms        
shm-uintr    4x4      512          194814.29      2.57ms         4.71ms        
shm-uintr    4x4      1024         185079.09      5.52ms         9.83ms        
shm-uintr    8x8      16           118474.62      135.71us       399.00us      
shm-uintr    8x8      32           126462.95      252.86us       637.00us      
shm-uintr    8x8      64           133912.71      475.52us       0.99ms        
shm-uintr    8x8      128          142998.07      0.89ms         1.69ms        
shm-uintr    8x8      256          150910.69      1.69ms         3.29ms        
shm-uintr    8x8      512          157924.34      3.22ms         6.11ms        
shm-uintr    8x8      1024         149588.37      6.75ms         10.90ms       
shm-uintr    16x16    16           79404.30       202.30us       551.00us      
shm-uintr    16x16    32           80399.17       399.00us       0.94ms        
shm-uintr    16x16    64           80557.39       794.42us       1.66ms        
shm-uintr    16x16    128          75449.81       1.69ms         3.29ms        
shm-uintr    16x16    256          82368.97       3.09ms         6.07ms        
shm-uintr    16x16    512          81718.63       6.13ms         10.65ms       
shm-uintr    16x16    1024         78881.63       12.52ms        19.75ms       
shm-uintr    32x32    16           50219.36       319.56us       805.00us      
shm-uintr    32x32    32           50076.15       648.06us       1.69ms        
shm-uintr    32x32    64           44366.26       1.46ms         3.54ms        
shm-uintr    32x32    128          42033.42       3.06ms         7.09ms        
shm-uintr    32x32    256          44955.22       5.70ms         11.57ms       
shm-uintr    32x32    512          44821.14       11.19ms        19.51ms       
shm-uintr    32x32    1024         44206.64       24.05ms        43.89ms       
shm-uintr    64x64    16           21724.66       742.35us       1.84ms        
shm-uintr    64x64    32           24174.14       1.35ms         3.34ms        
shm-uintr    64x64    64           23495.60       2.76ms         6.53ms        
shm-uintr    64x64    128          19884.93       6.46ms         12.78ms       
shm-uintr    64x64    256          19089.76       13.45ms        27.68ms       
shm-uintr    64x64    512          18966.45       26.44ms        48.92ms       
shm-uintr    64x64    1024         17433.34       56.95ms        105.45ms      
shm-uintr    128x128  16           3851.54        4.19ms         10.18ms       
shm-uintr    128x128  32           4434.98        7.26ms         16.04ms       
shm-uintr    128x128  64           5065.02        12.68ms        25.28ms       
shm-uintr    128x128  128          5010.70        25.92ms        57.22ms       
shm-uintr    128x128  256          5075.43        51.03ms        118.75ms      
shm-uintr    128x128  512          5010.29        100.42ms       201.16ms      
shm-uintr    128x128  1024         4980.31        195.30ms       425.60ms      
shm-uintr    256x256  16           472.52         34.25ms        84.36ms       
shm-uintr    256x256  32           643.90         49.83ms        110.14ms      
shm-uintr    256x256  64           649.44         98.12ms        188.62ms      
shm-uintr    256x256  128          679.92         187.50ms       443.49ms      
shm-uintr    256x256  256          697.19         369.16ms       1.10s         
shm-uintr    512x512  16           52.73          313.14ms       992.58ms      
shm-uintr    512x512  32           58.15          543.57ms       1.24s         
shm-uintr    512x512  64           62.25          984.38ms       1.72s         
shm-uintr    512x512  128          63.44          1.44s          1.99s         
shm-uintr    512x512  256          59.99          1.60s          2.00s         
```

# 每次调度都尝试settoken并唤醒

```
==========================================
All Matrix Multiplication Tests Complete!
==========================================

Results saved to: log/matrix_20260617-035836/

==========================================
Generating Summary Report...
==========================================
Summary saved to: log/matrix_20260617-035836/summary.txt
Matrix Multiplication Performance Comparison
===========================================
Date: Wed Jun 17 04:21:56 AM CST 2026

Matrix Sizes: 2 4 8 16 32 64 128 256 512
Concurrencies: 16 32 64 128 256 512 1024
Test Duration: 15s each

Transport    Matrix   Concurrency  QPS(req/s)     AvgLat(ms)     P99Lat(ms)    
---------    ------   ----------   ----------     ----------     ----------    


shm-uintr    2x2      16           91200.93       93.71us        231.00us      
shm-uintr    2x2      32           145657.20      219.27us       573.00us      
shm-uintr    2x2      64           168199.02      377.08us       0.86ms        
shm-uintr    2x2      128          179922.07      703.04us       1.40ms        
shm-uintr    2x2      256          205851.98      1.23ms         2.31ms        
shm-uintr    2x2      512          203962.65      2.45ms         4.49ms        
shm-uintr    2x2      1024         206913.18      4.78ms         7.58ms        
shm-uintr    4x4      16           95893.61       99.36us        257.00us      
shm-uintr    4x4      32           159142.06      199.74us       497.00us      
shm-uintr    4x4      64           153600.86      413.39us       0.91ms        
shm-uintr    4x4      128          164712.34      767.18us       20.74k        
shm-uintr    4x4      256          189264.10      1.34ms         2.51ms        
shm-uintr    4x4      512          188626.59      2.65ms         4.64ms        
shm-uintr    4x4      1024         186140.83      5.50ms         5.50ms        
shm-uintr    8x8      16           121976.34      131.87us       380.00us      
shm-uintr    8x8      32           125623.46      255.54us       649.00us      
shm-uintr    8x8      64           135505.23      468.15us       0.98ms        
shm-uintr    8x8      128          144501.89      0.88ms         1.65ms        
shm-uintr    8x8      256          155176.51      1.64ms         3.25ms        
shm-uintr    8x8      512          157401.13      3.18ms         6.40ms        
shm-uintr    8x8      1024         152101.09      6.43ms         10.53ms       
shm-uintr    16x16    16           73072.45       220.66us       597.00us      
shm-uintr    16x16    32           88937.72       361.48us       0.88ms        
shm-uintr    16x16    64           86465.73       739.12us       1.55ms        
shm-uintr    16x16    128          83206.53       1.54ms         3.01ms        
shm-uintr    16x16    256          82773.81       3.09ms         6.25ms        
shm-uintr    16x16    512          85111.04       5.90ms         10.32ms       
shm-uintr    16x16    1024         80610.77       12.35ms        18.56ms       
shm-uintr    32x32    16           37565.14       545.61us       4.18ms        
shm-uintr    32x32    32           52325.79       627.37us       1.82ms        
shm-uintr    32x32    64           45246.14       1.44ms         3.79ms        
shm-uintr    32x32    128          42114.41       3.05ms         7.17ms        
shm-uintr    32x32    256          45154.95       5.68ms         11.66ms       
shm-uintr    32x32    512          45405.51       11.22ms        19.84ms       
shm-uintr    32x32    1024         43787.63       22.65ms        35.62ms       
shm-uintr    64x64    16           20541.00       818.76us       2.89ms        
shm-uintr    64x64    32           21747.23       1.70ms         9.10ms        
shm-uintr    64x64    64           24357.34       2.64ms         6.20ms        
shm-uintr    64x64    128          20073.19       6.41ms         13.05ms       
shm-uintr    64x64    256          18692.84       13.67ms        26.44ms       
shm-uintr    64x64    512          17808.58       28.26ms        50.44ms       
shm-uintr    64x64    1024         16263.72       59.75ms        117.06ms      
shm-uintr    128x128  16           3849.62        4.54ms         18.43ms       
shm-uintr    128x128  32           5201.45        6.19ms         13.54ms       
shm-uintr    128x128  64           5212.86        12.35ms        25.86ms       
shm-uintr    128x128  128          5114.63        25.07ms        49.95ms       
shm-uintr    128x128  256          5179.82        48.41ms        96.23ms       
shm-uintr    128x128  512          5044.97        100.63ms       249.43ms      
shm-uintr    128x128  1024         4670.61        221.03ms       743.19ms      
shm-uintr    256x256  16           652.33         25.14ms        66.65ms       
shm-uintr    256x256  32           715.86         46.18ms        135.14ms      
shm-uintr    256x256  64           709.22         98.73ms        355.37ms      
shm-uintr    256x256  128          703.04         202.69ms       799.50ms      
shm-uintr    256x256  256          693.83         396.51ms       1.38s         
shm-uintr    512x512  16           61.33          276.57ms       981.73ms      
shm-uintr    512x512  32           65.22          491.33ms       1.82s         
shm-uintr    512x512  64           64.56          485.64ms       1.88s         
shm-uintr    512x512  128          63.39          614.18ms       1.91s         
shm-uintr    512x512  256          61.56          696.01ms       1.94s         
```