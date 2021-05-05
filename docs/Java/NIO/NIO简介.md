## NIO

Java中NIO类库是由Java1.4引入的异步IO,是非阻塞IO。
NIO的核心包括Buffer、Channel、Selector。

> 与旧的IO类库比较

- 旧的IO阻塞，NIO是非阻塞的。
- 旧的IO是基于字节流\字符流进行操作的，NIO是基于Buffer。
- 旧的IO只能顺序读取字节流或字符流的数据，不能改变指针的位置。NIO可以随意读取Buffer中的数据。

### Channel

NIO中，一个连接一个通道（Channel）,所有的IO操作都是从Channel中开始的，一个Channel相当于BIO中一个输入流和一个输出流的结合。

### Buffer

Buffer是NIO非阻塞的基础和前提，使用Buffer可以将数据从Buffer读取到Channel,也可以从Channel写入Buffer。

### Selector

Selector选择器是一个IO事件查询器，将Channel注册到Selector中，Selectot可以不断地查询一个或多个Channel的就绪状态（可读、可写、网络连接完成等），通过Selector可以使用一个线程管理多个Channel。