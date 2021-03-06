
xDS API协议定义中，除了各种资源的DS接口，还定义了ADS，Aggregated Discovery Service，即聚合发现服务。通过ADS，可以获取服务的多种信息，
而Pilot也是通过ADS接口为数据面提供信息的。

* 问题：没办法保证配置资源更新顺序

1. 在生产中需要部署多个Pilot来保证高可用，多个连接可以能会连接到多个Pilot，这样就没有办法保证配置资源的更新，比如LDS连接到Pilot1 ，
CDS连接到Pilot2，EDS连接到Pilot3，那么不同资源的动态获取资源的顺序无法保证的。有办法保证也需要耗费很大的代价。

2. 在分布式系统中多实例强一致性比较难保证。Pilot也没有提供这种机制，Pilot是一种无状态的应用，如果要保证强一致性必须要通过有状态的方式去实现，
多个Pilot还需要定义一种新的协议，需要同步Pilot节点之间的状态。Pilot实例之间的状态，增加Pilot很大的负担。

综合上面两个问题，就很容易出现配置更新过程中网络流量丢失带来网络错误（虚假的）404、503 。没办法保证配置更新过程，
没办法保证配置加载的过程就很容易出现404、503 是很常见的。404、503 并不是真正的错误，他是服务扩容和缩蓉带来的临时性的网络错误。

**ADS 允许单一管理服务器通过单个 gRPC 流**，提供所有的 API 更新。配合仔细规划的更新顺序，ADS 可规避更新过程中流量丢失。
使用 ADS，在单个流上可通过类型 URL 来进行复用多个独立的 DiscoveryRequest/DiscoveryResponse 序列。
对于任何给定类型的 URL，以上 DiscoveryRequest 和 DiscoveryResponse 消息序列都适用。

* 为什么ADS可以解决这种问题
1. ADS允许通过一条连接（gRPC的同一stream）,发送多种资源的请求和响应。
2. 能够保证请求一定落在同一Pilot上，解决多个管理服务器配置不一致的问题。
3. 通过顺序的配置分发，轻松解决资源更新顺序的问题。

> 结论： ADS主要用来解决多个Pilot情况下配置更新顺序的问题；即使使用ADS接口，还是要按照更新顺序（最后是RDS）来发送xDS。

