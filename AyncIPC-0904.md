# IPC的统一接口格式

## 基本形式

需要实现三个API，open，write_all()和 read()。

```rust
(rx, tx) = open();

tx.write_all(&chunk).await;

rx.read(&mut buf).await;
```


## 吞出量测试

假设共享缓冲区为16KB，发送方每次发送16KB，接收方每次读取16KB。测试发送256MB数据量所需要的时间。

所有测试都运行在单核上，通过taskset将发送方和接收方各自绑定在一个固定的核上。

```rust
const CHUNK: usize = 64 * 1024; // 单次写入 64 KB，因为内核pipe的默认环形缓冲区大小为 64KB
const TOTAL: usize = 256 * 1024 * 1024; // 总传输量 256 MB

fn sender()
{
    let (rx, tx) = open();

    let chunk = vec![0x5Au8; CHUNK];
    let mut sent = 0usize;

    let start = Instant::now();
    while sent < TOTAL {
        tx.write_all(&chunk).await.expect("write 失败");
        sent += CHUNK;
    }

    // to do: 等待receiver 退出
    let elapsed = start.elapsed();

    let secs = elapsed.as_secs_f64();
    let mib = TOTAL as f64 / (1024.0 * 1024.0);
    println!(
        "{name:<20} {mib:7.1} MiB / {secs:6.3} s = {:8.1} MiB/s",
        mib / secs
    );
}

fn receiver()
{
    let (rx, tx) = open();

    let mut buf = vec![0u8; CHUNK];
    let mut got = 0usize;
    while got < TOTAL {
        match rx.read(&mut buf).await {
            Ok(0) => break,
            Ok(n) => got += n,
            Err(e) => return Err(e),
        }
    }
}
```

## 延迟测试

发送方每次发送8 Bytes，接收方每次读取8 Bytes，然后将数据重新发送给发送方。测试来回发送的时间。

```rust
const WARMUP: usize = 2_000; // 延迟预热轮数（不计入统计）
const ROUNDS: usize = 100_000; // 延迟统计轮数
const MSG: [u8; 8] = [0xA5; 8];


/// 延迟测试：两条 pipe 全双工，服务端原样回显，统计 RTT。

/// 单方向 n 轮 ping-pong：写 8 字节 -> 读 8 字节回包。
fn ping_pong()
{
    let mut pong = [0u8; 8];
    for _ in 0..n {
        tx.write_all(&MSG).await?;
        rx.read_exact(&mut pong).await?;
    }
    Ok(())
}

fn client()
{
    let (server_rx, client_tx) = open();
    let (client_rx, server_tx) = open();

    let total = WARMUP + ROUNDS;

    // 预热（摊平首次注册/缺页等一次性开销）
    ping_pong(&mut client_tx, &mut client_rx, WARMUP).await?;

    let start = Instant::now();
    ping_pong(&mut client_tx, &mut client_rx, ROUNDS).await?;
    // to do: 等待receiver 退出
    let elapsed = start.elapsed();


    let rtt_us = elapsed.as_micros() as f64 / ROUNDS as f64;
    let msgs_per_s = ROUNDS as f64 / elapsed.as_secs_f64();
    println!(
        "{name:<20} RTT = {rtt_us:7.2} µs/轮  ({:9.0} msg/s)",
        msgs_per_s
    );
    Ok(())
}



fn server()
{
    let (server_rx, client_tx) = open();
    let (client_rx, server_tx) = open();

    for _ in 0..total {
        server_rx.read_exact(&mut buf).await?;
        server_tx.write_all(&buf).await?;
    }
}
```
