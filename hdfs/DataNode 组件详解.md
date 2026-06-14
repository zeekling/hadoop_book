# HDFS DataNode 组件详解

## 一、概述

DataNode 是 HDFS 的数据节点，负责存储实际的数据块，并处理来自客户端和 NameNode 的数据读写请求。DataNode 定期向 NameNode 发送心跳和块报告，以保持与 NameNode 的同步。

**核心职责**：
- 存储和管理数据块
- 处理数据读写请求
- 向 NameNode 发送心跳和块报告
- 执行块复制和恢复
- 管理存储空间
- 执行数据完整性校验

**位置**: `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/datanode/`

---

## 二、核心架构

### 2.1 类继承关系

```
DataNode
├── DataXceiverServer (数据传输服务)
│   ├── DataXceiver (数据接收器)
│   ├── BlockSender (块发送器)
│   └── BlockReceiver (块接收器)
├── BlockPoolManager (块池管理)
│   ├── BPServiceActor (块池服务执行器)
│   └── BPOfferService (块池服务)
├── DataStorage (数据存储)
│   ├── FsDatasetImpl (数据集实现)
│   ├── Replica (副本)
│   └── VolumeScanner (卷扫描器)
├── BlockScanner (块扫描器)
├── DirectoryScanner (目录扫描器)
└── DiskBalancer (磁盘均衡器)
```

### 2.2 主要组件

```
org.apache.hadoop.hdfs.server.datanode
├── DataNode.java                    - DataNode 主类
├── DataXceiverServer.java           - 数据传输服务
├── DataXceiver.java                 - 数据接收器
├── BlockSender.java                 - 块发送器
├── BlockReceiver.java               - 块接收器
├── BlockPoolManager.java             - 块池管理
├── BPServiceActor.java              - 块池服务执行器
├── BPOfferService.java              - 块池服务
├── DataStorage.java                 - 数据存储
├── FsDatasetImpl.java               - 数据集实现
├── Replica.java                      - 副本
├── ReplicaInPipeline.java            - 写入中的副本
├── FinalizedReplica.java             - 已完成的副本
├── ReplicaUnderRecovery.java        - 恢复中的副本
├── BlockScanner.java                - 块扫描器
├── VolumeScanner.java               - 卷扫描器
├── DirectoryScanner.java            - 目录扫描器
├── DiskBalancer.java                - 磁盘均衡器
└── BlockRecoveryWorker.java         - 块恢复工作器
```

---

## 三、主要组件详解

### 3.1 DataNode - DataNode 主类

**位置**: `DataNode.java:18`

**作用**: DataNode 的主类，负责启动和管理 DataNode 服务

**核心字段**:
```java
// 服务状态
private volatile boolean isRunning;               // 是否运行中
private final DataXceiverServer dataXceiverServer;  // 数据传输服务
private final BlockPoolManager blockPoolManager;    // 块池管理器

// 存储
private final DataStorage storage;                 // 数据存储
private final FsDatasetImpl dataset;                // 数据集

// 配置
private final Configuration conf;                  // 配置
private final DNConf dnConf;                       // DataNode 配置

// 统计
private final DataNodeMXBean metrics;              // 指标
```

**核心方法**:

#### 1. main - 主入口
```java
public static void main(String args[]) {
    // 1. 解析启动参数
    if (args.length != 0) {
        System.err.println("Usage: java DataNode");
        System.exit(-1);
    }
    
    // 2. 创建 DataNode
    DataNode datanode = createDataNode(args, null);
    
    // 3. 启动 DataNode
    datanode.runDatanodeDaemon();
}
```

#### 2. runDatanodeDaemon - 运行 DataNode 守护进程
```java
public void runDatanodeDaemon() throws IOException {
    // 1. 初始化存储
    storage.recoverTransitionRead();
    
    // 2. 启动数据传输服务
    startDataXceiverServer();
    
    // 3. 启动块池管理器
    blockPoolManager.start();
    
    // 4. 启动块扫描器
    blockScanner.start();
    
    // 5. 启动目录扫描器
    directoryScanner.start();
    
    // 6. 启动磁盘均衡器
    diskBalancer.start();
    
    // 7. 向 NameNode 注册
    register();
    
    // 8. 进入运行状态
    isRunning = true;
}
```

