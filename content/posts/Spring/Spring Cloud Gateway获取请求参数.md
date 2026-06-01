---
title: Spring Cloud Gateway获取请求参数
tags:
  - Spring Cloud
date: 2020-09-05
categories:
 - Spring
---

#### 问题描述

最近遇到了一个从`Spring Cloud Gateway`（后面简称`SCG`）获取请求体并进行校验的问题，被折磨了很久，网上大部分文章也都是千篇一律，各种坑。此处做记录，便于日后查看。也希望能够解决其他人同样的问题。我们都知道，`request body`中的数据读过一次之后就不能再读了，之前实现的方式是创建`request`包装对象缓存内容，如果`request body`中的数据获取为空则从`requestParam`里面获取。由于`SCG`采用的是`WebFlux`的方式，没有`httpServletRequest`给我们使用了，取而代之的是`ServerWebExchange`（这里面可以获取到`ServerHttpRequest`，注意区别）。

#### 问题解决

期间翻看了各种资料、官方文档、`GitHub`等，大概得出来以下几个结论

1. `SCG`中获取请求体的限制也是一样的，只能读一次，如果想读多次需要再次封装`Request`对象，过程及其麻烦，而且个人觉得很不优雅
2. `WebFlux`是非阻塞式的操作，不能用原来阻塞式的思维来处理
3. 常规的传值方式大概有`POST application/json、POST application/x-www-form-urlencoded、POST multipart/form-data、GET Query Params`，这里比较幸运的是，项目由于是前后端分离，所以基本上只有`application/json`的请求方式
4. 不同的请求方式在`SCG`获取参数的方式不一样

话不多说，直接上代码

1. 如果是`multipart/form-data`请求方式

   ```java
   public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
       //multipart/form-data方式
       return exchange.getMultipartData().flatMap(x -> {
           //获取该key下的第一个值
           Part value = x.getFirst("key");
   
           //value
           System.out.println(((FormFieldPart) value).value());
   
           //TODO 业务处理
   
           if(校验不通过) {
               //ResultUtil类的内容在下面
               return ResultUtil.errorResult(exchange, "异常描述");
           }
   
           return chain.filter(exchange);
       });
   }
   ```

   

2. 如果是`Query Params`

   ```java
   public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
   
       ServerHttpRequest request = exchange.getRequest();
   
       MultiValueMap<String, String> queryParams = request.getQueryParams();
   
       //获取该key下的第一个值
       String key = queryParams.getFirst("key");
   
       //value
       System.out.println(key);
   
       if (校验不通过) {
           //ResultUtil类的内容在下面
           return ResultUtil.errorResult(exchange, "异常描述");
       }
   
       return chain.filter(exchange);
   }
   ```

   

3. 如果是`application/x-www-form-urlencoded`的请求方式

   与`application/json`方式相同，只不过细节点略有不同，后面一起介绍

4. 如果是`application/json`

   获取`application/json`比较特殊，一开始并没有找到相关的解决方案，并且他是存在于`request body`中的，前面也说过了`request body`的数据读过一次就不能再读了，如果读取了还需要重新封装`request`，比较麻烦。后面在`GitHub issue`中看到了说`SCG`已经自带了一个过滤器来处理此问题（不过官方文档并没有说明），也就是`ReadBodyPredicateFactory`。进入到此类的源码，在`applyAsync`方法中可以看到相关代码

   ```java
   return ServerWebExchangeUtils.cacheRequestBodyAndRequest(exchange,
   							(serverHttpRequest) -> ServerRequest
   									.create(exchange.mutate().request(serverHttpRequest)
   											.build(), messageReaders)
   									.bodyToMono(inClass)
                                       //这里缓存了请求内容，key为CACHE_REQUEST_BODY_OBJECT_KEY
   									.doOnNext(objectValue -> exchange.getAttributes().put(
   											CACHE_REQUEST_BODY_OBJECT_KEY, objectValue))
   									.map(objectValue -> config.getPredicate()
   											.test(objectValue)));
   ```

   那么我们在使用的时候直接获取`key`就可以了

