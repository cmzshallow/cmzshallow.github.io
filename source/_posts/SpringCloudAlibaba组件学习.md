---
title: SpringCloudAlibaba组件学习
date: 2024-04-29 10:03
tags:
  - Spring Cloud Alibaba
categories:
  - 学习
---

# 系统架构的演变

单体应用架构 -> 垂直应用架构 -> 分布式架构 -> SOA架构 -> 微服务架构

- 单体应用架构：一般网站应用流量较小，只需要一个应用，将所有功能代码都部署在一起
  - 优点：
    - 开发、部署、维护成本低
  - 缺点：
    - 代码耦合性高

- 垂直应用架构：在单体应用的基础上，拆分出多个非耦合的系统
  - 优点：
    - 灵活、可以针对不同模块的系统进行优化和水平拓展
    - 系统之间不会互相影响，提高容错率
  - 缺点：
    - 重复代码多，每个模块可能都有很多重复的通用代码
- 分布式架构：
  - 优点：
    - 抽取公共的功能为服务层，提高代码复用性
  - 缺点：
    - 系统间耦合度变高，调用关系复杂
- SOA架构：
  - 优点：
    - 使用治理中心（ESB/dubbo）解决了服务间调用关系的自动调节
  - 缺点：
    - 服务间会有依赖关系，一旦某个环节出错影响较大（服务雪崩）
    - 服务关系复杂，运维、测试、部署困难
- 微服务架构：
  - 优点：
    - 服务原子化拆分，独立打包、部署和升级，保证每个微服务清晰划分，利于拓展
    - 微服务之间采用 Restful 等轻量级 http 协议相互调用
  - 缺点：
    - 分布式系统开发的技术成本高（容错、分布式事务）
    - 复杂性更高。各个微服务进行分布式独立部署，当进行模块调用的时候，分布式会变得更加麻烦

# Spring Cloud Alibaba 各组件作用

## Nacos：服务注册与配置中心
### Nacos 注册中心：
管理所有微服务，解决微服务之间调用关系错综复杂，难以维护的问题。

底层实现：
- Nacos 维护一个注册表
- 服务启动时，调用 Nacos 提供的注册接口，将自身服务的信息注册到 Nacos
- 服务启动后，定时调用 Nacos 提供的心跳接口（默认 5s），证明自己还活着。如果 Nacos 超过一定时间（默认 15s）没有收到该请求，则将服务置为下线状态。若更长时间没有收到（默认 30s），则直接将该服务的信息从注册表中删除
- 服务启动后，定时调用 Nacos 提供的服务获取接口，获取当前存在注册表中的服务信息，并进行缓存，当需要调用别的服务时，进行客户端的负载均衡
- 服务停止时，调用 Nacos 提供的注销接口，将自身服务信息从 Nacos 的注册表中删除

Nacos 与其他服务注册中心对比：
![image.png](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/fcd6b63e223a31d82f4d06d53d633b57.png)

### Nacos 配置中心
提供用于存储配置和其他元数据的 key/value 存储，为分布式系统中的外部化配置提供服务器端和客户端支持

Nacos 会定时从配置中心拉取最新的配置，但是通过 @Value 注解获取的值并不会实时刷新，需要在所用到的地方的类上加上 @RefreshScope 注解才会实时刷新

Nacos 与其他配置中心对比：
![image.png](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/61795841562c04cdebab43bca4a13b0e.png)

## 微服务负载均衡器
### 负载均衡的两种方式：
- 客户端的负载均衡：客户端从注册中心拉取服务提供方的信息，自己实现负载均衡
- 服务端的负载均衡：在服务器上配置 Nginx，由 Nginx 选择调用哪一个服务

