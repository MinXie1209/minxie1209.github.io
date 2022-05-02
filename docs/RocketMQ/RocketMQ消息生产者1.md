# RocketMQ 生产者

RocketMQ 生产者发送消息的方式有：同步发送、异步发送、单向发送。

-   同步：Producer 向 Broker 发送消息，等待 Broker 返回结果
-   异步：Producer 向 Broker 发送消息，指定回调函数，处理 Broker 返回的结果
-   单向：Producer 向 Broker 发送消息，不关注结果



## Message

先来看看生产者发送的消息的类结构：org.apache.rocketmq.common.message.Message

![image-20220502083239456](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uah2vx26j21ny0k642g.jpg)

Message 类内拥有成员变量如下：

-   body ：消息体
-   transactionId ：事务 id
-   properties ：消息的拓展属性
-   topic ：消息所属主题
-   flag ：消息的标志

跟踪 properties 引用的地方

![image-20220502083607038](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uah3fhumj21ny0k642g.jpg)

可以看到 properties 存储了很多属性

-   PROPERTY_KEYS（"KEYS"）：消息索引键，多个 key 用逗号分隔，RocketMQ 可以通过 key 快速查找消息
-   PROPERTY_TAGS（"TAGS"）：消息标签，在发送消息的时候可以给消息设置标签，消费端可以订阅指定的标签消费
-   PROPERTY_DELAY_TIME_LEVEL（"DELAY"）：消息延迟级别，用于发送延迟消息
-   PROPERTY_WAIT_STORE_MSG_OK（"WAIT"）：是否等待消息存储完成再返回
-   ...

## DefaultMQProducer

消息发送的入口类 ： org.apache.rocketmq.client.producer.DefaultMQProducer

这个类封装了各种发送消息的方法，包括发送同步消息、异步消息等等

![image-20220502085158624](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uah5q5y5j21ci0u042s.jpg)

### 查看 DefaultMQProducer 的继承关系

![image-20220502090004603](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uah4th04j21zw0q4acc.jpg)

DefaultMQProducer 继承类 ClientConfig ，并实现了 MQProducer 接口

-   ClientConfig ：底层网络连接的配置类

-   MQAdmin ：定义了很多方法如创建主题、查看消息等方法

    -   void **createTopic**(final String key, final String newTopic, final int queueNum) ：创建主题
    -   void **createTopic**(String key, String newTopic, int queueNum, int topicSysFlag) ：创建主题
    -   MessageExt **viewMessage**(final String offsetMsgId) ：通过msgId查询消息
    -   MessageExt **viewMessage**(String topic, String msgId) ：通过主题和消息id来查询消息
    -   long **maxOffset**(final MessageQueue mq) ：查询消息队列最大的偏移量
    -   long **minOffset**(final MessageQueue mq) ：查询消息队列最小的偏移量
    -   QueryResult **queryMessage**(final String topic, final String key, final int maxNum, final long begin, final long end) ： 查询一段时间内的消息
    -   long **searchOffset**(final MessageQueue mq, final long timestamp) ：查询某个时间（时间戳，毫秒）的消息队列偏移量
    -   long **earliestMsgStoreTime**(final MessageQueue mq) ：查询消息队列最早的时间（毫秒）

![image-20220502090503405](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uah351r2j21ny0k642g.jpg)

-   MQProducer ：消息生产者接口，定义发送消息等方法

    -   void **send**(final Collection<Message> msgs, final MessageQueue mq, final long timeout) ：批量发送同步消息，超时抛出异常
    -   void **sendOneway**(final Message msg, final MessageQueueSelector selector, final Object arg) ：发送单向消息，指定消息队列选择算法 ，覆盖默认算法
    -   void **sendOneway**(final Message msg) ：发送单向消息，使用默认负载均衡算法
    -   void **request**(final Message msg, final MessageQueue mq, final RequestCallback requestCallback, long timeout) ：异步发送消息，该方法立即返回，收到 Broker 的回复消息后，执行 requestCallback
    -   ...

