
## 简介

Namenode最重要的两个功能之一就是维护整个文件系统的目录树（即命名空间namesystem） 。
HDFS文件系统的命名空间（namespace） ， 也就是以“/”为根的整个目录树， 是通过FSDirectory类来管理的。 FSNamesystem也提供了管理目录树结构的方法。
FSNamesystem中的方法多是调用FSDirectory类的实现。FSNamesystem在FSDirectory类方法的基础上添加了editlog日志记录的功能。

FSDirectory的操作则全部是在内存中进行的， 并不进行editlog的日志记录。

## 参数

| 参数 | 默认值 | 描述 |
|----|----|----|
| 	dfs.permissions.enabled | true | 是否开启权限管理 |
| dfs.permissions.superusergroup | supergroup | 超级用户组 |
| dfs.namenode.acls.enabled | true | 设置为true以启用对HDFS ACL（访问控制列表）的支持。3.3.1版本默认启用ACL |
| dfs.namenode.posix.acl.inheritance.enabled | true | 是否启用POSIX格式的ACL权限 |
| dfs.namenode.xattrs.enabled | true | 是否支持扩展namenode的属性。 |
| dfs.namenode.fs-limits.max-xattr-size | 16384 | 以字节为单位的扩展属性的名称和值的最大组合大小。它应该大于0，小于或等于32768。 |
| dfs.namenode.accesstime.precision | 3600000 | HDFS文件访问时间的精确值，默认为1小时。当为0时，表示禁用。 |
| dfs.quota.by.storage.type.enabled | true | 如果为true，则启用基于存储类型的配额。 |
| dfs.ls.limit | 1000 | 限制ls打印的文件数。如果小于或等于零，最多将打印 DFS_LIST_LIMIT_DEFAULT (= 1000)。 |
| dfs.content-summary.limit | 5000 | 在一个锁定周期中允许的最大内容摘要计数。0或负数意味着没有限制。 |
| dfs.content-summary.sleep-microsec | 500 | 在内容汇总计算中，两次请求锁的时间。 |
| dfs.namenode.fs-limits.max-component-length | 255 | 定义路径中每个组件中UTF-8编码的最大字节数。0的值将禁用检查。  |
| dfs.namenode.fs-limits.max-directory-items | 1024\*1024 | 定义目录可能包含的最大项目数。无法将属性设置为小于1或大于6400000的值。|
| dfs.namenode.fs-limits.max-xattrs-per-inode | 32 | 每个索引节点的扩展属性的最大数目。 |
| dfs.protected.subdirectories.enable | false | 是否保护 fs.protected.directories 上设置的目录的子目录。 |
| dfs.namenode.name.cache.threshold | 10 | 经常访问的文件访问次数超过了这个阈值，缓存在FSDirectory nameCache中。 |
| dfs.namenode.quota.init-threads | 12 | quota初始化并发线程的数量。 |


## 常量

- INodeDirectory rootDir： 整个文件系统目录树的根节点， 是INodeDirectory类型的 。
- FSNamesystem namesystem： Namenode的门面类， 这个类主要支持对数据块进行操作的一些方法， 例如addBlock()。
- INodeMap inodeMap： 记录根目录下所有的INode,并维护INodeId ->INode的映射关系。
- ReentrantReadWriteLock dirLock： 对目录树以及inodeMap字段操作的锁。
- NameCache<ByteArray> nameCache： 将常用的name缓存下来， 以降低byte[]的使用， 并降低JVM heap的使用。
- SortedSet<String> protectedDirectories：使用 dfs.namenode.protected.directories 设置保护的一组目录。 这些目录不能被删除，除非它们是空的。
- FSEditLog editLog： 用于写editlog的类。
- HdfsFileStatus[] reservedStatuses： 待定。
- INodeAttributeProvider attributeProvider：用于实现权限管理。


## 操作类型

所有客户端的操作都是通过FSNamesystem.java 的。FSNamesystem.java 会调用具体操作的实现类的。再实现类里面操作时会使用到FSDirectory。

### 删除

**接口**：

```java
boolean delete(String src, boolean recursive, boolean logRetryCache) throws IOException {}
```

**简介**

