### 1. 建立连接

1. client向envoy发起连接，envoy的worker接收eventloop的callback， 触发 `listenCallback` (port: 15001)
2. 15001的 `useOriginalDst": true`  , `accept_filters_` 中会带有 `OriginalDstFilter`
3. 在 `OriginalDstFilter.OnAccept`中用 `os_syscalls.getsockopt(fd, SOL_IP, SO_ORIGINAL_DST, &orig_addr, &addr_len)` 获取在iptables修改之前dst ip
4. 通过`ConnectionHandlerImpl::findActiveListenerByAddress` 查到addr对应的Listener
   1. 先查找 `Listener.IP==addr.ip && Listener.Port==addr.port`的Listener
   2. 再查找`Listener.IP==0.0.0.0 && Listener.Port==addr.port`的Listener
5. `dispatcher.createServerConnection`  传入accept到的fd 创建Server连接对象 `ConnectionImpl` ， 并把`onFileEvent` 注册到eventloop，等待读写事件的到来，因为socket是由一个non-blocking listening socket创建而来，所以也是non-blocking
6. http的listener里filters为 `envoy.http_connection_manager` ， `buildFilterChain` 里会把`HTTP::ConnectionManagerImpl` 加入到 `upstream_filters_ `(list\<ActiveReadFilterPtr\>)中，这样在请求数据到达的时候，就可以使用http_connection_manager的`on_read` 方法
7. 目前envoy的ServerConnection会被设置 `TCP NO_DELAY`, 同时会把连接加入对应Listener的`connections_`中
8. 当连接刚刚加入eventloop的时候， Write Event会被立即触发，但因为`write_buffer_` 没有数据，所以不会写入任何数据

### 2. 接收请求

1. client开始向socket写入请求数据

2. eventloop在触发read event后，`transport_socket_.doRead` 中会循环读取加入`read_buffer_`，直到返回EAGAIN

3. 把buffer传入`Envoy::Http::ConnectionManagerImpl::onData` 进行HTTP请求的处理

4. 如果`codec_type` 是AUTO（HTTP1 or HTTP2 or AUTO）的情况下，会从请求中搜索 `PRI * HTTP/2` 来判断是否http2

5. 利用`http_parser` 进行http解析的callback, `ConnectionImpl::settings_` 静态初始化了parse各个阶段的callbacks

6. `onMessageBeginBase` 

   1. 创建`ActiveStream` , 保存downstream的信息，和对应的route信息
   2. 对于https，会把TLS握手的时候保存的SNI写入`ActiveStream.requested_server_name_`

7. `onHeaderField`, `onHeaderValue`

    迭代添加header到`current_header_map_` 中

8. `onHeadersComplete`

    把request中的一些字段（method, path, host ）加入headers中

