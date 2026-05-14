Gateway本身的功能并不复杂，主要就是监听端口，建立TCP连接，接受并解析HTTP，
其次就是路由转发，通过RPC或者HTTP协议将请求发送至后端的微服务。如果用AI实现，无非就是100多行代码。

Gateway本身的实现对应性能的影响(尤其是我们只是模拟简单的后端服务的场景下)不大。

Gateway和后端服务是否部署在同一台机器，其通信方式是HTTP还是RPC，以及RPC底层采用什么传输层对应web服务器的性能来说影响较大。

协议          吞吐量      延迟    数据大小
─────────────────────────────────────────   
HTTP/1.1+JSON  10,000/s   10ms    1.2KB
HTTP/2+JSON    15,000/s   8ms     1.2KB
gRPC           50,000/s   2ms     0.4KB
─────────────────────────────────────────
性能提升：
gRPC vs HTTP/1.1+JSON: 5倍吞吐量，5倍延迟降低，3倍数据压缩


# Gateway-gRPC

使用基于 tokio异步运行时的HTTP 库 Hyper处理客户端HTTP请求。然后Gateway将HTTP请求转换为gRPC。
gRPC​ 是 Google 基于 HTTP/2 和 Protocol Buffers 实现的具体 RPC 框架。

目前实现的三种模式，都是使用 tonic的 HTTP endpoint实现的通信。

# 关于RPC的相关工作

能找到的论文基本都是针对 remote这一点展开的，就是优化远程主机的访问。 Datacenter RPCs can be General and Fast和FaSST: Fast, Scalable and Simple Distributed Transactions with Two-Sided (RDMA) Datagram RPCs和HydraRPC: RPC in the CXL Era和In reference to RPC: it's time to add distributed memory和RDMA vs. RPC for Implementing Distributed Data Structures

RPC基于异步的工作。Asynchronous RPC Interface in Distributed Computing System
A survey of asynchronous remote procedure calls （1992年)

Rethinking RPC Communication for Microservices-based Applications

有个预印本 Efficient Asynchronous RPC Calls for Microservices: DeathStarBench Study 