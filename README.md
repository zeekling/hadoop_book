
# 简介
本仓库用于记录学习Hadoop相关笔记，菜就多做笔记。

# 编译

执行下面编译命令：
```bash
# -T 是编译的线程数，可以按照具体操作系统增加或者减少
mvn -T 8 package -Pdist,native -DskipTests -Dmaven.javadoc.skip=true
# or 忽略 本地方法编译
mvn -T 1C clean package -DskipTests -P\!sign -Pnative -P\!resource-bundle -PskipShade  -Dmaven.javadoc.skip=true -Dcheckstyle.skip=true -Dpmd.skip=true
```

# 知识树


## Common 模块

- [认证模块详解](./common/authenticated.md)
- [主备倒换模块详解](./common/zk_failover.md)


## HDFS 模块

- [HDFS概览](./hdfs/README.md)
- NameNode相关
   - [namenode全景](./hdfs/namenode全景.md)
   - [NameNode启动过程](./hdfs/nameNode启动过程.md)
   - [FsIMage](./hdfs/fsImages.md)
   - [NamenodeProtocols详解](./hdfs/NamenodeProtocols详解.md)
   - [leaseManager详解](./hdfs/leaseManager详解.md)
   - [FSDirectory详解](./hdfs/FSDirectory详解.md)
   - [ BPServiceActor详解](./hdfs/BPServiceActor详解.md)

- [文件上传](./hdfs/file_upload.md)
- DataNode相关
   - [dataNode启动过程](./hdfs/dataNode启动过程.md)
   - [BPServiceActor详解](./hdfs/BPServiceActor详解.md)
- Router相关
   - [router启动详解](./hdfs/router启动详解.md)


## Yarn 模块

- [Yarn概览](./yarn/README.md)
- [ResourceManager相关]
   - [ResourceManager详解](./yarn/resourcemanager.md)
   - [Capacity调度器](./yarn/capacity.md)
- [作业启动](./yarn/job_start.md)
- Yarn NodeManager相关
   - [containerManager 详解](./yarn/containerManager.md)

- Yarn 事件相关
   - [Yarn状态机](./yarn/yarn_event.md)
   - [Yarn事件处理机制](./yarn/yarn_event_detail.md)
- JobHistory 相关
  - [jobhistory 作业缓存](./yarn/jobhistory_cache.md)

- 工具相关
  - [Distcp详解](./yarn/distcp.md)


## Zookeeper模块
- [Zookeeper概览](./zookeeper/README.md)


## OZone 模块
- [OZone概览](./ozone/README.md)


