# 关于self-pipe 的一个改进点

如果在写入self-pipe 之前检查一下当前协程是否休眠在决定是否写，那么在 小计算量的情况下就可以大幅度减少 sys write，性能是完全优于 eventfd的。

但是当计算量大，协程大部分时间处于等待状态的时候，那么此时sys write的开销就不可避免。

```
log/matrix_20260615-202636

Matrix Multiplication Performance Comparison
===========================================
Date: Mon Jun 15 09:13:46 PM CST 2026

Matrix Sizes: 2 4 8 16 32 64 128 256 512
Concurrencies: 8 16 32 64 128 256 512
Test Duration: 15s each

Transport    Matrix   Concurrency  QPS(req/s)     AvgLat(ms)     P99Lat(ms)    
---------    ------   ----------   ----------     ----------     ----------    

shm-eventfd  2x2      8            116094.11      76.67us        440.00us      
shm-eventfd  2x2      16           147920.21      112.76us       452.00us      
shm-eventfd  2x2      32           143910.07      223.74us       603.00us      
shm-eventfd  2x2      64           151236.57      420.53us       0.93ms        
shm-eventfd  2x2      128          169381.20      750.09us       1.50ms        
shm-eventfd  2x2      256          178396.65      1.43ms         2.75ms        
shm-eventfd  2x2      512          181243.72      2.77ms         4.90ms        
shm-eventfd  4x4      8            106888.40      82.14us        438.00us      
shm-eventfd  4x4      16           131913.62      126.47us       487.00us      
shm-eventfd  4x4      32           131581.35      244.33us       639.00us      
shm-eventfd  4x4      64           140758.69      452.31us       0.97ms        
shm-eventfd  4x4      128          156379.46      813.46us       1.61ms        
shm-eventfd  4x4      256          168096.20      1.51ms         2.83ms        
shm-eventfd  4x4      512          169463.61      2.95ms         4.80ms        
shm-eventfd  8x8      8            98510.73       87.50us        438.00us      
shm-eventfd  8x8      16           118500.25      139.70us       499.00us      
shm-eventfd  8x8      32           119874.01      268.37us       674.00us      
shm-eventfd  8x8      64           126812.52      503.52us       1.08ms        
shm-eventfd  8x8      128          138760.37      0.92ms         1.74ms        
shm-eventfd  8x8      256          144261.63      1.77ms         3.24ms        
shm-eventfd  8x8      512          145462.66      3.44ms         5.63ms        
shm-eventfd  16x16    8            74676.37       114.52us       498.00us      
shm-eventfd  16x16    16           89976.36       183.16us       592.00us      
shm-eventfd  16x16    32           95127.17       338.62us       0.85ms        
shm-eventfd  16x16    64           93402.37       685.88us       1.45ms        
shm-eventfd  16x16    128          94639.48       1.35ms         2.58ms        
shm-eventfd  16x16    256          94186.39       2.70ms         4.95ms        
shm-eventfd  16x16    512          92673.72       5.40ms         8.31ms        
shm-eventfd  32x32    8            44273.62       184.88us       582.00us      
shm-eventfd  32x32    16           56111.95       291.09us       837.00us      
shm-eventfd  32x32    32           61435.74       527.44us       1.34ms        
shm-eventfd  32x32    64           56351.55       1.15ms         2.73ms        
shm-eventfd  32x32    128          52356.00       2.44ms         5.07ms        
shm-eventfd  32x32    256          52189.94       4.90ms         9.81ms        
shm-eventfd  32x32    512          50162.03       10.18ms        17.33ms       
shm-eventfd  64x64    8            16836.91       483.31us       1.27ms        
shm-eventfd  64x64    16           24187.89       676.52us       1.96ms        
shm-eventfd  64x64    32           29018.77       1.13ms         3.22ms        
shm-eventfd  64x64    64           23284.11       3.03ms         12.81ms       
shm-eventfd  64x64    128          26919.00       4.81ms         10.86ms       
shm-eventfd  64x64    256          23702.13       10.91ms        22.00ms       
shm-eventfd  64x64    512          22642.62       22.28ms        40.07ms       
shm-eventfd  128x128  8            3176.38        2.62ms         8.05ms        
shm-eventfd  128x128  16           5290.69        3.12ms         8.30ms        
shm-eventfd  128x128  32           6011.07        5.37ms         11.29ms       
shm-eventfd  128x128  64           5917.58        10.94ms        24.06ms       
shm-eventfd  128x128  128          5851.24        22.23ms        55.34ms       
shm-eventfd  128x128  256          5799.72        45.81ms        150.34ms      
shm-eventfd  128x128  512          5688.26        90.85ms        290.72ms      
shm-eventfd  256x256  8            410.24         20.56ms        61.24ms       
shm-eventfd  256x256  16           716.15         23.02ms        62.43ms       
shm-eventfd  256x256  32           757.09         42.38ms        87.13ms       
shm-eventfd  256x256  64           741.11         86.46ms        182.38ms      
shm-eventfd  256x256  128          752.33         170.33ms       368.93ms      
shm-eventfd  256x256  256          735.24         346.42ms       882.42ms      
shm-eventfd  512x512  8            45.26          188.36ms       711.77ms      
shm-eventfd  512x512  16           65.09          263.20ms       971.94ms      
shm-eventfd  512x512  32           66.79          477.50ms       1.21s         
shm-eventfd  512x512  64           65.68          935.48ms       1.64s         
shm-eventfd  512x512  128          65.11          1.59s          1.99s         
shm-eventfd  512x512  256          61.81          1.75s          1.99s         

shm-uintr    2x2      8            114453.62      74.23us        383.00us      
shm-uintr    2x2      16           162213.50      102.36us       407.00us      
shm-uintr    2x2      32           165393.58      195.95us       576.00us      
shm-uintr    2x2      64           177210.52      359.76us       833.00us      
shm-uintr    2x2      128          197710.40      637.84us       1.26ms        
shm-uintr    2x2      256          214211.92      1.18ms         2.24ms        
shm-uintr    2x2      512          214606.54      2.36ms         3.79ms        
shm-uintr    4x4      8            109182.69      77.17us        383.00us      
shm-uintr    4x4      16           148943.88      111.28us       429.00us      
shm-uintr    4x4      32           154463.66      209.24us       584.00us      
shm-uintr    4x4      64           164231.98      388.53us       0.86ms        
shm-uintr    4x4      128          185104.45      685.63us       1.33ms        
shm-uintr    4x4      256          199790.64      1.26ms         2.37ms        
shm-uintr    4x4      512          201418.28      2.48ms         4.05ms        
shm-uintr    8x8      8            102217.29      82.23us        389.00us      
shm-uintr    8x8      16           132957.76      123.76us       432.00us      
shm-uintr    8x8      32           140239.62      230.66us       627.00us      
shm-uintr    8x8      64           147195.17      433.90us       0.93ms        
shm-uintr    8x8      128          158284.97      802.62us       1.50ms        
shm-uintr    8x8      256          165833.56      1.52ms         1.52ms        
shm-uintr    8x8      512          164818.50      3.09ms         5.04ms        
shm-uintr    16x16    8            73876.55       113.11us       9.28k         
shm-uintr    16x16    16           98181.90       167.61us       558.00us      
shm-uintr    16x16    32           106412.38      304.20us       13.37k        
shm-uintr    16x16    64           100338.29      636.19us       1.29ms        
shm-uintr    16x16    128          100377.76      1.27ms         2.43ms        
shm-uintr    16x16    256          99360.47       2.57ms         4.74ms        
shm-uintr    16x16    512          95268.62       5.32ms         8.27ms        
shm-uintr    32x32    8            42625.10       191.96us       594.00us      
shm-uintr    32x32    16           56422.02       291.90us       0.87ms        
shm-uintr    32x32    32           64504.29       506.59us       1.36ms        
shm-uintr    32x32    64           61287.90       1.05ms         2.51ms        
shm-uintr    32x32    128          56511.20       2.28ms         4.92ms        
shm-uintr    32x32    256          54588.34       4.65ms         8.84ms        
shm-uintr    32x32    512          52046.76       9.65ms         15.22ms       
shm-uintr    64x64    8            16344.10       491.33us       1.21ms        
shm-uintr    64x64    16           22465.18       733.19us       2.19ms        
shm-uintr    64x64    32           27631.40       1.19ms         3.50ms        
shm-uintr    64x64    64           29293.40       2.20ms         5.15ms        
shm-uintr    64x64    128          27785.45       4.69ms         10.36ms       
shm-uintr    64x64    256          24201.19       10.52ms        20.60ms       
shm-uintr    64x64    512          22594.34       22.98ms        46.46ms       
shm-uintr    128x128  8            3004.60        2.77ms         8.32ms        
shm-uintr    128x128  16           4711.99        3.47ms         8.96ms        
shm-uintr    128x128  32           5544.00        5.81ms         12.04ms       
shm-uintr    128x128  64           5672.57        11.35ms        22.37ms       
shm-uintr    128x128  128          5744.51        22.46ms        44.03ms       
shm-uintr    128x128  256          5640.54        46.45ms        132.00ms      
shm-uintr    128x128  512          5471.99        93.26ms        255.47ms      
shm-uintr    256x256  8            405.79         20.50ms        62.15ms       
shm-uintr    256x256  16           694.56         23.69ms        63.86ms       
shm-uintr    256x256  32           733.03         43.85ms        95.30ms       
shm-uintr    256x256  64           730.18         87.57ms        172.53ms      
shm-uintr    256x256  128          727.25         175.46ms       375.94ms      
shm-uintr    256x256  256          719.99         352.32ms       885.17ms      
shm-uintr    512x512  8            41.02          206.22ms       719.31ms      
shm-uintr    512x512  16           63.85          267.44ms       982.45ms      
shm-uintr    512x512  32           65.10          488.41ms       1.26s         
shm-uintr    512x512  64           64.71          944.59ms       1.75s         
shm-uintr    512x512  128          62.56          1.49s          1.98s         
shm-uintr    512x512  256          63.89          1.41s          1.98s         

```

