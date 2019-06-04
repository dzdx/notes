#  简介 

Envoy是istio的核心组件之一，以sidecar的方式与服务运行在一起，对服务的流量进行拦截转发。 具有路由，流量控制等等强大特性。

Envoy利用libevent实现了基于事件触发的异步架构，所有的网络阻塞操作包括 accept，read, connect, write 都是由eventloop进行callback触发

本文以istio1.1所对应的Envoy版本进行源码流程分析

# 名词解释

- 下游: 发送请求给Envoy的服务，client
- 上游：接收Envoy发送的请求，并返回响应的服务， server

# Envoy并发架构

![](https://picgo-1259280442.cos.ap-shanghai.myqcloud.com/20190604135318.png)

> 摘自 <https://blog.envoyproxy.io/envoy-threading-model-a8d44b922310>

- Envoy进程由一个Main Thread和多个Worker Thread 组成

- 每个Main和Worker包含一个eventloop，所有的处理都是由eventloop触发开始

- Main负责xDS等功能，Worker负责处理连接和请求

- 当一个client向Envoy建立连接的时候，因为所有Worker的EventLoop都注册了listening fd，会由内核决定分配给哪个Worker

- 当一个下游client连接到了Envoy，在保持连接不断的情况下，会和同一个Worker进行通讯

  

# libevent函数

- `evconnlistener_new`

  把一个 **已经bind port** 的listening fd和callback注册到eventloop，当accept到新连接的时候会触发callback。Envoy里面采用这个函数把同一个listening fd注册到所有的Worker的eventloop中，当新连接来的时候，由内核选择应该分发给哪个Worker

- `event_assign`

  把一个fd和一个callback注册到eventloop，当read write closed事件触发的时候触发callback

- `event_active`

  立即触发一个eventloop中的event，执行callback

# 事件触发各阶段

1. client向Envoy建立连接

   ![](https://picgo-1259280442.cos.ap-shanghai.myqcloud.com/20190603220321.png)

2. client发送请求到Envoy，Envoy挑选节点向上游Server建立连接(如果连接池有空闲连接直接发送请求)

![](https://picgo-1259280442.cos.ap-shanghai.myqcloud.com/20190603233221.png)

3. Envoy向上游建连接成功，发送请求

   

   ![](https://picgo-1259280442.cos.ap-shanghai.myqcloud.com/20190603220357.png)

   

4. 上游server返回响应给Envoy

   ![](https://picgo-1259280442.cos.ap-shanghai.myqcloud.com/20190603220430.png)

5. Envoy返回请求给下游的client

   ![](https://picgo-1259280442.cos.ap-shanghai.myqcloud.com/20190603220452.png)


