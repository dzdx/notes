# Envoy 核心模块

## libevent

1. `event_base_new()` 

    创建event_base对象

2. `event_base_loop(base, flag)`

    创建event loop， 接受event的注册和处理callback

   base: even_base对象

   flag:

      默认（0x00）是当前event_base没有事件注册的时候返回

   ​      Worker 都是使用了默认flag

     `EVLOOP_ONCE: 0x01` 等待event变active, 运行后退出

     `EVLOOP_NONBLOCK: 0x02`  不会等待，如果有active event，会立即执行对应callback，执行完毕会立刻退出

     `EVLOOP_NO_EXIT_ON_EMPTY: 0x04 `  即使没有事件注册也不会退出，直到调用了 `event_base_loopbreak`, `event_base_loopexit` 或者出现错误

   ​    GuardDog使用了`EVLOOP_NO_EXIT_ON_EMPTY` flag

   参考: 
   <http://www.wangafu.net/~nickm/libevent-book/Ref3_eventloop.html>

3. `event_base_loopexit(base, timeval)` 

     使一个event_base退出event loop

      timevar: 多长时间后event_base会被终止， NULL就表示运行完所有当前active events就会退出

4. `event_assign(ev, base, fd, events, callback, arg)`

     设置一个event关联callback

     ev: event struct, 标识符

     fd:  监听的文件描述符

     events:  监听的事件， 类似 EV_READ, EV_WRITE

     callback: event发生的时候触发的回调函数

      args: 传递给callback的参数

5. `evtimer_assign(ev, base, callback, arg)`

   ​    `event_assign(ev, base, -1, 0, callback, arg)`

6. `evsignal_assign(ev, base, signal, callback, arg)`

   ​    `event_assign(ev, base, signal, EV_SIGNAL|EV_PERSIST, cb, arg)`

7. `evconnlistener_new(base, callback, ptr, flags, backlog, fd)`

      会对一个已经bind的fd利用event loop 进行listen操作, 并设置backlog

   ​    ptr:  传递给callback的参数

   ​    flags:   <https://github.com/libevent/libevent/blob/master/include/event2/listener.h#L62>

   ​    backlog: 已经tcp握手完成，但还没有被accept的fd队列大小

8. `event_active(event, res, ncalls)`

     立即active一个event

     event:  active的event对象

     res: 事件的类型，会传递给callback （EV_TIMEOUT， EV_READ，EV_WRITE …）

     ncalls:  废弃字段

9. `event_add(event, timeout)`

    把event加入pending队列， 并设置timeout

     timeout:  event会被执行当timeout过期或者特定条件触发的时候，NULL 永不过期，只有当event_assign或者event_new时设置的条件触发的时候才会执行

10. `event_del(event)`

     把一个event从监听的events中删除

​    

11. `evbuffer_new()`

     返回一个空的evbuffer对象

12. `evbuffer_add(buf, data, size)`

     把data加到buf的尾部

     buf:  evbuffer对象

13. `evbuffer_add_reference(outbuf, data, size, cleanupfn, cleanupfn_arg)`

     无拷贝的引用一段内存到evbuffer

    cleanupfn: 当内存不再被evbuffer应用的时候会调用

14. `evbuffer_prepend(buf, data, size)`

     把data加到buf头部

15.  `evbuffer_reserve_space(buf, size, vec, n_vec)`

     在evbuffer的链种保留一段大小的空间，在 `evbuffer_commit_space` 之前都无法读取

      vec:  保留内存的数据，`evbuffer_iovc` 结构

      n_vec: vec的个数， 最少1

16. `evbuffer_commit_space(buf, vec, n_vecs)`

     commit之前reserve的space

​    

17. `evbuffer_peek(buf, len, start_at, vec_out, n_vec)`

     peek 到evbuffer中指定位置

18. `evbuffer_drain(buf, len)`

     移除evbuffer中头部指定长度的数据

     

evbuffer在envoy中的用法主要是read和write