#### 3. register - 向 NameNode 注册
```java
public void register() throws IOException {
    // 1. 创建注册信息
    DatanodeRegistration registration = createRegistration();
    
    // 2. 向 NameNode 发送注册请求
    namenode.registerDatanode(registration);
    
    // 3. 启动心跳线程
    startHeartbeatThread();
    
    // 4. 启动块报告线程
    startBlockReportThread();
}
```

---

### 3.2 DataXceiverServer - 数据传输服务

**位置**: `DataXceiverServer.java`

**作用**: 处理数据传输请求，包括数据块的读取、写入、复制等

**核心字段**:
```java
// 服务配置
private final ServerSocketChannel ss;              // 服务器套接字
private final int maxXceiverCount;                  // 最大接收器数量
private final int maxXceiverCountPerPeer;           // 每个对等节点的最大接收器数量

// 接收器管理
private final Map<Peer, DataXceiver> peers;          // 对等节点映射
private final AtomicInteger xceiverCount;            // 接收器计数
```

**核心方法**:

#### 1. run - 运行服务
```java
@Override
public void run() {
    // 1. 绑定端口
    ss = ServerSocketChannel.open();
    ss.bind(new InetSocketAddress(port));
    
    // 2. 开始监听
    while (isRunning) {
        // 3. 接受连接
        SocketChannel s = ss.accept();
        
        // 4. 创建对等节点
        Peer peer = new Peer(s);
        
        // 5. 创建数据接收器
        DataXceiver xceiver = new DataXceiver(peer, this);
        
        // 6. 启动数据接收器
        xceiver.start();
    }
}
```

#### 2. addPeer - 添加对等节点
```java
public void addPeer(Peer peer, DataXceiver xceiver) {
    // 1. 检查接收器数量
    if (xceiverCount.get() >= maxXceiverCount) {
        throw new IOException("Too many xceivers");
    }
    
    // 2. 检查对等节点数量
    if (peers.containsKey(peer)) {
        throw new IOException("Peer already exists");
    }
    
    // 3. 添加到映射
    peers.put(peer, xceiver);
    
    // 4. 增加计数
    xceiverCount.incrementAndGet();
}
```

#### 3. removePeer - 移除对等节点
```java
public void removePeer(Peer peer) {
    // 1. 从映射中移除
    DataXceiver xceiver = peers.remove(peer);
    
    // 2. 减少计数
    if (xceiver != null) {
        xceiverCount.decrementAndGet();
    }
}
```

---

### 3.3 DataXceiver - 数据接收器

**位置**: `DataXceiver.java`

**作用**: 处理单个数据传输请求

**核心字段**:
```java
// 对等节点
private final Peer peer;                           // 对等节点
private final DataXceiverServer parent;            // 父服务

// 数据传输
private final DataTransferProtocol transferProtocol;  // 数据传输协议
private final Sender sender;                        // 发送器
```

**核心方法**:

#### 1. run - 运行接收器
```java
@Override
public void run() {
    // 1. 初始化数据传输协议
    transferProtocol = new DataTransferProtocol(peer);
    
    // 2. 处理请求
    while (isRunning) {
        // 3. 读取操作码
        byte op = transferProtocol.readOp();
        
        // 4. 处理操作
        switch (op) {
            case DataTransferProtocol.OP_READ_BLOCK:
                processReadBlock();
                break;
            case DataTransferProtocol.OP_WRITE_BLOCK:
                processWriteBlock();
                break;
            case DataTransferProtocol.OP_COPY_BLOCK:
                processCopyBlock();
                break;
            case DataTransferProtocol.OP_REPLACE_BLOCK:
                processReplaceBlock();
                break;
            default:
                throw new IOException("Unknown operation: " + op);
        }
    }
}
```

