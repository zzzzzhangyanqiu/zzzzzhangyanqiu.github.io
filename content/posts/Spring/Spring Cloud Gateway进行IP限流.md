---
title: Spring Cloud Gateway进行IP限流
tags:
  - Spring Cloud
date: 2020-09-20
categories:
 - Spring
---

#### 前言

限流指限制数据流量的速率，大白话来讲就是说通过某种方式控制请求数量，如果请求数量高过预设阈值，则拒绝后续请求，限流是高并发场景下保护系统的一种有效手段，首先了解一下常用的几种限流算法

#### 限流算法

##### 1、计数器（固定窗口）算法

计数器算法比较简单，设置一个单位时间和一个阈值，如果有请求则使用计数器`+1`，如果计数器到了阈值则拒绝该请求，到了下一个单位时间则重置计数器。

该算法有一个很大的问题，也就是临界问题。比如我们设置的阈值是`120`，单位时间为`1min`，那么在`0:55-1:00`和`1:00-1:05`这两段时间内，分别涌入了`120`个请求，虽然没有超出限制量，但是在`10`秒钟内总共接收到了`240`个请求，已经远远超出了服务器的处理能力。所以该算法的弊端就是控制的不够精细。

##### 2、计数器（滑动窗口）算法

滑动窗口算法中的滑动窗口就是单位时间，原理为把一个单位时间平均分成若干个格子，窗口每次滑动一个格，并把前面的格丢弃掉，每次只需要计算当前窗口内的所有格子请求总数就可以了，如下图所示

![image-20200924165120726](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/PicGo/image-20200924165120726.png)

##### 3、漏桶算法

漏桶算法比较简单，可以结合下图理解，水（请求）进入漏桶中，漏桶以匀速流出水（服务器处理请求），如果漏桶满了，这时候新流入的水（新接收到的请求）则会被丢弃掉

![image-20200924165538535](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/PicGo/image-20200924165538535.png)

##### 4、令牌桶算法

令牌桶算法和漏桶算法正好是相反的，大概意思是设置一个桶，系统会匀速向桶内装填令牌，服务器接收到请求时会从桶中拿一个令牌，如果接收到请求时桶内没有令牌，则丢弃掉该请求。

#### 使用Spring Cloud Gateway限流

做为网关的技术选型，`Spring`官方已经给我们自带了限流器[RequestRateLimiter GatewayFilter Factory](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.1.5.RELEASE/single/spring-cloud-gateway.html#_requestratelimiter_gatewayfilter_factory)，该限流器依赖`redis`记录相关信息，使用令牌桶算法

首先在`pom`文件中引入`gateway`和`redis`依赖

```xml

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

在`Java config`中注册`KeyResolver`，该类会作为流量的标识

```java
@Bean
public KeyResolver ipKeyResolver() {
    //使用hostname限流
    //return exchange -> Mono.just(exchange.getRequest().getRemoteAddress().getHostName());
    //IP限流
    return exchange -> Mono.just(exchange.getRequest().getRemoteAddress().getAddress().getHostAddress());
    //使用其它信息作为流量标记也是一样的，如使用userId作为标记，这里就直接从request中获取对应的参数就好了
}
```

最后修改配置文件

```yml
spring:
  application:
    name: gateway
  cloud:
    gateway:
      routes:
        - id: test
          uri: no://op
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 1		##令牌桶填充速率
                redis-rate-limiter.burstCapacity: 3	##令牌桶总容量
                key-resolver: "#{@ipKeyResolver}"  ##这里配置的是刚才我们注册的bean名称
```

随后启动启动项目，同时发送五个请求，就可以看到其中有两个请求返回的是`HTTP 429 - Too Many Requests`

需要注意的是，如果该限流器的规则不能满足，你也可以可以定义一个实现了`RateLimiter`接口的`bean`，然后在配置文件中指定

```yml
spring:
  application:
    name: gateway
  cloud:
    gateway:
      routes:
        - id: test
          uri: no://op
          filters:
            - name: RequestRateLimiter
              args:
                rate-limiter: "#{@myRateLimiter}"   ##自定义bean名称
                key-resolver: "#{@userKeyResolver}"	 ##keyResolver
```



