---
layout: post
title: "从零开始实现RPC框架--(3)服务注册中心"
date: 2016-02-03 17:03:39 +0800
comments: true
categories: 中间件,rpc
---
上一篇文章中，我们实现了RPC框架客户端和服务端的通信，可以完成一次完整的RPC调用。   
但是在生产环境中，服务集群会经常进行上下线和扩容的变动，显然在客户端配置服务端地址的方式是不适用的。   
我们需要的是一个注册中心组件来维护服务器列表，并且在服务器上下线、扩容的时候同步变更，定时通知客户端更新列表，来看看该如何实现。
<!--more-->

#概述
关于注册中心，在第一章中提到过：注册中心必须保证高可用，不能由于一两台机器挂掉就变得不可用，也就是说需要集群冗余的支持，幸运的是并不用我们自己来实现这样的组件，业界已经有成熟的框架zookeeper，其高可用性，支持集群冗余，订阅/发布的功能都是注册中心所需要的。

zookeeper是Apache下的一个分布式协调框架，它的特点如下：
*  它有一套选举机制可以保证在zookeeper集群只要有一半以上机器可用就可以正常提供服务，可用性非常高
*  其数据存储在内存当中，读写速度很快
*  zookeeper使用起来很像一个文件系统，其节点Znode就像是文件系统中的文件夹和文件，节点的路径很像文件的路径比如/root/node1/node1_1，一个Znode下有自己的数据，同时还可以有子节点
*  订阅者/发布者 当客户端连接zookeeper并注册了所关注的节点/路径，在节点/路径发生变化时，客户端会收到一个通知
*  可以用来保存分布式应用的一些状态，实现统一的配置中心，分布式锁等

#服务注册中心的实现
再说回来我们RPC框架的注册中心，其主要功能是：   
1.服务端启动的时候进行注册，下线时注销   
2.客户端上线时从注册中心拉取对应服务的列表，在服务端列表发生变更时同步变更

如何用zookeeper实现呢？大体思路是我们在zookeeper服务器上建立对应的以服务全名命名的节点，路径如
```
/zing/service/org.zxc.zing.demo.api.DemoService
```
然后在该节点下建立子节点代表能提供该服务的服务器的信息
```
/zing/service/org.zxc.zing.demo.api.DemoService/127.0.0.1:4080
```
新的服务器上线时就新建对应的节点，下线时去除对应的节点。   
客户端启动时，拉取service对应zookeeper下的节点，初始化服务列表，在节点发生变化时，客户端收到对应的变更事件，同步更新这个列表。

假设我们已经有一个zookeeper集群在运行了，要操作zookeeper，我们需要在代码中调用zookeeper的客户端，本框架中使用的是Netfilx的Curator框架，它在zookeeper客户端的基础上做了很多封装，可以简化zookeeper客户端的使用。


##1.服务端启动的时候进行注册，下线时注销   

首先，服务端启动时需要连接zookeeper:
```java
public class RegistryManager{
	...
	public static void start() throws Exception{
    	String zookeeperAddress = ConfigManager.getInstance().getProperty(Constants.ZOOKEEPER_ADDRESS);
		client = CuratorFrameworkFactory.newClient(zookeeperAddress, new ExponentialBackoffRetry(1000, 3));
		client.start();
		...
		started = client.blockUntilConnected(1000, TimeUnit.MILLISECONDS);
    }
}
```
这里的zookeeper地址及端口需要通过配置文件的形式填入，ConfigManager会在初始化时读取classpath下的zing.properties配置文件registry.zookeeper.address的值，形式如"registry.zookeeper.address=127.0.0.1:2181"   

在服务端启动后，加载各个service实现类时，即可向服务注册中心注册该服务器可提供的服务：
```java
...
String currentServerPath = ZookeeperPathUtils.formatProviderPath(serviceName, ip, port);
String result = client.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL).forPath(currentServerPath);
...
```
根据当前加载的serviceBean得到serviceName、当前机器IP和rpc端口生成对应的zookeeper路径，调用curator创建节点，这里创建的zookeeper节点类型是EPHEMERAL的，即临时节点，当服务端和zookeeper的连接断开时，该节点即会自动从zookeeper去除，这样在服务端下线、宕机、网络连接出现问题而不能正常提供服务时，注册中心就会自动将该服务器地址移除。

PS：由于目前服务端所在的环境可能是物理机，也可能是虚拟机或者docker这样的容器等，当前机器一般会有多个IP地址，需要把希望提供服务的本机IP地址和端口以配置文件的形式存储在机器的配置文件/data/env下，形式如"server.address.ip=127.0.0.1"，"server.address.port=4080"。

##2.客户端上线时从注册中心拉取对应服务的列表，在服务端列表发生变更时同步变更

客户端上线时需要拉到service对应的provider列表，可以在客户端的service proxy bean在spring初始化的时候来进行   

```java
Class<?> serviceClass = Class.forName(serviceName);
ServiceProviderManager.initServerListOfService(serviceName);
return Reflection.newProxy(serviceClass, new ServiceProxy(serviceName));
```

底层调用了curator拉取了/zing/service/org.zxc.zing.demo.api.DemoService下的所有子节点，转换成service provider list存储在内存中。   

```java
List<String> stringList = client.getChildren().watched().forPath(String.format(Constants.SERVICE_ZK_PATH_FORMAT, serviceName));
```

在客户端启动的时候需要监控zookeeper的/zing/service路径，当这个路径下的节点发生变化时，客户端需要能够得到通知。

```java
String zookeeperAddress = ConfigManager.getInstance().getProperty(Constants.ZOOKEEPER_ADDRESS);
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
client = CuratorFrameworkFactory.newClient(zookeeperAddress, retryPolicy);
client.start();

TreeCache treeCache = TreeCache.newBuilder(client, Constants.SERVICE_ZK_PATH_PREFIX).setCacheData(false).build();
treeCache.getListenable().addListener(new ProviderNodeEventListener(), curatorEventThreadPool);
treeCache.start();

started = client.blockUntilConnected(1000, TimeUnit.MILLISECONDS);
```
这里在启动，注册一个TreeCache，关注/zing/service路径下节点的变化，当这个路径下有节点创建/去除时，ProviderNodeEventListener会收到对应的事件。

```java
public void childEvent(CuratorFramework client, TreeCacheEvent event) throws Exception {
if (!isServiceProviderNodeChangeEvent(event)) {
    return;
}
String serviceName = ZookeeperPathUtils.getServiceNameFromProviderZkPath(event.getData().getPath());
if (Strings.isNullOrEmpty(serviceName)) {
    return;
}
ProviderStateListenerManager.getInstance().onProviderChange(serviceName);
}
```
收到zookeeper节点变动事件时过滤一层，只关心/zing/service路径下的，且必须是服务器地址节点这一层的新增/删除事件，这样拿到事件就可以知道是哪个service对应的provider list发生了变化，和启动时一样，重新拉取一次该service下的节点，覆盖原来的provider list。


#总结
至此，一个rpc框架的基本功能就已经实现了，这个版本的tag是"registry-center"。
一个强大的rpc框架还需要一些诸如服务监控、限流、隔离、降级，心跳检测等，之后有需要会逐步完善。如果大家有什么问题和建议，欢迎一起讨论。


> 文章欢迎转载，转载时请保留作者与原文链接  
> 作者：赵轩辰   
> 本文原文地址：<http://zxcpro.github.io{{ page.url }}>