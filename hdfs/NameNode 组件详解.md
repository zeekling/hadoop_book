# HDFS NameNode 组件详解

## 一、概述

NameNode 是 HDFS 的主节点，负责管理文件系统的命名空间和元数据。NameNode 维护整个文件系统的目录树、文件到块的映射、块到 DataNode 的映射等关键信息。

**核心职责**：
- 管理文件系统命名空间
- 管理块到 DataNode 的映射
- 处理客户端请求
- 管理 DataNode
- 处理安全模式
- 管理编辑日志和镜像

**位置**: `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/namenode/`

---

## 二、核心架构

### 2.1 类继承关系

```
NameNode
├── NameNodeRpcServer (RPC 服务)
├── FSNamesystem (命名空间管理)
│   ├── INodeMap (INode 映射)
│   ├── BlockManager (块管理)
│   ├── DatanodeManager (DataNode 管理)
│   ├── LeaseManager (租约管理)
│   ├── SafeMode (安全模式)
│   ├── SnapshotManager (快照管理)
│   ├── CacheManager (缓存管理)
│   └── EncryptionZoneManager (加密区域管理)
├── FSImage (文件系统镜像)
│   ├── NNStorage (存储管理)
│   ├── FSEditLog (编辑日志)
│   └── Checkpointer (检查点)
└── HAContext (高可用上下文)
    ├── ActiveState (活跃状态)
    └── StandbyState (待机状态)
```

### 2.2 主要组件

```
org.apache.hadoop.hdfs.server.namenode
├── NameNode.java                    - NameNode 主类
├── FSNamesystem.java               - 命名空间管理
├── NameNodeRpcServer.java           - RPC 服务
├── FSImage.java                     - 文件系统镜像
├── FSEditLog.java                   - 编辑日志
├── NNStorage.java                   - NameNode 存储
├── Checkpointer.java                - 检查点
├── SafeMode.java                    - 安全模式
├── LeaseManager.java                - 租约管理
├── INode.java                       - INode 基类
├── INodeDirectory.java              - 目录节点
├── INodeFile.java                   - 文件节点
├── INodeSymlink.java                - 符号链接节点
├── INodeMap.java                    - INode 映射
├── INodesInPath.java                - 路径中的节点
└── ha/                              - 高可用
    ├── HAContext.java               - 高可用上下文
    ├── ActiveState.java             - 活跃状态
    ├── StandbyState.java            - 待机状态
    ├── ZKFailoverController.java    - ZK 故障转移
    └── EditLogTailer.java           - 编辑日志跟踪器
```

---

## 三、主要组件详解

### 3.1 NameNode - NameNode 主类

**位置**: `NameNode.java:18`

**作用**: NameNode 的主类，负责启动和管理 NameNode 服务

**核心字段**:
```java
// 服务状态
private volatile HAState state;                    // HA 状态
private final HAContext haContext;                 // HA 上下文
private final NameNodeRpcServer rpcServer;          // RPC 服务
private final NameNodeHttpServer httpServer;        // HTTP 服务

// 命名空间
private final FSNamesystem namesystem;              // 命名空间
private final FSImage fsImage;                      // 文件系统镜像
private final FSEditLog editLog;                     // 编辑日志

// 配置
private final Configuration conf;                    // 配置
private final NamenodeRole role;                     // NameNode 角色

// 统计
private final NameNodeMetrics metrics;              // 指标
private final StartupProgress startupProgress;      // 启动进度
```

**核心方法**:

#### 1. main - 主入口
```java
public static void main(String argv[]) throws Exception {
    if (argv.length == 0) {
        System.err.println("Usage: java NameNode");
        System.exit(-1);
    }
    
    try {
        StringUtils.startupShutdownMessage(NameNode.class, argv, LOG);
        NameNode namenode = createNameNode(argv, null);
        if (namenode != null) {
            namenode.join();
        }
    } catch (Throwable e) {
        LOG.error("Exception in NameNode main", e);
        System.exit(-1);
    }
}
```

#### 2. createNameNode - 创建 NameNode
```java
public static NameNode createNameNode(String argv[], 
    Configuration conf) throws IOException {
    // 解析启动选项
    StartupOption startOpt = parseArguments(argv);
    
    // 设置配置
    if (startOpt == null) {
        startOpt = StartupOption.REGULAR;
    }
    
    // 创建 NameNode
    return new NameNode(conf, startOpt);
}
```

