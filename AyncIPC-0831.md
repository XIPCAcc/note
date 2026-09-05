taskset -c 15 ./target/release/uart-matmul --mode ipc-sender -c 100000 --msg-size 256
taskset -c 14 ./target/release/uart-matmul --mode ipc-receiver

[IPC] sender: 25600000 bytes in 0.043s -> 570.94 MiB/s
[receiver] senduipi(notify) 次数: 1637
[receiver] 睡眠唤醒次数: 664

[sender] senduipi(notify) 次数: 79832
[sender] 睡眠唤醒次数: 317


taskset -c 15 ./target/release/ipc-tokio --mode sender -c 100000 --msg-size 
taskset -c 14 ./target/release/uart-matmul --mode ipc-receiver

[sender] 25600000 bytes in 0.087s -> 282.17 MiB/s
[sender] senduipi(notify) 次数: 71932
[sender] 睡眠唤醒次数: 0

[receiver] senduipi(notify) 次数: 1637
[receiver] 睡眠唤醒次数: 664



 taskset -c 14 ./target/release/ipc-tokio --mode receiver
[receiver] senduipi(notify) 次数: 183
[receiver] 睡眠唤醒次数: 53

taskset -c 15 ./target/release/ipc-tokio --mode sender -c 100 --msg-size 65536
[sender] 6553600 bytes in 0.002s -> 3411.14 MiB/s
[sender] senduipi(notify) 次数: 58
[sender] 睡眠唤醒次数: 88

taskset -c 15 ./target/release/uart-matmul --mode ipc-sender -c 100 --msg-size 65536
[IPC] sender: 6553600 bytes in 0.002s -> 3667.75 MiB/s
[sender] senduipi(notify) 次数: 101
[sender] 睡眠唤醒次数: 1


taskset -c 14 ./target/release/uart-matmul --mode ipc-receiver
[receiver] senduipi(notify) 次数: 15
[receiver] 睡眠唤醒次数: 101

