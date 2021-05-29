## 组件

### Bootstrap
引导接口，存储用于启动程序的几个重要组件，包括EventLoopGroup、Channel、

#### Bootstrap
客户端启动引导程序

#### ServerBootstrap
服务端启动引导程序

### EventLoopGroup
线程组，Bootstrap只需要配置一个；ServerBootstrap需要配置两个，第一个用于处理服务端通道NioServerSocketChannel,第二个处理子通道，也就是每个客户端连接时产生的NioChannel。

### EventLoop
EventLoopGroup内的成员变量，EventLoopGroup存储一个EventLoop数组，默认是CPU核心数*2，也可实例化EventLoopGroup时指定。

### Channel

#### NioServerSocketChannel
NioServerSocketChannel是通过SelectorProvider.openServerSocketChannel()来打开服务器套接字通道，相当于调用java.nio.channels.ServerSocketChannel.open()。

#### NioSocketChannel
NioSocketChannel是通过SelectorProvider.openSocketChannel()来打开套接字通道，相当于调用java.nio.channels.SocketChannel.open()。


### Handler



### ChannelFuture

### ChannelFutureListener

### ChannelPromise

### ChannelHandler

### ChannelHandlerContext

### ChannelPipeline

### Buffer
