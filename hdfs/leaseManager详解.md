
# 简介

HDFS作为一个分布式文件系统，只允许一个客户端同时对一个文件进行修改操作。租约就是为了实现独占的写操作的机制。
HDFS租约的主要实现类是LeaseManager。

Lease 的使用场景如下：

![lease_quest](https://pan.zeekling.cn/zeekling/hadoop/namenode/lease_request.png)

- 客户端在申请创建新的文件或者向文件追加都会先向NameNode申请获得inode或者最后一个块的信息
- 在NameNode中FSNamesystem会调用recoverLeaseInternal检查文件是否是UnderConstruction，是UnderConstruction的前提下，在leaseManager中是否这个client已经持有租约，如果有则抛出已经持有租约的异常
- 再检查文件的原来的租约持有者的的租约是否超过了软限制，如果超过了软限制则执行租约恢复internalReleaseLease进行租约恢复。
- 因为在文件是UnderConstruction前提下检查，文件必定有一个租约持有者，所以，直接抛出已经有另一个租约持有者的异常。
- 如果文件不是在UnderConstruction状态，则直接为这个发起请求的客户端构造租约，加入到LeaseManager的租约维护的集合中。
- 在NameNode中租约持有者DFSClient并不是DFSClient类，而是clientName，他的生成规则如下
```java
# 其中dfsClientConf.taskId是mapreduce.task.attempt.id 配置获取默认为NONMAPREDUCE
clientName = "DFSClient_" + dfsClientConf.taskId + "_" + DFSUtil.getRandom().nextInt()  + "_" + Thread.currentThread().getId();
```

## leaseManager

-  软限制 & 硬限制

  - 软限制是能容忍的客户端刷新租约的最长时间限制，为60s不可更改，如果客户端的租约超过60s未更新，则其他客户端请求文件就可以执行租约恢复操作
  - 硬限制就是namenode能容忍的文件最长不放开租约的时间，在超过软限制后，并没有客户端请求更改文件导致没有触发租约恢复，那么只能等待LeaseManager的周期线程检查这个超过这个时限的租约强制进行租约恢复。恢复的角色也会变成namenode。

- LeaseManager 主要用户租约的管理，其实就是保存 用户 + 文件 + 租约的集合，LeaseManager内部的集合有2个（Hadoop 3.3.1版本）
  - leases：为一个map，记录clientName 对应的Lease。
  - leasesById：以路径字典序保存了文件的nodeId与租约的对应关系，用来其他类快速获取UnderConstruction的文件。
- 用户为DFSClient 索引者一个租约，一个租约下面挂载了多个文件，也就是说一个客户端操作多个文件租约还是同一个。
- 内部线程周期调度检查是否超出硬限制，如果超过硬限制，则将该租约下的所有文件都执行租约恢复，恢复的执行者为HDFS_NameNode。

