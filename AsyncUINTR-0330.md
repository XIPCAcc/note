# 使用协程方式唤醒异步用户态中断协程的方式

采用这种方式目前backend还是会经常进入睡眠状态

#[derive(Clone)]
pub struct UintrToken {
    inner: Arc<Inner>,
    name: String,
}

/// 内部状态
struct Inner {
    /// 是否已经收到一次中断
    pending: Mutex<bool>,
    /// 当前在等这个中断的任务的 waker（最多一个）
    waker: Mutex<Option<Waker>>,
}

impl UintrToken {
    /// 创建新的UintrToken
    /// 
    /// # 参数
    /// 
    /// * `name` - Token名称，用于调试
    /// 
    /// # 返回
    /// 
    /// 返回新创建的UintrToken实例
    pub fn new(name: &str) -> Self {
        Self {
            inner: Arc::new(Inner {
                pending: Mutex::new(false),
                waker: Mutex::new(None),
            }),
            name: name.to_string(),
        }
    }
    
    /// 获取Token名称
    pub fn name(&self) -> &str {
        &self.name
    }
}

// ============================================================================
// Future实现
// ============================================================================

/// UINTR 异步 Future
/// 
/// 这个结构体实现了Future trait，用于异步等待中断
pub struct UintrFuture {
    token: UintrToken,
}

impl Future for UintrFuture {
    type Output = UintrResult<()>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // 先检查是否有 pending 中断，避免时序问题
        let mut pending = self.token.inner.pending.lock().unwrap();
        if *pending {
            *pending = false;
            Poll::Ready(Ok(()))
        } else {
            // 没有 pending，保存 waker 并返回 Pending
            *self.token.inner.waker.lock().unwrap() = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}

// ============================================================================
// 公共API
// ============================================================================

/// 异步等待 UINTR 中断
/// 
/// # 参数
/// 
/// * `token` - UintrToken实例
/// 
/// # 返回
/// 
/// 成功返回Ok(())，失败返回错误
pub async fn uintr(token: UintrToken) -> UintrResult<()> {
    UintrFuture { token }.await
}



#[no_mangle]
pub extern "C" fn rust_interrupt_callback(_handler_name: *const libc::c_char, vector: u64) {
    unsafe {
        if let Some(ref token) = TOKEN_OBJ {
            let mut pending = token.inner.pending.lock().unwrap();
            *pending = true;
        }
    }
}

/// 处理UINTR wakers
/// 
/// 这个函数检查是否有待处理的中断，如果有则唤醒相应的waker
/// 
/// # 返回
/// 
/// 返回唤醒的waker数量
#[no_mangle]
pub extern "C" fn process_uintr_wakers(token) -> u32 {
    unsafe {
            let should_wake = {
                let pending = token.inner.pending.lock().unwrap();
                *pending
            };
            
            if should_wake {
                if let Some(waker) = token.inner.waker.lock().unwrap().take() {
                    waker.wake();
                    waker_count += 1;
                }
            }
        waker_count
    }
}

// 创建定期唤醒任务，确保异步运行时能够及时响应中断
    tokio::spawn(async {
        let mut interval = tokio::time::interval(tokio::time::Duration::from_micros(5));
        let mut count = 0;
        loop {
            interval.tick().await;
            count += 1;
            if count % 10000000 == 0 {
                println!("Periodic wakeup: count={}", count);
            }
            // 调用process_uintr_wakers来处理中断
            process_uintr_wakers(token.clone());
        }
    });


    uintr_await(token).await?