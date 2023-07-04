# 探一下Netty的高性能

Netty 应用很广泛，像我们常用的 Dubbo、Zookeeper、RocketMQ 等中间件中都有它的身影。

那 Netty 到底有什么魔力吸引这么多开发者的选择？

今天就简单来探究一下

## 借由 Dubbo 对 Netty 的应用讲起

在 Dubbo 框架中，底层的网络通讯框架也选择了 Netty，其中比较值得探究的一部分是其关于线程模型的设计



Dubbo 的线程模型实现位于类包 **org.apache.dubbo.remoting.transport.dispatcher**

官网的文档介绍（文档有滞后性，Dubbo2.x 和 Dubbo3.x 线程模型有实现上的差别） [线程模型](https://cn.dubbo.apache.org/zh-cn/overview/mannual/java-sdk/advanced-features-and-usage/performance/threading-model/provider/)

本文的讲述的线程模型是基于 Dubbo3.x 版本

Dubbo 中有几个值得关注的线程池

- NettyServerBoss
- NettyServerWorker
- 自定义线程池



NettyServerBoss 用于监听客户端的连接请求事件，这里配置的是一个线程

NettyServerWorker 用于监听客户端连接后的IO读写事件 ，这里配置了系统的（CPU核心数+1）个线程

自定义线程池 根据配置的策略不同，作用也有一些差别



### 线程模型 -all

Dubbo 中默认配置的线程模型为"all"，实现类为 **org.apache.dubbo.remoting.transport.dispatcher.all.AllChannelHandler**

在这种模型下，**网络连接、连接断开、消息接受、异常处理**这几种类型的事件都使用 共享的线程池 进行处理

而 **消息发送** 交由 IO线程池 负责



### 线程模型 -connection

"connection" 类型，其实现类为 **org.apache.dubbo.remoting.transport.dispatcher.connection.ConnectionOrderedChannelHandler**

在这种模式下，**消息接受、异常处理**这两种类型的事件使用的是 共享线程池 进行处理

而 **网络连接、连接断开**这两种类型的事件使用的是一个 独立的线程池 进行处理，而这个线程池配置的线程数大小是1

**消息发送**则交由 IO线程池 负责

> 这种模型的好处在于，连接事件和业务处理进行线程资源隔离



### 线程模型 -direct

"direct" 类型，其实现类为 **org.apache.dubbo.remoting.transport.dispatcher.direct.DirectChannelHandler**

在这种模型下，仅当自定义线程池配置的类型是 ThreadlessExecutor，消息接受的事件才交由共享线程池进行处理

其它情况由IO线程进行处理



> ThreadlessExecutor



