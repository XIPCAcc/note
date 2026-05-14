# 微服务架构模式 Gateway（HTTP）+ App（RPC）

API Gateway (HTTP接入层)处理所有客户端（Web/App/第三方）的HTTP(S)请求。根据URL、Header等将请求路由到正确的后端服务。

典型的Gateway

Kong / Apache APISIX：基于Nginx/OpenResty，插件生态丰富，性能极高，是当前最热门的选择。
Spring Cloud Gateway：基于Spring生态，Java友好。
Envoy：偏向Service Mesh的数据平面。

API Gateway 产品
├── 开源自托管
│   ├── 传统代理类：Nginx, Apache
│   ├── 新兴网关类：Kong, Apache APISIX, Tyk
│   ├── 云原生类：Envoy, Traefik, Gloo
│   └── Java生态类：Spring Cloud Gateway, Zuul
├── 云厂商托管
│   ├── AWS API Gateway
│   ├── 阿里云 API 网关
│   ├── 腾讯云 API 网关
│   └── Google Cloud Endpoints
└── 硬件/商业方案
    ├── F5 BIG-IP
    ├── Citrix ADC
    └── HAProxy Enterprise

RPC 框架选型
gRPC：Google发布，基于HTTP/2和ProtoBuf，高性能、跨语言支持好，是现代微服务的主流选择。
Apache Dubbo：阿里发布，Java生态功能丰富，国内企业级应用广泛。
Thrift：Facebook出品，跨语言，性能好，但生态略逊于gRPC。

# 架构的演进

客户端请求
     ↓
┌─────────────────────────────────────┐
│         Gateway（API网关）           │
│ 1. 接收HTTP请求（南北流量入口）         │
│ 2. 路由到对应RPC服务                  │
│ 3. 协议转换（HTTP↔RPC）               │
└─────────────────────────────────────┘
     ↓ （通过IPC：UDS/TCP/共享内存/UINTR）
┌─────────────────────────────────────┐
│         App（RPC服务）               │
│ 纯业务逻辑，不处理HTTP细节              │
└─────────────────────────────────────┘

100万用户请求
       ↓
┌─────────────────┐
│ Gateway集群     │ ← 100台机器分担压力
│ - 验身份证(认证) │ 
│ - 发排队号(限流) │ ← 只放10万人进入
│ - 指引去哪个柜台 │ 
└─────────────────┘
       ↓ （每人发个“办理业务”的纸条）
┌─────────────────┐
│ App集群         │ ← 1000台机器真正处理
│ - 减库存         │
│ - 生成订单       │
│ - 扣款          │
└─────────────────┘

用户浏览器
       ↓（直接访问）
┌──────────────┐
│ 博客服务器    │ ← 一个 Spring Boot 程序搞定
│ - 展示文章    │    所有功能
│ - 发表评论    │
│ - 用户登录    │
└──────────────┘

用户浏览器
       ↓
┌──────────────┐
│ Nginx        │ ← 只做反向代理和静态文件
│（简单转发）   │
└──────────────┘
       ↓
┌──────────────┐
│ 电商应用      │ ← 还是单体，包含所有功能
│ - 商品展示    │
│ - 购物车     │
│ - 订单支付    │
└──────────────┘

用户浏览器/APP
       ↓
┌─────────────────┐
│ API Gateway     │ ← Spring Cloud Gateway
│ - 统一认证      │    或 Kong 开源版
│ - 简单限流      │
└─────────────────┘
       ↓
┌─────────────────┐
│ 微服务集群       │ ← 拆成了3-5个服务
│ - 用户服务       │
│ - 订单服务       │
│ - 支付服务       │
└─────────────────┘

用户浏览器/APP
       ↓
┌─────────────────────────────────┐
│ 全局负载均衡 + CDN              │
└─────────────────────────────────┘
       ↓
┌─────────────────────────────────┐
│ 区域网关集群（多机房部署）        │
│ - 精细限流（用户/API/IP）        │
│ - 智能路由（就近、权重）          │
│ - API 编排（组合多个服务）        │
│ - 协议转换（HTTP/gRPC/WebSocket）│
└─────────────────────────────────┘
       ↓
┌─────────────────────────────────┐
│ 服务网格（数百个微服务）          │
│ - 服务发现                       │
│ - 熔断降级                       │
│ - 分布式追踪                     │
└─────────────────────────────────┘


# 实验设计

Client -> Gateway -> App

