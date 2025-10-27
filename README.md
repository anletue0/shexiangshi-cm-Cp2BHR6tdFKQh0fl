[合集 - Java烘焙师的架构笔记(20)](https://github.com)

1.架构师必备：限流方案选型（使用篇）10-27

[2.架构师必备：缓存更新模式总结09-15](https://github.com/toplist/p/19066474)[3.架构师必备：实时对账与离线对账08-04](https://github.com/toplist/p/19009976)[4.架构师必备：业务扩展模式选型07-07](https://github.com/toplist/p/18955480)[5.Java反射原理和实际用法2022-08-13](https://github.com/toplist/p/16513490.html)[6.架构师必备：系统容量现状checklist2022-06-13](https://github.com/toplist/p/16366857.html)[7.架构师必备：HBase行键设计与应用2022-06-07](https://github.com/toplist/p/16343200.html)[8.架构师必备：多维度查询的最佳实践2022-05-22](https://github.com/toplist/p/16295666.html)[9.架构师必备：Redis的几种集群方案2022-04-30](https://github.com/toplist/p/16128352.html)[10.Spring cache源码分析2022-03-30](https://github.com/toplist/p/16032955.html)[11.架构师必备：本地缓存原理和应用2022-02-28](https://github.com/toplist/p/15941303.html)[12.架构师必备：系统性解决幂等问题2022-01-14](https://github.com/toplist/p/15780103.html)[13.架构师必备：如何做容量预估和调优2021-12-24](https://github.com/toplist/p/15659580.html):[狗狗加速](https://gougoujs.com)[14.架构师必备：巧用Canal实现异步、解耦的架构2021-11-27](https://github.com/toplist/p/15449549.html)[15.架构师必备：MySQL主从延迟解决办法2021-10-16](https://github.com/toplist/p/15391534.html)[16.架构师必备：MySQL主从同步原理和应用2021-10-08](https://github.com/toplist/p/15365460.html)[17.应用开发中的存储架构进化史——从起步到起飞2021-09-27](https://github.com/toplist/p/15341529.html)[18.Spring @Async的异常处理2017-10-28](https://github.com/toplist/p/7747225.html)[19.Spring Boot应用中的异常处理2017-10-01](https://github.com/toplist/p/7616570.html)[20.Java子线程中的异常处理（通用）2017-09-25](https://github.com/toplist/p/7594557.html)

收起

大家好，我是Java烘焙师。为了避免突增流量引起服务雪崩，需要对接口、存储资源做限流保护，根据系统负载情况设置合适的限流值。下面结合笔者的经验和思考，对主要限流方案的选型做一下总结，本篇先看如何使用，下一篇再看背后的原理。

下面介绍几种常见限流方案的使用方法、优缺点：

* 单机限流：Guava RateLimiter
* 同时支持单机限流、集群限流：Sentinel
* 分布式限流：Redisson RateLimiter

# 1. 单机限流：Guava RateLimiter

## 使用

官网API文档：[https://guava.dev/releases/snapshot-jre/api/docs/com/google/common/util/concurrent/RateLimiter.html](https://github.com)

Guava是Google开源的Java类库，提供了很多实用的工具，包括集合、本地缓存、并发编程工具等。Guava RateLimiter就是其中的单机限流工具，核心概念如下：

* 每秒允许通过的最大permits：
  + 当请求获取1个permit时，就相当于是qps限流
  + 当请求获取多个permits时，代表该请求需要消耗多少资源，比如想限定更新DB的SQL qps为1000，则需获取1000 permits
* 支持预热：在预热期内逐步支持到最大permits，适用于需要预热缓存资源的场景
* 允许突发流量：每次请求是否被限流，不取决于该次请求permits的多少，而是取决于前一次请求，前一次请求的permits越多，则后续请求需等待的时间越长
  + 比如有一个空闲的RateLimiter，第一次请求无论是获取1 permit、还是10000 permits，都能立刻成功，但是会计算出下一次permits可用的时间。那么第一次请求是10000 permits时，第二次请求即使只需10 permits也可能要长时间等待

如下分别是阻塞、非阻塞方式，使用Guava RateLimiter的示例。

```
复制代码
```

* 1
* 2
* 3
* 4
* 5
* 6
* 7
* 8
* 9
* 10
* 11
* 12
* 13
* 14
* 15
* 16
* 17
* 18
* 19
* 20

 `// 创建一个每秒允许10个permits的限流器
RateLimiter rateLimiter = RateLimiter.create(10.0);
// 1. 阻塞的方式
long startTime = System.currentTimeMillis();
// 获取5 permits；如果充足，则获取成功，否则阻塞等待
rateLimiter.aquire(5);
long endTime = System.currentTimeMillis();
// 打印阻塞等待的时间
System.out.println("Processing business logic. Waiting time ms: " + (endTime - startTime));
// 2. 非阻塞的方式
// 尝试获取1 permit
if (rateLimiter.tryAcquire()) {
// 如果能获取到令牌，则继续处理业务逻辑
System.out.println("Processing business logic...");
} else {
// 否则快速失败，返回友好提示或执行降级逻辑
System.out.println("Too many requests.");
}`

## 优点

* 性能极高：纯计算，无网络开销
* 实现简单

## 缺点

* 无法跨实例协同，从整体层面控制限流

# 2. 同时支持单机限流、集群限流：Sentinel

## 使用

官网文档：[https://sentinelguard.io/zh-cn/docs/introduction.html](https://github.com)

Sentinel是阿里开源的流量治理组件，在国内使用较广泛。核心概念如下：

* 资源：可以是接口（调入、调出均可），也可以是任何一段代码逻辑。可通过sentinel注解、或者sentinel API包装成受保护的资源
* 规则：围绕资源的实时状态设定的规则，包括流量控制规则、熔断降级规则以及系统保护规则，所有规则都可以动态实时调整。
  + 常见的流量控制规则：
    - 限流阈值类型：支持QPS、线程数模式，默认是按QPS来限流
    - 流控效果：即超过限流阈值时的行为，支持直接拒绝、排队等待、慢启动模式，默认是直接拒绝
  + 常见的熔断降级规则：
    - 熔断策略：即在什么情况下熔断降级，支持慢调用比例、异常比例、异常数策略，默认是慢调用比例
    - 熔断阈值：慢调用比例模式下为慢调用的RT平均响应时间，异常比例/异常数模式下为对应的阈值
* 集群限流：是为了解决流量在集群内各实例之间不均匀、导致触发单机限流，而达不到总QPS预期的问题。集群限流有2种模式：单机均摊、全局阈值，为了避免token server成为瓶颈，比较典型的是单机均摊。
  + 单机均摊：先设置一个集群限流阈值，然后再根据机器实例数均摊到单机上，即`单机限流值=集群限流值/机器实例数`
  + 全局阈值：集群流控中共有两种身份
    - Token Client：集群流控客户端，用于向所属 Token Server 通信请求 token。集群限流服务端会返回给客户端结果，决定是否限流
    - Token Server：即集群流控服务端，处理来自 Token Client 的请求，根据配置的集群规则判断是否应该发放 token（是否允许通过）

如下分别是通过sentinel注解、sentinel API方式的示例。

* sentinel注解一般加在受保护的接口、访问存储资源的方法上
* sentinel API更加灵活，可包住受保护的业务代码片段；更进一步，结合业务入参，可实现热点参数限流

```
复制代码
```

* 1
* 2
* 3
* 4
* 5
* 6
* 7
* 8
* 9
* 10
* 11
* 12
* 13
* 14
* 15
* 16
* 17
* 18
* 19
* 20
* 21
* 22
* 23
* 24
* 25
* 26
* 27
* 28
* 29
* 30
* 31
* 32
* 33
* 34
* 35
* 36
* 37
* 38
* 39

 `// 1. sentinel注解：在sentinel控制台上可针对定义的资源名配置限流规则
@Service
@SentinelResource(value = "protectedMethod")
public String protectedMethod(Request request) {
}
// 2. sentinel API
// 2.1 抛异常方式定义资源
try (Entry entry = SphU.entry("protectedResource1")) {
// 被保护的逻辑
System.out.println("protected resource 1");
} catch (BlockException ex) {
// 处理被流控的逻辑
System.out.println("blocked!");
}
// 2.2 返回布尔值方式定义资源
if (SphO.entry("protectedResource2")) {
// 务必保证finally会被执行
try {
// 被保护的业务逻辑
} finally {
SphO.exit();
}
} else {
// 资源访问阻止，被限流或被降级
// 进行相应的处理操作
}
// 2.3 热点参数限流
try (Entry entry = SphU.entry("hotspotResource", EntryType.IN, 1, paramA, paramB)) {
// 易出现热点参数的业务逻辑
} catch (BlockException ex) {
// 热点参数限流
} finally {
if (entry != null) {
entry.exit(1, paramA, paramB);
}
}`

## 优点

* 易于与Java框架集成，提高应用开发效率：如Spring Cloud、Dubbo等
* 功能强大：在控制台页面，可动态配置限流规则、熔断降级规则

## 缺点

* sentinel的集群限流，在单机均摊模式下最终会换算为单机限流，当集群限流值较低、机器实例数较多时，计算出的单机限流值可能不准，无法精准控制总的集群限流
  + 例如：有100台机器，按30 qps来限流；则计算出的单机均摊qps不足1、按1处理，这样总的集群限流就变成了100 qps，大于预期的30 qps

# 3. 分布式限流：Redisson RateLimiter

## 使用

官网API文档：[https://redisson.pro/docs/data-and-services/objects/#ratelimiter](https://github.com)

Redisson是一个高性能的Redis客户端框架，提供了基于redis实现的分布式工具，如分布式集合、分布式锁、布隆过滤器等，借助Redisson RateLimiter可以实现分布式限流。核心概念如下：

* 允许在一段时间间隔内，最多有n个permits通过
* 通过redis lua脚本实现限流逻辑、以及原子操作

如下是使用Redisson RateLimiter的示例。

```
复制代码
```

* 1
* 2
* 3
* 4
* 5
* 6
* 7
* 8
* 9
* 10
* 11
* 12
* 13

 `// 使用Redisson的分布式限流器
RRateLimiter limiter = redisson.getRateLimiter("myLimiter");
// 限定每60秒100个permits
limiter.trySetRate(RateType.OVERALL, 100, 60, RateIntervalUnit.SECONDS);
// 阻塞式获取5 permits
limiter.acquire(5);
// 非阻塞式获取1 permit
if (limiter.tryAcquire(1)) {
// 获取到permit，执行业务逻辑
} else {
// 被限流
}`

## 优点

* 基于redis实现了分布式限流，能较精准地控制低qps场景的限流
* 与特定框架/限流组件无关

## 缺点

* 引入了外部依赖，有网络开销，系统稳定性易受影响

## 其它分布式限流方案

笔者还了解到有一个redis模块 [redis-cell](https://github.com) 可做分布式限流。不过有以下问题，不是很建议使用：

* 需要额外运维部署，成本较高
* 项目由个人维护，目前已基本停滞

# 结论

* Java应用开发，一般情况下，可引入Sentinel做接口、存储资源的限流控制
* 单机任务，如手动执行或定期执行的task，可引入Guava RateLimiter做简易的限流控制
* 低qps场景的限流、或不想接入特定框架/限流组件的服务，可引入Redisson RateLimiter来实现较精准的分布式限流控制
