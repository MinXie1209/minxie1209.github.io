# 基于 Javassist 实现 Java 动态代理



之前写过一篇 [基于 ASM 实现 Java 动态代理](https://juejin.cn/post/7043401155753279524) 的文章,这篇文章使用 Javassist 实现同样的效果

「本文共 2025 字，预计阅读需要 5 分钟」

阅读完本文你能 get 到的知识点

- 什么是 Javassist
- JDK 动态代理
- 使用 Javassist 实现和 JDK 一样的效果

## 什么是Javassist

很多同学估计会对这个词有点陌生,但随着你关注的博主越来越多,知道的也越来越多,马上这篇文章就带你走进 Javassist 的世界

Javassist 和 ASM 一样是操作字节码的框架, Javassist 诞生于 1999 年,多少有点年头

使用 Javassist 可以在运行时定义一个新类,可以在 JVM 加载类文件时修改类文件

而且 Javassist 提供不同类型的API: **源码级别** 和 **字节码级别** 

本文使用源码级别的 API ,所以你甚至可以在不懂字节码的前提下使用它,入手相对简单

但在性能上略逊于 ASM

## JDK 动态代理

话不多说,先来回顾一下我们平时是怎么使用 JDK 动态代理的

### Proxy

JDK 提供一个类 Proxy 用于生成代理类

调用类方法 newProxyInstance,传入的参数

- ClassLoader loader : 定义代理类的类加载器
- Class<?>[] interfaces : 代理类需要实现的所有接口
- InvocationHandler h : 调用处理程序,方法调用都会分派到这里

![image-20221120153820639](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bmttrgp5j31z4066dgt.jpg)

### InvocationHandler

InvocationHandler 是一个接口类,定义了调用方法

- Object proxy : 生成的代理对象
- Method method : 接口方法实例
- Object[] args : 方法调用中传递的参数值

![image-20221120153844889](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bmu8pvscj31z4066aan.jpg)



## 如何调用

调用 Proxy.newProxyInstance 生成代理对象, 传入参数接口InvocationHandler实现类的对象处理代理的逻辑

![image-20221120153916843](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bmussdu0j31z407875h.jpg)





## 代码设计

在动手写代码之前,我们先花几分钟在脑海中设想一下我们需要生成的代理类是什么样子的?

这里先揭晓了

假设我们定义了一个接口类 LoginService

- 定义的接口类

![image-20221120153938033](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bmv64jzuj31z408874w.jpg)



那么我们需要生成一个大概是这样的代理类

- 首先必须得实现了定义的接口 LoginService
- 接口的所有方法实现都调用都代理到 InvocationHandler 中

![image-20221120153957614](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bmvijicrj31z40hk42t.jpg)



### 几个重要的类

从上面生成的代理类入手,我们生成的类

继承了父类 **MObject** 

**实现**了需要代理的**接口** LoginService

生成了**类成员变量** LoginService_0, 这个对应 接口的定义的方法

**实现**了需要代理的**接口方法** login

还有带参数 **MInvocationHandler** 的构造方法



### MObject

MObject 是生成的代理类需要继承的父类,它的作用是存储了 MInvocationHandler(处理程序接口)

![image-20221120154024984](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bmvzldb4j31z40ce0ts.jpg)



### MInvocationHandler

同 JDK 自带的接口 InvocationHandler ,用于实现代理方法的处理逻辑

![image-20221120154042266](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bmwac8ojj31z4088aaw.jpg)



### MProxy

同 JDK 自带的类 Proxy

提供生成代理对象的方法 newProxyInstance

![image-20221120154100479](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bmwlly6uj31z40ac3zj.jpg)



## 编码开始

在经过代码设计之后,我们的脑海里应该有思路了,那就开始动手了



整个过程中比较重要的部分应该就是 MProxy 类了

在 MProxy 里面我们需要实现两大功能:

- 生成代理类字节码
- 根据字节码生成对象

### 生成代理类字节码

1. 生成代理类名称
2. 生成空类
3. 给类设置需要实现的接口
4. 添加类成员变量
5. 实现接口方法

![image-20221120161826421](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bnzk7rbij31z40tydkj.jpg)


####	生成代理类名称

这一步相对简单,为了防止生成的代理类重名

这里拼接了所有需要代理的接口全限定类名,通过字符串 "_" 连接 

![image-20221120154119865](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bmwxm6glj31z40acmyu.jpg)



#### 生成空类

首先我们需要根据新的类名生成一个空的类,注意类名不要重复了,不然会污染了原有的类

![image-20221120161950788](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bo102q24j31z4066aap.jpg)


- ClassPool : 类池,存储所有类的信息,会将类名->类信息 存储到 HashTable 里, 可通过 ClassPool.getDefault() 获取实例
- CtClass: 代表一个类
- classPool.makeClass : 生成新的类

#### 给类设置需要实现的接口

这里同样通过 ClassPool 的 get 方法获取到所有传入接口的 CtClass 定义

再调用 setInterfaces 方法给生成的类设置多个接口

![image-20221120162137951](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bo2v7jx3j31z40egmzx.jpg)

#### 添加类成员变量

因为我们调用 MInvocationHandler 的 invoke 方法时需要传入的第二个参数是被代理方法的 Method实例

所以将这个方法的存储到类成员变量中

![image-20221120162951723](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bobg4brwj31z40pujzb.jpg)

CtField 代表着一个变量

传入类型、变量名 生成一个 CtField 实例

通过 setModifiers 方法设置变量的修饰符为 static + private



因为这里还要设置变量的值

调用 getFieldInitCode 生成初始化代码

![image-20221120163720117](https://tva1.sinaimg.cn/large/008vxvgGgy1h8boj8e25cj31z40qwwl2.jpg)

为了获取 Method 对象 ,这里生成了反射的代码去获取

实例: Class.forName(类名).getMethod(方法名, Class<?>... 方法的参数类型);



生成了类成员变量之后,接下来该到实现接口的方法了

#### 实现接口方法



![image-20221120164543838](https://tva1.sinaimg.cn/large/008vxvgGgy1h8boryl8fij31z40lotep.jpg)

![image-20221120170151450](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bp8qbwhij31z40kogqi.jpg)

这里需要实现接口的方法

可以通过 CtNewMethod.copy 方法去拷贝需要实现的方法,不要直接使用原来的 CtMethod , 防止污染

拿到新的 CtMethod ,我们需要设置它的方法体、设置修饰符为 public

重点来看看怎么生成方法体代码

这里根据方法返回的类型调用不同的方法

- void : 没有返回值的 调用 getMethodBodyCodeByVoid
- 基本数据类型 : 调用 getMethodBodyCodeByPrimitive
- 其它类型 : 调用 getMethodBodyCode



那按顺序来看,不需要返回值的

那我们需要生成的代码是长这样的

***super.h.invoke(this, 对应的类成员变量, new Object[]{方法参数});***

这个有个语法需要知道: \$0 代表这方法的第一个参数,懂字节码的应该知道非构造方法的第一个入参是 一个隐式的 this ,指向对象本身

new Object[]{} 里面就可以用 \$1 \$2 代表着方法的参数了


![image-20221120170820287](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bpfhefl5j31z40os0xt.jpg)



返回值是基本数据类型的,需要调用调用包装类型对应的拆箱方法 如 

***Boolean.parseBoolean()***

所以和上面生成步骤的区别在于 前后生成了对于基本类型的 parse 代码

![image-20221120171916747](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bpqvedh0j31ec0u0af5.jpg)



最后的返回其它类型的也比较简单

直接生成强转的代码 如 ***(String)***

![image-20221120172139919](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bptc19bvj31z40egjtt.jpg)



以上步骤走完,前期准备工作算是做完了,接下来就要根据生成的字节码来实例化对象了

### 根据字节码生成对象

要根据字节码来生成对象,第一步我们需要编写自定义的类加载器,通过类加载器加载字节码

1. 编写自定义加载器 MClassLoader ,继承类 ClassLoader
2. 提供 add 方法将类名映射到对应的字节数组
3. 重写 ClassLoader 类的 findClass 方法,使用我们生成的字节数组生成类

![image-20221120154143336](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bmxcpjy2j31z40mqjv4.jpg)



在 MProxy 中调用 MClassLoader 加载并实例化对象

1. 加载类 mClassLoader.loadClass(clasName)
2. 获取带 MInvocationHandler 参数类型的构造
3. 实例化对象 constructor.newInstance(h)

![image-20221120154200270](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bmxmwogej31z40kogqb.jpg)



## 效果演示

好了,上面的代码已经编写完了,那么现在就来对比一下 JDK 自带的 Proxy 和我们自己实现的 Proxy 的效果



### 我们定义一个需要代理的接口 LoginService

这里按照 Java 的基本数据类型 以及它们对应的包装类 定义了16个接口方法

![image-20221120154224566](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bmy1ztavj31gw0u0n24.jpg)



### 分别实现了代理类 CusMInvocationHandler 和 CusJdkInvocationHandler

这两个代理类的实现是一样的

区别在于实现的接口一个是我们定义的 MInvocationHandler

另一个是 JDK 的 InvocationHandler



![image-20221120154250947](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bmyiy96dj31700u0n0f.jpg)



### 执行入口类

Main 类分别生成了 Proxy 和 MProxy 的代理对象

然后执行代理对象的各个方法

![image-20221120154317206](https://tva1.sinaimg.cn/large/008vxvgGgy1h8bmyzd8qfj312y0u010o.jpg)



> 来看看实现的效果,左边是 JDK 的动态代理,右边是使用 Javassist 实现的动态代理

![image-20221117213245136](https://tva1.sinaimg.cn/large/008vxvgGgy1h88g7qvk5rj30ou0n2dj0.jpg)



最后,贴一下代码地址,感兴趣的同学可以给个 Star : https://github.com/MinXie1209/javassist-proxy