### 常见的负载均衡算法：
- 随机：通过随机选择服务进行调用，一般很少使用
- 轮询：负载均衡默认实现方式，请求来之后排队处理
- 加权轮训：通过对服务器性能的分析，给高配置，低负载的服务器分配更高的权重，均衡各个服务器的压力
- 地址 Hash：通过客户端请求的地址的 Hash 值取模块映射进行服务器调度
- 最小连接数：即使请求均衡了，压力不一定会均衡，最小连接数法就是根据服务器的情况，比如请求积压数等参数，将请求分配到当前压力最小的服务器上

### Ribbon（Nacos 默认引入，由 Netflix 提供）

#### Ribbon 提供的7种负载均衡策略：
##### 1.轮询策略（RoundRobinRule）
按照一定的顺序依次调用服务实例。比如一共有 3 个服务，第一次调用服务 1，第二次调用服务 2，第三次调用服务3，依次类推。此策略的配置设置如下：
```java
springcloud-nacos-provider: # nacos中的服务id
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule #设置负载均衡
```

##### 2.权重策略（WeightedResponseTimeRule）
根据每个服务提供者的响应时间分配一个权重，响应时间越长，权重越小，被选中的可能性也就越低。它的实现原理是，刚开始使用轮询策略并开启一个计时器，每一段时间收集一次所有服务提供者的平均响应时间，然后再给每个服务提供者附上一个权重，权重越高被选中的概率也越大。此策略的配置设置如下：
```java
springcloud-nacos-provider: # nacos中的服务id
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.WeightedResponseTimeRule
```

##### 3.随机策略（RandomRule）
从服务提供者的列表中随机选择一个服务实例。此策略的配置设置如下：
```java
springcloud-nacos-provider: # nacos中的服务id
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule #设置负载均衡
```

##### 4.最小连接数策略（BestAvailableRule）
也叫最小并发数策略，它是遍历服务提供者列表，选取连接数最小的⼀个服务实例。如果有相同的最小连接数，那么会调用轮询策略进行选取。此策略的配置设置如下：
```java
springcloud-nacos-provider: # nacos中的服务id
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.BestAvailableRule #设置负载均衡
```

##### 5.重试策略（RetryRule）
按照轮询策略来获取服务，如果获取的服务实例为 null 或已经失效，则在指定的时间之内不断地进行重试来获取服务，如果超过指定时间依然没获取到服务实例则返回 null。此策略的配置设置如下：
```java
ribbon:
  ConnectTimeout: 2000 # 请求连接的超时时间
  ReadTimeout: 5000 # 请求处理的超时时间
springcloud-nacos-provider: # nacos 中的服务 id
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule #设置负载均衡
```

##### 6.可用性敏感策略（AvailabilityFilteringRule）
先过滤掉非健康的服务实例，然后再选择连接数较小的服务实例。此策略的配置设置如下：
```java
springcloud-nacos-provider: # nacos中的服务id
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.AvailabilityFilteringRule
```

#### 7.区域敏感策略（ZoneAvoidanceRule）（默认策略）
根据服务所在区域（zone）的性能和服务的可用性来选择服务实例，在没有区域的环境下，该策略和轮询策略类似。此策略的配置设置如下：
```java
springcloud-nacos-provider: # nacos中的服务id
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.ZoneAvoidanceRule
```

#### 修改 Ribbon 负载均衡策略的两种方式
- 通过配置类，配置一个 bean：IRule
  - 如果客户端需要针对所有服务提供方都使用这个负载均衡配置，则直接放在 Spring 能扫描到的地方
  - 如果客户端只是针对某个服务提供方需要进行负载均衡策略配置，则只需要放在 Spring 扫描不到的地方，并在 SpringBoot 的启动类中使用 @RibbonClient 指定某个服务使用这个配置类
- 通过配置文件配置

### LoadBalancer（Spring Cloud 官方提供）
#### LoadBalancer 提供的两种负载均衡策略：
- 轮询
- 随机

