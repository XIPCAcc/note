# 15个 worker 

**INTERVAL = MAX**

```
==========================================
All Matrix Multiplication Tests Complete!
==========================================

Results saved to: log/matrix_20260621-021237/

==========================================
Generating Summary Report...
==========================================
Summary saved to: log/matrix_20260621-021237/summary.txt
Matrix Multiplication Performance Comparison
===========================================
Date: Sun Jun 21 02:31:14 AM CST 2026

Matrix Sizes: 2 4 8 16 256 512
Concurrencies: 8 16 256 512
Test Duration: 15s each

Transport    Matrix   Concurrency  QPS(req/s)     AvgLat(ms)     P99Lat(ms)    
---------    ------   ----------   ----------     ----------     ----------    

shm-eventfd  2x2      8            112349.42      78.47us        433.00us      
shm-eventfd  2x2      16           145547.62      115.70us       475.00us      
shm-eventfd  2x2      256          180152.18      1.40ms         2.58ms        
shm-eventfd  2x2      512          183371.64      2.73ms         4.46ms        
shm-eventfd  4x4      8            109223.31      80.53us        442.00us      
shm-eventfd  4x4      16           135076.52      124.37us       482.00us      
shm-eventfd  4x4      256          168136.34      1.51ms         2.75ms        
shm-eventfd  4x4      512          166965.87      3.00ms         4.80ms        
shm-eventfd  8x8      8            97601.75       89.87us        465.00us      
shm-eventfd  8x8      16           120608.27      138.04us       504.00us      
shm-eventfd  8x8      256          144392.74      1.77ms         3.21ms        
shm-eventfd  8x8      512          142587.72      3.51ms         5.39ms        
shm-eventfd  16x16    8            71404.77       119.38us       507.00us      
shm-eventfd  16x16    16           91064.30       181.11us       587.00us      
shm-eventfd  16x16    256          91728.64       2.77ms         5.13ms        
shm-eventfd  16x16    512          88768.44       5.65ms         8.81ms        
shm-eventfd  256x256  8            435.39         19.25ms        60.06ms       
shm-eventfd  256x256  16           714.40         23.09ms        63.60ms       
shm-eventfd  256x256  256          718.16         357.87ms       1.04s         
shm-eventfd  256x256  512          698.35         691.28ms       1.64s         
shm-eventfd  512x512  8            39.42          240.17ms       958.29ms      
shm-eventfd  512x512  16           63.72          268.10ms       977.83ms      
shm-eventfd  512x512  256          62.73          1.67s          1.99s         
shm-eventfd  512x512  512          59.63          0.00us         0.00us        

shm-uintr    2x2      8            117396.93      83.05us        538.00us      
shm-uintr    2x2      16           154080.45      117.71us       612.00us      
shm-uintr    2x2      256          205743.94      1.23ms         2.29ms        
shm-uintr    2x2      512          193226.45      2.62ms         5.86ms        
shm-uintr    4x4      8            106995.55      88.79us        534.00us      
shm-uintr    4x4      16           141898.35      126.39us       640.00us      
shm-uintr    4x4      256          190925.07      1.32ms         2.47ms        
shm-uintr    4x4      512          184502.97      2.74ms         5.01ms        
shm-uintr    8x8      8            94829.54       100.08us       567.00us      
shm-uintr    8x8      16           122739.49      143.23us       673.00us      
shm-uintr    8x8      256          158950.74      1.61ms         3.25ms        
shm-uintr    8x8      512          144599.02      3.50ms         6.21ms        
shm-uintr    16x16    8            69076.74       131.95us       631.00us      
shm-uintr    16x16    16           86260.67       199.19us       796.00us      
shm-uintr    16x16    256          90243.69       2.80ms         5.48ms        
shm-uintr    16x16    512          74013.52       6.86ms         16.98ms       
shm-uintr    256x256  8            431.01         19.38ms        59.56ms       
shm-uintr    256x256  16           684.76         24.07ms        65.70ms       
shm-uintr    256x256  256          680.68         379.53ms       1.18s         
shm-uintr    256x256  512          646.00         741.55ms       1.89s         
shm-uintr    512x512  8            46.19          185.49ms       736.68ms      
shm-uintr    512x512  16           61.44          275.72ms       980.39ms      
shm-uintr    512x512  256          58.97          1.06s          1.98s         
shm-uintr    512x512  512          48.91          598.82ms       1.96s  
```

