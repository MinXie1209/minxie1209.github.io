## Canal使用

之前业务中使用过canal监听binlog来实现业务的解耦合

## 什么是Canal

canal是基于MySQL数据库的增量日志(binlog)解析,提供增量数据的订阅消费

由阿里巴巴开源的框架,通过对数据库的日志解析实现对增量数据变更的同步处理,由此衍生出大量的数据库增量订阅和消费业务:

- 数据库镜像
- 数据库实时备份
- 索引构建和实时维护(拆分异构索倒排索引等)
- 业务缓存刷新
- 带业务逻辑的增量数据处理(*)





## Canal工作原理

![image-20221029202240586](https://tva1.sinaimg.cn/large/008vxvgGgy1h7mfex4rmsj319s0lwdhu.jpg)

- MySQL master 将数据变更写入bin log日志中
- MySQL slave 将binlog拷贝到它的中继日志(relay log)中
- MySQL slave 重放中继日志中的事件,将数据的变更反映出它自己的数据



canal伪装为slave ,向master发送dump协议





## 步骤



### 开启MySQL 的 bin log 



```
[mysqld]
log-bin=mysql-bin # 开启 binlog
binlog-format=ROW # 选择 ROW 模式
server_id=1 # 配置 MySQL replaction 需要定义，不要和 canal 的 slaveId 重复
```





### 安装canal server



### 将消息投递到RocketMQ



### 监听消费

