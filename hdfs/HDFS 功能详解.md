# HDFS 功能详解

## 一、概述

HDFS（Hadoop Distributed File System）是 Hadoop 的核心分布式文件系统，设计用于存储和处理大规模数据集。HDFS 提供高吞吐量的数据访问，适合大规模数据集的应用程序。

**核心目标**：
- 存储和处理大规模数据集
- 提供高吞吐量的数据访问
- 容错性和高可用性
- 简化的数据一致性模型
- 运行在廉价的硬件上

**位置**: `hadoop-hdfs-project/hadoop-hdfs/`

---

## 二、核心架构

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                        HDFS 客户端                            │
│  - DistributedFileSystem                                     │
│  - DFSClient                                                │
│  - DFSInputStream / DFSOutputStream                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      NameNode (主节点)                       │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  NameNodeRpcServer (RPC 服务)                        │   │
│  │  - ClientProtocol                                    │   │
│  │  - DatanodeProtocol                                 │   │
│  │  - NamenodeProtocol                                 │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  FSNamesystem (命名空间管理)                        │   │
│  │  - INodeMap (INode 映射)                            │   │
│  │  - BlockManager (块管理)                            │   │
│  │  - DatanodeManager (DataNode 管理)                   │   │
│  │  - LeaseManager (租约管理)                          │   │
│  │  - SafeMode (安全模式)                              │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  FSImage (文件系统镜像)                             │   │
│  │  - NNStorage (存储管理)                             │   │
│  │  - FSEditLog (编辑日志)                             │   │
│  │  - Checkpointer (检查点)                            │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  HAContext (高可用上下文)                           │   │
│  │  - ActiveState (活跃状态)                          │   │
│  │  - StandbyState (待机状态)                          │   │
│  │  - ZKFailoverController (ZK 故障转移)               │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    DataNode (数据节点)                       │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  DataXceiverServer (数据传输服务)                    │   │
│  │  - DataXceiver (数据接收器)                          │   │
│  │  - BlockSender (块发送器)                           │   │
│  │  - BlockReceiver (块接收器)                         │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  BlockPoolManager (块池管理)                        │   │
│  │  - BPServiceActor (块池服务)                        │   │
│  │  - BPOfferService (块池服务)                        │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  DataStorage (数据存储)                              │   │
│  │  - FsDatasetImpl (数据集实现)                       │   │
│  │  - Replica (副本)                                   │   │
│  │  - VolumeScanner (卷扫描器)                         │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  BlockScanner (块扫描器)                            │   │
│  │  - DirectoryScanner (目录扫描器)                    │   │
│  │  - DiskBalancer (磁盘均衡器)                       │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 核心组件

**NameNode 组件**:
- `NameNode` - 主节点类
- `FSNamesystem` - 命名空间管理
- `NameNodeRpcServer` - RPC 服务
- `FSImage` - 文件系统镜像
- `FSEditLog` - 编辑日志
- `BlockManager` - 块管理
- `DatanodeManager` - DataNode 管理
- `LeaseManager` - 租约管理
- `SafeMode` - 安全模式

**DataNode 组件**:
- `DataNode` - 数据节点类
- `DataXceiverServer` - 数据传输服务
- `BlockPoolManager` - 块池管理
- `DataStorage` - 数据存储
- `BlockScanner` - 块扫描器
- `VolumeScanner` - 卷扫描器
- `DiskBalancer` - 磁盘均衡器

**客户端组件**:
- `DistributedFileSystem` - 分布式文件系统
- `DFSClient` - DFS 客户端
- `DFSInputStream` - DFS 输入流
- `DFSOutputStream` - DFS 输出流
- `DataStreamer` - 数据流发送器

---

## 三、核心功能模块

### 3.1 命名空间管理

**位置**: `FSNamesystem.java`

**作用**: 管理 HDFS 的命名空间，包括目录结构、文件元数据、权限等

**核心功能**:
- 文件和目录的创建、删除、重命名
- 权限管理（ACL、权限位）
- 配额管理（空间配额、名称配额）
- 快照管理
- 加密区域管理
- 擦除码策略管理

**核心类**:
- `INode` - 节点基类
- `INodeDirectory` - 目录节点
- `INodeFile` - 文件节点
- `INodeSymlink` - 符号链接节点
- `INodeMap` - 节点映射
- `INodesInPath` - 路径中的节点

---

### 3.2 块管理

**位置**: `BlockManager.java`

**作用**: 管理 HDFS 的块，包括块的分配、复制、恢复、删除等

