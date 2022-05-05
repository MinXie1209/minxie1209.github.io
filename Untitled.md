sendKernelImpl 分析

![image-20220505102801532](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xbjnshz5j22s00kkjww.jpg)

获取 Broker 对应的地址



![image-20220505102910007](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xbksdm6oj22o20b0tad.jpg)

Broker 地址为空，抛出异常信息



![image-20220505102945020](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xble137oj22pm06ujsv.jpg)

![image-20220505103119772](https://tva1.sinaimg.cn/large/e6c9d24ely1h1xbn1otb4j22r00jgdj8.jpg)

如果开通 VIP 通道，获取的VIP地址，是在原地址上端口号减 2 生成



![image-20220505104114953](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xbxcs4jgj22p20ligpk.jpg)

如果不是批量发送，设置消息的唯一ID

如果有命名空间，设置实例ID的值为和命名空间一样



![image-20220505104554853](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xc2823l6j22qi0jyn0w.jpg)

![image-20220505104757891](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xc4cjjl5j223z0u042q.jpg)

设置系统标志 sysFlag 

是否对消息体进行压缩

是否是事务第一阶段消息



![image-20220505104909866](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xc5m1vy6j21ly0u0gsh.jpg)

钩子函数执行



![image-20220505105204781](https://tva1.sinaimg.cn/large/e6c9d24egy1h1xc8ne07jj21xn0u0aia.jpg)

设置请求头信息

