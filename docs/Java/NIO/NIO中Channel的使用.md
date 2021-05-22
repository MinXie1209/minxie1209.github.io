## Channel

### 通道特点：
- 通道可以读也可以写
- 通道可以异步读写
- 通道基于缓存区Buffer进行读写

### 类型：
- FileChannel：文件读写
- SocketChannel：TCP/IP 数据读写
- ServerSocketChannel：监听TCP连接，每个请求会创建一个SocketChannel
- DatagramChannel：UDP数据读写

### FileChannel

FileChannel是操作文件的通道，可以通过FileChannel读取文件数据，也可以写入数据。FileChannel不能设置为非阻塞。

- 打开文件

```java
FileChannel fileChannel = new RandomAccessFile("test.txt", "rw").getChannel();
```

- 读取文件


```java
ByteBuffer buffer = ByteBuffer.allocate(1024);
fileChannel.read(buffer);
```

- 写入文件


```java
String input = "this is a file";
ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
byteBuffer.put(input.getBytes());
byteBuffer.flip();
while (byteBuffer.hasRemaining()) {
    fileChannel.write(byteBuffer);
}
```

- 关闭通道

```java
fileChannel.close();
```

- 强制刷新

```java
fileChannel.force();
```

> 出于性能原因，调用write方法不一定将通道的数据写入磁盘，操作系统可能将数据缓存到内存中，force()方法可以强制将通道的数据刷新写入到磁盘中。


### SocketChannel

SocketChannel用于基于TCP通信的通道。

- 创建SocketChannel
```java
SocketChannel socketChannel=SocketChannel.open();
socketChannel.connect(new InetSocketAddress("www.baidu.com",80));
```

- 连接校验

```java
socketChannel.isOpen();//通道是否打开
socketChannel.isConnected();//通道是否已经进行TCP连接
socketChannel.isConnectionPending();//通道是否正在进行连接
socketChannel.finishConnect();//通道是否完成连接
```

- 读写模式
SocketChannel有阻塞和非阻塞模式，通过configureBlocking方法进行配置

```java
try (SocketChannel socketChannel = SocketChannel.open()) {
    socketChannel.connect(new InetSocketAddress("www.baidu.com", 80));
    while (!socketChannel.finishConnect()) {
        System.out.println("等待连接。。。");
        //等待2s
        Thread.sleep(2000);
    }
    ByteBuffer byteBuffer = ByteBuffer.allocate(16);
    socketChannel.read(byteBuffer);
    System.out.println("test end!");
} catch (IOException e) {
    e.printStackTrace();
}
```
> 阻塞模式下，调用read方法会阻塞程序，直到接受到数据。

```java
try (SocketChannel socketChannel = SocketChannel.open()) {
    socketChannel.connect(new InetSocketAddress("www.baidu.com", 80));
    socketChannel.configureBlocking(false);//设置为非阻塞模式
    while (!socketChannel.finishConnect()) {
        System.out.println("等待连接。。。");
        //等待2s
        Thread.sleep(2000);
    }
    ByteBuffer byteBuffer = ByteBuffer.allocate(16);
    socketChannel.read(byteBuffer);
    System.out.println("程序执行结束!");
} catch (IOException e) {
    e.printStackTrace();
}
```
> 非阻塞模式下，会直接向下运行，程序打印“程序执行结束!”

### ServerSocketChannel

ServerSocketChannel通过监听TCP连接来生成SocketChannel对象。

- open()：打开ServerSocketChannel通道
- socket().bind():绑定端口号
- accept(): 阻塞，当有连接建立时，返回连接SocketChannel通道。
> 如果设置通道为非阻塞，调用accept方法时，如果没有连接到来，则返回null。

```java

try (ServerSocketChannel socketChannel = ServerSocketChannel.open()) {
    socketChannel.socket().bind(new InetSocketAddress(9090));
    while (true) {
        System.out.println("等待连接。。。");
        SocketChannel accept = socketChannel.accept();
        new Thread(() -> {
            System.out.println("监听服务端发送的数据");
            ByteBuffer in = ByteBuffer.allocate(1024);
            int read;
            try {
                while ((read = accept.read(in)) != -1) {
                    if (read == 0) {
                        Thread.sleep(1);
                        continue;
                    }
                    in.flip();
                    while (in.hasRemaining()) {
                        System.out.print((char) in.get());
                    }
                    in.clear();
                    send(accept, "i accept your msg,i am server");
                        }
            } catch (IOException | InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
        send(accept, "success");
    }
} catch (IOException e) {
    e.printStackTrace();
}
//把数据写入到SocketChannel通道
private static void send(SocketChannel accept, String msg) {
    ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
    byteBuffer.put(msg.getBytes());
    byteBuffer.flip();
    while (byteBuffer.hasRemaining()) {
        try {
            accept.write(byteBuffer);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

}
```

### DatagramChannel

DatagramChannel用来处理UDP请求。

- 打开连接

```java
try (DatagramChannel datagramChannel = DatagramChannel.open()) {
    //绑定端口号
    datagramChannel.bind(new InetSocketAddress(8080));
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    while (true) {
        System.out.println("wait");
        datagramChannel.receive(buffer);
        System.out.println("rec:" + buffer);
        buffer.flip();
        while (buffer.hasRemaining()) {
            System.out.print((char) buffer.get());
        }
        buffer.clear();
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

- 发送数据

```java
try (DatagramChannel datagramChannel = DatagramChannel.open()) {
    String msg = "UDP packet send from client";
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    buffer.put(msg.getBytes());
    buffer.flip();
    datagramChannel.send(buffer, new InetSocketAddress("127.0.0.1", 8080));
} catch (IOException e) {
    e.printStackTrace();
}
```

> DatagramChannel不分客户端服务端（UDP不是面向连接的），通过open打开通道后就可以接受和发送数据了。

> 不过要接受数据，需要先通过bind方法绑定端口号，别的通道才能往这个端口发送数据。

> 通过receive()方法可以阻塞接受数据。通过send方法可以发送数据。