read:

   利用 `evbuffer_reserve_space` 和 `evbuffer_commit_space` 在evbuffer种申请内存， 用syscall readv, 读取指定fd到evbuffer中

write:

  利用`evbuffer_peek`和syscall writev 向fd写入evbuffer中的首部数据，并用`evbuffer_drain` 删除已写入的数据



## dispatcher

封装了libevent ，方便envoy worker使用。

envoy强依赖libevent进行事件处理， master和worker的功能分隔如下

![](https://ws3.sinaimg.cn/large/006tNc79gy1g2cli29m25j310n0u0gq7.jpg)

dispatcher为每个worker (包括master) 独立拥有， 即每一个worker一个eventloop。

Master:   <https://github.com/envoyproxy/envoy/blob/ac7aa5ac8a815e5277b4d4659c5c02145fa1d56f/source/server/server.cc#L62>

Worker:

<https://github.com/envoyproxy/envoy/blob/ac7aa5ac8a815e5277b4d4659c5c02145fa1d56f/source/server/worker_impl.cc#L17>



1. `createTimer(callback)`

     调用 `evtimer_assign` 注册一个callback到event_base， 并返回一个TimerImpl 包含了event对象，并没有真正执行callback

2. `TimerImpl::enableTimer(milliseconds)`

     通过 `event_active` (立即active) 和 `event_add` (延迟active)来调用TimerImpl上event对应的callback

3. `post(callback)`

     把一个callback加入post_callbacks_ 列表，当前post_callbacks_ 有任务, 就通过active post_timer来调用 `runPostCallbacks` 执行所有的callback

4. `createFileEvent(fd, callback, trigger, events)`

     当fd 发生events的事件（可读或可写）的时候会触发

      fd:  监控的fd， file, socket都可以

      callback：触发时调用的函数

      trigger:

   ​       Level: 水平触发

   ​       Edge: 边沿触发：用于连接上事件的触发

      events:

   ​       Read or Write or Closed

5. `createClientConnection(address, source_address, transport_socket, options)`

    创建指向upstream的连接，会使用 `createFileEvent` 对连接的事件（Read|Write）进行注册，触发 `onFileEvent`

6. `createServerConnection(socket, transport_socket)`

    根据传入的downstream的socket，创建ConnectionImpl,  同时使用 `createFileEvent` 注册

7. `createListener(socket, callback, bind_to_port, hand_off_restored_destination_connections)`

     创建用于管理地址信息的 `ListenerImpl` 对象，并根据bind_to_port (istio架构下的15001)， 使用`evconnListener_new` 注册了listenCallback， 用于downstream请求的handle

8. `listenForSignal(signal_num, callback)` 

     利用`evsignal_assign`和 `evsignal_add` 注册signal handler到eventloop

9. `run(type)`

    调用 `event_base_loop` ， 根据type的参数，会阻塞于这个函数



## TLS(Thread Local Storage)

tls为master和worker共享一个 `ThreadLocal::Instance` 对象，利用c++的thread_local keyword进行隔离

1. `registerThread(dispatcher, main_thread)`

     只能在master调用，可以把worker的dispatcher加入 `registered_threads_` 进行管理

2. `runOnAllThreads(callback)`

    只能在master调用，把指定callback在  `registered_threads_` 内的dispatcher (worker dispatchers) 上执行, 常用于比如rds更新

3. `allocateSlot()`

    申请一个用于存放信息的 **位置**，所有的thread共享, slot有很多种，比如: 

   -  `AsyncClientManagerImpl::google_tls_slot_`
   - `RdsRouteConfigProviderImpl::tls`
   - `Config::upstream_drain_manager_slot_`

![img](https://cdn-images-1.medium.com/max/1600/1*fyx9IJBwbGDVtK_LhwQB6A.png)

4. `SlotImpl::set(callback)`

    传入一个callback, callback参数为Dispatcher, 在每个thread上都执行一遍，并把返回值设置给thread local的`thread_local_data_` 上slot的index指示处

5. `SlotImpl::get()`

    返回`thread_local_data_` 中存储的slot对应对象

   

#  启动流程

1. 初始化 thread local storage

    <https://github.com/envoyproxy/envoy/blob/ac7aa5ac8a815e5277b4d4659c5c02145fa1d56f/source/exe/main_common.cc#L70>

     ```
   tls_ = std::make_unique<ThreadLocal::InstanceImpl>();
    ```

2. 初始化master的dispatcher

    <https://github.com/envoyproxy/envoy/blob/ac7aa5ac8a815e5277b4d4659c5c02145fa1d56f/source/server/server.cc#L62>

   ```
   dispatcher_(api_->allocateDispatcher()),
   ```

3. Load bootstrap config

    <https://github.com/envoyproxy/envoy/blob/ac7aa5ac8a815e5277b4d4659c5c02145fa1d56f/source/server/server.cc#L237>

   ```
   InstanceUtil::loadBootstrapConfig(bootstrap_, options);
   bootstrap_config_update_time_ = time_system_.systemTime();
   ```

4. 初始化Workers

   *根据机器的硬件线程数目创建worker*

   <https://github.com/envoyproxy/envoy/blob/ac7aa5ac8a815e5277b4d4659c5c02145fa1d56f/source/server/listener_manager_impl.cc#L679>

   ```
   for (uint32_t i = 0; i < server.options().concurrency(); i++) {
       workers_.emplace_back(worker_factory.createWorker(server.overloadManager()));
   }
   ```

5. 根据bootstrap config中的静态配置初始化 secrets, cluster, listener等配置

   <https://github.com/envoyproxy/envoy/blob/ac7aa5ac8a815e5277b4d4659c5c02145fa1d56f/source/server/configuration_impl.cc#L46>

   ```
   onst auto& secrets = bootstrap.static_resources().secrets();
     ENVOY_LOG(info, "loading {} static secret(s)", secrets.size());
     for (ssize_t i = 0; i < secrets.size(); i++) {
       ENVOY_LOG(debug, "static secret #{}: {}", i, secrets[i].name());
       server.secretManager().addStaticSecret(secrets[i]);
     }
   
     ENVOY_LOG(info, "loading {} cluster(s)", bootstrap.static_resources().clusters().size());
     cluster_manager_ = cluster_manager_factory.clusterManagerFromProto(bootstrap);
   
     const auto& listeners = bootstrap.static_resources().listeners();
     ENVOY_LOG(info, "loading {} listener(s)", listeners.size());
     for (ssize_t i = 0; i < listeners.size(); i++) {
       ENVOY_LOG(debug, "listener #{}:", i);
       server.listenerManager().addOrUpdateListener(listeners[i], "", false);
     }
   ```

   

6. 初始化ClusterManager

   1. load static cluster (exclude eds)
   2. 初始化ads grcp client
   3. load static cluster (eds)  (在v2版本的配置中，eds cluster会依赖于non-eds的cluster)
   4. 初始化 `ThreadLocalClusterManager` ，提供从中央cluster manager更新的cached cluster data， 维护load balanacer状态和连接池
   5. 通过 `onClusterInit` 把static cluster 更新到 ThreadLocalClusterManager中
   6. 启动ads grpc双向stream， 并发送ads请求

   <https://github.com/envoyproxy/envoy/blob/ac7aa5ac8a815e5277b4d4659c5c02145fa1d56f/source/common/upstream/cluster_manager_impl.cc#L196>

   
   
7.  当第一次rds synced的时候, 会启动所有的worker

   1. 把active_listeners_ 加入所有的worker
   2. 用thread启动每一个worker

    <https://github.com/envoyproxy/envoy/blob/ac7aa5ac8a815e5277b4d4659c5c02145fa1d56f/source/server/server.cc#L419>

   ```
   init_watcher_("RunHelper", [&instance, workers_start_cb]() {
           if (!instance.isShutdown()) {
             workers_start_cb();
           }
         }) 
   ```

   

# ADS

###依赖触发流程

1. 根据bootstrap config 初始化grcp client 的 `remote_cluster_name_` 等配置

   <https://github.com/envoyproxy/envoy/blob/ac7aa5ac8a815e5277b4d4659c5c02145fa1d56f/source/common/upstream/cluster_manager_impl.cc#L212>

   ```
   ads_mux_ = std::make_unique<Config::GrpcMuxImpl>(
           local_info,
           Config::Utility::factoryForGrpcApiConfigSource(
               *async_client_manager_, bootstrap.dynamic_resources().ads_config(), stats)
               ->create(),
           main_thread_dispatcher,
           *Protobuf::DescriptorPool::generated_pool()->FindMethodByName(
               "envoy.service.discovery.v2.AggregatedDiscoveryService.StreamAggregatedResources"),
           random_, stats_,
           Envoy::Config::Utility::parseRateLimitSettings(bootstrap.dynamic_resources().ads_config()));
   ```

   

2. 建立grpc连接的时候，根据1中设置的 `remote_cluster_name_`  在`ThreadLocalClusterManagerImpl` 中取出对应http client， 并发出grpc请求
   <https://github.com/envoyproxy/envoy/blob/7de2b39eeb7d0929fecb00e7b81c70236c3a4869/source/common/upstream/cluster_manager_impl.cc#L738>

```
ThreadLocalClusterManagerImpl& cluster_manager = tls_>getTyped<ThreadLocalClusterManagerImpl>();
  auto entry = cluster_manager.thread_local_clusters_.find(cluster);
  if (entry != cluster_manager.thread_local_clusters_.end()) {
    return entry->second->http_async_client_;
  } else {
    throw EnvoyException(fmt::format("unknown cluster '{}'", cluster));
  }
```

3. 首先发送 `type.googleapis.com/envoy.api.v2.Cluster` (cds)

Pilot 那边会把该envoy所对应的所有Cluster一次性返回

<https://github.com/istio/istio/blob/f839501d87667181926f3e9d9de4a76c35ba2a3d/pilot/pkg/proxy/envoy/v2/cds.go#L59>

```
	rawClusters, err := s.generateRawClusters(con.modelNode, push)
	if err != nil {
		return err
	}
	if s.DebugConfigs {
		con.CDSClusters = rawClusters
	}
	response := con.clusters(rawClusters)
	err = con.send(response)
```

4. 与当前ClusterManager中的Cluster进行对比，得到 to_add和to_remove
   <https://github.com/envoyproxy/envoy/blob/af26857cead7501d622d1a94d15dfa2d6dc99e3b/source/common/upstream/cds_api_impl.cc#L45>

5. 此时暂停发送eds请求 ，并在cds更新完成后统一发送 `ClusterLoadAssignment` (eds)

   ```
   cm_.adsMux().pause(Config::TypeUrl::get().ClusterLoadAssignment);
   Cleanup eds_resume([this] { cm_.adsMux().resume(Config::TypeUrl::get().ClusterLoadAssignment); });
     
   ---
   // RAII cleanup via functor.
   class Cleanup {
   public:
     Cleanup(std::function<void()> f) : f_(std::move(f)) {}
     ~Cleanup() { f_(); }
   
   private:
     std::function<void()> f_;
   };
   ```

6. 在更新cluster的时候，会更新到每个worker的`ThreadLocalClusterManagerImpl` 中 (启动的时候只有add， update的逻辑较复杂)
   <https://github.com/envoyproxy/envoy/blob/7de2b39eeb7d0929fecb00e7b81c70236c3a4869/source/common/upstream/cluster_manager_impl.cc#L476>

```
  if (use_active_map) {
    ENVOY_LOG(info, "add/update cluster {} during init", cluster_name);
    auto& cluster_entry = active_clusters_.at(cluster_name);
    createOrUpdateThreadLocalCluster(*cluster_entry);
    init_helper_.addCluster(*cluster_entry->cluster_);
```



7. 当所有的cluster初始化完毕的时候，会挨个触发`ClusterLoadAssignment` 请求

```
Envoy::Config::GrpcMuxImpl::subscribe grpc_mux_impl.cc:80
Envoy::Config::GrpcMuxSubscriptionImpl::start grpc_mux_subscription_impl.h:41
Envoy::Upstream::EdsClusterImpl::startPreInit eds.cc:34
Envoy::Upstream::ClusterImplBase::initialize(std::function<void ()>) upstream_impl.cc:711
Envoy::Upstream::ClusterManagerInitHelper::maybeFinishInitialize cluster_manager_impl.cc:122
Envoy::Upstream::ClusterManagerInitHelper::removeCluster cluster_manager_impl.cc:93
Envoy::Upstream::ClusterManagerInitHelper::onClusterInit cluster_manager_impl.cc:70
```

8. eds的更新也会post到每个worker去执行， 更新到`ThreadLocalClusterManagerImpl` 上
   <https://github.com/envoyproxy/envoy/blob/7de2b39eeb7d0929fecb00e7b81c70236c3a4869/source/common/upstream/cluster_manager_impl.cc#L355>

```
  for (auto& host_set : cluster.prioritySet().hostSetsPerPriority()) {
    if (host_set->hosts().empty()) {
      continue;
    }
    postThreadLocalClusterUpdate(cluster, host_set->priority(), host_set->hosts(), HostVector{});
  }
```



9. 当 `ClusterManagerImpl` 初始化完成(cds和eds同步完成)的时候会暂时暂停rds更新, 并发送lds的请求
   <https://github.com/envoyproxy/envoy/blob/e1450a1ca006f7d8b118a63b0f7e30be2639b881/source/server/server.cc#L452>

```
  cm.setInitializedCb([&instance, &init_manager, &cm, this]() {
    if (instance.isShutdown()) {
      return;
    }

    // Pause RDS to ensure that we don't send any requests until we've
    // subscribed to all the RDS resources. The subscriptions happen in the init callbacks,
    // so we pause RDS until we've completed all the callbacks.
    cm.adsMux().pause(Config::TypeUrl::get().RouteConfiguration);

    ENVOY_LOG(info, "all clusters initialized. initializing init manager");
    init_manager.initialize(init_watcher_);

    // Now that we're execute all the init callbacks we can resume RDS
    // as we've subscribed to all the statically defined RDS resources.
    cm.adsMux().resume(Config::TypeUrl::get().RouteConfiguration);
  });
```



10. lds的初始化callback是作为init manager一个handle加入，并在init done的时候回调，发送 `type.googleapis.com/envoy.api.v2.Listener` (lds) 请求

```
Envoy::Init::ManagerImpl::add manager_impl.cc:21
Envoy::Server::LdsApiImpl::LdsApiImpl lds_api.cc:31
std::make_unique<Envoy::Server::LdsApiImpl, envoy::api::v2::core::ConfigSource const&, Envoy::Upstream::ClusterManager&, Envoy::Event::Dispatcher&, Envoy::Runtime::RandomGenerator&, Envoy::Init::Manager&, Envoy::LocalInfo::LocalInfo const&, Envoy::Stats::Store&, Envoy::Server::ListenerManager&, Envoy::Api::Api&> unique_ptr.h:831
Envoy::Server::ProdListenerComponentFactory::createLdsApi listener_manager_impl.h:49
Envoy::Server::ListenerManagerImpl::createLdsApi listener_manager_impl.h:116
Envoy::Server::InstanceImpl::initialize server.cc:339
```



11. pilot会把对应的Listener全部push给envoy

 <https://github.com/istio/istio/blob/a9aac591d43c38fb39a14a86f6494015d54092f9/pilot/pkg/proxy/envoy/v2/lds.go#L37-L36>



12. lds的更新期间会暂停rds请求的发送
    <https://github.com/envoyproxy/envoy/blob/af26857cead7501d622d1a94d15dfa2d6dc99e3b/source/server/lds_api.cc#L37>

```
void LdsApiImpl::onConfigUpdate(const Protobuf::RepeatedPtrField<ProtobufWkt::Any>& resources,
                                const std::string& version_info) {
  cm_.adsMux().pause(Config::TypeUrl::get().RouteConfiguration);
  Cleanup rds_resume([this] { cm_.adsMux().resume(Config::TypeUrl::get().RouteConfiguration); });

```



13. 接收到`bindToPort` 的Listener后，会在master进行端口bind

    ```
    Envoy::Network::ListenSocketImpl::doBind listen_socket_impl.cc:20
    Envoy::Network::ListenSocketImpl::setupSocket listen_socket_impl.cc:46
    Envoy::Network::NetworkListenSocket<Envoy::Network::NetworkSocketTrait<(Envoy::Network::Address::SocketType)0> >::NetworkListenSocket listen_socket_impl.h:90
    __gnu_cxx::new_allocator<Envoy::Network::NetworkListenSocket<Envoy::Network::NetworkSocketTrait<(Envoy::Network::Address::SocketType)0> > >::construct<Envoy::Network::NetworkListenSocket<Envoy::Network::NetworkSocketTrait<(Envoy::Network::Address::SocketType)0> >, std::shared_ptr<Envoy::Network::Address::Instance const>&, std::shared_ptr<std::vector<std::shared_ptr<Envoy::Network::Socket::Option const>, std::allocator<std::shared_ptr<Envoy::Network::Socket::Option const> > > > const&, bool&> new_allocator.h:136
    std::allocator_traits<std::allocator<Envoy::Network::NetworkListenSocket<Envoy::Network::NetworkSocketTrait<(Envoy::Network::Address::SocketType)0> > > >::construct<Envoy::Network::NetworkListenSocket<Envoy::Network::NetworkSocketTrait<(Envoy::Network::Address::SocketType)0> >, std::shared_ptr<Envoy::Network::Address::Instance const>&, std::shared_ptr<std::vector<std::shared_ptr<Envoy::Network::Socket::Option const>, std::allocator<std::shared_ptr<Envoy::Network::Socket::Option const> > > > const&, bool&> alloc_traits.h:475
    std::_Sp_counted_ptr_inplace<Envoy::Network::NetworkListenSocket<Envoy::Network::NetworkSocketTrait<(Envoy::Network::Address::SocketType)0> >, std::allocator<Envoy::Network::NetworkListenSocket<Envoy::Network::NetworkSocketTrait<(Envoy::Network::Address::SocketType)0> > >, (__gnu_cxx::_Lock_policy)2>::_Sp_counted_ptr_inplace<std::shared_ptr<Envoy::Network::Address::Instance const>&, std::shared_ptr<std::vector<std::shared_ptr<Envoy::Network::Socket::Option const>, std::allocator<std::shared_ptr<Envoy::Network::Socket::Option const> > > > const&, bool&> shared_ptr_base.h:545
    std::__shared_count<(__gnu_cxx::_Lock_policy)2>::__shared_count<Envoy::Network::NetworkListenSocket<Envoy::Network::NetworkSocketTrait<(Envoy::Network::Address::SocketType)0> >, std::allocator<Envoy::Network::NetworkListenSocket<Envoy::Network::NetworkSocketTrait<(Envoy::Network::Address::SocketType)0> > >, std::shared_ptr<Envoy::Network::Address::Instance const>&, std::shared_ptr<std::vector<std::shared_ptr<Envoy::Network::Socket::Option const>, std::allocator<std::shared_ptr<Envoy::Network::Socket::Option const> > > > const&, bool&> shared_ptr_base.h:677
    std::__shared_ptr<Envoy::Network::NetworkListenSocket<Envoy::Network::NetworkSocketTrait<(Envoy::Network::Address::SocketType)0> >, (__gnu_cxx::_Lock_policy)2>::__shared_ptr<std::allocator<Envoy::Network::NetworkListenSocket<Envoy::Network::NetworkSocketTrait<(Envoy::Network::Address::SocketType)0> > >, std::shared_ptr<Envoy::Network::Address::Instance const>&, std::shared_ptr<std::vector<std::shared_ptr<Envoy::Network::Socket::Option const>, std::allocator<std::shared_ptr<Envoy::Network::Socket::Option const> > > > const&, bool&> shared_ptr_base.h:1342
    std::shared_ptr<Envoy::Network::NetworkListenSocket<Envoy::Network::NetworkSocketTrait<(Envoy::Network::Address::SocketType)0> > >::shared_ptr<std::allocator<Envoy::Network::NetworkListenSocket<Envoy::Network::NetworkSocketTrait<(Envoy::Network::Address::SocketType)0> > >, std::shared_ptr<Envoy::Network::Address::Instance const>&, std::shared_ptr<std::vector<std::shared_ptr<Envoy::Network::Socket::Option const>, std::allocator<std::shared_ptr<Envoy::Network::Socket::Option const> > > > const&, bool&> shared_ptr.h:359
    std::allocate_shared<Envoy::Network::NetworkListenSocket<Envoy::Network::NetworkSocketTrait<(Envoy::Network::Address::SocketType)0> >, std::allocator<Envoy::Network::NetworkListenSocket<Envoy::Network::NetworkSocketTrait<(Envoy::Network::Address::SocketType)0> > >, std::shared_ptr<Envoy::Network::Address::Instance const>&, std::shared_ptr<std::vector<std::shared_ptr<Envoy::Network::Socket::Option const>, std::allocator<std::shared_ptr<Envoy::Network::Socket::Option const> > > > const&, bool&> shared_ptr.h:706
    std::make_shared<Envoy::Network::NetworkListenSocket<Envoy::Network::NetworkSocketTrait<(Envoy::Network::Address::SocketType)0> >, std::shared_ptr<Envoy::Network::Address::Instance const>&, std::shared_ptr<std::vector<std::shared_ptr<Envoy::Network::Socket::Option const>, std::allocator<std::shared_ptr<Envoy::Network::Socket::Option const> > > > const&, bool&> shared_ptr.h:722
    Envoy::Server::ProdListenerComponentFactory::createListenSocket listener_manager_impl.cc:144
    Envoy::Server::ListenerManagerImpl::addOrUpdateListener listener_manager_impl.cc:819
    Envoy::Server::LdsApiImpl::onConfigUpdate lds_api.cc:73
    ```

14. 在 master接受到LDS，初始化 `ListenerImpl`的时候，会把listener加到所有worker中，同时会把`bind_to_port==true` 的 listener的fd加入到所有worker的event loop中，绑定 `ListenCallback`


    <https://github.com/envoyproxy/envoy/blob/814dd9facff27391a42744309f7765dc94971e40/source/server/listener_manager_impl.cc#L948>

    ```
    void ListenerManagerImpl::onListenerWarmed(ListenerImpl& listener) {
      // The warmed listener should be added first so that the worker will accept new connections
      // when it stops listening on the old listener.
      for (const auto& worker : workers_) {
        addListenerToWorker(*worker, listener);
      }
    ```

    

    <https://github.com/envoyproxy/envoy/blob/b2149d1e0cc617a4763246daaeafed9e95677f98/source/common/network/listener_impl.cc#L52>

```
listener_.reset(
      evconnlistener_new(&dispatcher.base(), listenCallback, this, 0, -1, socket.ioHandle().fd()));
```

15. 在`ListenerImpl` 初始化的时候，对应的Rds的handle会调用对应handle, 发送 `type.googleapis.com/envoy.api.v2.RouteConfiguration` (rds)请求

```
Envoy::Config::GrpcMuxImpl::subscribe grpc_mux_impl.cc:80
Envoy::Config::GrpcMuxSubscriptionImpl::start grpc_mux_subscription_impl.h:41
Envoy::Router::RdsRouteConfigSubscription::<lambda()>::operator()(void) const rds_impl.cc:64
std::_Function_handler<void(), Envoy::Router::RdsRouteConfigSubscription::RdsRouteConfigSubscription(const envoy::config::filter::network::http_connection_manager::v2::Rds&, uint64_t, Envoy::Server::Configuration::FactoryContext&, const string&, Envoy::Router::RouteConfigProviderManagerImpl&)::<lambda()> >::_M_invoke(const std::_Any_data &) std_function.h:297
std::function<void ()>::operator()() const std_function.h:687
Envoy::Init::TargetImpl::<lambda(Envoy::Init::WatcherHandlePtr)>::operator()(Envoy::Init::WatcherHandlePtr) const target_impl.cc:29
std::_Function_handler<void(std::unique_ptr<Envoy::Init::WatcherHandle, std::default_delete<Envoy::Init::WatcherHandle> >), Envoy::Init::TargetImpl::TargetImpl(absl::string_view, Envoy::Init::InitializeFn)::<lambda(Envoy::Init::WatcherHandlePtr)> >::_M_invoke(const std::_Any_data &, std::unique_ptr<Envoy::Init::WatcherHandle, std::default_delete<Envoy::Init::WatcherHandle> > &&) std_function.h:297
std::function<void (std::unique_ptr<Envoy::Init::WatcherHandle, std::default_delete<Envoy::Init::WatcherHandle> >)>::operator()(std::unique_ptr<Envoy::Init::WatcherHandle, std::default_delete<Envoy::Init::WatcherHandle> >) const std_function.h:687
Envoy::Init::TargetHandleImpl::initialize target_impl.cc:16
Envoy::Init::ManagerImpl::add manager_impl.cc:28
Envoy::Router::RouteConfigProviderManagerImpl::createRdsRouteConfigProvider rds_impl.cc:202
Envoy::Router::RouteConfigProviderUtil::create rds_impl.cc:35
Envoy::Extensions::NetworkFilters::HttpConnectionManager::HttpConnectionManagerConfig::HttpConnectionManagerConfig config.cc:167
Envoy::Extensions::NetworkFilters::HttpConnectionManager::HttpConnectionManagerFilterConfigFactory::createFilterFactoryFromProtoTyped config.cc:92
Envoy::Extensions::NetworkFilters::Common::FactoryBase<envoy::config::filter::network::http_connection_manager::v2::HttpConnectionManager, envoy::config::filter::network::http_connection_manager::v2::HttpConnectionManager>::createFilterFactoryFromProto factory_base.h:29
Envoy::Server::ProdListenerComponentFactory::createNetworkFilterFactoryList_ listener_manager_impl.cc:70
Envoy::Server::ProdListenerComponentFactory::createNetworkFilterFactoryList listener_manager_impl.h:57
Envoy::Server::ListenerImpl::ListenerImpl listener_manager_impl.cc:283
Envoy::Server::ListenerManagerImpl::addOrUpdateListener listener_manager_impl.cc:751
Envoy::Server::LdsApiImpl::onConfigUpdate lds_api.cc:73
```

16. RDS的更新很简单，master分发给所有worker，设置 `ThreadLocalConfig.config_` 的变量
    <https://github.com/envoyproxy/envoy/blob/814dd9facff27391a42744309f7765dc94971e40/source/common/router/rds_impl.cc#L170>

```
void RdsRouteConfigProviderImpl::onConfigUpdate() {
  ConfigConstSharedPtr new_config(
      new ConfigImpl(subscription_->route_config_proto_, factory_context_, false));
  tls_->runOnAllThreads(
      [this, new_config]() -> void { tls_->getTyped<ThreadLocalConfig>().config_ = new_config; });
}
```

### 总结

envoy的ADS更新，完全由master负责，使用grpc建立stream拉取配置，更新的顺序为:

CDS -> EDS -> LDS -> RDS

这些配置经过master的处理后，会用每个worker的dispatcher分发给所有的worker上的TLS

