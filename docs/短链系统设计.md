# 短链系统设计

最近,在分析慢接口的时候发现,耗时慢的位置在于 调用了第三方的短链系统

有多慢? 基本耗时好几秒吧

这是 trace 的截图

![image-20220924105904825](https://tva1.sinaimg.cn/large/e6c9d24egy1h6hifpk2f7j221o0h4n1l.jpg)

这慢得实在是惨不忍睹,那怎么办呢?自己设计一个吧!

## 开始设计

首先我们得清楚短链是啥?

顾名思义,‘短’是精髓,短链就是将 长链接 转为 短链接

如: https://juejin.cn/post/7043401155753279524 -->  http://minxie.space/a/1L9zO9R

## 清楚要解决的问题

1. 将长链接转为短链接
2. 访问短链接时可以重定向到长链接

主要就是上面两个问题,解决上面两个问题,一个简单的短链系统就出来了

### 如何将长链接转为短链接?

给长链接分配一个自增数字,再将 10进制数字转为 62进制的字符

int 最大的值 2147483647 转为 62进制为 2lkCB1 

给长链接分配 int 类型的数字,可以生成21亿多条的长链接,转化的短链长度为6位,21亿的数量级 一般的公司已经够用了

如果不使用 int类型,使用 long类型

那 long 最大的值 922 3372 0368 5477 5807L 转为 62进制的字符为 aZl8N0y58M7

给长链接分配 long 类型的数字,可以存储无数个的长链接,转化的短链长度为11位,而这个数量级就大到数不来了

### 如何将短链接映射到长链接?

那就需要将长短文本的关系存储起来了,可以使用数据库存储长短链的关系



## 动手干活了

按照上面的设计,我们第一步得实现长链转短链的功能,再实现短链重定向到长链

### 自增数字生成

这里我们得先给长链接生成一个数字

那实现的方式有多种

- 雪花算法

  可以使用雪花算法生成,雪花算法生成需要依赖数据id和时间,但时间如果回拨的话,会造成一定的数据重复,不能完全保证唯一

  但如果能判断ID重复可以再生成,些许麻烦,不推荐

- MySQL自增ID

  创建一个自增表,主键id为bigint类型,自增

  每次生成一条记录,取自增的id

  

- Redis自增

  可以通过Redis原子自增命令生成自增ID,不但高性能,占用的存储空间还小,五星推荐

### 10进制数字转62进制字符

要想实现10进制转62进制,我们得先了解一下62进制是什么?

![image-20220925003110571](https://tva1.sinaimg.cn/large/e6c9d24egy1h6i5wnkhqgj211y0g2jt4.jpg)

emmm,没想到百度百科没有收录62进制的词条

那直接贴个10进制转换62进制的栗子

![image-20220925003039582](https://tva1.sinaimg.cn/large/e6c9d24egy1h6i5w6dt51j21ak0u0dit.jpg)

10进制的10000 转换62进制的结果是 2Bi

首先62位的组成是 数字 + 小写字母 + 大写字母

10个数字加26个小写字母加26个大写字母刚刚好62位

0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ

![image-20220925004511354](https://tva1.sinaimg.cn/large/e6c9d24egy1h6i6b8mcl2j20zu0domyg.jpg)



```kotlin
class Base62Utils {
    companion object {
        // 62个字符 10个数字+大小写英文字母
        private const val CHARS = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

        // 字符数组长度
        private const val CHARS_LEN = CHARS.length

        fun genBase62(input: Long): String {
            var newInput = input
            val sb = StringBuilder("")
            if (newInput == 0L) {
                return "0"
            }
            while (newInput > 0) {
                newInput = genBase62(newInput, sb)
            }
            return sb.reverse().toString()
        }

        private fun genBase62(input: Long, sb: StringBuilder): Long {
            val rem = (input % CHARS_LEN).toInt()
            sb.append(CHARS[rem])
            return input / CHARS_LEN
        }

    }
}
```



### 存储长短链映射关系

前面已经可以通过长链生成了短链,那么接下来我们需要创建一个表来存储长短链的关系



```sql
create table long_short_url_relation
(
    id          bigint auto_increment comment '自增id' primary key,
    long_url    varchar(255) not null comment '长链',
    short_url   varchar(255) not null comment '短链',
    create_time datetime     not null default current_timestamp comment '创建时间',
    update_time datetime     not null default current_timestamp on update current_timestamp comment '更新时间'
) comment '长短链映射表' engine = innodb;

create unique index uk_long_url_short_url
    on long_short_url_relation (long_url, short_url) comment '唯一键';
create index idx_short_url
    on long_short_url_relation (short_url) comment '索引';
```



这里创建一个表 long_short_url_relation

字段有long_url、short_url

唯一索引(long_url,short_url)

索引(short_url)



### 处理逻辑

1. 暴露入参 longUrl
2. 根据 longUrl 去数据库查找看有没有对应的记录
3. 有,则返回对应的短链 shortUrl
4. 没有,则先从 Redis 里生成相应的数字,再根据数字生成 62进制 的字符串,拼上前缀,生成对应的短链,再存储到数据库中



### 优化点

当对应的长链没有生成短链时,我们需要先查找数据库,再插入数据库,两次的数据库操作,虽然还好,但有没有优化的空间呢?

答案是有,这时候我们熟知的“布隆过滤器”又可以出场了,上一篇[换个思路, 省下 Redis 10G 内存](https://juejin.cn/post/7142511895965073444)中它可是主角啊

那么怎么合理使用布隆过滤器来优化我们的性能呢?

- 先从布隆过滤器中查找对应的长链是否存在
- 不存在,则生成短链,并插入数据库记录和布隆过滤器中
- 存在,则查询数据库,但这里需要注意布隆过滤器有一定的误杀,所以查询数据库未必就一定存在记录
- 存在记录,则返回短链
- 不存在,同样的 生成短链,并插入数据库记录和布隆过滤器中



### 思考点

这里需要考虑不同的长链生成的短链会有冲突的情况吗?

首先短链是通过数字转为62进制而来,只要我们保证不同的长链生成的数字绝对不会重复,那么我们生成的短链就不会重复

如果真的不同的长链生成的数字重复了,那么我们重新生成新的数字就解决了



> 到这里,基本上可以解决 长链生成短链 的问题了,那这个问题就过了



## 如何将短链接映射到长链接?

前面已经是通过长链生成了对应的短链,而且存储到我们的数据库表中

那么当用户拿到了生成的短链后,如何通过短链去访问到实际的长链内容呢?



这里我们需要了解的就是 Http的重定向

### Http的重定向

![image-20220925094756638](https://tva1.sinaimg.cn/large/e6c9d24egy1h6ilzyt5uuj21370u0n0f.jpg)

那重定向包含哪些呢?

- 301、308: 永久重定向,它表示原URL不再使用,这里和我们的场景不符
- 302、303、307: 临时重定向

这里使用临时重定向我们的短链



### 代码实现

由于我们使用的是 SpringBoot 框架,该框架本身就实现了重定向的功能,所以很简单

```kotlin
override fun short2Long(shortUrl: String): Any {
    val longShortUrlRelation = longShortUrlRelationMapper.selectByShortUrl("$basePath$shortUrl")
    if (longShortUrlRelation == null) {
        return ""
    }
    val redirectView = RedirectView(longShortUrlRelation.longUrl, true)
    redirectView.setStatusCode(HttpStatus.FOUND)
    return redirectView
}
```



传入长链构造对应的RedirectView对象,再设置状态码为 302 即可实现短链重定向到长链



## 总结思考

思路很粗糙,还有很多可以思考的空间留给大家

- 当生成的数据越来越多时,如何保证查询的性能,是分库分表呢,还是引入缓存呢?
- 生成的短链应该是永久的吗?还是有一定的实时性,比如一个月期限过期
- 除了 SpringBoot 框架,是否可以使用别的技术框架实现
- ...

