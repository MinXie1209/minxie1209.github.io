# MappedByteBuffer

在 RocketMQ 的 Broker 模块中,使用了顺序写,随机读的机制

那是怎样去设计的呢?

我们先来了解一下Java NIO 中操作文件的类 java.nio.MappedByteBuffer



## FileChannel

