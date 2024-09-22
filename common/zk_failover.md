

# 简介

HDFS的NameNode、Yarn的ResourceManager都是依靠ZK实现主备倒换的。核心的类为：ZKFailoverController.java,
选举的核心类为ActiveStandbyElector.java

## 主备选举

主备选举的核心类是ActiveStandbyElector。在初始化的时候需要创建zk连接并且尝试在zk上面创建文件。在创建连接或者创建文件的时候都会有回调事件。

回调处理的函数主要包含：


### 创建node节点回调

入口函数如下：
```java
public synchronized void processResult(int rc, String path, Object ctx,
      String name) {
// .....
}
```

处理流程图如下：

![zk_failver_001](https://pan.zeekling.cn//zeekling/hadoop/common/zk_failover_001.png)



### 监控回调

入口函数如下：
```java
public synchronized void processResult(int rc, String path, Object ctx,
      Stat stat) {
// ...
}
```

处理流程如下：

![zk_failver_003](https://pan.zeekling.cn//zeekling/hadoop/common/zk_failover_003.png)

### 事件回调

入口函数如下：

```java
synchronized void processWatchEvent(ZooKeeper zk, WatchedEvent event) {
 // ..
}
```

![zk_failver_002](https://pan.zeekling.cn//zeekling/hadoop/common/zk_failover_002.png)