```java
//此字段从ReadBodyPredicateFactory复制出来  
private static final String CACHE_REQUEST_BODY_OBJECT_KEY = "cachedRequestBodyObject";   

public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {

    ServerHttpRequest request = exchange.getRequest();

    MediaType contentType = request.getHeaders().getContentType();

    String attribute = exchange.getAttribute(CACHE_REQUEST_BODY_OBJECT_KEY);

    //application/x-www-form-urlencoded
    if (MediaType.APPLICATION_JSON.isCompatibleWith(contentType) || MediaType.APPLICATION_JSON_UTF8.isCompatibleWith(contentType)) {
        //如果是application/json 直接转换为json就可以了
        JSONObject jsonObject = JSONObject.parseObject(attribute);

        System.out.println(jsonObject.getString("key"));

        //TODO 业务处理
        if (校验不通过) {
            //ResultUtil类的内容在下面
            return ResultUtil.errorResult(exchange, "异常描述");
        }

    } else if(MediaType.APPLICATION_FORM_URLENCODED.isCompatibleWith(contentType)) {
        //application/x-www-form-urlencoded

        /*
         这里要注意
         与application/json不同的是
         这里获取到的值是这种格式：key=value&key1=value1  需要自行处理，本篇文章不在处理
        */

        //TODO 业务处理
        if (校验不通过) {
            //ResultUtil类的内容在下面
            return ResultUtil.errorResult(exchange, "异常描述");
        }

    }

    return chain.filter(exchange);
}
```

然后在配置文件中加入如下配置

```yml
spring:
  cloud:
    routes:
        - id: test
        uri: no://op
        predicates:
        - Path=/test/**
        - name: ReadBodyPredicateFactory  #使用ReadBodyPredicateFactory断言，将body读入缓存
            args:
            inClass: '#{T(String)}'
            ##  这里需要在java config类中配置一个bean
            predicate: '#{@bodyPredicate}' #注入实现predicate接口类
```

```java
public class GateWayConfig {

    /**
     * 用于readBody断言，可配置到yml
     **/
    @Bean
    public Predicate bodyPredicate() {
        return o -> true;
    }
}
```

这样一来，我们每次都是从缓存中读取的，不需要重新封装`request`，也不会引起什么异常了

`ResultUtil`类代码如下

```java
public class ResultUtil {

    public static Mono<Void> errorResult(ServerWebExchange exchange, String message) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.OK);
        byte[] bytes = error(message).getBytes(StandardCharsets.UTF_8);
        DataBuffer buffer = response.bufferFactory().wrap(bytes);
        return exchange.getResponse().writeWith(Flux.just(buffer));
    }

    public static String error(String message) {
        JSONObject jo = new JSONObject();
        jo.put("code", -1);
        jo.put("msg", message);
        return jo.toJSONString();
    }
}

```



不过此种方案有一个弊端，如果`request body`的内容是空，`ReadBodyPredicateFactory`会直接返回`false`，解决方案就是我们新创建一个类，把`ReadBodyPredicateFactory`的`applyAsync()`方法稍作改动，这里只展示关键代码，如下

```java
public class GwReadBodyPredicateFactory extends AbstractRoutePredicateFactory<GwReadBodyPredicateFactory.Config> {
    //省略...
    @Override
    @SuppressWarnings("unchecked")
    public AsyncPredicate<ServerWebExchange> applyAsync(GwReadBodyPredicateFactory.Config config) {
        
         return new AsyncPredicate<ServerWebExchange>() {
            @Override
            public Publisher<Boolean> apply(ServerWebExchange exchange) {
				
                if (cachedBody != null) {
                   //省略...
                } else {
                    return ServerWebExchangeUtils.cacheRequestBodyAndRequest(exchange,
                            (serverHttpRequest) -> ServerRequest
                                    .create(exchange.mutate().request(serverHttpRequest)
                                            .build(), messageReaders)
                                    .bodyToMono(inClass)
                                       .doOnNext(objectValue -> exchange.getAttributes().put(
                                            CACHE_REQUEST_BODY_OBJECT_KEY,
                                            objectValue.toString()))
                                    .map(objectValue -> config.getPredicate()
                                            .test(objectValue))
                                    //主要修改这里，总是返回true
                                    .thenReturn(true));
                }
        
    }
    
}
```

然后将修改配置文件

```yml
spring:
  cloud:
    routes:
        - id: test
        uri: no://op
        predicates:
        - Path=/test/**
        - name: GwReadBodyPredicateFactory  #这里修改成刚刚定义的类
            args:
            inClass: '#{T(String)}'
            ##  这里需要在java config类中配置一个bean
            predicate: '#{@bodyPredicate}' #注入实现predicate接口类
```



`OK`，问题解决

#### 小结

可以看到在`SCG`中获取`request`请求参数也是非常麻烦，而且一系列操作也比较浪费性能，所以建议大家最好不要这么做，而是将`token`等信息放在`request header`中