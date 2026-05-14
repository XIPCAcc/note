为什么在gateway和backend的场景下 uint_wait 老是返回not support

返回 -EOPNOTSUPP 表示 当前进程未注册为用户中断接收者 。

调用 uintr_wait 之前，必须先调用 uintr_register_handler 注册为接收者

关键问题是： UINTR 注册是线程相关的 ！

### 调用流程分析
1. 我们在 main.rs 的主线程中调用 initialize_uintr()
2. initialize_uintr() 在主线程中调用 UintrServer::with_socket_path()
3. UintrServer::with_socket_path() 在主线程中注册中断处理程序
4. 然后我们在 tokio::spawn 的 后台线程 中调用 wait_for_notification()
5. 后台线程调用 wait_interrupt() ，但后台线程从未注册过中断处理程序！

### 修复方案
在 uintr/src/api.rs 中：

1. 添加了线程局部变量 HANDLER_REGISTERED 来跟踪每个线程是否已注册处理程序
2. 修改了 UintrServer::wait_interrupt() 和 UintrClient::wait_interrupt() 方法
3. 在每次调用 wait_interrupt() 时：
   - 检查当前线程是否已注册
   - 如果未注册，重新注册中断处理程序并启用中断
   - 标记当前线程为已注册

   在uintr_wait 中注册中断处理程序是一个好办法，但是如何没有进入uintr_wait 就收到了用户态中断的线程是不是还会出错


rm /dev/shm/gateway_shm_uintr_* 
# 如果需要停止进程，可以使用以下命令
pkill -f "backend --transport shm-uintr"
pkill -f "gateway --transport shm-uintr"

# 使用 info 日志级别运行 backend
RUST_LOG=info ./target/debug/backend --transport shm-uintr --shm-name gateway_shm_uintr_test_manual

# 使用 info 日志级别运行 gateway
RUST_LOG=info ./target/debug/gateway --transport shm-uintr --shm-name gateway_shm_uintr_test_manual

# 发送测试请求
curl -s "http://127.0.0.1:8080/echo?message=test_uintr"

wrk -t8 -c256 -d30s --latency -s "scripts/wrk.lua" "http://127.0.0.1:8080/api/echo"


2026-03-30T12:28:02.534758Z  INFO gateway::shm_transport_uintr: wait_for_uintr: 开始异步等待 UINTR 中断...
2026-03-30T12:28:02.532179Z  INFO backend::shm_server_uintr: Request handler: 已发送 UINTR 响应通知，请求 9df07f4f-55ed-4944-ab70-55faed3c1620
2026-03-30T12:28:52.464136Z  INFO backend::shm_server_uintr: Request handler: 已发送 UINTR 响应通知，请求 20e3132a-7a79-424a-b5d3-e1af3a5c0e28
