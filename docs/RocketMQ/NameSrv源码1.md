# 简介

之前的文章中有介绍 RocketMQ 中的模块结构，感兴趣的同学可以点进去看一看 

 [RocketMQ源码架构说明](https://juejin.cn/post/7090923709468246030)

其中 NameSrv 组件算是这个系统中很重要的一环，简单画个图

![image-20220429202216898](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn9kmw6kj21bi0eg75x.jpg)

NameSrv 负责存储 生产者、消费者、存储组件 Broker 的路由信息

1. **想一下为什么会这么去设计？能不能让生产者、消费者、Broker存储各自的路由信息 然后进行通信？**

   单机下直接通讯是 OK 的，但分布式消息组件承载的数据量通常情况下是非常大的，难免需要进行集群部署

   

2. **那么 Producer 怎样感知 Broker 集群中有多少实例呢？**

   这时往往需要中间组件的介入，用于解耦系统

   通常在分布式系统中都会引入服务注册发现组件，业界中常用的组件有 Zookeeper 、Eureka 、Nacos 等
   
   

# 进入正题

为什么设计 RocketMQ 时不使用 Zookeeper 组件，而是重新开发一个组件 NameSrv 呢？接下来一起来研究一下 NameSrv 的源码吧

## NameSrv启动流程

启动入口类在 **org.apache.rocketmq.namesrv.NamesrvStartup**

### 构造 NamesrvController

1. **解析命令行参数**

   - p printConfigItem ：打印配置并退出程序
   - c configFile ： 指定配置文件路径
   - h help ：打印帮助并退出程序
   
2. **构造 NamesrvConfig**

   - 根据configFile解析
   - 根据命令行参数解析
   - 检查环境变量 **ROCKETMQ_HOME**
   

![image-20220429225917257](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn9jnivhj21ci0u0gqs.jpg)

>  NamesrvConfig成员变量
>
>  - rocketmqHome : 指定主目录路径
>
>  - kvConfigPath ： 指定存储配置属性的路径
>
>  - configStorePath ： 配置文件路径
>
>  - productEnvName ： 生产环境名 默认 center
>
>  - clusterTest : 集群测试 默认 false
>
>  - orderMessageEnable : 开启顺序消息 默认开启

  

3. **构造 NettyServerConfig**
   - 设置服务器暴露端口号为 **9876**
   - 根据配置文件、命令行参数来设置 NettyServerConfig 的成员变量
   

![image-20220430001726865](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn9ixwe8j21ci0u0tdj.jpg)


4. Logback 日志组件初始化

   指定日志配置位置：ROCKETMQ_HOME + "/conf/logback_namesrv.xml"

5. 调用构造方法

![image-20220429212555954](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tn9lhecxj21x60nyjv9.jpg)

> NettyServerConfig 的成员变量
> - listenPort ：监听端口，默认 **8888**
> - serverWorkerThreads  ： Netty业务线程数，默认 **8** 个
> - serverCallbackExecutorThreads  ：Netty 任务线程数，默认 **0** 个
> - serverSelectorThreads  ： IO线程数，解析网络协议，转发到业务线程进行处理，默认 **3** 个
> - serverOnewaySemaphoreValue  ： send one way 并发度，默认 **256**
> - serverAsyncSemaphoreValue ： 异步发送消息并发度，默认 **64**
> - serverChannelMaxIdleTimeSeconds ： 网络连接最大空闲时间，默认 **120** 秒
> - serverSocketSndBufSize ： 网络连接发送最大缓存数，默认为 **0**
> - serverSocketRcvBufSize ： 网络连接接受最大缓存数，默认为 **0**
> - writeBufferHighWaterMark ：写缓存最大的阈值 ，默认为 **0**
> - writeBufferLowWaterMark : 写缓存最小的阈值，默认为 **0**
> - serverSocketBacklog ：待接受连接的最大队列长度 ，默认为 **1024**
> - serverPooledByteBufAllocatorEnable ： 是否开启缓存
>
### 启动 NamesrvController

1. 调用 NamesrvController 对象的 initialize() 方法初始化

   - 加载 kvConfigManager

   - 构造 remotingServer ：Netty 服务端

   - 构造 remotingExecutor： Netty 线程池

   - registerProcessor ： 注册处理器到 Netty 服务端 

   - 注册定时任务 routeInfoManager::scanNotActiveBroker 每10秒一次

   - 注册定时任务 kvConfigManager::printAllPeriodically 每10分钟一次

2. 注册程序关闭的钩子

   在程序关闭前会触发注册的方法，可以在 shutdown 方法里释放网络连接、线程池资源等

   - Runtime.getRuntime().addShutdownHook

3. 启动 NamesrvController
   - remotingServer.start() 启动 Netty 服务器
   - 启动守护线程 fileWatchService