#### 3. serviceInit - 服务初始化
```java
@Override
protected void serviceInit(Configuration conf) throws IOException {
    // 1. 初始化配置
    this.conf = conf;
    
    // 2. 初始化 HA 上下文
    this.haContext = createHAContext();
    
    // 3. 初始化 RPC 服务
    this.rpcServer = createRpcServer();
    
    // 4. 初始化 HTTP 服务
    this.httpServer = createHttpServer();
    
    // 5. 初始化命名空间
    this.namesystem = createNamesystem();
    
    // 6. 初始化文件系统镜像
    this.fsImage = createFSImage();
    
    // 7. 初始化编辑日志
    this.editLog = createEditLog();
    
    // 8. 初始化指标
    this.metrics = createMetrics();
    
    // 9. 初始化启动进度
    this.startupProgress = createStartupProgress();
    
    super.serviceInit(conf);
}
```

#### 4. serviceStart - 服务启动
```java
@Override
protected void serviceStart() throws Exception {
    // 1. 启动 RPC 服务
    rpcServer.start();
    
    // 2. 启动 HTTP 服务
    httpServer.start();
    
    // 3. 启动命名空间
    namesystem.start();
    
    // 4. 启动文件系统镜像
    fsImage.start();
    
    // 5. 启动编辑日志
    editLog.start();
    
    // 6. 启动指标
    metrics.start();
    
    // 7. 启动 HA 上下文
    haContext.start();
    
    super.serviceStart();
}
```

---

### 3.2 FSNamesystem - 命名空间管理

**位置**: `FSNamesystem.java:18`

**作用**: 管理文件系统的命名空间，包括目录结构、文件元数据、权限等

**核心字段**:
```java
// INode 管理
private final INodeMap inodeMap;                    // INode 映射
private final NameCache nameCache;                  // 名称缓存

// 块管理
private final BlockManager blockManager;            // 块管理器
private final DatanodeManager datanodeManager;      // DataNode 管理器

// 租约管理
private final LeaseManager leaseManager;            // 租约管理器

// 安全模式
private final SafeMode safeMode;                    // 安全模式

// 快照管理
private final SnapshotManager snapshotManager;      // 快照管理器

// 缓存管理
private final CacheManager cacheManager;            // 缓存管理器

// 加密区域管理
private final EncryptionZoneManager encryptionZoneManager;  // 加密区域管理器

// 配额管理
private final Quota quota;                          // 配额管理器

// 锁管理
private final FSNLockManager lockManager;           // 锁管理器
```

**核心方法**:

#### 1. mkdirs - 创建目录
```java
public void mkdirs(String src, PermissionStatus permissions,
    boolean createParent) throws IOException {
    // 1. 检查权限
    checkPermission(src, permissions);
    
    // 2. 解析路径
    INodesInPath iip = getINodesInPath(src, true);
    
    // 3. 创建目录
    FSDirMkdirOp.mkdir(this, iip, permissions, createParent);
    
    // 4. 记录编辑日志
    getEditLog().logMkDir(src, permissions);
}
```

#### 2. createFile - 创建文件
```java
public HdfsFileStatus createFile(String src, PermissionStatus permissions,
    String holder, boolean overwrite, boolean createParent,
    short replication, long blockSize, CryptoProtocolVersion protocolVersion,
    String encryptionZoneName) throws IOException {
    // 1. 检查权限
    checkPermission(src, permissions);
    
    // 2. 解析路径
    INodesInPath iip = getINodesInPath(src, true);
    
    // 3. 创建文件
    FSDirWriteFileOp.createFile(this, iip, permissions, holder,
        overwrite, createParent, replication, blockSize,
        protocolVersion, encryptionZoneName);
    
    // 4. 记录编辑日志
    getEditLog().logCreateFile(src, permissions, replication,
        blockSize, protocolVersion, encryptionZoneName);
    
    // 5. 返回文件状态
    return getFileInfo(src);
}
```

