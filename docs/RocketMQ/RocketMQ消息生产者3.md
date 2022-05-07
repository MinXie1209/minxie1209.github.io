今晚放假了，继续进行 RocketMQ 的源码分析吧，可不能落下了

消息生产者前面已经输出了两篇文章，追踪了消息生产的大概流程以及发送消息时如何选择要发送的消息队列。

今天继续，研究一波消息生产的发送消息的细节



第二篇 [RocketMQ 源码分析 之 消息生产者（二）负载均衡](https://juejin.cn/post/7093479659035164703) 讲了方法 org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#sendDefaultImpl

在sendDefaultImpl 里面调用了 sendKernelImpl 方法去发送消息



## sendKernelImpl 分析

![image-20220505102801532](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xbjnshz5j22s00kkjww.jpg)

![image-20220505102910007](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xbksdm6oj22o20b0tad.jpg)

第一步，通过 BrokerName 去查找对对应的 Broker 地址

如果 Broker 地址不存在，则抛出异常信息



![image-20220505102945020](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xble137oj22pm06ujsv.jpg)

![image-20220505103119772](https://tva1.sinaimg.cn/large/e6c9d24ely1h1xbn1otb4j22r00jgdj8.jpg)

如果生产者配置发送的通道是 VIP 通道，则需要计算出 VIP 通道的地址

VIP 通道的地址的计算很简单，就是对 Broker 地址的端口号减 2 就是了



![image-20220505104114953](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xbxcs4jgj22p20ligpk.jpg)

接下来，开始设置消息的唯一 ID ，批量消息是不需要设置的

如果生产者设置有命名空间，则需要把实例 ID 的值设置和命名空间一样



![image-20220505104554853](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xc2823l6j22qi0jyn0w.jpg)

![image-20220505104757891](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xc4cjjl5j223z0u042q.jpg)

下一步，设置系统标志 sysFlag 

消息的系统标志可以标志 

- 是否压缩消息体
- 是否是多标签消息
- 是否是事务消息
- ...

具体可查看类 org.apache.rocketmq.common.sysflag.MessageSysFlag



![image-20220505104909866](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xc5m1vy6j21ly0u0gsh.jpg)

下一步，如果有设置一些钩子函数，则需要先执行



![image-20220505105204781](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xc8ne07jj21xn0u0aia.jpg)

然后封装生产消息的请求头，

请求头存储的消息包括

- 消息生产者组名
- 消息主题
- 默认的主题
- 默认的消息队列数
- 消息队列的 ID
- 消息的系统标志
- 消息生产时间
- 消息的附加属性
- 重投次数
- ...

 这里还有重投消息的处理



![image-20220507223448918](https://tva1.sinaimg.cn/large/e6c9d24egy1h207shjp8cj21l30u0gq4.jpg)

![image-20220507223517227](https://tva1.sinaimg.cn/large/e6c9d24egy1h207sxh1pvj21k10u0429.jpg)

最后，根据不同的消息发送方式，分别处理，同步发送方式会将发送结果赋值到 sendResult 变量中

单向发送、异步发送返回的 sendResult 在这里的值是 null



那么异步发送的结果是怎样处理的呢？

![image-20220507225656328](https://tva1.sinaimg.cn/large/e6c9d24egy1h208fgyrltj21cm0u0n2u.jpg)

![image-20220507225950610](https://tva1.sinaimg.cn/large/e6c9d24egy1h208ihyy7wj21cm0u0gtp.jpg)

在发送的时候生成一个 opaque , 是自增的数字，代表这个请求的唯一ID 

然后将 opaque 和 异步处理函数存储在 responseTable 里面



![image-20220507230054841](https://tva1.sinaimg.cn/large/e6c9d24egy1h208jls4anj21cm0u0453.jpg)

当 Broker 发送 response 过来的时候，通过 opaque 去responseTable里面去找到处理函数进行处理