![image-20220502131827527](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uah8l2pjj21ci0u078l.jpg)

### DefaultMQProducer核心属性

![image-20220502153042379](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uah9f7ozj21ci0u0788.jpg)

-   Set<Integer> **retryResponseCodes** ：重试响应码

    -   ResponseCode.TOPIC_NOT_EXIST（17）：主题不存在
    -   ResponseCode.SERVICE_NOT_AVAILABLE（14）：服务不可用
    -   ResponseCode.SYSTEM_ERROR（1）：系统错误
    -   ResponseCode.NO_PERMISSION（16）：没有权限
    -   ResponseCode.NO_BUYER_ID（204）：没有BUYER ID
    -   ResponseCode.NOT_IN_CURRENT_UNIT（205）：不在当前单位

-   String **createTopicKey** ：默认 Topic（"TBW102"）

-   int **compressMsgBodyOverHowmuch** ：压缩消息体的阈值，默认 4K 大小

-   int **maxMessageSize** ：允许发送的最大消息大小，默认 4M

-   int **retryTimesWhenSendFailed** ：默认 2 ，同步模式下发送失败重试的次数，可能会导致消息重复，需要应用开发者自己处理

-   int **retryTimesWhenSendAsyncFailed** ：默认 2 ，异步模式下发送失败重试次数，同上

-   boolean **retryAnotherBrokerWhenNotStoreOK** ：默认 false ，在**同步**发送消息失败时，是否重试发送到另一个 Broker

-   ...

![image-20220502165801994](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uah42futj222c0puwi4.jpg)

### 消息生产者启动流程追踪

看个使用生产者的例子，在发送消息之前会先实例化 DefaultMQProducer 对象，然后设置一些属性如 NameSrv 的服务地址、压缩消息体的阈值等等

在设置完这些属性后，会调用 start() 方法启动消息生产者

![image-20220502172414737](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uah9wl0zj21ci0u0td6.jpg)

DefaultMQProducer 内部包装了实现类 DefaultMQProducerImpl，所有操作都是直接调用这个对象的方法

![image-20220502173342637](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uahbbjhsj21ci0u0gpk.jpg)

![image-20220502173423194](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uahbrqjjj21ci0u00wc.jpg)

**defaultMQProducerImpl** 是在 DefaultMQProducer 构造方法时实例化的，类型是 DefaultMQProducerImpl

所以我们先追踪子类 **DefaultMQProducerImpl** 的 start() 方法

![image-20220502172930352](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uahadipmj21ci0u0gps.jpg)

**start** 方法

-   **检查参配置是否合法** ：producerGroup

![image-20220502185951965](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uah6qelhj21u00dy0u7.jpg)

-   **修改实例名称** ：PID + "#" + System.nanoTime()

![image-20220502190219188](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uahdo903j21we08egmn.jpg)

-   创建或者获取 MQClientInstance ：底层网络通讯的类

    通过（IP+实例名称+UnitName）来区别获取不同的 MQClientInstance

所以哪怕是相同的进程中，因为实例名称是PID+时间戳来生成的，只要是不是同一时刻生成的实例

获取的 MQClientInstance 不会是同一个

-   将生产者注册到 MQClientInstance 中进行管理

![image-20220502192818629](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uahcvqraj225o0fqtbv.jpg)

-   加入当前生产者的生产主题到 topicPublishInfoTable 中 ： topicPublishInfoTable 缓存主题发布的路由信息
-   启动 MQClientInstance 实例

![image-20220502192858509](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uah5bxbhj221y0hago6.jpg)

![image-20220502192642649](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uahatyx0j21lm068dgk.jpg)

![image-20220502192708884](https://tva1.sinaimg.cn/large/e6c9d24egy1h1uahc8nnqj21ge0kkdhu.jpg)

### 总结
本章讲了消息生产者的属性参数、消息体的结构、生产者启动的大概流程，下一章会讲消息发送的过程