#### 2. processReadBlock - 处理读块请求
```java
private void processReadBlock() throws IOException {
    // 1. 读取请求参数
    Block block = transferProtocol.readBlock();
    long offset = transferProtocol.readOffset();
    long length = transferProtocol.readLength();
    
    // 2. 获取块数据
    Replica replica = dataset.getReplica(block);
    
    // 3. 创建块发送器
    BlockSender blockSender = new BlockSender(replica, offset, length);
    
    // 4. 发送数据
    blockSender.sendBlock(peer);
}
```

#### 3. processWriteBlock - 处理写块请求
```java
private void processWriteBlock() throws IOException {
    // 1. 读取请求参数
    Block block = transferProtocol.readBlock();
    long offset = transferProtocol.readOffset();
    long length = transferProtocol.readLength();
    
    // 2. 创建块接收器
    BlockReceiver blockReceiver = new BlockReceiver(block, offset, length);
    
    // 3. 接收数据
    blockReceiver.receiveBlock(peer);
    
    // 4. 保存数据
    dataset.addReplica(block, blockReceiver.getReplica());
}
```

---

### 3.4 BlockPoolManager - 块池管理

**位置**: `BlockPoolManager.java`

**作用**: 管理多个块池，每个块池对应一个命名空间

**核心字段**:
```java
// 块池映射
private final Map<String, BPOfferService> bpOfferServices;  // 块池服务映射
private final Map<String, BPServiceActor> bpServiceActors;    // 块池服务执行器映射

// 配置
private final Configuration conf;                  // 配置
```

**核心方法**:

#### 1. addBlockPool - 添加块池
```java
public void addBlockPool(String bpid) throws IOException {
    // 1. 检查是否已存在
    if (bpOfferServices.containsKey(bpid)) {
        return;
    }
    
    // 2. 创建块池服务
    BPOfferService bpOfferService = new BPOfferService(bpid, conf);
    bpOfferServices.put(bpid, bpOfferService);
    
    // 3. 创建块池服务执行器
    BPServiceActor bpServiceActor = new BPServiceActor(bpid, conf);
    bpServiceActors.put(bpid, bpServiceActor);
    
    // 4. 启动服务
    bpOfferService.start();
    bpServiceActor.start();
}
```

#### 2. removeBlockPool - 移除块池
```java
public void removeBlockPool(String bpid) {
    // 1. 停止块池服务
    BPOfferService bpOfferService = bpOfferServices.remove(bpid);
    if (bpOfferService != null) {
        bpOfferService.stop();
    }
    
    // 2. 停止块池服务执行器
    BPServiceActor bpServiceActor = bpServiceActors.remove(bpid);
    if (bpServiceActor != null) {
        bpServiceActor.stop();
    }
}
```

---

### 3.5 BPServiceActor - 块池服务执行器

**位置**: `BPServiceActor.java`

**作用**: 处理块池的服务请求，包括心跳、块报告等

**核心字段**:
```java
// 块池信息
private final String blockPoolId;                   // 块池 ID
private final BPOfferService bpOfferService;        // 块池服务

// 状态
private volatile boolean isRunning;               // 是否运行中
private volatile boolean isAlive;                  // 是否存活

// 心跳
private final long heartbeatInterval;              // 心跳间隔
private final long blockReportInterval;             // 块报告间隔
```

**核心方法**:

#### 1. run - 运行服务执行器
```java
@Override
public void run() {
    // 1. 进入运行状态
    isRunning = true;
    isAlive = true;
    
    // 2. 发送初始块报告
    sendInitialBlockReport();
    
    // 3. 进入主循环
    while (isRunning) {
        try {
            // 4. 发送心跳
            sendHeartbeat();
            
            // 5. 发送块报告
            sendBlockReport();
            
            // 6. 等待下一个周期
            Thread.sleep(heartbeatInterval);
        } catch (InterruptedException e) {
            break;
        }
    }
    
    // 7. 退出运行状态
    isRunning = false;
    isAlive = false;
}
```

