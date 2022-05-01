上一篇文章大概说了 NameSrv 启动的整个流程 [RocketMQ 源码分析 之 斗胆研究一波 NameSrv 源码 （一）](https://juejin.cn/post/7092065695373983774)

这一篇文章会更深入讲一些细节

# NamesrvController

在 NamesrvController 构造方法里，除了上一篇文章中讲到的 NamesrvConfig 、NettyServerConfig ，还有其它几个类也很重要

## RouteInfoManager

路由信息管理的类，管理 Topic 、Broker 等信息

RouteInfoManager 是数据核心管理类

![image-20220430102857920](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn91t0uxj223g0u078f.jpg)

-   **clusterAddrTable** : Broker 集群信息
-   **brokerLiveTable** : Broker 状态信息
-   **brokerAddrTable** : Broker 基础信息
-   **topicQueueTable** : 主题的队列信息
-   **filterServerTable** : Broker 中 FilterServer 列表

### 就 topicQueueTable 来分析一下

1.  **方法调用链分析**

![image-20220430122212071](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn8uq7poj21ny0qadln.jpg)

![image-20220430122251713](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn8y1thoj21ci0u0af7.jpg)

![image-20220430123855828](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn8z1k9bj21ci0u0win.jpg)

![image-20220430124055564](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn8yjktmj21ci0u0gqc.jpg)

往上定位到调用方法 org.apache.rocketmq.namesrv.processor.DefaultRequestProcessor#processRequest

这个方法会处理底层的网络请求

根据不同的请求 **code** 调用不同的处理方法

这个请求的 code 是**103**（注册 Broker 的请求）

![image-20220430124338617](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn8wtzdij21ci0u0wk0.jpg)

这里推荐安装协议解析插件，可以很直观地观察 RocketMQ 底层的请求数据

[「推荐」好用到爆的 RocketMQ 协议解析插件](https://juejin.cn/post/7091641992450408461)

2.  **深入 registerBroker 方法**

![image-20220430133502400](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn8zheglj21ci0u0wk5.jpg)

![image-20220430134110837](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn8v5eh8j21ny0qadln.jpg)

结合抓包插件解析请求数据

-   **clusterAddrTable** 变更

![image-20220430135533016](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn90hrpuj21ci0u0q74.jpg)

根据传入的 clusterName（集群名称，这里传入的是 "DefaultCluster" ）去 **clusterAddrTable** 中查找有没有对应的 value

没有则设置一个空的 HashSet

然后往 Set 中添加 BrokerName（ Broker 的名称，这里的值是 "broker-a" ）

从这里可以推断出 clusterAddrTable 存储的数据结构

HashMap 里面的 key 是集群的名称

value 是一个 Set ，存储 Broker 的名称

-   **接着往下看 brokerAddrTable 的操作**

![image-20220430141818551](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn92gp3hj21ci0u0dke.jpg)

根据传入的 brokerName 去 **brokerAddrTable** 中查找有没有对应的值

如果没有就添加

brokerAddrTable 存储的是每个 Broker 的路由信息

key 是 broker 的名称

value 是 BrokerData 结构信息

BrokerData 里面存储 集群名称、Broker 名称、brokerAddrs 路由信息

brokerAddrs key 是 brokerId（这里传入的值是 0 ） ，value 是 broker 的地址（这里传入的地址是 192.168.43.114:10911 ）

-   **topicQueueTable 操作**

![image-20220430165105396](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn90vknrj21ci0u0n16.jpg)

![image-20220430165122452](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn8vqh8tj21ci0u0aej.jpg)

先是通过数据版本号判断需不需要变动数据

**topicQueueTable** 存储 key 是 Topic 的名称，value 是一个 map

这个 map 的 key 是 broker 名称，value 是 QueueData 对象

QueueData 存储 brokerName（Broker名称）、readQueueNums （主题写队列数）、writeQueueNums （读队列数）、perm（权限）、topicSysFlag（标志）

![image-20220430170628276](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn91e0kkj21ci0u0787.jpg)

-   **brokerLiveTable操作**

![image-20220430163845193](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn8zxc5wj21ci0u0tcv.jpg)

再往下，这里会往 **brokerLiveTable** 中插入数据，key 是 Broker 的地址，value 是 BrokerLiveInfo 对象

BrokerLiveInfo 存储 lastUpdateTimestamp（最后更新时间戳）、dataVersion（版本号）、Channel（网络连接的通道）、haServerAddr（HA服务地址）

brokerLiveTable 存储的就是 Broker 对应的状态信息、网络通道

-   **filterServerTable操作**

![image-20220430164513368](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn8xppxkj21ci0u0dju.jpg)

再往下就是 **filterServerTable** 变量的操作了

如果没有传入 filterServerList 的大小是空的 ，则将 Broker 对应的 filterServerList 删除

filterServerList 不为空，则更新相应 Broker 的 filterServerList

filterServerTable 存储结构

key 是 Broker 地址

value 是过滤的服务地址列表



## 最后，总结一波

在源码分析的时候，像想要分析某个变量的含义，可以先从引用它的地方入手

从下而上，再从上而下，这么一回合，整条链路已经整得明明白白

> 还有，内容太多，下一节继续研究