#### 3. delete - 删除文件或目录
```java
public boolean delete(String src, boolean recursive) throws IOException {
    // 1. 检查权限
    checkPermission(src);
    
    // 2. 解析路径
    INodesInPath iip = getINodesInPath(src, false);
    
    // 3. 删除文件或目录
    FSDirDeleteOp.delete(this, iip, recursive);
    
    // 4. 记录编辑日志
    getEditLog().logDelete(src, recursive);
    
    return true;
}
```

#### 4. getFileInfo - 获取文件信息
```java
public HdfsFileStatus getFileInfo(String src) throws IOException {
    // 1. 解析路径
    INodesInPath iip = getINodesInPath(src, false);
    
    // 2. 获取 INode
    INode inode = iip.getLastINode();
    
    // 3. 检查权限
    checkPermission(inode);
    
    // 4. 返回文件状态
    return FSDirStatAndListingOp.getFileInfo(this, inode, src);
}
```

#### 5. getBlockLocations - 获取块位置
```java
public LocatedBlocks getBlockLocations(String src, long offset,
    long length) throws IOException {
    // 1. 解析路径
    INodesInPath iip = getINodesInPath(src, false);
    
    // 2. 获取 INode
    INode inode = iip.getLastINode();
    
    // 3. 检查权限
    checkPermission(inode);
    
    // 4. 获取块位置
    return FSDirStatAndListingOp.getBlockLocations(this, inode,
        src, offset, length);
}
```

---

### 3.3 NameNodeRpcServer - RPC 服务

**位置**: `NameNodeRpcServer.java`

**作用**: 处理客户端和 DataNode 的 RPC 请求

**核心字段**:
```java
// RPC 服务
private final Server clientRpcServer;              // 客户端 RPC 服务
private final Server serviceRpcServer;             // 服务 RPC 服务
private final Server lifelineRpcServer;            // 生命线 RPC 服务

// 协议实现
private final ClientProtocol clientProtocol;      // 客户端协议
private final DatanodeProtocol datanodeProtocol;    // DataNode 协议
private final NamenodeProtocol namenodeProtocol;    // NameNode 协议

// 配置
private final Configuration conf;                    // 配置
```

**核心方法**:

#### 1. create - 创建文件
```java
@Override
public HdfsFileStatus create(String src, FsPermission masked,
    String clientName, EnumSetWritable<CreateFlag> flag,
    boolean createParent, short replication, long blockSize,
    CryptoProtocolVersion[] supportedVersions,
    String storagePolicy) throws IOException {
    // 1. 检查权限
    checkPermission(src);
    
    // 2. 调用 FSNamesystem 创建文件
    return namesystem.createFile(src, masked, clientName,
        flag, createParent, replication, blockSize,
        supportedVersions, storagePolicy);
}
```

#### 2. delete - 删除文件或目录
```java
@Override
public boolean delete(String src, boolean recursive) throws IOException {
    // 1. 检查权限
    checkPermission(src);
    
    // 2. 调用 FSNamesystem 删除文件或目录
    return namesystem.delete(src, recursive);
}
```

#### 3. getBlockLocations - 获取块位置
```java
@Override
public LocatedBlocks getBlockLocations(String src, long offset,
    long length) throws IOException {
    // 1. 检查权限
    checkPermission(src);
    
    // 2. 调用 FSNamesystem 获取块位置
    return namesystem.getBlockLocations(src, offset, length);
}
```

---

### 3.4 FSImage - 文件系统镜像

**位置**: `FSImage.java`

**作用**: 管理文件系统镜像，包括镜像的加载、保存、检查点等

**核心字段**:
```java
// 存储
private final NNStorage storage;                    // NameNode 存储
private final FSEditLog editLog;                     // 编辑日志

// 镜像信息
private final FSImageFormat imageFormat;              // 镜像格式
private final FSImageCompression compression;         // 压缩

// 检查点
private final Checkpointer checkpointer;              // 检查点

// 状态
private final FSImageStorageInspector inspector;      // 存储检查器
```

**核心方法**:

#### 1. loadFSImage - 加载文件系统镜像
```java
public void loadFSImage(StartupOption startOpt) throws IOException {
    // 1. 检查存储目录
    storage.checkConsistent();
    
    // 2. 加载镜像
    FSImageFormat format = FSImageFormat.newInstance(conf);
    format.load(this, startOpt);
    
    // 3. 加载编辑日志
    editLog.load();
    
    // 4. 恢复命名空间
    recover(startOpt);
}
```