**核心功能**:
- 块分配和放置策略
- 块复制和恢复
- 块报告和心跳
- 块失效和重新复制
- 块缓存管理
- 擦除码块管理

**核心类**:
- `BlockManager` - 块管理器
- `DatanodeManager` - DataNode 管理器
- `BlockPlacementPolicy` - 块放置策略
- `ReplicaRecovery` - 副本恢复
- `UnderReplicatedBlocks` - 欠复制块
- `PendingReplicationBlocks` - 待复制块

---

### 3.3 数据存储

**位置**: `DataStorage.java`

**作用**: 管理 DataNode 的数据存储

**核心功能**:
- 数据卷管理
- 块存储和检索
- 副本管理
- 存储空间管理
- 数据完整性校验

**核心类**:
- `DataStorage` - 数据存储
- `FsDatasetImpl` - 数据集实现
- `Replica` - 副本
- `ReplicaInPipeline` - 写入中的副本
- `FinalizedReplica` - 已完成的副本
- `ReplicaUnderRecovery` - 恢复中的副本

---

### 3.4 数据传输

**位置**: `DataXceiverServer.java`

**作用**: 处理数据传输请求

**核心功能**:
- 数据块读取
- 数据块写入
- 数据块复制
- 数据块恢复
- 短路读取

**核心类**:
- `DataXceiverServer` - 数据传输服务
- `DataXceiver` - 数据接收器
- `BlockSender` - 块发送器
- `BlockReceiver` - 块接收器
- `BlockChecksumHelper` - 块校验和助手

---

### 3.5 高可用性

**位置**: `ha/` 目录

**作用**: 实现 HDFS 的高可用性

**核心功能**:
- 主备切换
- 故障转移
- 共享编辑日志
- ZK 故障转移控制器
- 观察者模式

**核心类**:
- `HAContext` - 高可用上下文
- `ActiveState` - 活跃状态
- `StandbyState` - 待机状态
- `ZKFailoverController` - ZK 故障转移控制器
- `EditLogTailer` - 编辑日志跟踪器
- `StandbyCheckpointer` - 待机检查点

---

### 3.6 安全性

**位置**: `security/` 目录

**作用**: 实现 HDFS 的安全功能

**核心功能**:
- 认证和授权
- 访问控制列表（ACL）
- 加密
- 令牌管理
- 审计日志

**核心类**:
- `PermissionChecker` - 权限检查器
- `AclStorage` - ACL 存储
- `EncryptionZoneManager` - 加密区域管理器
- `DelegationTokenSecretManager` - 委托令牌密钥管理器
- `AuditLogger` - 审计日志记录器

---

### 3.7 快照

**位置**: `snapshot/` 目录

**作用**: 实现 HDFS 的快照功能

**核心功能**:
- 快照创建和删除
- 快照差异
- 快照恢复
- 快照路径解析

**核心类**:
- `SnapshotManager` - 快照管理器
- `Snapshot` - 快照
- `Diff` - 差异
- `INodeDirectorySnapshottable` - 可快照目录

---

### 3.8 缓存

**位置**: `server/blockmanagement/CacheManager.java`

**作用**: 实现 HDFS 的集中式缓存管理

**核心功能**:
- 缓存指令管理
- 缓存池管理
- 缓存块管理
- 缓存统计

**核心类**:
- `CacheManager` - 缓存管理器
- `CachePool` - 缓存池
- `CachedBlock` - 缓存块
- `CacheDirective` - 缓存指令

---

### 3.9 擦除码

**位置**: `erasurecoding/` 目录

**作用**: 实现 HDFS 的擦除码功能

**核心功能**:
- 擦除码策略管理
- 擦除码块管理
- 擦除码恢复
- 擦除码读取

**核心类**:
- `ErasureCodingPolicyManager` - 擦除码策略管理器
- `StripedBlockInfo` - 条带块信息
- `ErasureCodingWork` - 擦除码工作
- `ReconstructStripedBlock` - 重建条带块

---

### 3.10 Web UI

**位置**: `web/` 目录

**作用**: 提供 HDFS 的 Web 界面

**核心功能**:
- 文件系统浏览
- 集群状态监控
- 块报告查看
- 日志查看

**核心类**:
- `NameNodeHttpServer` - NameNode HTTP 服务
- `DfsServlet` - DFS Servlet
- `FsckServlet` - Fsck Servlet
- `ImageServlet` - 镜像 Servlet

---

## 四、工作流程

