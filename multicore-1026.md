

# 性能数据汇总

## 被测对象

| 代号 | 出处 | 队列 | worker 线程 | 唤醒 |
|---|---|---|---|---|
| a448631a8901f41db76a1d1a6cb798e7a4ae779b | commit a448631（log 已关闭） | `Mutex<VecDeque>` | `std::thread` 动态 | eventfd poke |
| **raw** | 裸 clone3 + 栈缓存 | Vyukov 无锁环（原子指令和自旋） | 裸 `clone3` 动态 | 无（超时轮询） |
| **thread** | std::thread | 同上 | `std::thread` 动态 | 同上 |
| **pool** | 常驻池 | 同上 | 常驻池 | `park`/`unpark` |


## 一、系统调用（320 请求＝32×10 轮，strace -f 逐行计数）

| syscall/请求级指标 | 1 | 2 | 3 | 4 |
|---|---|---|---|---|
| 总 syscall | 8,957（28.0/req） | 4,153（13.0/req） | **3,920（12.3/req）** | 7,763（24.3/req） |
| futex | 3,044 | 0 | 17 | 3,841 |
| fcntl | 1,304 | 0 | 0 | 0 |
| write | 1,411 | 641 | 641 | 641 |
| epoll_ctl | 983 | 982 | 982 | 982 |
| clone3 | 69 | 77 | 68 | 15 |
| mmap/mprotect/munmap | 96/88/77 | **26/17/14** | 97/88/75 | 59/53/28 |
| eventfd2 | 1 | 0 | 0 | 0 |

## 二、CPU（burst=1000×5＝5000 请求，多轮区间）

| 方案 | CPU/请求 | 说明 |
|---|---|---|
| 1 | 54–60 µs | Mutex 竞争时线程睡 futex，不占 CPU |
| 2 | 72–106 µs | 无锁队列 CAS 在高竞争下空转烧 CPU |
| 3 | 74–84 µs | 同上 + NPTL 自举税被栈缓存摊薄 |
| 4 | 108–158 µs | park/unpark futex 税随任务数线性增长 |

