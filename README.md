
# 简介
本仓库用于记录学习Hadoop相关笔记，菜就多做笔记。

# 编译

执行下面编译命令：
```bash
# -T 是编译的线程数，可以按照具体操作系统增加或者减少
mvn -T 8 package -Pdist,native -DskipTests -Dmaven.javadoc.skip=true
```

# 知识树

## HDFS
- [HDFS概览](./hdfs/README.md)
- NameNode相关
   - [namenode全景](./hdfs/namenode全景.md)
   - [NameNode启动过程](./hdfs/nameNode启动过程.md)
   - [FsIMage](./hdfs/fsImages.md)
   - [NamenodeProtocols详解](./hdfs/NamenodeProtocols详解.md)
   - [leaseManager详解](./hdfs/leaseManager详解.md)
   - [FSDirectory详解](./hdfs/FSDirectory详解.md)
- [文件上传](./hdfs/file_upload.md)
- DataNode相关
   - [dataNode启动过程](./hdfs/dataNode启动过程.md)
   - [BPServiceActor详解](./hdfs/BPServiceActor详解.md)


## Yarn
- [Yarn概览](./yarn/README.md)
- [ResourceManager详解](./yarn/resourcemanager.md)
- [作业启动](./yarn/job_start.md)
- Yarn NodeManager相关
   - [containerManager 详解](./yarn/containerManager.md)

- Yarn 事件相关
   - [Yarn状态机](./yarn/yarn_event.md)
   - [Yarn事件处理机制](./yarn/yarn_event_detail.md)
- JobHistory 相关
  - [jobhistory 作业缓存](./yarn/jobhistory_cache.md)


## Zookeeper
- [Zookeeper概览](./zookeeper/README.md)


## OZone
- [OZone概览](./ozone/README.md)