## OpenFeign （远程调用组件）
### Java 项目中实现远程调用的几种方式：
- HttpClient：Apache Jakarta Common 下的子项目，提供高效的、最新的、功能丰富的支持 HTTP 协议的客户端变成工具包，并且支持 HTTP 协议最新版本和建议。HttpClient 相比传统 JDK 自带的 URLConnection，提升了易用性和灵活性，使客户端发送 HTTP 请求变得容易，提高了开发效率
- OkHttp：一个处理网络请求的开源项目，是安卓端最火的轻量级框架，由 Square 公司贡献，用于替代 HttpUrlConnection 和 Apache HttpClient。OkHttp 拥有简洁的 API、高效的性能，并且支持多重协议（HTTP/2 和 SPDY）
- HttpUrlConnection：HttpURLConnection 是 Java 的标准类，继承自 URLConnection，可用于向指定网站发送 GET、POST 请求。使用起来比较复杂，不像 HttpClient 那样容易使用
- RestTemplate：Spring 提供的用于访问 Rest 服务的客户端，RestTemplate 提供了多种便捷访问远程 HTTP 服务的方法，能够大大提高客户端的编写效率
- WebClient：相较于 RestTemplate，它是异步的。RestTemplate 会阻塞直到请求返回，才会进行下一步的操作。WebClient 可以先进行其他操作，并直接返回响应给前端，等请求返回了再处理请求的部分，相当于启动了一个异步线程去操作其他服务的方法
- Feign：Netflix 开发的声明式、模板化的 HTTP 客户端，可以帮助我们更便捷、优雅地调用 HTTP API，调用远程方法就像调用接口一样
- OpenFeign：对 Feign 进行了增强，使其支持 Spring MVC 注解，另外还整合了 Ribbon 和 Nacos，从而使得 Feign 的使用更加方便

## Sentinel（面向分布式服务架构的高可用防护组件）

### 核心功能
- 流控
  - 模式（添加集群流控规则时，不能设置模式，默认直接，所以关联模式和链路模式都是针对单机的，也就是说，如果有两台 order 服务，其中一台 order 服务限流了不会影响另一台正常访问）
    - 直接：只针对当前请求
    - 关联：需要额外配置一个关联的资源，如果关联的资源达到流控的阈值，对当前的资源进行流控，主动让出系统资源给关联的资源
    - 链路：当有两个接口同时需要调用某个资源，例如 A 接口 和 B 接口都需要调用 C 方法，如果此时想限制 A 接口，让出一些性能给 B 接口，那么就可以将方法 C 设置成资源，并且将 A 设置成链路模式中的入口资源，当资源 A 的调用达到阈值后，就会针对资源 A 进行流控，而资源 B 怎么调用都不会进行流控（如果 A、B、C 中的某个资源没有注册到 Sentinel，需要用 @SentinelResource 注解注册到 Sentinel，同时还需要在配置文件中配置 ```web-context-unify: false``` 将调用链路展开）
  - 阈值类型
    - QPS：在指定间隔时间内可以通过的请求数
    - 线程数：在指定间隔时间内，能够用来处理请求的线程数量
  - 方式（当阈值类型为 QPS 才能设置不同的方式，线程数默认快速失败）
    - 快速失败：超出的请求直接返回定义的流控异常
    - Warm Up：也是快速失败，不过一开始允许通过的请求个数为阈值的 1/3，随后慢慢变成阈值
    - 匀速排队：超出的请求进行排队，如果轮到请求且请求排队时间未达到配置的超时时间，处理请求，否则返回定义的流控异常
  - 针对的对象
    - 网关中配置的 route id
    - API 分组管理中添加的路径分组
    - controller 中定义的路由
    - 针对以上的三种对象又可以进行进一步的针对性配置（假设以每 2 秒只能有 1 个请求通过，否则快速失败的流控规则进行配置）
      - Client IP：若不设置值则针对所有 IP，如果设置值，则针对指定 IP
      - Remote Host：若不设置值针对所有域名，如果设置值，则针对指定域名
      - Header：设置对应 header，若不设置值则针对所有带有设置的 header 的请求，没有带有设置的 header 的请求不进行针对。如果设置值，则带有设置的 header 且值与设置值相等的请求才会进行流控，其他不处理
      - URL 参数：和 Header 匹配规则一样
      - Cookie：和 Header 匹配规则一样