```
2x2:
    uintr/efd: avg 1.137x (over 7 concurrencies)
      c=    8: 0.986x
      c=   16: 1.097x  **
      c=   32: 1.149x  **
      c=   64: 1.172x ***
      c=  128: 1.167x ***
      c=  256: 1.201x ***
      c=  512: 1.184x ***

  4x4:
    uintr/efd: avg 1.150x (over 7 concurrencies)
      c=    8: 1.021x
      c=   16: 1.129x  **
      c=   32: 1.174x ***
      c=   64: 1.167x ***
      c=  128: 1.184x ***
      c=  256: 1.189x ***
      c=  512: 1.189x ***

  8x8:
    uintr/efd: avg 1.131x (over 7 concurrencies)
      c=    8: 1.038x
      c=   16: 1.122x  **
      c=   32: 1.170x ***
      c=   64: 1.161x ***
      c=  128: 1.141x  **
      c=  256: 1.150x  **
      c=  512: 1.133x  **

  16x16:
    uintr/efd: avg 1.060x (over 7 concurrencies)
      c=    8: 0.989x
      c=   16: 1.091x  **
      c=   32: 1.119x  **
      c=   64: 1.074x
      c=  128: 1.061x
      c=  256: 1.055x
      c=  512: 1.028x

  32x32:
    uintr/efd: avg 1.038x (over 7 concurrencies)
      c=    8: 0.963x
      c=   16: 1.006x
      c=   32: 1.050x
      c=   64: 1.088x  **
      c=  128: 1.079x
      c=  256: 1.046x
      c=  512: 1.038x

  64x64:
    uintr/efd: avg 1.023x (over 7 concurrencies)
      c=    8: 0.971x
      c=   16: 0.929x
      c=   32: 0.952x
      c=   64: 1.258x ***
      c=  128: 1.032x
      c=  256: 1.021x
      c=  512: 0.998x

  128x128:
    uintr/efd: avg 0.948x (over 7 concurrencies)
      c=    8: 0.946x
      c=   16: 0.891x
      c=   32: 0.922x
      c=   64: 0.959x
      c=  128: 0.982x
      c=  256: 0.973x
      c=  512: 0.962x

  256x256:
    uintr/efd: avg 0.976x (over 6 concurrencies)
      c=    8: 0.989x
      c=   16: 0.970x
      c=   32: 0.968x
      c=   64: 0.985x
      c=  128: 0.967x
      c=  256: 0.979x

  512x512:
    uintr/efd: avg 0.974x (over 6 concurrencies)
      c=    8: 0.906x
      c=   16: 0.981x
      c=   32: 0.975x
      c=   64: 0.985x
      c=  128: 0.961x
      c=  256: 1.034x
```