#### 2. saveFSImage - 保存文件系统镜像
```java
public void saveFSImage() throws IOException {
    // 1. 创建检查点
    saveNamespace();
    
    // 2. 保存镜像
    FSImageFormat format = FSImageFormat.newInstance(conf);
    format.save(this);
    
    // 3. 清理编辑日志
    editLog.purge();
}
```

#### 3. saveNamespace - 保存命名空间
```java
public void saveNamespace() throws IOException {
    // 1. 进入安全模式
    namesystem.enterSafeMode(false);
    
    // 2. 保存镜像
    saveFSImage();
    
    // 3. 退出安全模式
    namesystem.leaveSafeMode(false);
}
```

---

### 3.5 FSEditLog - 编辑日志

**位置**: `FSEditLog.java`

**作用**: 管理编辑日志，记录所有文件系统操作

**核心字段**:
```java
// 日志流
private final JournalSet journalSet;                // 日志集合
private final List<JournalAndStream> journals;       // 日志流列表

// 日志状态
private long lastWrittenTxId;                       // 最后写入的事务 ID
private long lastFlushedTxId;                        // 最后刷新的事务 ID

// 缓冲区
private final EditLogOutputStream editLogStream;     // 编辑日志输出流
private final EditLogAsync async;                    // 异步编辑日志
```

**核心方法**:

#### 1. logCreateFile - 记录创建文件
```java
public void logCreateFile(String path, PermissionStatus permissions,
    short replication, long blockSize,
    CryptoProtocolVersion protocolVersion,
    String encryptionZoneName) {
    // 1. 创建编辑日志操作
    FSEditLogOp op = FSEditLogOp.CreateOp.newInstance(path,
        permissions, replication, blockSize,
        protocolVersion, encryptionZoneName);
    
    // 2. 写入编辑日志
    logEdit(op);
}
```

#### 2. logDelete - 记录删除
```java
public void logDelete(String path, boolean recursive) {
    // 1. 创建编辑日志操作
    FSEditLogOp op = FSEditLogOp.DeleteOp.newInstance(path, recursive);
    
    // 2. 写入编辑日志
    logEdit(op);
}
```

#### 3. logMkDir - 记录创建目录
```java
public void logMkDir(String path, PermissionStatus permissions) {
    // 1. 创建编辑日志操作
    FSEditLogOp op = FSEditLogOp.MkDirOp.newInstance(path, permissions);
    
    // 2. 写入编辑日志
    logEdit(op);
}
```

#### 4. logEdit - 记录编辑日志
```java
private void logEdit(FSEditLogOp op) {
    // 1. 分配事务 ID
    long txid = allocateTransactionId();
    
    // 2. 设置事务 ID
    op.setTransactionId(txid);
    
    // 3. 写入编辑日志
    editLogStream.write(op);
    
    // 4. 更新最后写入的事务 ID
    lastWrittenTxId = txid;
}
```

---

### 3.6 SafeMode - 安全模式

**位置**: `SafeMode.java`

**作用**: 管理 NameNode 的安全模式

**核心字段**:
```java
// 安全模式状态
private boolean safeMode;                            // 是否在安全模式
private long resourcesLow;                          // 资源不足标志

// 阈值
private double threshold;                           // 阈值
private int datanodeThreshold;                       // DataNode 阈值

// 统计
private int blockTotal;                              // 总块数
private int blockSafe;                               // 安全块数
private int blockReported;                          // 已报告块数
```

**核心方法**:

#### 1. enter - 进入安全模式
```java
public synchronized void enter() {
    // 1. 设置安全模式标志
    safeMode = true;
    
    // 2. 重置统计
    blockTotal = 0;
    blockSafe = 0;
    blockReported = 0;
    
    // 3. 通知监听器
    notifyAll();
}
```

#### 2. leave - 退出安全模式
```java
public synchronized void leave() {
    // 1. 清除安全模式标志
    safeMode = false;
    
    // 2. 通知监听器
    notifyAll();
}
```

#### 3. checkSafeMode - 检查安全模式
```java
public synchronized void checkSafeMode() throws SafeModeException {
    // 1. 检查是否在安全模式
    if (!safeMode) {
        return;
    }
    
    // 2. 检查是否满足退出条件
    if (canLeave()) {
        leave();
        return;
    }
    
    // 3. 抛出安全模式异常
    throw new SafeModeException("Name node is in safe mode");
}
```