- 熔断降级
  - 策略
    - 慢调用比例：设置最大响应时间（RT），比例阈值（0-1），熔断时长，最小请求数。假设最大响应时间为 100ms，比例阈值为 0.1，熔断时长为 10s，最小请求数为 10。当单位时间内请求数达到 10 个及以上，其中响应时间大于 100ms 的请求数大于 10 * 0.1 = 1 个时，就会进入熔断状态 10s，10s 内都会执行定义好的降级方法。当超过 10s 后，服务会进入半开状态，此时第一个进入的请求如果响应时间仍然超过 100ms，则直接进入熔断状态，而不会等满足设置的条件再进入。
    - 异常比例：设置比例阈值，熔断时长，最小请求数。和慢调用比例一样，当单位时间内请求数达到设置的最小请求数，且异常的请求数比例达到阈值后，就进入熔断状态。
    - 异常数：和异常比例一样，只不过不是设置比例阈值，而是直接设置异常数，熔断时长，最小请求数。
- 热点规则
  - 设置需要流控的参数的下标索引（下标从 0 开始，方法中的第一个参数的下标则为 0），并设置好单机阈值和统计窗口时长后，进入热点规则页面可以进行额外的配置。假设方法的第一个参数类型为 int，且值为 1 的数据时热点数据，那么就可以配置参数例外项的类型为 int，参数值为 1，并设置限流阈值。当访问这个接口时，第一个参数且值为 1 的请求的限流阈值为参数例外项的限流阈值，而其他请求则是一开始设置好的参数阈值
