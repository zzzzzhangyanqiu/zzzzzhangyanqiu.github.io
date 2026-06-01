---
title: Spring Cloud Gateway网关无法返回响应数据
tags:
  - Spring Cloud
date: 2020-10-25
categories:
 - Spring
---

### 问题产生

首先介绍一下项目背景，服务端使用`haproxy`做代理，`Spring Cloud Gateway`做网关层，`c#`端使用`HttpWebRequest`调用后端接口。问题来源于`c#`同事反映后台接口全部报错`503 bad gateway`，问题来的很奇怪，因为同样的接口和参数使用`postman`访问就是好的，而且前端同事也访问没问题。于是开始了漫长的问题排查之旅（没想到这一排查就是两天）

### 问题排查

首先在本地写了一段`c#`测试代码，如下

```c#
        static void Main(string[] args)
        {
            string url = "";
            HttpWebRequest request = (HttpWebRequest)WebRequest.Create(url);
            request.ContentType = "application/json;charset=UTF-8";
            request.Method = "POST";
            request.Accept = "application/json, text/plain, */*";
            request.UserAgent = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.111 Safari/537.36";
            request.Headers.Add("Accept-Language", "zh-CN");

            using (var streamWriter = new StreamWriter(request.GetRequestStream()))
            {
                string json = new JavaScriptSerializer().Serialize(new
                {
                    token = "xxxxxxx",
                    userId = 7,             
                    projectId = 2,         
                    loginType = "web"
                });

                streamWriter.Write(json);
            }
            var response = (HttpWebResponse)request.GetResponse();
            using (var streamReader = new StreamReader(response.GetResponseStream()))
            {
                var result = streamReader.ReadToEnd();
                Console.WriteLine(result);
            }
            Console.ReadLine();
        }
```

开始访问服务器接口，`OK`百分百复现，然后开启了`haproxy`和`Spring Cloud Gateway`的日志，开始继续访问，也没有相关日志打出。

先使用排除法，将`haproxy`的地址直接改为`Spring Cloud Gateway`的地址，这次不报错了，但是程序会一直阻塞住直到抛出超时异常，那基本就可以确定了是`Spring Cloud Gateway`的问题导致的`haproxy`。但是让人奇怪的是`Spring Cloud Gateway`并没有输出任何错误日志。

尝试使用`jstack`分析是否有死锁情况，发现了如下输出

```java
"Signal Dispatcher" #4 daemon prio=9 os_prio=0 tid=0x00007f385417d000 nid=0xc7388 runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
	- None

"Finalizer" #3 daemon prio=8 os_prio=0 tid=0x00007f385414a000 nid=0xc7387 in Object.wait() [0x00007f3841090000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:143)
	- locked <0x00000000e0264f00> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:164)
	at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:209)

   Locked ownable synchronizers:
	- None

"Reference Handler" #2 daemon prio=10 os_prio=0 tid=0x00007f3854145800 nid=0xc7386 in Object.wait() [0x00007f3841191000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	at java.lang.Object.wait(Object.java:502)
	at java.lang.ref.Reference.tryHandlePending(Reference.java:191)
	- locked <0x00000000e0264f58> (a java.lang.ref.Reference$Lock)
	at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:153)

   Locked ownable synchronizers:
	- None
```

心里一阵激动，以为要找到问题的原因了，但是上网了解了一下，发现这些输出跟问题一点关系都没有，看来还需要再尝试新的办法

查看`Spring Cloud Gateway`的`issue`，并且搜索了`blocking request`类似的关键字后，也没有找到相关的信息，后来忽然想到，不管`c#`代码如何，都是通过`http`协议调用的， 所以使用`fiddler`开始抓包，奇怪的是直接访问`Spring Cloud Gateway`的时候，`fiddler`并没有抓到任何请求，更改请求地址为`haproxy`后，`fiddler`上终于如愿以偿的显示了本次请求。随后使用`postman`又发起了一次正常的请求，使用`fidder`比较两次请求，发现`c#`的请求头上多了一个`Expect:100-continue`，这里简单说下含义，大概意思就是`c#`不会直接就发起`POST`请求, 而是会分为2步：

- 发送一个请求，包含一个`Expect:100-continue`，询问`Server`是否愿意接受数据。
- 接收到`Server`返回的`100-continue`应答以后, 才把数据`POST`给`Server`。

对于`100-continue`这个字段，`RFC`文档（http://www.w3.org/Protocols/rfc2616/rfc2616-sec8.html#sec8.2.3）是这么解释的：它可以让客户端在发送请求数据之前去判断服务器是否愿意接收该数据，如果服务器愿意接收，客户端才会真正发送数据。这么做的原因是如果客户端直接发送请求数据，但是服务器又将该请求拒绝的话，这种行为将带来很大的资源开销。

这时候心里已经觉得大概率是这个原因导致的，尝试了一下，问题终于如愿以偿的解决了。

### 解决办法

#### （1）阻止Gateway转发Expect header请求参数

可以在`Spring Cloud Gateway`中阻止转发`Expect`请求参数，在路由中添加如下代码即可：

```java
.filters((t)->t.removeRequestHeader("Expect"))
```

#### （2）在application.yml 配置文件中添加配置

```yml
spring: 
	cloud: 
		gateway: 
			routes: 
				- id: removerequestheader_route 
				  uri: https://example.org 
				  filters: 
				  	## 去掉Expect请求头
				  	- RemoveRequestHeader=Expect
```

#### （3）在发起请求时，取消Expect请求参数

在`c#`端添加如下代码

```c#
request.ServicePoint.Expect100Continue = false;
```

### 总结

1. 其实除了上述解决步骤，还有很多，比如百度、必应、`google`等，还找过运维同事，可惜都没有解决问题
2. 正常情况下`Spring Boot`是可以处理这个问题的，猜测可能是因为`Spring Cloud Gateway`是使用`Netty`实现的，所以算是`Netty`的问题，而之前一直都被`Spring Cloud Gateway`带偏了，所以没有找到相关资料
3. 问题解决之后，还是觉得自己技术不够强，早就应该想到抓包然后查看差异的，而且也暴露了自己对于网络的薄弱，继续学习吧













