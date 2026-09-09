# 异步命名管道

```rust
pub struct PipeSender {
    fd: RawFd,          // FIFO 写端（O_WRONLY | O_NONBLOCK）
    uipi_index: u64,    // 接收方：写完数据后 senduipi 通知有数据
    token: UintrToken,  // 等待接收方读走后发来的"有空间"背压中断
}

pub struct PipeReceiver {
    fd: RawFd,          // FIFO 读端（O_RDONLY | O_NONBLOCK）
    uipi_index: u64,    // 发送方：读出数据后 senduipi 通知有空间
    token: UintrToken,  // 等待发送方写来的"有数据"中断
}
```


```rust
pub async fn poll_write(&self, buf: &[u8]) -> io::Result<usize> {
    loop {
        let n = libc::write(self.fd, buf...);      // 非阻塞写
        if n > 0 {
            senduipi(self.uipi_index);             // 写成功，通知接收方
            return Ok(n);
        }
        if err == WouldBlock /* EAGAIN */ {
            uintr(self.token.clone()).await?;      // FIFO 满，挂起等待
            continue;
        }
        return Err(...);
    }
}
pub async fn write_all(&self, buf) { 循环 poll_write 直到写完 }
```

```rust
pub async fn poll_read(&self, buf: &mut [u8]) -> io::Result<usize> {
    loop {
        let n = libc::read(self.fd, buf...);       // 非阻塞读
        if n > 0 {
            senduipi(self.uipi_index);             // 读成功 → 通知发送方
            return Ok(n);
        }
        if n == 0 { return Ok(0); }                // EOF：所有写端已关闭
        if err == WouldBlock {
            uintr(self.token.clone()).await?;      // 无数据挂起等待
            continue;
        }
    }
}
```
