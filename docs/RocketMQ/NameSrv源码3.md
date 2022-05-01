上一篇文章讲了 NameSrv 中很重要的部分，Broker、Topic 的路由信息管理，主要讲注册 Broker 的逻辑

[RocketMQ 源码分析 之 斗胆研究一波 NameSrv 源码 （二）](https://juejin.cn/post/7092322627586523167)

那这一篇来讲讲移除 Broker 的逻辑

在 org.apache.rocketmq.namesrv.NamesrvController#initialize 方法中启动了一个定时任务

scanNotActiveBroker，该任务10秒钟执行一次

![image-20220430220220314](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn7lx7qgj21ci0u0wif.jpg)

进入方法 scanNotActiveBroker 里面看看

![image-20220430220555773](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn7jdy80j21ci0u0dji.jpg)

scanNotActiveBroker 方法做的事情很简单，就是剔除不活跃的 Broker

那多久算是不活跃呢？**BROKER_CHANNEL_EXPIRED_TIME** 的值是 1000 * 60 * 2 ，也就是两分钟

brokerLiveTable 在上一章有介绍过，里面存储 key 是 Broker 的地址，value 存储的是 上一次更新时间（ lastUpdateTimestamp ）、版本号（ dataVersion ）、网络通道（ channel ）等

这里通过判断上一次更新时间是不是已经超过两分钟，如果是，取出网络通道进行关闭操作并移除该记录

![image-20220430220726324](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn7j0r9xj217e05mt93.jpg)

再来看看和Broker有关的数据结构，按理说，剔除 Broker 后，这些相关的信息都要清除掉

![image-20220430223306979](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/11a18d81989f4cd9879f31469cbae891~tplv-k3u1fbpfcp-zoom-1.image)

scanNotActiveBroker方法中已经将 **brokerLiveTable** 中 Broker 信息清除了

进入方法 onChannelDestroy

-   **移除 filterServerTable**

![image-20220430224227185](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn7kj1ojj21ci0u077h.jpg)

**filterServerTable** 和 **brokerLiveTable** 的 key 都是 brokerAddr，这里直接调用 remove 移除

-   **移除 brokerAddrTable**

![image-20220430224751480](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn7mfzvdj21ci0u0n15.jpg)

**brokerAddrTable** 的存储的 key 是 brokerName ，value 是 BrokerData 对象

BrokerData 对象里面的属性 brokerAddrs 存储的才是 broker 地址列表

所以先把要移除的地址从 brokerAddrs 中移除

然后如果移除之后这个 brokerAddrs 没有其它地址了，这个 BrokerData 就没有存在的意义了

那也把 BrokerData 移除掉

-   **移除clusterAddrTable**

![image-20220430225430592](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn7leqq5j21ci0u0wii.jpg)

**clusterAddrTable** 存储的 key 是clusterName，value 是 BrokerName

如果前面移除 brokerAddrTable 时有把 brokerName 也移除掉的

那么这里也要把 brokerName给移除掉

-   **移除topicQueueTable**

![image-20220430232338192](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn7k1pudj217e05mt93.jpg)

topicQueueTable 的 key 是 topic ，value 是一个 Map 结构

Map 的 key 是 brokerName ，value 是 QueueData 对象

如果前面有移除brokerName的，这里Map也要移除掉对应的 key

移除后如果 Map 的大小是 0 ，则 topicQueueTable 需要移除掉对应 topic 的数据