9. `onMessageComplete`

   1. 默认不支持http/1.0和http/0.9协议，快速返回426，可以用过配置`accept_http_10`来开启

   2. 调用`refreshCachedRoute` 在route中查询cluster

   3. route查找逻辑，缓存在`cached_route_` 中

      1. 根据host查找对应virtualhost

         1. 如果只有一个domain为`*`的virtualhost，快速返回
         2. 在整个route中，跨越多个virtualhost，对于domains用http host进行equal匹配，返回匹配到的domain所属virtualhost
         3. domains中匹配通配符放在开始(*.foo.bar)的最长的domain
         4. domains中匹配通配符放在最后(foo.bar.*)的最长的domain
         5. 以上逻辑都没有匹配后，就返回`*` 的virtualhost(需要自己定义)

      2. 通过配置的header和query params在virtualhost的routes中匹配

         1. Method等属性也是存放在header内(`:method`)

         2. header match 

              `Value` 等于，`Regex` 正则，`Range` 在指定范围内，value需要是int，`Present`  存在就可以，`Prefix`前缀，`Suffix` 后缀，`Invert` 上面的match取反。除了`Invert`，上面只有一个能生效

         3. query param match， 只有regex和value

         4. Path match， 只有 `path` (exact), `prefix`, `regex` 

         5. 可以在request中指定cluster name，返回指定cluster

         6. 会根据stream_id在weighted_clusters中进行取余按照概率返回一个cluster  (stream_id为随机数)

   4. 通过route上的cluster name从ThreadLocalClusterManager中查找cluster, 缓存在 `cached_cluster_info_` 中

   5. 根据配置构造在route上的filterChain (具体的filter实现是通过 `registerFactory`方法注册进去，在`createFilterChain` 的时候根据名称构造，比如istio-proxy的mixer)

   6. 如果对应http connection manager上有trace配置

      1. request header中有trace，就创建子span, sampled跟随parent span
      2. 如果header中没有trace，就创建root span, 并设置sampled

   7. 根据http connection manager上配置的filters (`mixer`, `envoy.cors`, `envoy.fault`, `envoy.router`)，一个个执行`decodeHeaders` 

      这里主要写一下`envoy.fault`和`envoy.router`

      1. `envoy.fault`

         1. 如果设置了delay, 会创建一个指定timeout的定时器，并中断decodeHeaders, 当定时器超时的时候会触发继续decode。`decodeHeaders`会从参数filter的下一个开始执行，超时callback会把fault filter传入，这样就可以继续恢复中断之前迭代到的地方
         2. 如果设置了abort，会直接返回响应，并中断decode
         3. 没有情况发生就继续一下个filter

      2. `envoy.router`

         1. 没有route能匹配请求

            返回 404 `no cluster match for URL`

         2. 有配置`directResponseEntry` 

            直接返回

         3. route上的clustername在clustermanager上找不到对应cluster

            返回配置的 `clusterNotFoundResponseCode` 

         4. 当前处于`maintenanceMode`

             503  `maintenance mode`

         5. 调用`getConnPool` 获取upstream conn pool

            1. 根据 cluster上的`features`配置和 `USE_DOWNSTREAM_PROTOCOL` 来确定使用http1还是http2协议向上游发送请求
            2. 在 **ThreadLocalClusterManager** 上根据cluster name查询cluster
            3. 根据loadbalancer算法挑选节点（此处worker之间的负载均衡数据是独立的，比如round robin，只有同一个Worker上的才是严格的顺序）
            4. 根据节点和协议拿到连接池 (连接池由ThreadLocalClusterManager管理，各个Worker不共享)

         6. 如果没有找到对应conn pool

             503 no healthy upstream

         7. 根据配置（timeout, perTryTimeout）确定本次请求的timeout

         8. 检查有无 `x-envoy-upstream-rq-timeout-alt-response` 这个header，如果有timeout的时候用204代替504

         9. 把之前生成的trace写入request header

         10. 对request做一些最终的修改，`headers_to_remove` `headers_to_add` `host_rewrite` `rewritePathHeader`

         11. 构造 retry和shadowing的对象

   ### 3. 发送请求

   发送请求部分也是在`envoy.router`中的逻辑

   1. 查看当前conn pool是否有空闲client
      - 如果存在空闲连接
        1. 根据downstream request和tracing等配置构造发往upstream的请求buffer
        2. 把buffer一次性移入`write_buffer_`, 立即触发Write Event
        3. `ConnectionImpl::onWriteReady` 随后会被触发
        4. 把`write_ buffer_`的内容写入socket发送出去
      - 如果不存在空闲连接
        1. 根据`max_pending_requests`和`max_connections` 判断是否可以创建新的连接（此处的指标为worker间共享）
        2. 根据配置设置新连接的socket options, 使用`dispatcher.createClientConnection` 创建连接上游的连接，并绑定到eventloop
        3. 新建`PendingRequest` 并加到`pending_requests_` 头部
        4. 当连接成功建立的时候，会触发`ConnectionImpl::onFileEvent` 
        5. 在`onConnected`的回调中
           1. 停止`connect_timer_`
           2. 复用存在空闲连接时的逻辑，发送请求
   2. 在 `onRequestComplete`里调用`maybeDoShadowing` 进行流量复制
      1. shadowing流量并不会返回错误
      2. shadowing 流量为asynclient发送，不会阻塞downstream，timeout也为`global_timeout_`
      3. shadowing 会修改request header里的host 和 authority 添加 `-shadow` 后缀
   3. 根据 `global_timeout_` 启动响应超时的定时器

   

   ### 4. 接收响应

   1. eventloop 触发 `ClientConnectionImpl.ConnectionImpl`上的 `onFileEvent` 的read ready事件
   2. 经过http_parser  execute后触发 `onHeadersComplete` 后执行到`UpstreamRequest::decodeHeaders` 
   3. `upstream_request_->upstream_host_->outlierDelector().putHttpResponseCode` 写入status code，更新外部检测的状态
   4. 根据返回结果、配置和 `retries_remaining_`判断是否应该retry，retry的间隔时间在一定范围内成指数级变大
   5. 根据 `internal_redirect_action` 的配置和response来确定是否需要redirect到新的host

   ### 5. 返回响应

   1. 停止 `request_timer` , 重置 `idle_timer`
   2. 和向upstream发送请求一样的逻辑，发送响应给downstream