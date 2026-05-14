

因为想要使用Rust tokio自带的gRPC
所以需要安装protobuf

wget https://github.com/protocolbuffers/protobuf/archive/v3.15.8.tar.gz
tar -xvzf v3.15.8.tar.gz
sudo apt-get update
sudo apt-get install -y autoconf automake libtool curl make g++ unzip
cd protobuf-3.15.8
# 生成配置文件
./autogen.sh
# 配置安装路径
./configure --prefix=/usr/local
# 编译
make -j$(nproc)
# 安装
sudo make install
# 更新库缓存
sudo ldconfig
protoc --version

# gateway

用 Tokio + tonic + Kairos-rs 组合出一套带 gRPC 的微服务 + 网关架构。压测客户端采用  (wrk/vegeta/ghz, 控制并发和QPS)    模拟后端服务 (固定延迟响应，如 echo 或计算服务)

# gateway-gRPC

用 Tokio + tonic 组合出一套带 gRPC 的微服务 + 网关架构。压测客户端采用  (wrk/vegeta/ghz, 控制并发和QPS)    模拟后端服务 (固定延迟响应，如 echo 或计算服务)
评测维度
1. 端到端延迟（HTTP 视角）
对每一种 Gateway ↔ App 的实现（UDS / TCP / 共享内存+eventfd / 共享内存+UINTR）：
测 P50 / P90 / P95 / P99 / P99.9 延迟；
在不同并发连接数下曲线（例如 C=64 / 256 / 1024）。


2. 吞吐量（QPS）
在保持 P99 < 某个阈值（例如 10ms）的前提下，比较各实现的最大稳定 QPS。


3. 系统资源开销
使用 perf stat 或 pidstat 统计：
系统调用次数；
上下文切换次数；
CPU user / kernel 时间比例；


# gateway-kairos-rs

使用Kairos-rs API Gateway `https://docs.rs/kairos-rs/latest/kairos_rs/` 构建一个服务器，要求要有后端模拟的微服务

> sudo apt  install golang-go

使用apache 
> sudo apt-get install apache2-utils

使用 wrk 测试

> sudo apt-get install wrk
> wrk -t12 -c400 -d60s http://127.0.0.1:5900/health
Running 1m test @ http://127.0.0.1:5900/health
  12 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.86ms    2.65ms 105.00ms   92.44%
    Req/Sec    24.70k     2.52k   50.07k    80.49%
  17721068 requests in 1.00m, 2.56GB read
Requests/sec: 294831.86
Transfer/sec:     43.58MB

> sudo apt install golang-go
> curl -sSL https://raw.githubusercontent.com/voidint/g/master/install.sh | bash
> g install 1.25.8
> go env -w GOPROXY=https://goproxy.cn,direct
> go install github.com/rakyll/hey@latest
> go install github.com/bojand/ghz/cmd/ghz@latest


## Kairos-rs

使用HTTP 和客户端通信

客户端应用
    ↓
1. 发送 HTTP 请求
    ↓
2. Gateway 接收请求
    ↓
3. 路由匹配
    ↓
4. 转发到后端服务
    ↓
5. 接收后端响应
    ↓
6. 返回给客户端

Gateway 与后端微服务的通信
通信协议：HTTP/HTTPS

// 创建路由处理器
let route_handler = RouteHandler::new(config.routers, 30);

// 处理请求并转发
route_handler.handle_request(req, body).await

┌─────────────────────────────────────┐
│  RouteHandler.handle_request()      │
├─────────────────────────────────────┤
│ 1. 匹配路由规则                      │
│    /users/123 → /api/v1/user/123    │
│                                     │
│ 2. 构造后端请求                      │
│    GET http://127.0.0.1:8002/       │
│        api/v1/user/123              │
│                                     │
│ 3. 发送 HTTP 请求到后端              │
│    使用 reqwest 或 hyper 客户端      │
│                                     │
│ 4. 接收后端响应                      │
│    HTTP 200 + JSON 数据              │
│                                     │
│ 5. 返回响应给客户端                  │
└─────────────────────────────────────┘


┌─────────────────────────────────────────────────┐
│         handle_request() 内部流程                │
├─────────────────────────────────────────────────┤
│                                                 │
│  1. 解析客户端请求                               │
│     ├─ 提取 HTTP 方法 (GET/POST/PUT/DELETE)     │
│     ├─ 提取请求路径                 │
│     ├─ 提取请求头                  │
│     └─ 提取请求体                       │
│                                                 │
│  2. 路由匹配                                     │
│     ├─ 遍历配置的路由规则                        │
│     ├─ 匹配 external_path                       │
│     ├─ 提取路径参数 (如 {id})                    │
│     └─ 转换为 internal_path                     │
│                                                 │
│  3. 构造后端 HTTP 请求                           │
│     ├─ 目标 URL: http://host:port/internal_path │
│     ├─ HTTP 方法: 与客户端相同                   │
│     ├─ 请求头: 转发或添加                        │
│     └─ 请求体: 转发                             │
│                                                 │
│  4. 发送 HTTP 请求到后端服务 ⭐                   │
│     └─ 使用 HTTP 客户端     │
│                                                 │
│  5. 接收后端 HTTP 响应                           │
│     ├─ 状态码 (200, 404, 500...)                │
│     ├─ 响应头                                    │
│     └─ 响应体 (JSON)                            │
│                                                 │
│  6. 转换为 Actix HttpResponse                   │
│     └─ 返回给客户端                              │
│                                                 │
└─────────────────────────────────────────────────┘

kairos-rs 使用 Reqwest v0.12 作为 HTTP 客户端库
kairos-rs
    ↓
reqwest v0.12  ← 高级 HTTP 客户端
    ↓
hyper v1.8     ← 底层 HTTP 实现
    ↓
tokio          ← 异步运行时

# HTTP 和 RPC的对比

相同硬件环境下的基准测试：

协议          吞吐量      延迟    数据大小
─────────────────────────────────────────
HTTP/1.1+JSON  10,000/s   10ms    1.2KB
HTTP/2+JSON    15,000/s   8ms     1.2KB
gRPC           50,000/s   2ms     0.4KB
─────────────────────────────────────────

性能提升：
gRPC vs HTTP/1.1+JSON: 5倍吞吐量，5倍延迟降低，3倍数据压缩

目前这个项目采用 http + json形式纯粹是因为简单容易实现，如果优化应该用 rpc + protobuf