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



### ServerSocketChannel

### DatagramChannel