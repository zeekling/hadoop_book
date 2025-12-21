# 简介

本仓库用于记录学习Hadoop相关笔记，菜就多做笔记。

# 编译

执行下面编译命令：

- Linux 下的编译命令如下：

```bash
# -T 是编译的线程数，可以按照具体操作系统增加或者减少
mvn -T 8 package -Pdist,native -DskipTests -Dmaven.javadoc.skip=true
# or 忽略 本地方法编译
mvn -T 1C clean package -DskipTests -P\!sign -Pnative -P\!resource-bundle -PskipShade  -Dmaven.javadoc.skip=true -Dcheckstyle.skip=true -Dpmd.skip=true
```

- Windows 下面的编译命令如下（Windows下的二进制编译未调试通过）：

```bash
# 本地编译，忽略二进制编译
mvn -T 1C clean install -DskipTests -PskipShade -P\!native-win -PskipShade  -Dmaven.javadoc.skip=true -Dcheckstyle.skip=true -Dpmd.skip=true
```

# 知识目录树

| 知识库名称 |  二级分类 | 所属模块 |
|---|---|----|
| [认证模块详解](./common/authenticated.md)                 | --          | [Common 模块](./common/README.md) 
| [主备倒换模块详解](./common/zk_failover.md)                |  --        | [Common 模块](./common/README.md) 
| [Fair Call Queue](./common/fair_call_queue.md)            | 队列       | [Common 模块](./common/README.md)  
| [NativeIO](./common/NativeIO.md)                          | IO 相关    | [Common 模块](./common/README.md)  
| [namenode全景](./hdfs/namenode全景.md)                    |  NameNode相关    | [HDFS模块](./hdfs/README.md)  
| [NameNode启动过程](./hdfs/nameNode启动过程.md)             |  NameNode相关   | [HDFS模块](./hdfs/README.md)  
| [FsImage详解](./hdfs/fsImages.md)                         |  NameNode相关   | [HDFS模块](./hdfs/README.md)  
| [NamenodeProtocols详解](./hdfs/NamenodeProtocols详解.md)   |  NameNode相关   | [HDFS模块](./hdfs/README.md)  
| [leaseManager详解](./hdfs/leaseManager详解.md)             |  NameNode相关   | [HDFS模块](./hdfs/README.md)  
| [FSDirectory详解](./hdfs/FSDirectory详解.md)               |  NameNode相关   | [HDFS模块](./hdfs/README.md)
| [ BPServiceActor详解](./hdfs/BPServiceActor详解.md)        |  NameNode相关   | [HDFS模块](./hdfs/README.md) 
| [文件上传](./hdfs/file_upload.md)                          |  ---            | [HDFS模块](./hdfs/README.md) 
| [dataNode启动过程](./hdfs/dataNode启动过程.md)              |  DataNode相关   | [HDFS模块](./hdfs/README.md) 
| [BPServiceActor详解](./hdfs/BPServiceActor详解.md)         |  DataNode相关   | [HDFS模块](./hdfs/README.md) 
| [router启动详解](./hdfs/router启动详解.md)                  |  Router相关    | [HDFS模块](./hdfs/README.md) 
| [webhdfs](./hdfs/webhdfs详解.md)                           |  Router相关    | [HDFS模块](./hdfs/README.md) 
| [ResourceManager详解](./yarn/resourcemanager.md)         |  ResourceManager相关    | [Yarn&Mapreduce模块](./yarn/README.md)
| [Capacity调度器](./yarn/capacity.md)                     |  ResourceManager相关    | [Yarn&Mapreduce模块](./yarn/README.md)
| [作业启动](./yarn/job_start.md)                          |  ---                    | [Yarn&Mapreduce模块](./yarn/README.md)
| [containerManager 详解](./yarn/containerManager.md)      |  NodeManager相关        | [Yarn&Mapreduce模块](./yarn/README.md)
| [container-executor详解](./yarn/container-executor.md)   |  NodeManager相关        | [Yarn&Mapreduce模块](./yarn/README.md)
| [Yarn状态机](./yarn/yarn_event.md)                       |  Yarn事件相关           | [Yarn&Mapreduce模块](./yarn/README.md)
| [Yarn事件处理机制](./yarn/yarn_event_detail.md)           |  Yarn事件相关           | [Yarn&Mapreduce模块](./yarn/README.md)
| [jobhistory 作业缓存](./yarn/jobhistory_cache.md)        |  JobHistory 相关        | [Yarn&Mapreduce模块](./yarn/README.md)
| [Distcp详解](./yarn/distcp.md)                           |  工具相关               | [Yarn&Mapreduce模块](./yarn/README.md)
| [Zookeeper启动源码详解](./zookeeper/Zookeeper启动源码详解.md)          |  ----               | [Zookeeper模块](./zookeeper/README.md)
| [RollingLevelDBTimeStore详解](./common/applicationhistoryservice/RollingLevelDBTimelineStore.md) | applicationhistoryservice相关   | [applicationhistoryservice](./common/applicationhistoryservice/README.md)