#### 2. sendHeartbeat - 发送心跳
```java
private void sendHeartbeat() throws IOException {
    // 1. 创建心跳请求
    DatanodeCommand[] cmds = new DatanodeCommand[0];
    
    // 2. 向 NameNode 发送心跳
    DatanodeCommand[] response = namenode.sendHeartbeat(
        blockPoolId, storageInfo, cmds);
    
    // 3. 处理响应
    processCommands(response);
}
```

#### 3. sendBlockReport - 发送块报告
```java
private void sendBlockReport() throws IOException {
    // 1. 获取所有块
    List<Block> blocks = dataset.getBlocks(blockPoolId);
    
    // 2. 创建块报告
    StorageBlockReport[] reports = createBlockReports(blocks);
    
    // 3. 向 NameNode 发送块报告
    namenode.blockReport(blockPoolId, reports);
}
```

---

### 3.6 DataStorage - 数据存储

**位置**: `DataStorage.java`

**作用**: 管理 DataNode 的数据存储

**核心字段**:
```java
// 存储目录
private final List<StorageLocation> storageLocations;  // 存储位置列表
private final Map<String, FsVolumeImpl> volumes;        // 卷映射

// 存储统计
private final long capacity;                         // 总容量
private final long used;                              // 已使用空间
private final long available;                         // 可用空间
```

**核心方法**:

#### 1. getStorageLocations - 获取存储位置
```java
public List<StorageLocation> getStorageLocations() {
    return storageLocations;
}
```

#### 2. getVolume - 获取卷
```java
public FsVolumeImpl getVolume(String volumeId) {
    return volumes.get(volumeId);
}
```

#### 3. getCapacity - 获取容量
```java
public long getCapacity() {
    return capacity;
}
```

---

### 3.7 FsDatasetImpl - 数据集实现

**位置**: `FsDatasetImpl.java`

**作用**: 实现数据集接口，管理数据块的存储和检索

**核心字段**:
```java
// 块映射
private final Map<String, Map<Block, Replica>> blockMap;  // 块映射

// 卷管理
private final Map<String, FsVolumeImpl> volumes;        // 卷映射

// 统计
private final long capacity;                         // 总容量
private final long used;                              // 已使用空间
```

**核心方法**:

#### 1. getReplica - 获取副本
```java
public Replica getReplica(Block block) throws IOException {
    // 1. 获取块池 ID
    String bpid = block.getBlockPoolId();
    
    // 2. 获取块池的块映射
    Map<Block, Replica> poolMap = blockMap.get(bpid);
    if (poolMap == null) {
        throw new IOException("Block pool not found: " + bpid);
    }
    
    // 3. 获取副本
    Replica replica = poolMap.get(block);
    if (replica == null) {
        throw new IOException("Replica not found: " + block);
    }
    
    return replica;
}
```

#### 2. addReplica - 添加副本
```java
public void addReplica(Block block, Replica replica) throws IOException {
    // 1. 获取块池 ID
    String bpid = block.getBlockPoolId();
    
    // 2. 获取块池的块映射
    Map<Block, Replica> poolMap = blockMap.get(bpid);
    if (poolMap == null) {
        poolMap = new HashMap<>();
        blockMap.put(bpid, poolMap);
    }
    
    // 3. 添加副本
    poolMap.put(block, replica);
    
    // 4. 更新统计
    updateStatistics(replica);
}
```

#### 3. removeReplica - 移除副本
```java
public void removeReplica(Block block) throws IOException {
    // 1. 获取块池 ID
    String bpid = block.getBlockPoolId();
    
    // 2. 获取块池的块映射
    Map<Block, Replica> poolMap = blockMap.get(bpid);
    if (poolMap == null) {
        return;
    }
    
    // 3. 移除副本
    Replica replica = poolMap.remove(block);
    
    // 4. 更新统计
    updateStatistics(replica);
}
```

---

### 3.8 Replica - 副本

**位置**: `Replica.java`

**作用**: 表示数据块的副本

