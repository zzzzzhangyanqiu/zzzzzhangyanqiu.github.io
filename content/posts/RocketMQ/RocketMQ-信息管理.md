---
title: RocketMQ-信息管理
date: 2024-03-13
categories:
 - RocketMQ
---

基本上每个消息队列中间件都会有一个**命名服务**的组件用来服务发现、服务注册、存储Topic信息等。如Kafka中使用ZooKeeper、RabbitMQ中的Exchange（类似）等。RocketMQ中的命名服务叫做NameServer。

## 主要功能

对于消息队列的信息存储，我整理了下面几个问题。

如何存储信息？

如何更新已经存储的信息？

集群模式下如何存储信息？

### 信息管理

NameServer主要负责维护Broker信息、Topic信息、队列信息等

**Broker信息**

当Broker启动时，会向NameServer注册自己的地址，NameServer存储到内存变量中，变量为Map集合，类似于下面的结构

```json
{
    "broker-a": {
        "brokerAddrs": {
            0: "10.213.0.12:10911"      //key为brokerID，value为broker地址
        },
        "brokerName": "broker-a",       //broker名称
        "cluster": "DefaultCluster",    //集群名称
        "enableActingMaster": false     //是否启用代理主机，用于旧版本兼容
    }
}
```

**Topic信息**

创建Topic后，也会由NameServer存储，与Broker信息一样存储在内存变量Map集合中

```json
{
    "TopicTest": {          //topic名称
        "broker-a": {
            "brokerName": "broker-a",   //所在的broker
            "perm": 6,
            "readQueueNums": 4,     //写入队列的数量
            "topicSysFlag": 0,
            "writeQueueNums": 4     //读取队列的数量
        }
    }
}
```

**核心代码**

Broker注册时，由Broker向NameServer发送请求，参数大致如下

```json
{
    "bodyCrc32": 23305878,
    "brokerAddr": "ip:port",
    "brokerId": 0,
    "brokerName": "broker-a",       
    "clusterName": "DefaultCluster",    
    "compressed": false,   
    "enableActingMaster": false,
    "haServerAddr": "ip:port"
}
```



其中注册Broker部分的核心代码如下（精简），会保存Broker信息、Broker存活状态、Topic及分区信息等

```java
//DefaultRequestProcessor.java
RegisterBrokerResult result = this.namesrvController.getRouteInfoManager().registerBroker(
);

//RouteInfoManager.java
public RegisterBrokerResult registerBroker(
    final String clusterName,
    final String brokerAddr,
    final String brokerName,
    final long brokerId,
    final String haServerAddr,
    final String zoneName,
    final Long timeoutMillis,
    final Boolean enableActingMaster,
    final TopicConfigSerializeWrapper topicConfigWrapper,
    final List<String> filterServerList,
    final Channel channel) {
    RegisterBrokerResult result = new RegisterBrokerResult();
    try {
        //写锁
        this.lock.writeLock().lockInterruptibly();

        //init or update the cluster info
        Set<String> brokerNames = ConcurrentHashMapUtils.computeIfAbsent((ConcurrentHashMap<String, Set<String>>) this.clusterAddrTable, clusterName, k -> new HashSet<>());
        brokerNames.add(brokerName);

        boolean registerFirst = false;

        //注册broker
        BrokerData brokerData = this.brokerAddrTable.get(brokerName);
        if (null == brokerData) {
            registerFirst = true;
            brokerData = new BrokerData(clusterName, brokerName, new HashMap<>());
            this.brokerAddrTable.put(brokerName, brokerData);
        }

        //主备切换相关
        //Switch slave to master: first remove <1, IP:PORT> in namesrv, then add <0, IP:PORT>
        //The same IP:PORT must only have one record in brokerAddrTable
        brokerAddrsMap.entrySet().removeIf(item -> null != brokerAddr && brokerAddr.equals(item.getValue()) && brokerId != item.getKey());

        if (null != topicConfigWrapper && (isMaster || isPrimeSlave)) {

            ConcurrentMap<String, TopicConfig> tcTable =
                topicConfigWrapper.getTopicConfigTable();

            if (tcTable != null) {
                //处理topic及分区数据，存在topicQueueTable中
                for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {
                    if (registerFirst || this.isTopicConfigChanged(clusterName, brokerAddr,
                        this.createAndUpdateQueueData(brokerName, topicConfig);
                    }
                }

            }
        }

        //broker存活列表
        BrokerAddrInfo brokerAddrInfo = new BrokerAddrInfo(clusterName, brokerAddr);
        BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddrInfo,
            new BrokerLiveInfo(
                System.currentTimeMillis(),
                timeoutMillis == null ? DEFAULT_BROKER_CHANNEL_EXPIRED_TIME : timeoutMillis,
                topicConfigWrapper == null ? new DataVersion() : topicConfigWrapper.getDataVersion(),
                channel,
                haServerAddr));
        if (null == prevBrokerLiveInfo) {
            log.info("new broker registered, {} HAService: {}", brokerAddrInfo, haServerAddr);
        }

    } catch (Exception e) {
        log.error("registerBroker Exception", e);
    } finally {
        this.lock.writeLock().unlock();
    }

    return result;
}

```

