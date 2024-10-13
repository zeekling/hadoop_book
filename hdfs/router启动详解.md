
# 简介
为了解决HDFS的水平扩展性问题，社区从Apache Hadoop 0.23.0版本开始引入了HDFS federation。HDFS Federation是指 HDFS集群可同时存在多个NameNode/Namespace，每个Namespace之间是互相独立的；
单独的一个Namespace里面包含多个 NameNode，其中一个是主，剩余的是备，这个和上面我们介绍的单Namespace里面的架构是一样的。这些Namespace共同管理整个集群的数据，每个Namespace只管理一部分数据，之间互不影响。

集群中的DataNode向所有的NameNode注册，并定期向这些NameNode发送心跳和块信息，同时DataNode也会执行NameNode发送过来的命令。集群中的NameNodes共享所有DataNode的存储资源。HDFS Federation的架构如下图所示：

![pic](https://pan.zeekling.cn/zeekling/hadoop/router/router_0001.png)

# 子模块

## 一、State Store模块

当前模块主要用于保存Router状态信息，提供了文件系统(HDFS、本地)、Mysql、Zookeeper。个人觉得一般使用Zookeeper比较合理。

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

其中initRecordStorage针对每个record stores都需要调用，如下：
```java
for (Class<? extends BaseRecord> cls : records) {
  String recordString = StateStoreUtils.getRecordName(cls);
  if (!initRecordStorage(recordString, cls)) {
    LOG.error("Cannot initialize record store for {}", cls.getSimpleName());
    return false;
  }
}
```

#### StateStoreFileImpl or StateStoreFileSystemImpl

##### initDriver

对于当前的StateStore初始化比较简单，主要是检查本地文件夹是否存在，不存在就创建。大致代码如下：

```java
public boolean initDriver() {
    String rootDir = getRootDir();
      if (rootDir == null) {
        LOG.error("Invalid root directory, unable to initialize driver.");
        return false;
      }

      // Check root path
      if (!exists(rootDir)) {
        if (!mkdir(rootDir)) {
          LOG.error("Cannot create State Store root directory {}", rootDir);
          return false;
        }
      }
    // ... 省略 ...
    int threads = getConcurrentFilesAccessNumThreads();
      this.concurrentStoreAccessPool =
          new ThreadPoolExecutor(threads, threads, 0L, TimeUnit.MILLISECONDS,
              new LinkedBlockingQueue<>(),
              new ThreadFactoryBuilder()
                  .setNameFormat("state-store-file-based-concurrent-%d")
                  .setDaemon(true).build());
    return true;
  }
```

##### initRecordStorage
和initDriver类似，支持针对每个State Store创建对应的目录，目录名称使用state store的className。

```java
public <T extends BaseRecord> boolean initRecordStorage(
      String className, Class<T> recordClass) {
    String dataDirPath = getRootDir() + "/" + className;
    // Create data directories for files
    if (!exists(dataDirPath)) {
      LOG.info("{} data directory doesn't exist, creating it", dataDirPath);
      if (!mkdir(dataDirPath)) {
        LOG.error("Cannot create data directory {}", dataDirPath);
        return false;
      }
    }
    return true;
  }
```


#### StateStoreMySQLImpl

##### initDriver
核心逻辑就是创建Mysql连接。Mysql连接封装为类MySQLStateStoreHikariDataSourceConnectionFactory。
```java
MySQLStateStoreHikariDataSourceConnectionFactory(Configuration conf) {
    Properties properties = new Properties();
    properties.setProperty("jdbcUrl", conf.get(StateStoreMySQLImpl.CONNECTION_URL));
    properties.setProperty("username", conf.get(StateStoreMySQLImpl.CONNECTION_USERNAME));
    properties.setProperty("password", conf.get(StateStoreMySQLImpl.CONNECTION_PASSWORD));
    properties.setProperty("driverClassName", conf.get(StateStoreMySQLImpl.CONNECTION_DRIVER));

    // Include hikari connection properties
    properties.putAll(conf.getPropsWithPrefix(HIKARI_PROPS));

    HikariConfig hikariConfig = new HikariConfig(properties);
    this.dataSource = new HikariDataSource(hikariConfig);
}
```

##### initRecordStorage

在StateStoreMySQLImpl当中，每个state store 对应一张表。建表语句如下：
```sql
CREATE TABLE <className> (
   recordKey VARCHAR (255) NOT NULL,
   recordValue VARCHAR (2047) NOT NULL,
   PRIMARY KEY(recordKey))
)
```


#### StateStoreZooKeeperImpl

##### initDriver

对于StateStoreZooKeeperImpl，initDriver中主要是建立zk连接。核心代码如下：
```java
this.zkManager = new ZKCuratorManager(conf);
this.zkManager.start(zkHostPort);
this.zkAcl = ZKCuratorManager.getZKAcls(conf);
```
对于异步模式，会创建线程池，用于异步保存状态。
```java
ThreadFactory threadFactory = new ThreadFactoryBuilder()
    .setNameFormat("StateStore ZK Client-%d")
    .build();
this.executorService = new ThreadPoolExecutor(numThreads, numThreads,
    0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>(), threadFactory);
LOG.info("Init StateStoreZookeeperImpl by async mode with {} threads.", numThreads);
```

##### initRecordStorage

当前函数的实现比较简单，主要是在zk上面创建状态保存的目录。
```java
try {
  String checkPath = getNodePath(baseZNode, className);
  zkManager.createRootDirRecursively(checkPath, zkAcl);
  return true;
} catch (Exception e) {
  LOG.error("Cannot initialize ZK node for {}: {}",
      className, e.getMessage());
  return false;
}
```

## 二、ActiveNamenodeResolver
具体实现有配置项dfs.federation.router.namenode.resolver.client.class指定。默认为MembershipNamenodeResolver.class。目前最新的也只支持这一种。

### MembershipNamenodeResolver

初始化如下,主要是初始化NN缓存以及将当前注册到stateStore里面。

```java
this.stateStore = store;
this.cacheNS = new ConcurrentHashMap<>();
this.cacheBP = new ConcurrentHashMap<>();

if (this.stateStore != null) {
  // Request cache updates from the state store
  this.stateStore.registerCacheExternal(this);
}
```
注册到stateStore里面的逻辑比较简单：
```java
public void registerCacheExternal(StateStoreCache client) {
  this.cachesToUpdateExternal.add(client);
}
```

## 三、subclusterResolver
具体实现由配置项dfs.federation.router.file.resolver.client.class指定，默认为MountTableResolver.class。此外还支持MultipleDestinationMountTableResolver.class



