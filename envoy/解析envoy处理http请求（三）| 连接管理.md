#  简介 

Envoy是istio的核心组件之一，以sidecar的方式与服务运行在一起，对服务的流量进行拦截转发。 具有路由，流量控制等等强大特性。

本文以istio1.1所对应的Envoy版本进行源码流程分析



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

1. 每个Worker的连接池和负载均衡上下文都是独立的
2. 连接池内的连接都是同一worker，同一upstream host，同一协议
