# 关于后端是否要合并中断的验证

backend会在被唤醒后，批量读取共享内存的内容，对于每个req，单独生成一个协程处理。
如果不采用批量处理，那么每个协程处理完req并将resp写入共享内存后，就发送一个用户态中断。
如果采用批量处理，那么需要由创建req处理协程的父协程等待统一批次的协程完成后，统一发送一个中断。

预想中应该是非批量处理的延迟会更低。

## 实际效果

### 非批量处理

#### 2x2 8c
```
Summary:
  Success rate:	100.00%
  Total:	60003.2089 ms
  Slowest:	101.1201 ms
  Fastest:	0.0222 ms
  Average:	0.1202 ms
  Requests/sec:	65611.6744

Response time distribution:
  10.00% in 0.0630 ms
  25.00% in 0.0745 ms
  50.00% in 0.0917 ms
  75.00% in 0.1159 ms
  90.00% in 0.1525 ms
  95.00% in 0.2050 ms
  99.00% in 0.4429 ms
  99.90% in 0.6903 ms
  99.99% in 100.3131 ms
```

#### 4x4 32c
```
Summary:
  Success rate:	100.00%
  Total:	60002.1800 ms
  Slowest:	101.1910 ms
  Fastest:	0.0252 ms
  Average:	0.3007 ms
  Requests/sec:	105936.5176

Response time distribution:
  10.00% in 0.1673 ms
  25.00% in 0.2088 ms
  50.00% in 0.2589 ms
  75.00% in 0.3235 ms
  90.00% in 0.4119 ms
  95.00% in 0.4942 ms
  99.00% in 0.7317 ms
  99.90% in 2.0263 ms
  99.99% in 100.6570 ms
```

### 批量处理

#### 2x2 8c

Summary:
  Success rate: 100.00%
  Total:        60003.9310 ms
  Slowest:      100.7790 ms
  Fastest:      0.0191 ms
  Average:      0.1099 ms
  Requests/sec: 71672.8875

Response time distribution:
  10.00% in 0.0614 ms
  25.00% in 0.0721 ms
  50.00% in 0.0893 ms
  75.00% in 0.1148 ms
  90.00% in 0.1532 ms
  95.00% in 0.2074 ms
  99.00% in 0.4754 ms
  99.90% in 0.6720 ms
  99.99% in 2.3712 ms


#### 4x4 32c

```
Summary:
  Success rate: 100.00%
  Total:        60001.9570 ms
  Slowest:      101.4195 ms
  Fastest:      0.0281 ms
  Average:      0.2889 ms
  Requests/sec: 110222.9715

Response time distribution:
  10.00% in 0.1688 ms
  25.00% in 0.2132 ms
  50.00% in 0.2653 ms
  75.00% in 0.3271 ms
  90.00% in 0.4018 ms
  95.00% in 0.4677 ms
  99.00% in 0.6902 ms
  99.90% in 1.3123 ms
  99.99% in 7.2710 ms
```

## 猜想

也可能不是批量中断的问题，而是父协程等待子协程的的过程毕业避免进入uintr.await?

父协程等待子协程的过程中有中断出现，被唤醒后uintr.await 直接返回ready，

父协程等待子协程被唤醒的开销又比较小。

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

在小批量计算数据的时候等待可以提高性能，但是在高并发大批量计算数据的时候反而会削弱性能。

```

矩阵大小: 64x64
----------------------------------------------------------------------------------------------------
    并发  shm-uintr_QPS shm-eventfd_QPS    shm-uds_QPS shm-eventfd/shm-uintr shm-uds/shm-uintr shm-uds/shm-eventfd
----------------------------------------------------------------------------------------------------------------
     8        11514.0        16983.6        15674.3         1.48x         1.36x         0.92x
    16        13852.6        25535.6        22226.6         1.84x         1.60x         0.87x
    32        17427.2        29691.0        26362.6         1.70x         1.51x         0.89x
    64        21897.2        28115.6        21819.4         1.28x         1.00x         0.78x
   128        20514.9        24679.9        20018.0         1.20x         0.98x         0.81x
   256        17602.3        20144.4        19359.2         1.14x         1.10x         0.96x


矩阵大小: 128x128
----------------------------------------------------------------------------------------------------
    并发  shm-uintr_QPS shm-eventfd_QPS    shm-uds_QPS shm-eventfd/shm-uintr shm-uds/shm-uintr shm-uds/shm-eventfd
----------------------------------------------------------------------------------------------------------------
     8         1653.5         3397.0         3526.1         2.05x         2.13x         1.04x
    16         2654.9         5333.0         5030.7         2.01x         1.89x         0.94x
    32         4254.6         6575.9         6318.5         1.55x         1.49x         0.96x
    64         5509.1         6147.7         5032.2         1.12x         0.91x         0.82x
   128         5568.7         5354.9         5314.9         0.96x         0.95x         0.99x
   256         4315.3         4179.2         4972.0         0.97x         1.15x         1.19x
```