- 系统规则
  - LOAD（仅对 Linux/Unix-like 机器生效）：当系统 load1 超过设定的阈值，且系统当前的并发线程数超过估算的系统容量时才会触发系统保护（[参考：理解 linux 下的 load](https://www.cnblogs.com/gentlemanhai/p/8484839.html)）
  - RT：当单台机器上所有入口流量的平均 RT 达到阈值即触发系统保护（单位为 ms）
  - 线程数：当单台机器上所有入口流量的并发献策会给你数量达到阈值即触发系统保护
  - 入口 QPS：当单台机器上所有入口流量的 QPS 达到阈值即触发系统保护
  - CPU 使用率：当系统 CPU 使用率超过阈值（0-1）即触发系统保护

## Gateway
网关作为流量的入口，常用的功能包括路由转发、权限校验、限流等行为

### 核心功能：
- 路由：根据匹配的路由转发到对应的服务
- 断言：官方设定或自定义的路由匹配规则
  - 基于 Datetime 类型的断言工厂：-After=2019-12-32T23:59:59.789+08:00[Asia/Shanghai]
    - AfterRoutePredicateFactory: 接收一个日期参数，判断请求日期是否晚于指定日期
    - BeforeRoutePredicateFactory: 接收一个日期参数，判断请求日期是否早于指定日期
    - BetweenRoutePredicateFactory: 接收两个日期参数，判断请求日期是否在指定日期段内
  - 基于远程地址的断言工厂：-RemoteAddr=192.168.1.1/24
    - RemoteAddrRoutePredicateFactory：接收一个 IP 地址段，判断请求主机地址是否在地址段中
  - 基于 Cookie 的断言工厂：-Cookie=chocolate, ch.
    - CookieRoutePredicateFactory：接收两个参数，cookie 的名字和一个正则表达式，判断请求的 cookie 是否具有给定名称且值与正则表达式匹配
  - 基于 Header 的断言工厂：-Header=X-Request-Id, \d+
    - HeaderRoutePredicateFactory：接收两个参数，请求头名称和正则表达式，判断请求的 Header 是否具有给定名称且值与正则表达式匹配
  - 基于 Host 的断言工厂：-Host=**.testhost.org
    - HostRoutePredicateFactory：接收一个参数，主机名模式，判断请求的 Host 是否满足匹配规则
  - 基于 Method 请求方法的断言工厂：-Method=GET
    - MethodRoutePredicateFactory：接收一个参数，判断请求类型是否与指定的类型匹配
  - 基于 Path 请求路径的断言工厂：-Path=/foo/{segment}
    - PathRoutePredicateFactory：接收一个参数，判断请求的 URI 部分是否满足路径匹配规则
  - 基于 Query 请求参数的断言工厂：-Query=baz, ba.
    - QueryRoutePredicateFactory：接收两个参数，param 和正则表达式，判断 param 是否具有给定名称且值与正则表达式匹配
  - 基于路由权重的断言工厂：-Weight=group, 1
    - WeightRoutePredicateFactory：接收一个[组名，权重]，然后对于同一个组内的路由按权重转发
  - 自定义路由断言工厂：继承 AbstractRoutePredicateFactory类并重写 apply 方法逻辑。在 apply 方法中可以通过 exchange.getRequest() 拿到 ServerHttpRequest 对象，从而可以获取到请求的参数，请求方式，请求头等信息。需要注意的是，自定义断言工厂类的命名要以 RoutePredicateFactory 结尾
- 过滤器：转发请求前可以进行的一些操作
  - ![image.png](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/40056b5baee7b00d366caff3ae9bfac7.png)
  - ![image.png](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/d96261e631d4412862adf3302503276d.png)
  - ![image.png](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/f6be3debc8066ea1b1e83301a0903d03.png)
  - 自定义过滤器：继承 AbstractNameValueGatewayFilterFactory 类且自定义过滤器类名必须要以 GatewayFilterFactory 结尾
  - 全局过滤器：作用于所有路由转发请求前，implement GlobalFilter 类即可

## Seata（分布式事务）
### 常见分布式事务解决方案（以下都为 2PC 两阶段提交协议）：
- AT（Auto Transaction） - seata 阿里分布式事务框架
- TCC（Try Confirm Cancel） - 消息队列
- Saga
- XA

### 2PC 两阶段提交协议，分为 prepare 和 commit 阶段
- prepare：提交事务请求
  - 询问：协调者向所有参与者发送事务请求，询问是否可执行事务操作，然后等待各个参与者的响应。
  - 执行：各个参与者接收到协调者事务请求后，执行事务操作(例如更新一个关系型数据库表中的记录)，并将 undo 和 redo 信息记录到事务日志中
  - 响应：如果参与者成功执行了事务并写入 undo 和 redo 信息，则向协调者返 yes 响应，否侧返 no 响应，当然，参与者也可能宕机，从而不会返回响应
- commit：执行事务提交
  - commit：协调者向所有参与者发送 commit 请求
  - 事务提交：参与者收到 commit 请求后，执行事务提交，提交完成后释放事务执行期占用的所有资源
  - 反馈结果：参与者执行事务提交后向协调者发送 ack 响应
  - 完成事务：接收到所有参与者的 ack 响应后，完成事务提交
- rollback：在执行 prepare 步骤过程中，如果某些参与者执行事务失败、宕机或与协调者之间的网络中断，那么协调者就无法收到所有参与者的 yes 响应，或者某个参与者返回了 no 响应。此时，协调者就会进入回退流程，对事务进行回退
  - rollback：协调者向所有参与者发送 rollback 请求
  - 事务回滚：参与者收到 rollback 后，使用 prepare 阶段的 undo 日志执行事务回滚，完成后释放事务执行期占用的所有资源
  - 反馈结果：参与者执行事务回滚后向协调者发送 ack 响应
  - 中断事务：接收到所有参与者的 ack 响应后，完成事务中断

### 2PC 模式流程：
![image20230320113326678.png](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/0b6c9113caf957e58732f62a7b78e8d3.png)

### 2PC 存在的问题：
- 同步阻塞：所有参与者等待协调者命令时，参与者无法进行其他操作
- 依赖协调者：2PC 中一切请求来自协调者，如果协调者宕机，会导致参与者一直阻塞，占用事务资源
- 环境可靠性依赖：如果参与者和协调者网络中断，会导致协调者无法收到所有参与者的响应，导致协调者等待一定时间后，超时并触发事务中断
- 最终提交事务的时候可能出现异常，无法保证 100% 事务成功率，只能尽量保证。例如：如果未收到参与者的答应，进行补偿机制，比如重试。重试到一定次数后可以提醒开发人员。

### AT 模式流程：
底层实现：
- 事务协调者（Transaction Coordinator）：维护全局和分支事务的状态，驱动全局事务提交或回滚
- 事务管理器（Transaction Manager）：定义全局事务的范围，开始全局事务、提交或回滚全局事务
- 资源管理器（Resource Manager）：管理分支事务处理的资源，与 TC 交谈以注册分支实物和报告分支事务的状态，并驱动分支事务提交或回滚

#### 一阶段：
![1697302979167.jpg](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/e2bbb99544ce8153506e1a9a6401b6e2.jpg)
#### 二阶段提交：
![1697302990198.jpg](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/ea24043e7aebb9a9133bc609cdc79c6f.jpg)
#### 二阶段回滚：
![1697303017539.jpg](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/3a9a88c5aab32b264cb7696dabd27938.jpg)

### 优点：
- 对业务代码无侵入
- 较高的性能：AT模式在每个参与者的本地事务中执行操作，避免了全局锁和阻塞的问题，提高了系统的并发性能。
- 简化的实现：相对于XA模式，AT模式的实现相对简单，不需要涉及全局事务协调器，减少了开发和维护的复杂性。
- 本地事务的独立性：每个参与者在本地事务管理器中管理自己的事务，可以独立控制和优化本地事务的执行。
### 缺点：
- 弱一致性：AT 模式对一致性的要求相对较低，可能会出现数据不一致的情况。在某些场景下，可能需要更高的一致性保证，需要考虑其他分布式事务处理模式。
- 隔离级别限制：由于 AT 模式依赖于本地事务的隔离性，参与者的隔离级别受限于本地事务管理器支持的隔离级别，可能无法满足某些特定的隔离需求。
- 回滚异常：由于 AT 模式在一阶段结束后就已经让事务参与者提交本地事务并释放锁资源，导致二阶段进行回滚阶段前有可能某个事务参与者的参与本次事务的数据已被其他事务进行修改，回滚时由于和 undo_log 表中存储的原数据不一致导致一直进行回滚操作并且失败，需要开发者介入手动进行回滚操作

## TCC 模式流程：
![image20230320141325422.png](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/2b20f649d03d6d03abea9c6a60a8836a.png)

### 优点：
- 没有全局锁，效率高
### 缺点：
- 需要自己实现 try、confirm、cancel 三个操作，代码侵入性比较强

### 可靠消息最终一致性方案
![image20230320144533796.png](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/a0304b4ac04fc1dc528a7dfd9d02da6b.png)

## SkyWalking（性能监视工具）
底层实现：
- 服务引入 SkyWalking 依赖，其中的 SkyWalking Agent 负责收集各种监控数据
- SkyWalking OapService 负责处理监控数据。比如接收 SkyWalking Agent 的监控数据并存储在数据库中（MySQL、ElasticSearch 等）；接收 SkyWalking WebApp 的前端请求，从数据库查询数据并返回给前端。SkyWalking OapService 通常以集群的形式存在。SkyWalking OapService 会暴露 11800 和 12800 端口，11800 端口用于手机监控数据，12800 端口用于接收前端请求
- SkyWalking WebApp 负责展示数据
