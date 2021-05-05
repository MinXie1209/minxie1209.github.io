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
   
    private IntBuffer buffer;

    @Test()
    public void allocate() {
        buffer = IntBuffer.allocate(10);
        System.out.println("allocate:" + buffer);
    }
}
```

> 控制台输出：allocate:java.nio.HeapIntBuffer[pos=0 lim=10 cap=10]

> 调用IntBuffer的allocate方法会构造一个子类对象HeapIntBuffer : position=0 limit=10 capacity=10

- put(i): 往Buffer中写入数据

```java
public class BufferTest {
    
    private IntBuffer buffer;

    @Test()
    public void allocate() {
        buffer = IntBuffer.allocate(10);
        System.out.println("allocate:" + buffer);
    }

    @Test()
    public void put() {
        allocate();
        for (int i = 0; i < buffer.capacity() / 3; i++) {
            buffer.put(i);
            System.out.println("put:" + buffer);
        }

    }

}
```

> 控制台输出：<br/>
> put:java.nio.HeapIntBuffer[pos=1 lim=10 cap=10]<br/>
> put:java.nio.HeapIntBuffer[pos=2 lim=10 cap=10]<br/>
> put:java.nio.HeapIntBuffer[pos=3 lim=10 cap=10]<br/>

> 调用IntBuffer的put方法会写入数据，同时postion+1。

> 当position大于等于limit时，会抛出java.nio.BufferOverflowException异常，说明不能往Buffer写入数据了。

- flip(): 读写模式转换，当需要读取已经写入的数据时，需要调用flip()方法转换为读模式。
  
```java
public class BufferTest {

    private IntBuffer buffer;
    
    @Test()
    public void flip() {
        put();
        System.out.println("flip-before:" + buffer);
        buffer.flip();
        System.out.println("flip-after:" + buffer);
    }
}
```

> 控制台输出：<br />
> flip-before:java.nio.HeapIntBuffer[pos=3 lim=10 cap=10]<br />
> flip-after:java.nio.HeapIntBuffer[pos=0 lim=3 cap=10]<br />

> 切换模式时，将position赋值给limit,然后重置position和mark。
```java
    public final Buffer flip() {
        limit = position;
        position = 0;
        mark = -1;
        return this;
    }
```

- get(): 切换为读模式后就可以调用get()方法读取已经写入的数据了。

```java
public class BufferTest {
   
    private IntBuffer buffer;

    @Test()
    public void get() {
        flip();

        for (int i = buffer.position(); i < buffer.limit(); i++) {
            System.out.println(i + "-flip-before:" + buffer);
            System.out.println(buffer.get());
            System.out.println(i + "-flip-after:" + buffer);
            System.out.println("");
        }
    }
}
```
> 控制台输出：<br />
> 0-flip-before:java.nio.HeapIntBuffer[pos=0 lim=3 cap=10]<br />
> 0<br />
> 0-flip-after:java.nio.HeapIntBuffer[pos=1 lim=3 cap=10]<br />
> <br />
> 1-flip-before:java.nio.HeapIntBuffer[pos=1 lim=3 cap=10]<br />
> 1<br />
> 1-flip-after:java.nio.HeapIntBuffer[pos=2 lim=3 cap=10]<br />
> <br />
> 2-flip-before:java.nio.HeapIntBuffer[pos=2 lim=3 cap=10]<br />
> 2<br />
> 2-flip-after:java.nio.HeapIntBuffer[pos=3 lim=3 cap=10]<br />

> 可以看到，每执行一次get()方法，position都往后移动一步，也就是加1。

> 当position大于等于limit时，会抛出java.nio.BufferUnderflowException异常，说明不能从Buffer读取数据了。

- rewind(): 数据读完了，再调用get()方法就会抛出异常，如果想要重头再读一遍，则调用rewind()方法之后再调用get()方法去读取数据。

```java
public class BufferTest {

    private IntBuffer buffer;

    @Test()
    public void rewind() {
        get();
        System.out.println("rewind-before:" + buffer);
        buffer.rewind();
        System.out.println("rewind-after:" + buffer);
        
    }
}
```

> 控制台输出：<br />
> rewind-before:java.nio.HeapIntBuffer[pos=3 lim=3 cap=10]<br />
> rewind-after:java.nio.HeapIntBuffer[pos=0 lim=3 cap=10]<br />



- clear(): 重置成员变量

```java
    public final Buffer clear() {
        position = 0;
        limit = capacity;
        mark = -1;
        return this;
    }
```

> 在写模式下，已经写满数据了，调用clear()转换为读模式。

> 在读模式下，已经读完数据了，调用clear()转换为写模式。

- mark() reset(): mark()标记读取或写入的位置，reset()还原位置。

```java
public class BufferTest {
    
    private IntBuffer buffer;

    @Test()
    public void markAndReset() {
        flip();
        buffer.mark();
        System.out.println("mark:" + buffer);
        buffer.get();
        System.out.println("get:" + buffer);
        buffer.reset();
        System.out.println("reset:" + buffer);

    }
}
```

> 控制台输出：<br />
> mark:java.nio.HeapIntBuffer[pos=0 lim=3 cap=10]<br />
> get:java.nio.HeapIntBuffer[pos=1 lim=3 cap=10]<br />
> reset:java.nio.HeapIntBuffer[pos=0 lim=3 cap=10]<br />

> 可以看到调用mark()标记position的位置，然后读取数据后position的位置往后移动一位，最后调用reset()还原标记的position位置。