---

### 3.7 LeaseManager - 租约管理

**位置**: `LeaseManager.java`

**作用**: 管理文件租约，防止文件被多个客户端同时写入

**核心字段**:
```java
// 租约映射
private final SortedMap<Long, Lease> leases;        // 租约映射
private final Map<String, Lease> leaseMap;          // 文件到租约的映射

// 监控器
private final Monitor monitor;                      // 租约监控器
private final long softLimit;                        // 软限制
private final long hardLimit;                        // 硬限制
```

**核心方法**:

#### 1. addLease - 添加租约
```java
public Lease addLease(String holder, String src) {
    // 1. 检查是否已存在租约
    Lease lease = leaseMap.get(src);
    if (lease != null) {
        return lease;
    }
    
    // 2. 创建新租约
    lease = new Lease(holder);
    
    // 3. 添加到租约映射
    leaseMap.put(src, lease);
    leases.put(lease.getLeaseId(), lease);
    
    return lease;
}
```

#### 2. removeLease - 移除租约
```java
public void removeLease(Lease lease) {
    // 1. 从租约映射中移除
    leaseMap.remove(lease.getPath());
    leases.remove(lease.getLeaseId());
    
    // 2. 释放文件锁
    releaseFileLock(lease.getPath());
}
```

#### 3. checkLeases - 检查租约
```java
public void checkLeases() {
    // 1. 获取当前时间
    long now = Time.now();
    
    // 2. 检查每个租约
    for (Lease lease : leases.values()) {
        // 3. 检查是否过期
        if (lease.expiredHardLimit(now)) {
            // 4. 移除过期租约
            removeLease(lease);
        }
    }
}
```

---

### 3.8 INode - 节点基类

**位置**: `INode.java`

**作用**: 表示文件系统中的节点（文件、目录、符号链接）

**核心字段**:
```java
// 节点信息
private final long id;                              // 节点 ID
private final String name;                          // 节点名称
private final PermissionStatus permissions;          // 权限
private final long modificationTime;                 // 修改时间
private final long accessTime;                       // 访问时间

// 父节点
private INodeDirectory parent;                       // 父目录
```

**核心方法**:

#### 1. isDirectory - 是否为目录
```java
public boolean isDirectory() {
    return this instanceof INodeDirectory;
}
```

#### 2. isFile - 是否为文件
```java
public boolean isFile() {
    return this instanceof INodeFile;
}
```

#### 3. isSymlink - 是否为符号链接
```java
public boolean isSymlink() {
    return this instanceof INodeSymlink;
}
```

---

### 3.9 INodeDirectory - 目录节点

**位置**: `INodeDirectory.java`

**作用**: 表示文件系统中的目录

**核心字段**:
```java
// 子节点
private final List<INode> children;                 // 子节点列表
private final Map<String, INode> childrenMap;        // 子节点映射

// 配额
private final QuotaCounts quota;                     // 配额统计

// 快照
private final DirectorySnapshottableFeature snapshottable;  // 可快照特性
```

**核心方法**:

#### 1. addChild - 添加子节点
```java
public void addChild(INode child) {
    // 1. 检查是否已存在
    if (childrenMap.containsKey(child.getName())) {
        throw new IOException("Child already exists");
    }
    
    // 2. 添加到列表
    children.add(child);
    
    // 3. 添加到映射
    childrenMap.put(child.getName(), child);
    
    // 4. 设置父节点
    child.setParent(this);
    
    // 5. 更新配额
    quota.update(child);
}
```

#### 2. removeChild - 移除子节点
```java
public void removeChild(INode child) {
    // 1. 从列表中移除
    children.remove(child);
    
    // 2. 从映射中移除
    childrenMap.remove(child.getName());
    
    // 3. 清除父节点
    child.setParent(null);
    
    // 4. 更新配额
    quota.update(child);
}
```

#### 3. getChild - 获取子节点
```java
public INode getChild(String name) {
    return childrenMap.get(name);
}
```

---

### 3.10 INodeFile - 文件节点

**位置**: `INodeFile.java`

**作用**: 表示文件系统中的文件