### 信息更新

**30秒心跳机制**

Broker每隔30秒向NameServer发送自己的心跳信息，用来更新自己的存活状态，代码如下（精简）

```java
        scheduledFutures.add(this.scheduledExecutorService.scheduleAtFixedRate(new AbstractBrokerRunnable(this.getBrokerIdentity()) {
            @Override
            public void run0() {
                try {
                    BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
                } catch (Throwable e) {
                    BrokerController.LOG.error("registerBrokerAll Exception", e);
                }
            }
        }, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS));
```

虽然是心跳，但是可以看到，代码中是重新走了一次Broker注册的逻辑，在Broker注册的逻辑中更新对应的存活状态

本质上就是使用ScheduledExecutorService启动了一个定时任务，不断执行BrokerController.this.registerBrokerAll()方法

核心在于定时周期：Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)，

这段代码保证了发送心跳的时间最小为10秒，最大为60秒，既不会太频繁浪费性能，又不会太久导致误判

其中BrokerConfig.registerNameServerPeriod默认值为30秒

```java
    /**
     * This configurable item defines interval of topics registration of broker to name server. Allowing values are
     * between 10,000 and 60,000 milliseconds.
     */
    private int registerNameServerPeriod = 1000 * 30;
```



**120秒故障检查机制**

NameServer中有个定时任务，定期扫描自己的本地Broker存活列表，将过期的Broker从存活列表中踢出

```java
//NamesrvController.java
this.scheduledExecutorService.scheduleAtFixedRate(NamesrvController.this.routeInfoManager::scanNotActiveBroker, 5, 10, TimeUnit.SECONDS);


//RouteInfoManager.java
private final static long BROKER_CHANNEL_EXPIRED_TIME = 1000 * 60 * 2;

public int scanNotActiveBroker() {
    int removeCount = 0;
    Iterator<Entry<String, BrokerLiveInfo>> it = this.brokerLiveTable.entrySet().iterator();
    while (it.hasNext()) {
        Entry<String, BrokerLiveInfo> next = it.next();
        long last = next.getValue().getLastUpdateTimestamp();
        //上次更新时间 + 120秒如果小于当前时间，则认为是过期，将此节点移除
        if ((last + BROKER_CHANNEL_EXPIRED_TIME) < System.currentTimeMillis()) {
            RemotingUtil.closeChannel(next.getValue().getChannel());
            it.remove();
            log.warn("The broker channel expired, {} {}ms", next.getKey(), BROKER_CHANNEL_EXPIRED_TIME);
            this.onChannelDestroy(next.getKey(), next.getValue().getChannel());

            removeCount++;
        }
    }

    return removeCount;
}
```