删除文件或者文件夹，如果删除文件夹参数recursive必须为true。

**实现逻辑**

- 1、检查是否有写权限。具体可查看`checkOperation(OperationCategory.WRITE)`；
- 2、加全局锁。
- 3、再次检查是否有写权限。
- 4、调用FSDirDeleteOp.delete删除目录或者文件。
 ```java
 toRemovedBlocks = FSDirDeleteOp.delete(this, pc, src, recursive, logRetryCache);
 ```
- 5、释放全局锁。
- 6、同步editlog，并且记录审计日志。
- 7、将需要删除的块`toRemovedBlocks`添加到`markedDeleteQueue`队列里面，等待异步删除。

**FSDirDeleteOp.delete 实现逻辑**

- 检查权限：调用FSDirectory的函数checkPermission检查权限。
- 如果是非空的文件夹，检查是否有-r参数。如果没有-r参数则需要报错。
- 调用deleteInternal，开始删除文件夹。核心删除代码如下：

```java
List<INodeDirectory> snapshottableDirs = new ArrayList<>();
FSDirSnapshotOp.checkSnapshot(fsd, iip, snapshottableDirs);
ReclaimContext context = new ReclaimContext(
    fsd.getBlockStoragePolicySuite(), collectedBlocks, removedINodes,
    removedUCFiles);
// 更核心的删除代码再这个函数里面，会调用destroyAndCollectBlocks删除block,代码：targetNode.destroyAndCollectBlocks(reclaimContext);
if (unprotectedDelete(fsd, iip, context, mtime)) {
  filesRemoved = context.quotaDelta().getNsDelta();
  fsn.removeSnapshottableDirs(snapshottableDirs);
}
fsd.updateReplicationFactor(context.collectedBlocks()
                                .toUpdateReplicationInfo());
fsd.updateCount(iip, context.quotaDelta(), false);
```
- 删除EditLog。
- 调用incrDeletedFileCount更新metrics信息。

### 创建文件

**接口**

```java
HdfsFileStatus startFile(String src, PermissionStatus permissions,
      String holder, String clientMachine, EnumSet<CreateFlag> flag,
      boolean createParent, short replication, long blockSize,
      CryptoProtocolVersion[] supportedVersions, String ecPolicyName,
      String storagePolicy, boolean logRetryCache) throws IOException {
}
```

- 检查当前用户是否有写权限。
- 检查是否处于安全模式，如果处于安全模式，则不能进行当前操作。
- 检查是否对当前文件所在的目录是否有权限（开启权限管理的情况下）
- 调用FSDirWriteFileOp.startFile开始创建文件或者覆盖已有文件。
  - 对于已经存在的文件。在覆盖的场景下。主要核心代码如下，会将原来的文件删除。并且释放租约。
  ```java
  List<INode> toRemoveINodes = new ChunkedArrayList<>();
  List<Long> toRemoveUCFiles = new ChunkedArrayList<>();
  long ret = FSDirDeleteOp.delete(fsd, iip, toRemoveBlocks,
                                  toRemoveINodes, toRemoveUCFiles, now());
  if (ret >= 0) {
    iip = INodesInPath.replace(iip, iip.length() - 1, null);
    FSDirDeleteOp.incrDeletedFileCount(ret);
    fsn.removeLeasesAndINodes(toRemoveUCFiles, toRemoveINodes, true);
  }
  ```
  - 对于不需要覆盖的场景下，需要重新刷新租约信息。
   ```java
    fsn.recoverLeaseInternal(FSNamesystem.RecoverLeaseOp.CREATE_FILE, iip,
                                 src, holder, clientMachine, false);
   ```
  - 对于父文件夹存在的时候，将文件添加到父文件夹下面。
  ```java
  iip = addFile(fsd, parent, iip.getLastLocalName(), permissions,
      replication, blockSize, holder, clientMachine, shouldReplicate,
      ecPolicyName, storagePolicy);
  newNode = iip != null ? iip.getLastINode().asFile() : null;
  ```
 - 添加租约信息。
  ```java
  fsn.leaseManager.addLease(
     newNode.getFileUnderConstructionFeature().getClientName(),
     newNode.getId());
  ```
  - 返回文件信息。