**核心字段**:
```java
// 块信息
private final BlockInfo[] blocks;                   // 块信息
private final short replication;                     // 副本数
private final long preferredBlockSize;               // 首选块大小

// 文件状态
private final FileUnderConstructionFeature uc;       // 构建中特性
```

**核心方法**:

#### 1. getBlocks - 获取块信息
```java
public BlockInfo[] getBlocks() {
    return blocks;
}
```

#### 2. getReplication - 获取副本数
```java
public short getReplication() {
    return replication;
}
```

#### 3. getPreferredBlockSize - 获取首选块大小
```java
public long getPreferredBlockSize() {
    return preferredBlockSize;
}
```

---

## 四、工作流程

### 4.1 文件创建流程

```
1. 客户端调用 create()
   ↓
2. NameNodeRpcServer 接收请求
   ↓
3. 检查权限
   ↓
4. 解析路径
   ↓
5. 创建 INodeFile
   ↓
6. 分配块
   ↓
7. 记录编辑日志
   ↓
8. 返回文件状态
```

### 4.2 文件删除流程

```
1. 客户端调用 delete()
   ↓
2. NameNodeRpcServer 接收请求
   ↓
3. 检查权限
   ↓
4. 解析路径
   ↓
5. 删除 INode
   ↓
6. 释放块
   ↓
7. 记录编辑日志
   ↓
8. 返回结果
```

### 4.3 块分配流程

```
1. 客户端请求块
   ↓
2. NameNodeRpcServer 接收请求
   ↓
3. 检查配额
   ↓
4. 选择 DataNode
   ↓
5. 分配块
   ↓
6. 更新块信息
   ↓
7. 记录编辑日志
   ↓
8. 返回块位置
```

### 4.4 心跳处理流程

```
1. DataNode 发送心跳
   ↓
2. NameNodeRpcServer 接收心跳
   ↓
3. 更新 DataNode 状态
   ↓
4. 处理块报告
   ↓
5. 发送指令
   ↓
6. 返回响应
```

---

## 五、关键特性

### 5.1 高可用性

**优势**:
- 主备切换
- 故障自动转移
- 共享编辑日志

**实现**:
- HA 架构
- ZK 故障转移
- QJM 共享存储

### 5.2 安全模式

**优势**:
- 保护数据完整性
- 防止数据丢失
- 确保副本完整

**实现**:
- 块完整性检查
- DataNode 数量检查
- 副本数量检查

### 5.3 租约管理

**优势**:
- 防止并发写入
- 自动释放过期租约
- 支持租约续期

**实现**:
- 租约映射
- 租约监控
- 租约过期处理

### 5.4 编辑日志

**优势**:
- 记录所有操作
- 支持恢复
- 支持检查点

**实现**:
- 事务日志
- 日志分段
- 日志压缩

### 5.5 镜像管理

**优势**:
- 快速启动
- 数据持久化
- 支持检查点

**实现**:
- 镜像格式
- 镜像压缩
- 镜像加载

---

## 六、配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `dfs.namenode.name.dir` | file:///... | NameNode 存储目录 |
| `dfs.namenode.edits.dir` | file:///... | 编辑日志目录 |
| `dfs.namenode.safemode.threshold` | 0.999f | 安全模式阈值 |
| `dfs.namenode.safemode.min.datanodes` | 0 | 安全模式最小 DataNode 数 |
| `dfs.namenode.lease.recheck.interval` | 2000ms | 租约检查间隔 |
| `dfs.namenode.handler.count` | 10 | NameNode 处理线程数 |
| `dfs.namenode.replication.min` | 1 | 最小副本数 |
| `dfs.namenode.replication.max` | 512 | 最大副本数 |

---

## 七、总结

NameNode 是 HDFS 的核心组件，具有以下特点：

1. **集中式管理**: 管理所有元数据
2. **高可用性**: 支持主备切换
3. **数据一致性**: 编辑日志和镜像
4. **安全模式**: 保护数据完整性
5. **租约管理**: 防止并发写入

**关键设计思想**:
- 主从架构
- 编辑日志和镜像
- 安全模式保护
- 租约管理机制
- 高可用架构

通过深入理解 NameNode 的实现，可以更好地优化 HDFS 的性能和可靠性。
