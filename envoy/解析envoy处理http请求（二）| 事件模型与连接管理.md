#  简介 

Envoy是istio的核心组件之一，以sidecar的方式与服务运行在一起，对服务的流量进行拦截转发。 具有路由，流量控制等等强大特性。

Envoy利用libevent实现了基于事件触发的异步架构，所有的网络阻塞操作包括 accept，read, connect, write 都是由eventloop进行callback触发

本文以istio1.1所对应的Envoy版本进行源码流程分析

# 名词解释

- 下游: 发送请求给Envoy的服务，client
- 上游：接收Envoy发送的请求，并返回响应的服务， server
- Eventloop: libevent中的eventloop，可以用于注册事件的callback并触发回调
- fd: Envoy创建的监听Connection和用于传输上下游数据的文件描述符

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





# Cluster管理 (HTTP1)

## 层次结构图

- 上面的实线表示下方的头部是上方的属性之一
- 虚线表示两者相关
- `ClusterManagerImpl`是Envoy内的单例，用于管理多个Worker上`ThreadLocalClusterManagerImpl`
- `ThreadLocalCusterManagerImpl` 每个Worker都拥有一个，用于管理上游的连接和负载均衡上下文

![](https://picgo-1259280442.cos.ap-shanghai.myqcloud.com/20190531180804.png)

## 连接结构简介

1. 负载均衡器只挑选host，然后会从conn_pool_map取出对应host的连接池

2. 每个worker都都包含自己独立的连接池和负载均衡上下文

   （此处有发现设置RoundRobin负载均衡策略的时候，只有client保持长连接（不换worker）的情况下，才是严格的轮询）

3. 同一个上游节点的不同协议（http10, http11, http2, tcp）的连接池都是分开的



## 连接管理

对于同一个Worker，同一个Host，同一个协议，Envoy会维护一个连接池，连接池中http1有关属性如下（一下情况没有对Limit做说明，实际各个阶段会有stats和config limit来进行限制）：

- `busy_clients_ ` 当前发送了请求，还没处理响应的connection

  变多：

  - 有新的请求需要发送的时候，如果`ready_clients`非空， 就会把`ready_clients_.front()`移到`busy_clients.front()`， 并发送请求
  - 向上游的连接connected时，如果 `(!pending_requests_.empty() && !ready_clients_.empty()) ` 就会把`ready_clients_.front()` 移到`busy_clients_`
  - 当有连接close的时候，如果`pending_requests_.size() > (ready_clients_.size() + busy_clients_.size())`，就会创建新的connection到`busy_clients_`并发送请求

  变少：

  - connpool被删除，`busy_clients_`会被清空
  - 连接关闭时，如果是正在包含stream_wrapper (正在发送请求）的client，就会从`busy_clients`移除

- `ready_clients_`  空闲连接

  变多：

  - ((连接connected的时候 || 有响应完成的时候) &&  `pending_requests_` 为空) ，就会加入`ready_clients`

  变少：

  - 在有新的请求需要发送的时候，如果`ready_clients`非空， 就会把`ready_clients_.front()`移到`busy_clients.front()`

  - 向上游的连接connected时，如果 `(!pending_requests_.empty() && !ready_clients_.empty()) ` 就会把`ready_clients_.front()` 移到`busy_clients_`

  - connpool被删除，`ready_clients_`会被清空

  - 空闲的连接被关闭的时候，会从`ready_clients_` 中移除

    

- `pending_requests_` 代发送的请求

  变多：

  - `ready_clients` 为空的时候，有新的请求，就会加入到`pending_requests`

  变少：

  - 当上游连接connected时，发送请求到连接，并从`pending_requests` 中移除
  - 请求被终止的时候（超时，或者收到上游的响应结束 ），就会从`pending_requests` 中移除



# 总结

1. Envoy的处理请求过程依赖libevent的事件触发，包括从建立连接到断开连接的全部过程
2. 每个Worker的连接池和负载均衡上下文都是独立的
3. 连接池内的连接都是同一worker，同一upstream host，同一协议