在RocketMQ5.0版本后，这里的逻辑发生了一些变化

```java
//NamesrvController.java
//变化1：定时任务由原来的每10秒执行一次变为scanNotActiveBrokerInterval配置
//scanNotActiveBrokerInterval默认值为5秒
        this.scanExecutorService.scheduleAtFixedRate(NamesrvController.this.routeInfoManager::scanNotActiveBroker,
            5, this.namesrvConfig.getScanNotActiveBrokerInterval(), TimeUnit.MILLISECONDS);

//RouteInfoManager.java
public void scanNotActiveBroker() {
  try {
    log.info("start scanNotActiveBroker");
    for (Entry<BrokerAddrInfo, BrokerLiveInfo> next : this.brokerLiveTable.entrySet()) {
      long last = next.getValue().getLastUpdateTimestamp();
      //变化2：活动时间由写死的120秒改为了BrokerLiveInfo.heartbeatTimeoutMillis
      long timeoutMillis = next.getValue().getHeartbeatTimeoutMillis();
      if ((last + timeoutMillis) < System.currentTimeMillis()) {
        RemotingHelper.closeChannel(next.getValue().getChannel());
        log.warn("The broker channel expired, {} {}ms", next.getKey(), timeoutMillis);
        this.onChannelDestroy(next.getKey());
      }
    }
  } catch (Exception e) {
    log.error("scanNotActiveBroker exception", e);
  }
}


//重新注册的逻辑
BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddrInfo,
    new BrokerLiveInfo(
        System.currentTimeMillis(),
        //BrokerLiveInfo.heartbeatTimeoutMillis会在重新注册时判断，如果传了timeoutMillis则使用
        //没传则还是默认的120s
        //timeoutMillis就是注册时broker传过来的requestHeader.getHeartbeatTimeoutMillis()
        timeoutMillis == null ? DEFAULT_BROKER_CHANNEL_EXPIRED_TIME : timeoutMillis,
        topicConfigWrapper == null ? new DataVersion() : topicConfigWrapper.getDataVersion(),
        channel,
        haServerAddr));
```

关于注册时传的requestHeader.getHeartbeatTimeoutMillis参数，涉及到RocketMQ的一个新功能，**Slave代理Master模式**

如果开启了此模式，那心跳的过期时间需要缩短为10秒

```java
//BrokerController.class  注册Broker时候传的参数，也就是requestHeader.getHeartbeatTimeoutMillis()
//enableSlaveActingMaster默认为false
//brokerNotActiveTimeoutMillis默认为10秒
this.brokerConfig.isEnableSlaveActingMaster() ? this.brokerConfig.getBrokerNotActiveTimeoutMillis() : null
```

综上所述，如果从Broker允许故障时代理主Broker，那么心跳的过期时间为10秒，默认不允许代理主Broker，心跳的过期时间为120秒。NameServer每隔scanNotActiveBrokerInterval（默认值为5秒）会检查一下本地Broker存活列表，如果超过时间没有心跳，则会将对应的Broker节点踢出



**轻量级心跳**

RocketMQ5.0版本后，新推出了**Slave代理Master模式**和**Controller 模式**（先简单理解为两种集群部署模式），在该两种模式下，RocketMQ需要及时发现Broker的上下线动作及对应信息更新。原本Broker与NameServer的心跳依赖于registerBroker操作，但是该方法操作逻辑太重，而且注册间隔过于长，所以新增了一个轻量级的心跳

```java
//BrokerController.java
//如果开启了Slave代理Master模式
if (this.brokerConfig.isEnableSlaveActingMaster()) {
    //发送轻量级心跳
    scheduleSendHeartbeat();
}

//如果开启了controller模式
if (this.brokerConfig.isEnableControllerMode()) {
    //发送轻量级心跳
    scheduleSendHeartbeat();
}
```

