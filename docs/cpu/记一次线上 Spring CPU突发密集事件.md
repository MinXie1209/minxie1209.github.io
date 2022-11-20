# 记一次线上 Spring CPU 突发密集事件

这是一篇 CPU 突发密集的事件, 经过就是 CPU 突然就跑了很高,而且是间歇性的,同时计算耗时特别长 ,这是下图CPU的使用率

![image-20220907215444564](https://tva1.sinaimg.cn/large/e6c9d24egy1h5yduoo53ej213y0e8whf.jpg)

看起来CPU耗时很高,为了分析CPU的使用情况,当时就登录了一台CPU繁忙的机器

同时祭出大招,使用 Arthas 的 profiler 的命令生成火焰图

- 下载arthas : curl -O https://arthas.aliyun.com/arthas-boot.jar

- 执行arthas: java -jar arthas-boot.jar

- 生成火焰图
  - profiler start 开始收集
  - profiler stop 停止收集
  - profiler getSamples 查看收集的数量



生成了火焰图,那接下来就分析具体耗时的位置

![image-20220907220921351](https://tva1.sinaimg.cn/large/e6c9d24egy1h5ye9uzjjwj20u019l48h.jpg)

这是生成的火焰图

y轴是调用的堆栈

下面调用的方法入口,y越大,方法堆栈越深



x轴越宽,越耗时

我们需要注意是平顶

也就是x轴很宽,y轴很大的方法

这里的平顶是 putVal方法,往下定位到具体的业务代码



修复上线,再观察线上CPU的使用率情况

![image-20220907221855797](https://tva1.sinaimg.cn/large/e6c9d24egy1h5yejtespoj213s0c675w.jpg)