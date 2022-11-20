在我们排查线上问题的时候,可能会发现在一些很重要的地方并没有打印日志?

那么如果在不重新发布应用的前提下,打印我们需要的日志信息呢?



先简单写个程序

```kotlin
package space.minxie.arthasstudy.service

import org.slf4j.LoggerFactory
import org.springframework.stereotype.Service

interface TestService {
    fun checkName(name: String): Any
}

@Service
class TestServiceImpl : TestService {
    private val log = LoggerFactory.getLogger(TestServiceImpl::class.java)
    override fun checkName(name: String): Any {
        if (name == "MinXie") {
            return true
        }
        return false
    }
}
```

但传入的name等于MinXie的时候,返回true

假设我们需要知道返回false的时候传入的name是什么?

这时候除了改代码重新打包发布之外,还有没有更优雅的方案呢?



答案当然有



arthas retransform 搞起

jad --source-only space.minxie.arthasstudy.service.TestServiceImpl > /tmp/TestServiceImpl.java

mc /tmp/TestServiceImpl.java -d /tmp

retransform /tmp/space/minxie/arthasstudy/service/TestServiceImpl.class



```
retransform --deleteAll
```