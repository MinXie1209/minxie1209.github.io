> asm是一款编写字节码的框架，熟练使用可以加深对字节码指令的掌握。

## Java的动态代理
Java动态代理是基于接口代理的，所以首先我们得定义一个公共接口。

现在代理用户接口，实现登陆逻辑和来打印登录的花费时间


```java
public interface UserService {

    boolean login(String username, String password);
}

```
再来看看Proxy的使用方法，newProxyInstance方法需要传三个参数，第一个类加载器，第二个需要代理的接口数组，第三个参数是调用方法处理器，也是我们写代理逻辑的需要实现的接口。


![image.png](https://tva1.sinaimg.cn/large/e6c9d24egy1h1to5mh3u0j211w0fy0u7.jpg)

实现InvocationHandler，判断传入的username 和password是否等于admin，而且打印调用方法耗时。
```java
public class UserServiceInvocationHandler implements InvocationHandler {

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws InvocationTargetException, IllegalAccessException {
        long start = System.currentTimeMillis();
        System.out.println("invoke:" + proxy.getClass().getSimpleName() + "." + method.getName() + ":" + (System.currentTimeMillis() - start) + "ms");
        return "admin".equals(args[0]) && "admin".equals(args[1]);
    }
}
```

生成代理类

```java
import java.lang.reflect.Proxy;

public class App {
    public static void main(String[] args) {
        UserService userServiceProxy = (UserService) Proxy.newProxyInstance(App.class.getClassLoader(), new Class[]{UserService.class}, new UserServiceInvocationHandler());
        System.out.println(userServiceProxy.getClass());
        System.out.println(userServiceProxy.login("admin", "admin"));
        System.out.println(userServiceProxy.login("admin", "admin1"));
    }
}
```
调用main方法，打印结果

![image.png](https://tva1.sinaimg.cn/large/e6c9d24egy1h1to5m6e94j211w0fy0u7.jpg)


## 使用ASM实现
首先我们先看一下生成的代理类最终的样子
```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import proxy.ASMProxy;
import proxy.UserService;

public class $Proxy0 extends ASMProxy implements UserService {
    public static Method _UserService_0 = Class.forName("proxy.UserService").getMethod("login", Class.forName("java.lang.String"), Class.forName("java.lang.String"));

    public $Proxy0(InvocationHandler var1) {
        super(var1);
    }

    public boolean login(String var1, String var2) throws Exception {
        return (Boolean)super.h.invoke(this, _UserService_0, new Object[]{var1, var2});
    }

}
```
三个要点：
- InvocationHandler保存在ASMProxy中。
- 要实现的接口方法Method使用静态字段保存。
- 实现接口方法内实际调用父类的InvocationHandler的invoke方法。

具体看看实现步骤

### ASMProxy
```java
package proxy;

import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.util.concurrent.atomic.AtomicInteger;

public class ASMProxy {
    protected InvocationHandler h;
    //代理类名计数器
    private static final AtomicInteger PROXY_CNT = new AtomicInteger(0);
    private static final String PROXY_CLASS_NAME_PRE = "$Proxy";

    public ASMProxy(InvocationHandler var1) {
        h = var1;
    }

    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
            throws Exception {
        //生成代理类Class
        Class<?> proxyClass = generate(interfaces);
        Constructor<?> constructor = proxyClass.getConstructor(InvocationHandler.class);
        return constructor.newInstance(h);
    }

    /**
     * 生成代理类Class
     *
     * @param interfaces
     * @return
     */
    private static Class<?> generate(Class<?>[] interfaces) throws ClassNotFoundException {
        String proxyClassName = PROXY_CLASS_NAME_PRE + PROXY_CNT.getAndIncrement();
        byte[] codes = ASMProxyFactory.generateClass(interfaces, proxyClassName);
      

        //使用自定义类加载器加载字节码
        ASMClassLoader asmClassLoader = new ASMClassLoader();
        asmClassLoader.add(proxyClassName, codes);
        return asmClassLoader.loadClass(proxyClassName);
    }
}
```
ASMProxy的主要功能一个是作为代理类需要继承的父类，接着提供一个和Proxy同样的静态方法newProxyInstance。<br>
newProxyInstance里面调用ASMProxyFactory生成字节码二进制流，然后调用自定义的类加载器来生成Class。<br>
最后反射生成代理类的实例，返回对象。



#### ASMProxyFactory

接着看看最核心的部分，ASMProxyFactory是怎样生成字节码的，分几个步骤：<br>
1. 创建初始化方法
2. 声明静态字段
3. 创建静态方法
4. 实现接口方法

```java
package proxy;

import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.Type;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.util.Arrays;
import java.util.stream.Collectors;

public class ASMProxyFactory {
    private static final Integer DEFAULT_NUM = 1;

    public static byte[] generateClass(Class<?>[] interfaces, String proxyClassName) {
        //创建一个ClassWriter对象，自动计算栈帧和局部变量表大小
        ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES | ClassWriter.COMPUTE_MAXS);
        //创建的java版本、访问标志、类名、父类、接口
        cw.visit(Opcodes.V1_8, Opcodes.ACC_PUBLIC, proxyClassName, null, Type.getInternalName(ASMProxy.class), getInterfacesName(interfaces));
        //创建<init>
        createInit(cw);
        //创建static
        addStatic(cw, interfaces);
        //创建<clinit>
        addClinit(cw, interfaces, proxyClassName);
        //实现接口方法
        addInterfacesImpl(cw, interfaces, proxyClassName);
        cw.visitEnd();
        return cw.toByteArray();
    }
    
    private static String[] getInterfacesName(Class<?>[] interfaces) {
        String[] interfacesName = new String[interfaces.length];
        return Arrays.stream(interfaces).map(Type::getInternalName).collect(Collectors.toList()).toArray(interfacesName);
    }
    
     /**
     * 创建init方法
     * 调用父类的构造方法
     * 0 aload_0
     * 1 aload_1
     * 2 invokespecial #1 <proxy/ASMProxy.<init> : (Ljava/lang/reflect/InvocationHandler;)V>
     * 5 return
     *
     * @param cw
     */
    private static void createInit(ClassWriter cw) {
        MethodVisitor mv = cw.visitMethod(Opcodes.ACC_PUBLIC, "<init>", "(Ljava/lang/reflect/InvocationHandler;)V", null, null);
        mv.visitCode();
        //将this入栈
        mv.visitVarInsn(Opcodes.ALOAD, 0);
        //将参数入栈
        mv.visitVarInsn(Opcodes.ALOAD, 1);
        //调用父类初始化方法
        mv.visitMethodInsn(Opcodes.INVOKESPECIAL, Type.getInternalName(ASMProxy.class), "<init>", "(Ljava/lang/reflect/InvocationHandler;)V", false);
        // 返回
        mv.visitInsn(Opcodes.RETURN);
        mv.visitMaxs(2, 2);
        mv.visitEnd();
    }
    
    /**
     * 创建static字段
     *
     * @param cw
     * @param interfaces
     */
    private static void addStatic(ClassWriter cw, Class<?>[] interfaces) {
        for (Class<?> anInterface : interfaces) {
            for (int i = 0; i < anInterface.getMethods().length; i++) {
                String methodName = "_" + anInterface.getSimpleName() + "_" + i;
                cw.visitField(Opcodes.ACC_PUBLIC | Opcodes.ACC_STATIC, methodName, Type.getDescriptor(Method.class), null, null);
            }
        }
    }
    
    private static void addClinit(ClassWriter cw, Class<?>[] interfaces, String proxyClassName) {
        //_UserService_0 = Class.forName("proxy.UserService").getMethod("login", String.class, String.class);
        MethodVisitor mv = cw.visitMethod(Opcodes.ACC_STATIC, "<clinit>", "()V", null, null);
        mv.visitCode();
        for (Class<?> anInterface : interfaces) {
            for (int i = 0; i < anInterface.getMethods().length; i++) {
                Method method = anInterface.getMethods()[i];
                String methodName = "_" + anInterface.getSimpleName() + "_" + i;
                mv.visitLdcInsn(anInterface.getName());
                mv.visitMethodInsn(Opcodes.INVOKESTATIC, Type.getInternalName(Class.class), "forName", "(Ljava/lang/String;)Ljava/lang/Class;", false);
                mv.visitLdcInsn(method.getName());
                if (method.getParameterCount() == 0) {
                    mv.visitInsn(Opcodes.ACONST_NULL);
                } else {
                    switch (method.getParameterCount()) {

                        case 1:
                            mv.visitInsn(Opcodes.ICONST_1);
                            break;
                        case 2:
                            mv.visitInsn(Opcodes.ICONST_2);
                            break;
                        case 3:
                            mv.visitInsn(Opcodes.ICONST_3);
                            break;
                        default:
                            mv.visitVarInsn(Opcodes.BIPUSH, method.getParameterCount());
                            break;
                    }
                    mv.visitTypeInsn(Opcodes.ANEWARRAY, Type.getInternalName(Class.class));
                    for (int paramIndex = 0; paramIndex < method.getParameterTypes().length; paramIndex++) {
                        Class<?> parameter = method.getParameterTypes()[paramIndex];
                        mv.visitInsn(Opcodes.DUP);
                        switch (paramIndex) {
                            case 0:
                                mv.visitInsn(Opcodes.ICONST_0);
                                break;
                            case 1:
                                mv.visitInsn(Opcodes.ICONST_1);
                                break;
                            case 2:
                                mv.visitInsn(Opcodes.ICONST_2);
                                break;
                            case 3:
                                mv.visitInsn(Opcodes.ICONST_3);
                                break;
                            default:
                                mv.visitVarInsn(Opcodes.BIPUSH, paramIndex);
                                break;
                        }
                        mv.visitLdcInsn(parameter.getName());
                        mv.visitMethodInsn(
                                Opcodes.INVOKESTATIC, Type.getInternalName(Class.class),
                                "forName",
                                "(Ljava/lang/String;)Ljava/lang/Class;",
                                false
                        );
                        mv.visitInsn(Opcodes.AASTORE);
                    }
                }

//                invokevirtual #13 <java/lang/Class.getMethod : (Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;>
                mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, Type.getInternalName(Class.class), "getMethod", "(Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;", false);
                //putstatic #3 <proxy/$Proxy0._UserService_0 : Ljava/lang/reflect/Method;>
                mv.visitFieldInsn(Opcodes.PUTSTATIC, proxyClassName, methodName, Type.getDescriptor(Method.class));
            }
            mv.visitInsn(Opcodes.RETURN);
        }
        mv.visitMaxs(DEFAULT_NUM, DEFAULT_NUM);
        mv.visitEnd();
    }
    
    private static void addInterfacesImpl(ClassWriter cw, Class<?>[] interfaces, String proxyClassName) {
        for (Class<?> anInterface : interfaces) {
            for (int i = 0; i < anInterface.getMethods().length; i++) {
                Method method = anInterface.getMethods()[i];
                String methodName = "_" + anInterface.getSimpleName() + "_" + i;
                MethodVisitor mv = cw.visitMethod(Opcodes.ACC_PUBLIC, method.getName(), Type.getMethodDescriptor(method), null, new String[]{Type.getInternalName(Exception.class)});
                mv.visitCode();
                mv.visitVarInsn(Opcodes.ALOAD, 0);
                mv.visitFieldInsn(Opcodes.GETFIELD, Type.getInternalName(ASMProxy.class), "h", "Ljava/lang/reflect/InvocationHandler;");
                mv.visitVarInsn(Opcodes.ALOAD, 0);
                mv.visitFieldInsn(Opcodes.GETSTATIC, proxyClassName, methodName, Type.getDescriptor(Method.class));
                //
                switch (method.getParameterCount()) {
                    case 0:
                        mv.visitInsn(Opcodes.ICONST_0);
                        break;
                    case 1:
                        mv.visitInsn(Opcodes.ICONST_1);
                        break;
                    case 2:
                        mv.visitInsn(Opcodes.ICONST_2);
                        break;
                    case 3:
                        mv.visitInsn(Opcodes.ICONST_3);
                        break;
                    default:
                        mv.visitVarInsn(Opcodes.BIPUSH, method.getParameterCount());
                        break;
                }
                mv.visitTypeInsn(Opcodes.ANEWARRAY, Type.getInternalName(Object.class));
                //     * 12 dup
                //     * 13 iconst_0
                //     * 14 aload_1
                //     * 15 aastore
                for (int paramIndex = 0; paramIndex < method.getParameterCount(); paramIndex++) {
                    mv.visitInsn(Opcodes.DUP);
                    switch (paramIndex) {
                        case 0:
                            mv.visitInsn(Opcodes.ICONST_0);
                            break;
                        case 1:
                            mv.visitInsn(Opcodes.ICONST_1);
                            break;
                        case 2:
                            mv.visitInsn(Opcodes.ICONST_2);
                            break;
                        case 3:
                            mv.visitInsn(Opcodes.ICONST_3);
                            break;
                        default:
                            mv.visitVarInsn(Opcodes.BIPUSH, paramIndex);
                            break;
                    }
                    mv.visitVarInsn(Opcodes.ALOAD, paramIndex + 1);
                    mv.visitInsn(Opcodes.AASTORE);
                }
// * 20 invokeinterface #5 <java/lang/reflect/InvocationHandler.invoke : (Ljava/lang/Object;Ljava/lang/reflect/Method;[Ljava/lang/Object;)Ljava/lang/Object;> count 4
//     * 25 checkcast #6 <java/lang/Boolean>
//     * 28 invokevirtual #7 <java/lang/Boolean.booleanValue : ()Z>
                mv.visitMethodInsn(Opcodes.INVOKEINTERFACE, Type.getInternalName(InvocationHandler.class), "invoke",
                        "(Ljava/lang/Object;Ljava/lang/reflect/Method;[Ljava/lang/Object;)Ljava/lang/Object;", true);
                addReturn(mv, method.getReturnType());
//                mv.visitFrame(Opcodes.F_FULL, 0, null, 0, null);
                mv.visitMaxs(DEFAULT_NUM, DEFAULT_NUM);
                mv.visitEnd();
            }
        }
    }
    
    //添加方法返回
    private static void addReturn(MethodVisitor mv, Class<?> returnType) {
        if (returnType.isAssignableFrom(Void.class)) {
            mv.visitInsn(Opcodes.RETURN);
            return;
        }
        if (returnType.isAssignableFrom(boolean.class)) {
            //checkcast #6 <java/lang/Boolean>
            //     * 28 invokevirtual #7 <java/lang/Boolean.booleanValue : ()Z>
            mv.visitTypeInsn(Opcodes.CHECKCAST, Type.getInternalName(Boolean.class));
            mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, Type.getInternalName(Boolean.class), "booleanValue", "()Z", false);
            mv.visitInsn(Opcodes.IRETURN);
        } else if (returnType.isAssignableFrom(int.class)) {
            mv.visitTypeInsn(Opcodes.CHECKCAST, Type.getInternalName(Integer.class));
            mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, Type.getInternalName(Integer.class), "intValue", "()I", false);
            mv.visitInsn(Opcodes.IRETURN);
        } else if (returnType.isAssignableFrom(long.class)) {
            mv.visitTypeInsn(Opcodes.CHECKCAST, Type.getInternalName(Long.class));
            mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, Type.getInternalName(Long.class), "longValue", "()J", false);
            mv.visitInsn(Opcodes.JRETURN);
        } else if (returnType.isAssignableFrom(short.class)) {
            mv.visitTypeInsn(Opcodes.CHECKCAST, Type.getInternalName(Short.class));
            mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, Type.getInternalName(Short.class), "shortValue", "()S", false);
            mv.visitInsn(Opcodes.IRETURN);
        } else if (returnType.isAssignableFrom(byte.class)) {
            mv.visitTypeInsn(Opcodes.CHECKCAST, Type.getInternalName(Byte.class));
            mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, Type.getInternalName(Byte.class), "byteValue", "()B", false);
            mv.visitInsn(Opcodes.IRETURN);
        } else if (returnType.isAssignableFrom(char.class)) {
            mv.visitTypeInsn(Opcodes.CHECKCAST, Type.getInternalName(Character.class));
            mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, Type.getInternalName(Character.class), "charValue", "()C", false);
            mv.visitInsn(Opcodes.IRETURN);
        } else if (returnType.isAssignableFrom(float.class)) {
            mv.visitTypeInsn(Opcodes.CHECKCAST, Type.getInternalName(Float.class));
            mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, Type.getInternalName(Float.class), "floatValue", "()F", false);
            mv.visitInsn(Opcodes.FRETURN);
        } else if (returnType.isAssignableFrom(double.class)) {
            mv.visitTypeInsn(Opcodes.CHECKCAST, Type.getInternalName(Double.class));
            mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, Type.getInternalName(Double.class), "doubleValue", "()D", false);
            mv.visitInsn(Opcodes.DRETURN);
        } else {
            mv.visitTypeInsn(Opcodes.CHECKCAST, Type.getInternalName(returnType));
            mv.visitInsn(Opcodes.ARETURN);
        }
    }
    
        
}
```
#### ASMClassLoader

自定义类加载器，提供add方法，添加<类名、字节码>映射关系<br>
覆写findClass方法，当类名能找到对应字节码时，调用defineClass生成Class。

```
package proxy;

import java.util.HashMap;
import java.util.Map;

public class ASMClassLoader extends ClassLoader {
    private final Map<String, byte[]> classMap = new HashMap<>();

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        if (classMap.containsKey(name)) {
            byte[] bytes = classMap.get(name);
            classMap.remove(name);
            return defineClass(name, bytes, 0, bytes.length);
        }
        return super.findClass(name);
    }

    public void add(String name, byte[] bytes) {
        classMap.put(name, bytes);
    }
}
```

#### App
```java
package proxy;

import java.lang.reflect.Proxy;

public class App {
    public static void main(String[] args) throws Throwable {
        System.out.println("Java动态代理===========================");
        UserService userServiceProxy = (UserService) Proxy.newProxyInstance(App.class.getClassLoader(), new Class[]{UserService.class}, new UserServiceInvocationHandler());
        System.out.println(userServiceProxy.getClass());
        System.out.println(userServiceProxy.login("admin", "admin"));
        System.out.println(userServiceProxy.login("admin", "admin1"));


        System.out.println("ASM动态代理===========================");
        UserService userServiceAsm = (UserService) ASMProxy.newProxyInstance(App.class.getClassLoader(), new Class[]{UserService.class}, new UserServiceInvocationHandler());
        System.out.println(userServiceAsm.getClass());
        System.out.println(userServiceAsm.login("admin", "admin"));
        System.out.println(userServiceAsm.login("admin", "admin1"));
    }
}
```
运行App:打印两种代理方式的结果

![image.png](https://tva1.sinaimg.cn/large/e6c9d24egy1h1to5n4pudj21420nwtau.jpg)