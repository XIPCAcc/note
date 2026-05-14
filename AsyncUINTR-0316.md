# Kairos-rs

目前 Rust 生态里比较完整、明确标榜为“网关”的项目是 Kairos-rs：

项目定位：生产可用的 HTTP/API 网关和反向代理
技术栈：
Rust 编写
以 Actix Web 作为 HTTP 服务框架
明确使用 Tokio 作为异步运行时：
文档中的示例用 #[tokio::main] 启动入口函数；


# gRPC：tonic
官方描述：“A Rust implementation of gRPC …”，是高性能 gRPC 客户端/服务端实现[1]。
依赖 Tokio 作为异步运行时（tokio、tokio-stream 等依赖可见于 crates.io 元数据）。
提供：
基于 HTTP/2 + Protobuf 的高性能 RPC
代码生成（从 .proto 生成 Rust 接口和消息类型）
双向流、服务器流、客户端流等 gRPC 特性
典型用法：
#[tokio::main] 启动服务
Server::builder().add_service(...).serve(addr).await 跑在 Tokio runtime 上


如何用 Tokio + tonic + Kairos-rs 组合出一套带 gRPC 的微服务 + 网关架构

https://github.com/hyperium/tonic

Tonic是一种基于HTTP/2的gRPC实现，注重高性能、互操作性和灵活性。

