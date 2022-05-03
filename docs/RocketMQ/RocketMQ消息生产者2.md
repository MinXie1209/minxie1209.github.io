# 生产者发送消息流程

这篇文章会从消息发送如果选择消息队列的角度切入研究消息发送的细节

主要分析包括

- **发送消息如何选择需要发送到哪个消息队列**
- **开启消息发送延迟故障的场景，怎样选择消息队列**



## 如何选择发送的消息队列

以同步方式发送消息为切入点，追踪其中实现的细节



### send(Message msg) 切入

![image-20220503142050966](https://tva1.sinaimg.cn/large/e6c9d24egy1h1v719xzhtj21w40pqte2.jpg)

先对 Message 的 topic 进行重新包装，如果生产者有设置命名空间，新的主题名称是 命名空间 + "%" + 主题名称

然后再调用 DefaultMQProducerImpl 的 send(Message msg) 方法



![image-20220502212700371](https://tva1.sinaimg.cn/large/e6c9d24egy1h1udqcde83j21r60fgacl.jpg)

![image-20220502213945138](https://tva1.sinaimg.cn/large/e6c9d24egy1h1ue3lw85rj21sc0cadid.jpg)

send(Message msg) 方法里再调用 带有超时参数的 send 方法，超时时间默认设置为 3 秒

然后再调用 **sendDefaultImpl** 方法，传入参数 同步发送方式 CommunicationMode.SYNC ，同步发送是不需要回调的，所以sendCallBack 不需要传参



### **sendDefaultImpl细节追踪**

sendDefaultImpl 是整个调用链路中比较底层的方法，不管是同步发送、异步发送、单向发送，最后都会调用到这个方法

![image-20220503142155355](https://tva1.sinaimg.cn/large/e6c9d24egy1h1v72e0axvj21wi0eaadf.jpg)

![image-20220502215224458](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uegrtx3hj21wa0fkgom.jpg)

第一步，调用**makeSureStateOk()** ，检验状态，判断生产者的状态是不是在运行中，不是则抛出异常



![image-20220502215407300](https://tva1.sinaimg.cn/large/e6c9d24egy1h1ueikogsbj21il0u0afx.jpg)

![image-20220502215835714](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uen7m6iyj21wk0b20va.jpg)

![image-20220502215901561](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uennp989j21wg06sab9.jpg)

![image-20220502215930573](https://tva1.sinaimg.cn/large/e6c9d24egy1h1ueo6rd8oj21wq0cejv4.jpg)

第二步，检验参数，调用**checkMsg(msg)** ，校验 Message 内各项参数是否合法

- 主题不能为空
- 主题字符长度不能大于 127
- 主题字符合法
- 消息体不能为空
- 消息体长度不能大于生产者设置的最大消息大小

- 不能将主题设置为系统内置的主题之一



![image-20220502231416149](https://tva1.sinaimg.cn/large/e6c9d24egy1h1ugtz5cfuj21ws0a0771.jpg)

![image-20220502231443544](https://tva1.sinaimg.cn/large/e6c9d24egy1h1ugufiy13j21w60r6wlf.jpg)

接着，调用 **tryToFindTopicPublishInfo** 方法获取主题的路由信息

如果主题信息是空，则调用 mQClientFactory.updateTopicRouteInfoFromNameServer(topic) 方法去获取

如果没有主题路由信息或者队列为空的

调用 updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer)  方法获取



![image-20220502232033148](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uh0i30mrj21ws0jg77y.jpg)

![image-20220502232515573](https://tva1.sinaimg.cn/large/e6c9d24egy1h1v6dell52j21wi0getbt.jpg)

**TopicPublishInfo** ，这个类存储了主题的相关信息

- orderTopic ： 是否是顺序的消息
- hasTopicRouterInfo ：是否有主题路由信息
- messageQueueList ：主题的消息队列
- sendWhichQueue ：每次发送消息该值会自增 1 ，用于选择要发送的消息队列
- topicRouteData ： 主题的路由信息

**TopicRouteData** ，存储了消息队列信息、Broker路由信息、过滤服务器地址等



![image-20220503105222747](https://tva1.sinaimg.cn/large/e6c9d24egy1h1v10cyfqlj21ci0u0gsr.jpg)

获取到主题对应的路由信息后，调用 **selectOneMessageQueue** 去选择一个需要发送的消息队列 MessageQueue 



![image-20220503105346619](https://tva1.sinaimg.cn/large/e6c9d24egy1h1v11swc6qj21vo08kgnk.jpg)

![image-20220503142257362](https://tva1.sinaimg.cn/large/e6c9d24egy1h1v73gmzmuj21w80k8427.jpg)

如果发送延迟故障开关（sendLatencyFaultEnable） 关闭，调用 TopicPublishInfo.selectOneMessageQueue(lastBrokerName) 方法



![image-20220503105846309](https://tva1.sinaimg.cn/large/e6c9d24egy1h1v170dz68j21w60rqjwl.jpg)

如果 lastBrokerName 是 null ，调用 selectOneMessageQueue() 方法

如果不是 null，则遍历消息队列，每一次遍历 sendWhichQueue 都自增1，然后找出一个 MessageQueue

如果这个 MessageQueue 的 brokerName 是和 lastBrokerName 相同的，则跳过该 MessageQueue

如果遍历整个消息队列列表都找不到一个合适的消息队列，则调用 selectOneMessageQueue() 方法

![image-20220503111132510](https://tva1.sinaimg.cn/large/e6c9d24egy1h1v1ka1i7mj21w20c4jtq.jpg)

selectOneMessageQueue 就是 sendWhichQueue 自增，然后对数组取模之后返回数组对于位置的消息队列



### 小总结

总结一下，在没有开启发送延迟故障开关的场景下，会从消息队列数组中选择一个 brokerName 不和上一次发送失败的 brokerName 相同的消息队列，如果数组中都是一样的 brokerName ，那最后就选择其中一个消息队列作为兜底策略



## 发送延迟故障处理

再来看看如果开启发送延迟故障的场景下是怎样选择消息队列的



### selectOneMessageQueue

![image-20220503142354955](https://tva1.sinaimg.cn/large/e6c9d24egy1h1v74geivpj21vy0lq79d.jpg)

同样的也是 sendWhichQueue 自增，然后遍历队列数组，但这里不是简单判断 brokerName 是否相同

而是调用方法 latencyFaultTolerance.isAvailable(mq.getBrokerName()) 去判断 broker 是否可用

broker 可用就返回该消息队列



![image-20220503112242883](https://tva1.sinaimg.cn/large/e6c9d24egy1h1v1vx013rj21wa0s2wki.jpg)

接着如果找不到合适的消息队列，往下走，调用 latencyFaultTolerance.pickOneAtLeast() ，从故障 Broker 列表中获取一个Broker，但如果列表为空，返回 null

然后获取该 Broker 的写队列数，如果写队列数大于 0 ，从 topic 中获取一个消息队列，并更新 Broker 为前面获取到的 Broker

否则将该 Broker 从 latencyFaultTolerance 中移除，在执行方法 selectOneMessageQueue 选择一个消息队列进行兜底发送



前面频繁调用到对象 latencyFaultTolerance 的一系列方法，追踪进行看看



### LatencyFaultTolerance

![image-20220503150052912](https://tva1.sinaimg.cn/large/e6c9d24egy1h1v86x1q0qj21wa0l8gos.jpg)

LatencyFaultTolerance 是一个接口，定义了几个接口方法

- updateFaultItem ：更新故障 Broker 信息，name 代表 BrokerName ，currentLatency 当前发送消息延迟（从发送到接受到结果的耗时），notAvailableDuration Broker 不可用时长
- isAvailable ：判断 Broker 是否可用
- remove ：移除故障 Broker 信息
- pickOneAtLeast ：从故障列表中获取一个 Broker



### LatencyFaultToleranceImpl

LatencyFaultToleranceImpl 是 LatencyFaultTolerance 的实现类，来看看各个实现方法的细节部分



![image-20220503151041290](https://tva1.sinaimg.cn/large/e6c9d24egy1h1v8h511o1j21w40tq0z9.jpg)

![image-20220503151313305](https://tva1.sinaimg.cn/large/e6c9d24egy1h1v8jrprtfj21vw0dqwgj.jpg)

**updateFaultItem** ，从 faultItemTable 中查找是否存储了相关Broker的故障信息

如果找到了，更新 FaultItem 的信息

如果没有，构建 FaultItem 对象，更新到 faultItemTable 中

FaultItem 的 name 是 BrokerName ，currentLatency 当前发送消息延迟，startTimestamp 等于（当前时间 + Broker 不可用时长），也就是说 startTimestamp 代表 Broker 恢复使用的时间点



![image-20220503151717968](https://tva1.sinaimg.cn/large/e6c9d24egy1h1v8o0el96j21vs0e0dhy.jpg)

![image-20220503151752242](https://tva1.sinaimg.cn/large/e6c9d24egy1h1v8olmwbgj21w0074my9.jpg)

**isAvailable** ，同样从 faultItemTable 中查找是否存储了相关Broker的故障信息

如果找不到，直接返回 true ，表示该 Broker 是可用的

如果找到了FaultItem ，调用 isAvailable ，判断 现在的时间是否大于 startTimestamp

如果大于，说明 Broker 已经恢复了



### updateFaultItem 调用

再回过头追踪一下代码在什么地方调用了 updateFaultItem 方法

最终追踪到方法 org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#sendDefaultImpl

![image-20220503152750065](https://tva1.sinaimg.cn/large/e6c9d24egy1h1v8yzpgl3j21ci0u0wn9.jpg)

![image-20220503193613479](https://tva1.sinaimg.cn/large/e6c9d24egy1h1vg5mzvclj21w60bu0vp.jpg)

每次发送完消息都会对 FaultItem 进行更新

但有几种异常处理传入参数 isolation 的值是 true ，包括 RemotingException 、MQClientException 、 MQBrokerException 

正常情况下和 InterruptedException 异常传入参数 isolation 的值是 false 

isolation 参数为 true 时，代表 Broker 需要被隔离，需要隔离 duration 值为 600 秒



![image-20220503194045414](https://tva1.sinaimg.cn/large/e6c9d24egy1h1vga5bpq1j21w20e8mza.jpg)

![image-20220503194427121](https://tva1.sinaimg.cn/large/e6c9d24egy1h1vge141pvj21v60amdi4.jpg)

computeNotAvailableDuration 计算规则

根据传入的 currentLatency 值去 latncyMax 数组中从后往前找到第一个比 currentLatency 的值小的

然后对应到 notAvailableDuration 数组的位置



像上面传入 currentLatency 值是 30000 ，从后往前找到第一个比 30000 大的就是 15000L 

对应的 notAvailableDuration 中的就是 600000L



### 小总结

发送延迟故障处理机制，每一次的消息发送都会根据发送接受的时延来维护 Broker 的故障时长，如果发送时延在 550 ms，则代表 Broker 是没有问题，等于 550 ms 或者大于 550 ms 都会代表该 Broker 是有故障的，故障时长的范围在 30 s 到 10分钟之间



## 总结

本文从最基本的以同步方式发送消息为切入点，追踪了

普通场景下如何选择主题消息队列，如何规避故障的 Broker 

以及在开启发送延迟故障开关的场景下怎么维护故障 Broker 的信息，最后怎样选择发送的信息队列







