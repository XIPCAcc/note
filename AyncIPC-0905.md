# 用户态共享内存IPC的设计

## 共享缓冲区的元数据

通过mmap创建出共享内存，其中包含唤醒缓冲区data（64KB），写位置write_pos，读位置read_pos。

该内存区域由发送方和接收方共享。

```rust
pub const RING_CAPACITY: usize = 1 << 16; // 64 KiB

pub struct ShmChannel {
    write_pos: AtomicU64,
    read_pos: AtomicU64,
    data: UnsafeCell<[u8; RING_CAPACITY]>,
}

```

## 发送方

发送方需要保存共享内存ShmChannel信息，发送中断的下标，接收中断的token等数据。

```rust
pub struct Sender {
    shm: &'static ShmChannel,
    /// 发送方发送用户态中断的index
    uipi_index: u64,
    /// UintrToken 用于存储用户态中断的等待信息，包括waker，seq等
    token: UintrToken,
}
```

### write_all

发送方进入睡眠等待状态的条件是发送方数据未完全写入但是缓冲区已满。

但是什么时候应该发送通知是需要考虑的事情。

最简单的做法是每次只要有数据的写入，就发送一次通知。

对于传统的IPC而言，比如pipe，pipe_write 会通过 was_empty 判断写之前缓冲区是否为空，如果为空，那么发送完数据后则唤醒，或者有epoll 在监听这个pipe，那么每次发送数据，必然唤醒。

```rust
`    pub async fn write_all(&self, buf: &[u8]) -> io::Result<()> {
        let mut written = 0;
        while written < buf.len() {
            // write_available 将数据写入缓冲区，返回写入的字节数
            let n = self.shm.write_available(&buf[written..]);
            if n > 0 {
                written += n;
                // 每次写入数据，必发送一次通知
                self.notify_receiver();
                continue;
            }

            uintr(self.token.clone()).await;
        }


        Ok(())
    }

    fn notify_receiver(&self) {
            unsafe { syscall::senduipi(self.uipi_index); }
    }

    fn write_available(&self, src: &[u8]) -> usize {
        let write_pos = self.write_pos.load(Ordering::Relaxed);
        let read_pos = self.read_pos.load(Ordering::Acquire);
        let used = write_pos.wrapping_sub(read_pos) as usize;
        let free = RING_CAPACITY - used;
        let n = free.min(src.len());
        if n == 0 {
            return 0;
        }
        let mask = RING_CAPACITY - 1;
        let start = (write_pos as usize) & mask;
        unsafe {
            let data = self.data.get() as *mut u8;
            for i in 0..n {
                *data.add((start + i) & mask) = src[i];
            }
        }
        self.write_pos.store(write_pos.wrapping_add(n as u64), Ordering::Release);
        n
    }
```

## 接收方

发送方同样需要保存共享内存ShmChannel信息，发送中断的下标，接收中断的token等数据。

```rust
pub struct Receiver {
    shm: &'static ShmChannel,
    uipi_index: u64,
    token: UintrToken,    
}
```

### read

```rust
    pub async fn read(&self, buf: &mut [u8]) -> io::Result<usize> {
        loop {
            let n = self.shm.read_available(buf);
            if n > 0 {
                // 每次读取一次数据，必然发送一次通知。
                self.notify_sender();
                return Ok(n);
            }

            uintr(self.token.clone()).await;
        }
    }

    fn notify_sender(&self) {
            unsafe { syscall::senduipi(self.uipi_index); }
    }

    fn read_available(&self, out: &mut [u8]) -> usize {
        let read_pos = self.read_pos.load(Ordering::Relaxed);
        let write_pos = self.write_pos.load(Ordering::Acquire);
        let available = write_pos.wrapping_sub(read_pos) as usize;
        let n = available.min(out.len());
        if n == 0 {
            return 0;
        }
        let mask = RING_CAPACITY - 1;
        let start = (read_pos as usize) & mask;
        unsafe {
            let data = self.data.get() as *const u8;
            for i in 0..n {
                out[i] = *data.add((start + i) & mask);
            }
        }
        self.read_pos.store(read_pos.wrapping_add(n as u64), Ordering::Release);
        n
    }
```