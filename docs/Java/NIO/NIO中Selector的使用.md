## Selector

Selector是选择器，也称多路复用器，使用Selector可以获取多个Channel的就绪状态，可读、可写、可连接等，这样通过单线程就可以管理多个Channel,也就是多个网络连接。减少线程上下文切换的系统开销。

### Selector使用方法

- 创建

```java
try (Selector selector = Selector.open()) {

} catch (IOException e) {
    e.printStackTrace();
}
```

- 注册Channel

```java

try (Selector selector = Selector.open()) {
    ServerSocketChannel socketChannel = ServerSocketChannel.open();
    socketChannel.bind(new InetSocketAddress(8080));
    socketChannel.configureBlocking(false);
    socketChannel.register(selector, SelectionKey.OP_ACCEPT);
} catch (IOException e) {
    e.printStackTrace();
}
```
调用channel的register方法可以将Channel注册到Selector中，第二个参数是要Selector监听的操作的就绪状态，这里是监听服务端通道的接受就绪状态。

可以监听的几种操作
- OP_READ: 可读的就绪状态
- OP_WRITE: 可写的就绪状态
- OP_CONNECT: 连接就绪
- OP_ACCEPT: 接受就绪

如果要注册多个操作，可以使用或运算符注册
```java
channel.register(selector, SelectionKey.OP_READ|SelectionKey.OP_WRITE);
```

> Selector只能注册非阻塞的通道，所以FileChannel是不能注册到Selector的。

- Selector选择就绪通道
向Selector注册多个通道后，可以调用select()方法返回就绪的通道。
    - int select(): 阻塞到至少有一个通道是就绪状态
    - int select(long timeout): 阻塞超时返回
    - int selectNow(): 不阻塞，返回就绪的通道数

```java
try (ServerSocketChannel socketChannel = ServerSocketChannel.open()) {
    socketChannel.bind(new InetSocketAddress(80));
    socketChannel.configureBlocking(false);
    //注册
    Selector selector = Selector.open();
    socketChannel.register(selector, SelectionKey.OP_ACCEPT);
    while (selector.isOpen() && selector.select() != 0) {
        //获取channel set
        Set<SelectionKey> selectionKeys = selector.selectedKeys();
        Iterator<SelectionKey> iterator = selectionKeys.iterator();
        while (iterator.hasNext()) {
            SelectionKey selectionKey = iterator.next();
            handler(selectionKey, selector);
            iterator.remove();
        }
    }
} catch (IOException e) {
    e.printStackTrace();
}
```
调用selectedKeys方法可以获取就绪状态的SelectionKey,通过SelectionKey可以获取对应的通道，处理完后要移除该SelectionKey,防止被重复处理。


```java
private static void handler(SelectionKey selectionKey, Selector selector) throws IOException {
    if (selectionKey.isAcceptable()) {
        ServerSocketChannel serverSocketChannel = (ServerSocketChannel) selectionKey.channel();
        SocketChannel socketChannel = serverSocketChannel.accept();
        socketChannel.configureBlocking(false);
        socketChannel.register(selector, SelectionKey.OP_READ);
    } else if (selectionKey.isReadable()) {
        //处理其它可读就绪的通道
    }
}
```
通过ServerSocketChannel可以接受到客户端连接产生的SocketChannel,将该通道注册到Selector中，让Selector检查其可读就绪状态。