## 正常的interval

```
==========================================
Generating Summary Report...
==========================================
Summary saved to: log/matrix_20260621-115013/summary.txt
Matrix Multiplication Performance Comparison
===========================================
Date: Sun Jun 21 12:08:51 PM CST 2026

Matrix Sizes: 2 4 8 16 256 512
Concurrencies: 8 16 256 512
Test Duration: 15s each

Transport    Matrix   Concurrency  QPS(req/s)     AvgLat(ms)     P99Lat(ms)    
---------    ------   ----------   ----------     ----------     ----------    

shm-eventfd  2x2      8            115709.94      75.83us        417.00us      
shm-eventfd  2x2      16           145662.19      116.33us       484.00us      
shm-eventfd  2x2      256          183227.82      1.38ms         2.52ms        
shm-eventfd  2x2      512          181128.20      2.75ms         4.43ms        
shm-eventfd  4x4      8            105703.42      82.43us        434.00us      
shm-eventfd  4x4      16           133644.17      125.70us       496.00us      
shm-eventfd  4x4      256          167593.51      1.51ms         2.74ms        
shm-eventfd  4x4      512          167491.13      2.98ms         4.68ms        
shm-eventfd  8x8      8            97455.72       88.75us        439.00us      
shm-eventfd  8x8      16           120266.18      137.63us       497.00us      
shm-eventfd  8x8      256          143856.04      1.75ms         3.18ms        
shm-eventfd  8x8      512          141144.99      3.54ms         5.65ms        
shm-eventfd  16x16    8            71883.34       119.24us       514.00us      
shm-eventfd  16x16    16           92825.45       177.81us       579.00us      
shm-eventfd  16x16    256          92079.48       2.75ms         4.85ms        
shm-eventfd  16x16    512          88547.13       5.68ms         9.01ms        
shm-eventfd  256x256  8            442.70         18.43ms        58.24ms       
shm-eventfd  256x256  16           711.15         23.13ms        62.62ms       
shm-eventfd  256x256  256          722.63         356.78ms       967.52ms      
shm-eventfd  256x256  512          720.72         664.79ms       1.48s         
shm-eventfd  512x512  8            42.13          212.24ms       947.47ms      
shm-eventfd  512x512  16           62.89          270.92ms       979.11ms      
shm-eventfd  512x512  256          61.32          1.57s          2.00s         
shm-eventfd  512x512  512          56.90          1.38s          2.00s         

shm-uintr    2x2      8            115324.39      81.14us        498.00us      
shm-uintr    2x2      16           151143.29      118.96us       596.00us      
shm-uintr    2x2      256          213929.82      1.18ms         2.23ms        
shm-uintr    2x2      512          201687.61      2.50ms         4.46ms        
shm-uintr    4x4      8            108783.22      88.04us        542.00us      
shm-uintr    4x4      16           140948.38      128.63us       641.00us      
shm-uintr    4x4      256          197318.18      1.30ms         2.43ms        
shm-uintr    4x4      512          185382.34      2.71ms         4.75ms        
shm-uintr    8x8      8            96471.97       98.61us        569.00us      
shm-uintr    8x8      16           126083.14      140.90us       654.00us      
shm-uintr    8x8      256          165576.10      1.53ms         3.05ms        
shm-uintr    8x8      512          152634.95      3.30ms         6.09ms        
shm-uintr    16x16    8            68379.79       131.71us       616.00us      
shm-uintr    16x16    16           85904.35       199.04us       776.00us      
shm-uintr    16x16    256          98190.11       2.61ms         5.27ms        
shm-uintr    16x16    512          93808.42       5.42ms         9.58ms        
shm-uintr    256x256  8            424.94         19.69ms        60.34ms       
shm-uintr    256x256  16           683.28         24.06ms        64.85ms       
shm-uintr    256x256  256          670.23         379.70ms       1.15s         
shm-uintr    256x256  512          649.70         734.79ms       1.92s         
shm-uintr    512x512  8            47.88          169.40ms       478.25ms      
shm-uintr    512x512  16           61.35          274.67ms       983.27ms      
shm-uintr    512x512  256          60.28          921.55ms       1.97s         
shm-uintr    512x512  512          55.53          1.29s          1.99s 
```