**核心字段**:
```java
// 块信息
private final Block block;                          // 块信息
private final long generationStamp;                // 生成时间戳

// 副本状态
private final ReplicaState state;                   // 副本状态
private final long bytesOnDisk;                     // 磁盘字节数
```

**核心方法**:

#### 1. getBlock - 获取块信息
```java
public Block getBlock() {
    return block;
}
```

#### 2. getGenerationStamp - 获取生成时间戳
```java
public long getGenerationStamp() {
    return generationStamp;
}
```

#### 3. getState - 获取副本状态
```java
public ReplicaState getState() {
    return state;
}
```

---

### 3.9 BlockScanner - 块扫描器

**位置**: `BlockScanner.java`

**作用**: 定期扫描数据块，检查数据完整性

**核心字段**:
```java
// 扫描配置
private final long scanPeriod;                      // 扫描周期
private final int maxRetries;                        // 最大重试次数

// 扫描状态
private volatile boolean isRunning;               // 是否运行中
private final Map<Block, Integer> scannedBlocks;    // 已扫描的块
```

**核心方法**:

#### 1. run - 运行扫描器
```java
@Override
public void run() {
    // 1. 进入运行状态
    isRunning = true;
    
    // 2. 进入主循环
    while (isRunning) {
        try {
            // 3. 扫描所有块
            scanAllBlocks();
            
            // 4. 等待下一个周期
            Thread.sleep(scanPeriod);
        } catch (InterruptedException e) {
            break;
        }
    }
    
    // 5. 退出运行状态
    isRunning = false;
}
```

#### 2. scanAllBlocks - 扫描所有块
```java
private void scanAllBlocks() throws IOException {
    // 1. 获取所有块
    List<Block> blocks = dataset.getBlocks();
    
    // 2. 扫描每个块
    for (Block block : blocks) {
        // 3. 检查块完整性
        checkBlock(block);
        
        // 4. 记录扫描结果
        scannedBlocks.put(block, 1);
    }
}
```

#### 3. checkBlock - 检查块完整性
```java
private void checkBlock(Block block) throws IOException {
    // 1. 获取副本
    Replica replica = dataset.getReplica(block);
    
    // 2. 计算校验和
    byte[] checksum = calculateChecksum(replica);
    
    // 3. 验证校验和
    if (!verifyChecksum(replica, checksum)) {
        // 4. 报告坏块
        reportBadBlock(block);
    }
}
```

---

### 3.10 VolumeScanner - 卷扫描器

**位置**: `VolumeScanner.java`

**作用**: 定期扫描存储卷，检查卷的健康状态

**核心字段**```java
// 扫描配置
private final long scanPeriod;                      // 扫描周期
private final int maxRetries;                        // 最大重试次数

// 扫描状态
private volatile boolean isRunning;               // 是否运行中
private final Map<String, VolumeHealth> volumeHealth;  // 卷健康状态
```

**核心方法**:

#### 1. run - 运行扫描器
```java
@Override
public void run() {
    // 1. 进入运行状态
    isRunning = true;
    
    // 2. 进入主循环
    while (isRunning) {
        try {
            // 3. 扫描所有卷
            scanAllVolumes();
            
            // 4. 等待下一个周期
            Thread.sleep(scanPeriod);
        } catch (InterruptedException e) {
            break;
        }
    }
    
    // 5. 退出运行状态
    isRunning = false;
}
```

#### 2. scanAllVolumes - 扫描所有卷
```java
private void scanAllVolumes() throws IOException {
    // 1. 获取所有卷
    List<FsVolumeImpl> volumes = dataset.getVolumes();
    
    // 2. 扫描每个卷
    for (FsVolumeImpl volume : volumes) {
        // 3. 检查卷健康
        checkVolume(volume);
        
        // 4. 记录扫描结果
        volumeHealth.put(volume.getVolumeId(), volume.getHealth());
    }
}
```

#### 3. checkVolume - 检查卷健康
```java
private void checkVolume(FsVolumeImpl volume) throws IOException {
    // 1. 检查卷是否可访问
    if (!volume.isAccessible()) {
        // 2. 标记卷为失败
        volume.markFailed();
        return;
    }
    
    // 3. 检查卷空间
    long available = volume.getAvailable();
    if (available < MIN_AVAILABLE_SPACE) {
        // 4. 标记卷为空间不足
        volume.markLowSpace();
    }
}
```

