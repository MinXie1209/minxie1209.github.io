## NIO-Buffer

Buffer是java.nio包下的一个抽象类，用作缓冲区。Buffer是一块可以写入数据，也可以读取数据的内存空间。

### 成员变量

Buffer在内部使用相应数组存储数据，使用不同的成员变量来标记数据读取、写入的位置等。
- mark: 保存position,调用mark()时将postion位置写入mark变量中。
- position: 当前数据位置（读取或写入的数组下标）。
- limit: 写入或读取的最大下标。
- capacity: 底层存储数据的数组最大容量，一旦指定就不能再更改，使用静态方法allocate()分配内存会根据capacity的大小分配相应大小的内存空间。

> mark <= position <= limit <= capacity

### 子类

根据不同的基本数据类型，Buffer有相应的子类对应，不同子类实际存储数据的数组类型不同。

![Buffer类图](/images/Buffer.png)

### Buffer中的方法

- allocate(capacity): 分配指定容量的Buffer，以IntBuffer为例：
```java
public class BufferTest {

    @Test()
    public IntBuffer allocate() {
        IntBuffer intBuffer = IntBuffer.allocate(10);
        System.out.println(intBuffer);
        return intBuffer;
    }
}
```

> 控制台输出：java.nio.HeapIntBuffer[pos=0 lim=10 cap=10]

> 调用IntBuffer的allocate方法会构造一个子类对象HeapIntBuffer : position=0 limit=10 capacity=10

- put(i): 往Buffer中写入数据

```java
public class BufferTest {

    @Test()
    public void put() {
        IntBuffer buffer = allocate();

        for (int i = 0; i < 3; i++) {
            buffer.put(i);
            System.out.println(buffer);
        }

    }

}
```

> 控制台输出：<br/>
> java.nio.HeapIntBuffer[pos=1 lim=10 cap=10]<br/>
> java.nio.HeapIntBuffer[pos=2 lim=10 cap=10]<br/>
> java.nio.HeapIntBuffer[pos=3 lim=10 cap=10]<br/>

> 调用IntBuffer的put方法会写入数据，同时postion+1。

> 当position大于等于limit时，会抛出java.nio.BufferOverflowException异常，说明不能往Buffer写入数据了。

- 