客户端选择benchmark：wrk / hey / vegeta
https://github.com/bojand/ghz
https://github.com/zhuweipu/load-testing-toolkit
https://github.com/BuoyantIO/strest-grpc

服务端：Gateway（HTTP） + App（RPC）的组合；
请求：简单 HTTP POST，body 几十字节到几 KB；可以有多个 API（例如 /echo, /calc）。

## 实现方案

| 实现方案 | Gateway ↔ App 通信方式 | 核心特点 | 预期优势 |
| :--- | :--- | :--- | :--- |
| **1. TCP** | Localhost TCP Socket | 标准网络栈，经过内核协议栈 | 基线对比 |
| **2. UDS** | Unix Domain Socket | 绕过内核网络栈，零拷贝（在内核内） | 比TCP延迟更低，开销更小 |
| **3. 共享内存 + eventfd** | Shared Memory + eventfd 通知 | 零拷贝（用户态），但需系统调用通知 | 延迟低，但仍有syscall开销 |
| **4. 共享内存 + UINTR** | Shared Memory + User Interrupt 通知 | 零拷贝，用户态直接中断唤醒 | 理论上延迟最低，syscall/cs最少，尾部延迟最稳定 |

┌─────────────────────────────────────────┐
│            压测客户端                    │
│  (wrk/vegeta/ghz, 控制并发和QPS)        │
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│           Gateway 测试对象               │
│  (Nginx/APISIX/Kong/Envoy/Spring Cloud) │
└─────────────────────────────────────────┐
                   ↓
┌─────────────────────────────────────────┐
│            模拟后端服务                  │
│  (固定延迟响应，如 echo 或计算服务)      │
└─────────────────────────────────────────┐

metrics:
  latency:  # 延迟分布
    - p50
    - p90
    - p95
    - p99
    - p99.9
    - max
  
  throughput:  # 吞吐量
    - rps: 每秒请求数
    - mbps: 网络吞吐量
  
  resource_usage:  # 资源使用
    - cpu_user: 用户态CPU
    - cpu_system: 内核态CPU
    - memory_rss: 常驻内存
    - context_switches: 上下文切换
    - syscalls: 系统调用数
  
  error_rate:  # 错误率
    - success_count
    - error_count
    - timeout_count

## 评测维度
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

4. 对比：
共享内存 + UINTR 方案是否显著减少 syscalls 和 cs。
尾部延迟在负载下的稳定性
增加业务处理时间抖动（比如 App 处理时随机 sleep(0~5ms)）；
观察不同实现的 P99.9 是否存在明显差距；
UINTR 路径短、用户态唤醒更快，有助于减小排队和层次叠加带来的长尾。


# 相关工作

2000-2010: 进程间通信优化，Unix IPC, 共享内存
    
2010-2015: 微服务间通信，REST, RPC框架
    
2015-2020: Sidecar通信优化，Envoy性能调优
    
2020-2023: 用户态IPC创新，io_uring, eBPF

Brendan Burns​ (2015). "Designing Distributed Systems: Patterns and Paradigms for Scalable, Reliable Services" - Sidecar模式定义

Adrian Cockcroft​ (2014). "Microservices" - Netflix微服务实践

"Envoy: A C++ L7 Proxy and Communication Bus"​ (2019) - Envoy设计哲学

"Cost of Service Mesh Sidecars"​ (USENIX ATC 2019) - 性能分析


"Accelerating Service Mesh Data Plane" (SIGCOMM 2021)
基于eBPF的零拷贝代理延迟降低60%，CPU使用率减少40%

"Zero Trust Service Mesh" (IEEE S&P 2022)
智能调度，mTLS自动轮换，基于身份的访问控制，实现动态信任评估

"AI-Driven Traffic Management" (NSDI 2023)
强化学习流量调度，QoS提升30%

外部用户/客户端
                     ↓
           ┌─────────────────┐
           │   API Gateway   │ ← Gateway 在这里
           │  (南北流量入口)  │
           └─────────────────┘
                     ↓
    ┌───────────────┬───────────────┐
    ↓               ↓               ↓
┌─────────┐    ┌─────────┐    ┌─────────┐
│Service A│    │Service B│    │Service C│
├─────────┤    ├─────────┤    ├─────────┤
│ Sidecar │    │ Sidecar │    │ Sidecar │ ← Sidecar 在这里
└─────────┘    └─────────┘    └─────────┘
    ↓               ↓               ↓
    └───────────────┴───────────────┘
            内部网络通信
            (东西流量)