### 4.1 文件写入流程

```
1. 客户端创建文件
   ↓
2. 调用 DistributedFileSystem.create()
   ↓
3. DFSClient 创建 DFSOutputStream
   ↓
4. 向 NameNode 申请块
   ↓
5. NameNode 分配块并返回 DataNode 列表
   ↓
6. 客户端建立数据管道
   ↓
7. 数据写入 DataNode 管道
   ↓
8. DataNode 确认接收
   ↓
9. 客户端继续写入下一个块
   ↓
10. 完成写入，关闭文件
   ↓
11. NameNode 提交编辑日志
```

### 4.2 文件读取流程

```
1. 客户端打开文件
   ↓
2. 调用 DistributedFileSystem.open()
   ↓
3. DFSClient 创建 DFSInputStream
   ↓
4. 向 NameNode 请求块位置
   ↓
5. NameNode 返回块位置信息
   ↓
6. 客户端从 DataNode 读取数据
   ↓
7. 如果读取失败，尝试其他副本
   ↓
8. 继续读取下一个块
   ↓
9. 完成读取，关闭文件
```

### 4.3 块恢复流程

```
1. DataNode 检测到块损坏
   ↓
2. 向 NameNode 报告坏块
   ↓
3. NameNode 标记块为损坏
   ↓
4. NameNode 选择源 DataNode
   ↓
5. NameNode 选择目标 DataNode
   ↓
6. 源 DataNode 复制块到目标 DataNode
   ↓
7. 目标 DataNode 确认接收
   ↓
8. NameNode 更新块信息
```

### 4.4 心跳和块报告流程

```
1. DataNode 启动
   ↓
2. 向 NameNode 注册
   ↓
3. 定期发送心跳
   ↓
4. 定期发送块报告
   ↓
5. NameNode 处理心跳
   ↓
6. NameNode 处理块报告
   ↓
7. NameNode 发送指令
   ↓
8. DataNode 执行指令
```

---

## 五、关键特性

### 5.1 高吞吐量

**优势**:
- 流式数据访问
- 批量数据处理
- 数据本地化

**实现**:
- 大块大小（默认 128MB）
- 数据本地化策略
- 流式数据传输

### 5.2 容错性

**优势**:
- 数据副本
- 自动故障恢复
- 数据完整性校验

**实现**:
- 默认 3 副本
- 心跳检测
- 校验和验证

### 5.3 高可用性

**优势**:
- 主备切换
- 故障自动转移
- 共享编辑日志

**实现**:
- HA 架构
- ZK 故障转移
- QJM 共享存储

### 5.4 可扩展性

**优势**:
- 水平扩展
- 线性扩展
- 负载均衡

**实现**:
- 分布式架构
- 数据分片
- 负载均衡策略

### 5.5 数据一致性

**优势**:
- 写一次读多次
- 强一致性
- 原子操作

**实现**:
- 写一次读多次模型
- 编辑日志
- 原子操作

---

## 六、配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `dfs.blocksize` | 128MB | 块大小 |
| `dfs.replication` | 3 | 副本数 |
| `dfs.namenode.name.dir` | file:///... | NameNode 存储目录 |
| `dfs.datanode.data.dir` | file:///... | DataNode 存储目录 |
| `dfs.heartbeat.interval` | 3s | 心跳间隔 |
| `dfs.blockreport.intervalMsec` | 21600000ms | 块报告间隔 |
| `dfs.namenode.handler.count` | 10 | NameNode 处理线程数 |
| `dfs.datanode.handler.count` | 10 | DataNode 处理线程数 |
| `dfs.namenode.safemode.threshold` | 0.999f | 安全模式阈值 |
| `dfs.namenode.safemode.min.datanodes` | 0 | 安全模式最小 DataNode 数 |

---

## 七、总结

HDFS 是 Hadoop 的核心分布式文件系统，具有以下特点：

1. **高吞吐量**: 流式数据访问，适合大规模数据处理
2. **容错性**: 数据副本和自动故障恢复
3. **高可用性**: 主备切换和故障自动转移
4. **可扩展性**: 水平扩展和线性扩展
5. **数据一致性**: 写一次读多次和强一致性

**关键设计思想**:
- 主从架构（NameNode/DataNode）
- 大块大小（减少元数据开销）
- 数据本地化（提高性能）
- 副本机制（容错性）
- 流式数据访问（高吞吐量）

通过深入理解 HDFS 的实现，可以更好地使用和优化 HDFS，提高数据处理效率。
