Broker 是 RocketMQ 中最核心的模块，也是最难啃的骨头

但是，没关系，单点突破，积少成多，一点点来攻破它

上一节中讲到了消息发送，那接着这个突破口，看看 Broker 中是怎样处理消息生产者发送过来的消息的



消息生产者 调用方法发送消息到 Broker ，方法 org.apache.rocketmq.remoting.netty.NettyRemotingAbstract#invokeSyncImpl

底层网络发送的数据全部封装到 RemotingCommand 类中



### RemotingCommand

![image-20220508111637881](https://tva1.sinaimg.cn/large/e6c9d24egy1h20tt3k87jj21v80h8q5s.jpg)

发送请求时调用 createRequestCommand 方法构造 RemotingCommand

属性 code 用来区别是什么类型的请求

像消息生产者生产消息的 code 就有很多种

- SEND_MESSAGE（10）:  发送消息 v1
- SEND_MESSAGE__V2（310）: 发送消息 v2
- SEND_BATCH_MESSAGE（320）: 发送批量消息
- ...    

![image-20220508112214106](https://tva1.sinaimg.cn/large/e6c9d24egy1h20tyxrzonj21u60pqwme.jpg)



### Broker 怎样处理发送消息的请求呢？



1. 注册处理器 

   方法 org.apache.rocketmq.broker.BrokerController#registerProcessor

   ![image-20220508114044620](https://tva1.sinaimg.cn/large/e6c9d24egy1h20ui6s3ypj21v00q27e6.jpg)

在这个方法中，对于生产消息的请求，都注册处理器 **SendMessageProcessor**



方法 org.apache.rocketmq.remoting.netty.NettyRemotingAbstract#processRequestCommand

![image-20220508114409688](https://tva1.sinaimg.cn/large/e6c9d24egy1h20ulrm6fkj21ee0u0wl0.jpg)

在 processsRequestCommand 方法中，会查找对应请求类型的处理器 

生产消息类的请求处理器都是 SendMessageProcessor



### SendMessageProcessor

![image-20220508201218700](https://tva1.sinaimg.cn/large/e6c9d24egy1h219aiudmaj21ug0gy773.jpg)

![image-20220508202031923](https://tva1.sinaimg.cn/large/e6c9d24egy1h219j1gvbsj21s00u0jyp.jpg)

继续追踪，调用方法 asyncSendMessage 异步处理请求



![image-20220508202304701](https://tva1.sinaimg.cn/large/e6c9d24egy1h219lob2j9j21um0de42u.jpg)

![image-20220508202338946](https://tva1.sinaimg.cn/large/e6c9d24egy1h219m9ktq3j21tu08yabo.jpg)

![image-20220508202407492](https://tva1.sinaimg.cn/large/e6c9d24egy1h219ms8ebjj21te07kab9.jpg)

追踪 asyncSendMessage ，可以发现方法调用层级还是挺深的，最后定位到方法 asyncPutMessages

org.apache.rocketmq.store.DefaultMessageStore#asyncPutMessages



### asyncPutMessages

![image-20220508202505459](https://tva1.sinaimg.cn/large/e6c9d24egy1h219pdr8rhj21960u0jxh.jpg)

asyncPutMessages 方法先判断存储组件是否可以存储数据



![image-20220508203311389](https://tva1.sinaimg.cn/large/e6c9d24egy1h219w77m35j217u0u0q7y.jpg)

不可以存储消息的几种情况：

- shutdown 状态
- Broker 是从机
- 磁盘写满了
- 操作系统页缓存繁忙



![image-20220508203821556](https://tva1.sinaimg.cn/large/e6c9d24egy1h21a1lj7fjj21te06egmx.jpg)

![image-20220508204229600](https://tva1.sinaimg.cn/large/e6c9d24egy1h21a5vk8ucj20y90u079s.jpg)

追踪方法 org.apache.rocketmq.store.CommitLog#asyncPutMessages 

在这个方法里，找到要写入的 MappedFile

再调用MappedFile 的 appendMessages 写入消息数据



![image-20220508204501859](https://tva1.sinaimg.cn/large/e6c9d24egy1h21a8im3n8j216c0u0wkt.jpg)

appendMessages 调用方法appendMessagesInner

先获取MappedFile的写指针

如果写指针大于等于文件大小，返回异常信息



如果写指针小于文件大小，通过 slice 方法获取一个原 ByteBuffer 共享的内存区域

但是新的 ByteBuffer 维护自己的 position、limit、capacity 属性

这里设置 position 为当前写指针位置

然后调用doAppend方法



### doAppend

![image-20220508224046823](https://tva1.sinaimg.cn/large/e6c9d24egy1h21dkyi9svj20w80u0q7j.jpg)

在 doAppend 方法中，调用 byteBuffer.put(preEncodeBuffer) 将消息数据写入内存中