BrokerController.scheduleSendHeartbeat()方法比较简单，通过java.util.concurrent.ScheduledExecutorService#scheduleAtFixedRate()设置定时心跳，间隔为1秒一次。

Broker的轻量级心跳会携带自身的clusterName、brokerAddr、brokerName等信息发送给NameServer

NameServer接到请求后，只有一个操作：将broker的上次活动时间改为当前时间。相对比与registerBroker逻辑，能够快速更新数据并响应请求

```java
//org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager#updateBrokerInfoUpdateTimestamp
public void updateBrokerInfoUpdateTimestamp(final String clusterName, final String brokerAddr) {
    //更新Broker存活列表的Broker活动时间
    BrokerAddrInfo addrInfo = new BrokerAddrInfo(clusterName, brokerAddr);
    BrokerLiveInfo prev = this.brokerLiveTable.get(addrInfo);
    if (prev != null) {
        prev.setLastUpdateTimestamp(System.currentTimeMillis());
    }
}
```



对于轻量级心跳的老版本的兼容问题，RocketMQ使用了原QUERY_DATA_VERSION操作来完成一次心跳

### 集群部署

NameServer是支持集群的，每个Broker启动的时候需要向所有的NameServer注册，代码如下（精简）

```java
//BrokerOuterAPI.registerBrokerAll()
final List<RegisterBrokerResult> registerBrokerResultList = new CopyOnWriteArrayList<>();
List<String> nameServerAddressList = this.remotingClient.getAvailableNameSrvList();

final CountDownLatch countDownLatch = new CountDownLatch(nameServerAddressList.size());
//遍历配置的NameServer列表，向所有的NameServer注册
for (final String namesrvAddr : nameServerAddressList) {
  brokerOuterExecutor.execute(new AbstractBrokerRunnable(brokerIdentity) {
      @Override
      public void run0() {
          try {
              RegisterBrokerResult result = registerBroker(namesrvAddr, oneway, timeoutMills, requestHeader, body);
              if (result != null) {
                  registerBrokerResultList.add(result);
              }
          } catch (Exception e) {
             
          } finally {
              countDownLatch.countDown();
          }
      }
  });
}
```



另外NameServer之间是不互相通信的，每个NameServer节点都存了一份所有的元数据，由于NameServer之间无需处理数据同步等问题，我理解这样的设计的原因应该是增加了高可用能力的同时极大降低了复杂度。

### 一些思考

**NameServer如果发生掉线或重启，其它客户端是怎么感知到的？**

org.apache.rocketmq.remoting.netty.NettyRemotingClient.scanAvailableNameSrv()方法会定时扫描目前的NameServer列表

```java
List<String> nameServerList = this.namesrvAddrList.get();

for (final String namesrvAddr : nameServerList) {
  scanExecutor.execute(new Runnable() {
      @Override
      public void run() {
          try {
              //如果成功获取通信通道，则代表连接成功
              Channel channel = NettyRemotingClient.this.getAndCreateChannel(namesrvAddr);
              if (channel != null) {
                  NettyRemotingClient.this.availableNamesrvAddrMap.putIfAbsent(namesrvAddr, true);
              } else {
                  Boolean value = NettyRemotingClient.this.availableNamesrvAddrMap.remove(namesrvAddr);
                  //如果不为空，证明之前是存在的，这次给移除了，也就是本次扫描新断开连接的
                  if (value != null) {
                      LOGGER.warn("scanAvailableNameSrv remove unconnected address {}", namesrvAddr);
                  }
              }
          } catch (Exception e) {
              LOGGER.error("scanAvailableNameSrv get channel of {} failed, ", namesrvAddr, e);
          }
      }
  });
}
```

**NameServer如果挂了，消息队列是否可以正常使用**

如果Producer、Consumer、Broker等无法请求NameServer，会优先使用本地缓存的配置进行通信，只不过NameServer不可用时，整个消息队列将无法进行信息更新，如Broker和Topic的增删改，Producer、Consumer、Broker的上下线等。