---

## 四、工作流程

### 4.1 数据写入流程

```
1. 客户端请求写入数据
   ↓
2. DataXceiver 接收请求
   ↓
3. 创建 BlockReceiver
   ↓
4. 接收数据块
   ↓
5. 保存到本地存储
   ↓
6. 确认接收
   ↓
7. 向 NameNode 报告
```

### 4.2 数据读取流程

```
1. 客户端请求读取数据
   ↓
2. DataXceiver 接收请求
   ↓
3. 创建 BlockSender
   ↓
4. 从本地存储读取数据
   ↓
5. 发送数据给客户端
   ↓
6. 完成读取
```

### 4.3 心跳流程

```
1. BPServiceActor 启动
   ↓
2. 发送初始块报告
   ↓
3. 进入主循环
   ↓
4. 发送心跳
   ↓
5. 处理 NameNode 响应
   ↓
6. 等待下一个周期
```

### 4.4 块报告流程

```
1. BPServiceActor 启动
   ↓
2. 获取所有块
   ↓
3. 创建块报告
   ↓
4. 向 NameNode 发送块报告
   ↓
5. 处理 NameNode 响应
   ↓
6. 等待下一个周期
```

---

## 五、关键特性

### 5.1 数据完整性

**优势**:
- 定期扫描数据块
- 校验和验证
- 坏块检测和报告

**实现**:
- BlockScanner 定期扫描
- 校验和计算和验证
- 坏块报告机制

### 5.2 存储管理

**优势**:
- 多卷支持
- 空间管理
- 卷健康检查

**实现**:
- VolumeScanner 定期扫描
- 空间统计和报告
- 卷健康状态管理

### 5.3 数据传输

**优势**:
- 高性能数据传输
- 流式数据传输
- 支持多种传输模式

**实现**:
- DataXceiverServer 处理传输
- BlockSender 和 BlockReceiver
- 支持短路读取

### 5.4 块池管理

**优势**:
- 支持多个命名空间
- 独立的块池服务
- 独立的心跳和块报告

**实现**:
- BlockPoolManager 管理多个块池
- BPServiceActor 处理块池服务
- BPOfferService 提供块池服务

### 5.5 磁盘均衡

**优势**:
- 自动磁盘均衡
- 负载均衡
- 性能优化

**实现**:
- DiskBalancer 定期均衡
- 数据迁移策略
- 性能监控

---

## 六、配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `dfs.datanode.data.dir` | file:///... | DataNode 存储目录 |
| `dfs.datanode.address` | 0.0.0.0:50010 | DataNode 地址 |
| `dfs.datanode.http.address` | 0.0.0.0:50075 | DataNode HTTP 地址 |
| `dfs.heartbeat.interval` | 3s | 心跳间隔 |
| `dfs.blockreport.intervalMsec` | 21600000ms | 块报告间隔 |
| `dfs.datanode.handler.count` | 10 | DataNode 处理线程数 |
| `dfs.datanode.max.transfer.threads` | 4096 | 最大传输线程数 |
| `dfs.datanode.directoryscan.interval` | 21600000ms | 目录扫描间隔 |
| `dfs.blockreport.initialDelay` | 0ms | 初始块报告延迟 |

---

## 七、总结

DataNode 是 HDFS 的核心组件，具有以下特点：

1. **数据存储**: 存储和管理数据块
2. **数据传输**: 处理数据读写请求
3. **数据完整性**: 定期扫描和校验
4. **存储管理**: 多卷支持和空间管理
5. **块池管理**: 支持多个命名空间

**关键设计思想**:
- 主从架构（NameNode/DataNode）
- 数据本地化
- 块池管理
- 心跳和块报告
- 数据完整性检查

通过深入理解 DataNode 的实现，可以更好地优化 HDFS 的存储性能和可靠性。
