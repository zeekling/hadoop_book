
# 简介
为了解决HDFS的水平扩展性问题，社区从Apache Hadoop 0.23.0版本开始引入了HDFS federation。HDFS Federation是指 HDFS集群可同时存在多个NameNode/Namespace，每个Namespace之间是互相独立的；
单独的一个Namespace里面包含多个 NameNode，其中一个是主，剩余的是备，这个和上面我们介绍的单Namespace里面的架构是一样的。这些Namespace共同管理整个集群的数据，每个Namespace只管理一部分数据，之间互不影响。

集群中的DataNode向所有的NameNode注册，并定期向这些NameNode发送心跳和块信息，同时DataNode也会执行NameNode发送过来的命令。集群中的NameNodes共享所有DataNode的存储资源。HDFS Federation的架构如下图所示：

![pic](https://pan.zeekling.cn/zeekling/hadoop/router/router_0001.png)

# 子模块

## State Store模块

### 初始化

初始化是从类Router的serviceInit函数触发的。

提供开关：dfs.federation.router.store.enable，默认开启。核心实现类是StateStoreService。

在serviceInit的时候初始化。
初始化store driver，通过配置dfs.federation.router.store.driver.class，默认为StateStoreZooKeeperImpl.class,通过反射机制初始化。目前默认支持的有：
- StateStoreFileImpl
- StateStoreFileSystemImpl
- StateStoreMySQLImpl
- StateStoreZooKeeperImpl

注册record stores，目前支持的有：
- MembershipStoreImpl
- MountTableStoreImpl
- RouterStoreImpl
- DisabledNameserviceStoreImpl

所有的record stores都保存在recordStores当中。

```java
// Add supported record stores
addRecordStore(MembershipStoreImpl.class);
addRecordStore(MountTableStoreImpl.class);
addRecordStore(RouterStoreImpl.class);
addRecordStore(DisabledNameserviceStoreImpl.class);
```

初始化定期检查任务

```java
// Check the connection to the State Store periodicallythis
this.monitorService = new StateStoreConnectionMonitorService(this);
this.addService(monitorService);
```

初始化缓存跟新服务

```java
// Cache update service
this.cacheUpdater = new StateStoreCacheUpdateService(this);
addService(this.cacheUpdater);
```

最后是初始化监控信息，核心的监控实现bean是StateStoreMBean。


### 启动

启动主要是Router的serviceStart函数触发，最终调用StateStoreDriver的init函数，用于初始化driver。核心函数为initDriver和initRecordStorage。

#### StateStoreFileImpl 

##### initDriver


##### initRecordStorage


#### StateStoreFileSystemImpl

initDriver

initRecordStorage


#### StateStoreMySQLImpl

initDriver

initRecordStorage


#### StateStoreZooKeeperImpl

initDriver

initRecordStorage


## ActiveNamenodeResolver


## subclusterResolver


## RPC 


## adminServer


## httpServer


## NameNode Heartbeat


## Router metrics system


## quota relevant service


## Safemode


## mount table cache update

## quota manager


