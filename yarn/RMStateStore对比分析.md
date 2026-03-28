# FileSystemRMStateStore、LeveldbRMStateStore、ZKRMStateStore 详细对比分析

## 一、概述对比

| 特性 | FileSystemRMStateStore | LeveldbRMStateStore | ZKRMStateStore |
|------|----------------------|---------------------|----------------|
| **存储后端** | Hadoop FileSystem (HDFS/S3/本地) | LevelDB 嵌入式数据库 | ZooKeeper 分布式协调服务 |
| **代码行数** | ~1018 行 | ~797 行 | ~1493 行 |
| **版本** | 1.3 | 1.1 | 1.5 |
| **依赖** | Hadoop FileSystem API | leveldbjni (JNI) | Apache Curator + Zookeeper |
| **架构类型** | 主从式 (共享存储) | 单机嵌入式 | 分布式共识 |
| **一致性模型** | 最终一致性 (依赖FS) | 强一致性 | 强一致性 (ZAB协议) |

---

## 二、存储架构对比

### 2.1 FileSystemRMStateStore - 目录/文件结构

```
hdfs://namenode:8020/rmstate/FSRMStateRoot/
├── RMVersionNode                    # 版本信息
├── EpochNode                       # Epoch计数器
├── RMDTSecretManagerRoot/         # 委托令牌
│   ├── DelegationKey_<id>
│   ├── RMDelegationToken_<seq>
│   └── RMDTSequenceNumber_<seq>
├── AMRMTokenSecretManagerRoot/    # AMRM令牌
│   └── AMRMTokenSecretManagerNode
├── RMAppRoot/                     # 应用状态
│   └── <ApplicationId>/
│       ├── <ApplicationId>
│       └── attempt_<id>
├── ReservationSystemRoot/         # 资源预留
│   └── <PlanName>/
└── ProxyCARoot/                   # CA证书
    ├── caCert
    └── caPrivateKey
```

**特点**：
- 树形目录结构
- 每个状态项对应一个文件
- 使用文件重命名保证原子性

### 2.2 LeveldbRMStateStore - Key-Value 结构

```
数据库: yarn-rm-state (LevelDB)

Key                                    | Value
--------------------------------------|------------------------------
RMVersionNode                          | VersionProto
EpochNode                              | EpochProto
RMDTSecretManagerRoot/DelegationKey_<id> | DelegationKey
RMDTSecretManagerRoot/RMDelegationToken_<seq> | TokenData
RMDTSecretManagerRoot/RMDTSequentialNumber | Int
RMAppRoot/application_<id>           | ApplicationStateDataProto
AMRMTokenSecretManagerRoot            | AMRMTokenStateProto
ProxyCARoot/caCert                    | X509Certificate (DER)
ProxyCARoot/caPrivateKey              | PrivateKey (DER)
ReservationSystemRoot/<plan>/<res_id> | ReservationProto
```

**特点**：
- 扁平化 Key-Value 存储
- Key 使用路径风格 (`/`)
- 内置 B+Tree 索引

### 2.3 ZKRMStateStore - Znode 层次结构

```
/yarn rm/ZKRMStateRoot/
├── VERSION_INFO                      # 版本
├── EPOCH_NODE                       # Epoch
├── RM_ZK_FENCING_LOCK              # 锁节点 (HA)
├── RMAppRoot/
│   ├── HIERARCHIES/                # 分层索引 (1-4)
│   │   ├── 1/application_1
│   │   │   └── application_1234_0001_01  # 尝试状态
│   │   ├── 2/application_12
│   │   │   └── application_34_0001_01
│   │   └── ...
│   └── application_<id>
│       └── attempt_<id>
├── RMDTSecretManagerRoot/
│   ├── RMDTSequentialNumber        # 序列号
│   ├── RMDTMasterKeysRoot/
│   │   └── Key_<id>
│   └── RMDelegationTokensRoot/
│       ├── 1/Token_1
│       │   └── Token_1234
│       └── ...
├── AMRMTokenSecretManagerRoot/
│   ├── currentMasterKey
│   └── nextMasterKey
├── ReservationSystemRoot/
│   └── <PlanName>/
│       └── <ReservationId>
└── ProxyCARoot/
    ├── caCert
    └── caPrivateKey
```

**特点**：
- 分层目录设计 (优化 ZK 节点数量)
- 支持节点分割 (split index 1-4)
- 内置 ACL 权限控制
- 临时节点用于 HA 锁

---

## 三、核心功能实现对比

### 3.1 应用状态管理

#### 存储操作

| 实现 | 存储方式 | 原子性保证 |
|------|----------|------------|
| **FileSystemRMStateStore** | 临时文件 + rename (2-3次IO) | 依赖文件系统 rename |
| **LeveldbRMStateStore** | 直接 db.put() (1次IO) | LevelDB 事务 |
| **ZKRMStateStore** | safeCreate() + 事务 (多次IO) | ZK 事务 (Multi) |

```java
// FileSystemRMStateStore - 临时文件策略
protected void writeFile(Path outputPath, byte[] data, boolean makeUnreadableByAdmin) {
    Path tempPath = new Path(outputPath.getParent(), outputPath.getName() + ".tmp");
    fsOut = fs.create(tempPath, true);
    fsOut.write(data);
    fsOut.close();
    fs.rename(tempPath, outputPath);  // 原子重命名
}

// LeveldbRMStateStore - 直接写入
protected void storeApplicationStateInternal(ApplicationId appId, ApplicationStateData appStateData) {
    db.put(bytes(key), appStateData.getProto().toByteArray());
}

// ZKRMStateStore - 安全创建
public synchronized void storeApplicationStateInternal(ApplicationId appId, 
    ApplicationStateData appStateDataPB) throws Exception {
    String nodeCreatePath = getLeafAppIdNodePath(appId.toString(), true);
    zkManager.safeCreate(nodeCreatePath, appStateData, zkAcl,
        CreateMode.PERSISTENT, zkAcl, fencingNodePath);
}
```

### 3.2 委托令牌管理

| 实现 | 令牌存储 | 序列号存储 | 原子性 |
|------|----------|------------|--------|
| **FileSystemRMStateStore** | 单独文件 | 单独文件 | ❌ 不保证 |
| **LeveldbRMStateStore** | WriteBatch | WriteBatch | ✅ 保证 |
| **ZKRMStateStore** | 事务 | 事务 | ✅ 保证 |

```java
// FileSystemRMStateStore - 分开存储，存在不一致风险
private void storeOrUpdateRMDelegationTokenState(...) {
    writeFileWithRetries(nodeCreatePath, identifierData.toByteArray(), true);
    // 序列号单独存储...
    if (dtSequenceNumberPath == null) {
        createFileWithRetries(latestSequenceNumberPath);
    } else {
        renameFileWithRetries(dtSequenceNumberPath, latestSequenceNumberPath);
    }
}

// LeveldbRMStateStore - 原子批量操作
private void storeOrUpdateRMDT(...) {
    try (WriteBatch batch = db.createWriteBatch()) {
        batch.put(bytes(tokenKey), tokenData.toByteArray());
        if (!isUpdate) {
            batch.put(bytes(RM_DT_SEQUENCE_NUMBER_KEY), bs.toByteArray());
        }
        db.write(batch);  // 原子提交
    }
}

// ZKRMStateStore - ZK 事务
protected synchronized void storeRMDelegationTokenState(...) {
    SafeTransaction trx = zkManager.createTransaction(zkAcl, fencingNodePath);
    trx.create(nodeCreatePath, identifierData.toByteArray(), zkAcl, CreateMode.PERSISTENT);
    trx.setData(dtSequenceNumberPath, seqOs.toByteArray(), -1);
    trx.commit();  // 原子提交
}
```

### 3.3 删除操作

| 实现 | 应用删除 | 尝试删除 | 批量删除 |
|------|----------|----------|----------|
| **FileSystemRMStateStore** | 递归删除目录 | 单独删除 | ❌ 无 |
| **LeveldbRMStateStore** | WriteBatch 删除 | WriteBatch 删除 | ✅ 支持 |
| **ZKRMStateStore** | safeDelete + 层级清理 | safeDelete | ✅ 支持 |

---

## 四、高可用特性对比

### 4.1 HA 支持

| 特性 | FileSystemRMStateStore | LeveldbRMStateStore | ZKRMStateStore |
|------|----------------------|---------------------|----------------|
| **HA 支持** | ✅ 支持 (共享存储) | ❌ 不支持 | ✅ 支持 |
| **Fencing 机制** | ❌ 无 | ❌ 无 | ✅ 有 |
| **Epoch 管理** | ✅ 有 | ✅ 有 | ✅ 有 |
| **脑裂防护** | 依赖共享存储 | 不支持 | ZK 锁 |

### 4.2 ZKRMStateStore 专用 HA 特性

```java
// Fencing 锁节点
private static final String FENCING_LOCK = "RM_ZK_FENCING_LOCK";
private String fencingNodePath;

// 验证活跃状态线程
private class VerifyActiveStatusThread extends Thread {
    @Override
    public void work() {
        while (!isFencedState()) {
            // 周期性创建/删除 fence 节点
            zkManager.createTransaction(zkAcl, fencingNodePath).commit();
            Thread.sleep(zkSessionTimeout);
        }
    }
}

// 安全事务 - 带 fencing
public void safeCreate(String path, ...) {
    SafeTransaction trx = zkManager.createTransaction(zkAcl, fencingNodePath);
    trx.create(path, data, acl, mode);
    trx.commit();
}
```

### 4.3 ZKRMStateStore 分层优化

```java
// 应用ID分层存储 - 避免单节点children过多
// 配置: yarn.resourcemanager.zk-appid-node-split-index
// splitIndex=2 时: application_12 / application_34_0001

private Map<Integer, String> rmAppRootHierarchies;
for (int splitIndex = 1; splitIndex <= 4; splitIndex++) {
    rmAppRootHierarchies.put(splitIndex,
        getNodePath(hierarchiesPath, Integer.toString(splitIndex)));
}
```

---

## 五、可靠性机制对比

### 5.1 原子性保证

| 实现 | 机制 | 依赖 |
|------|------|------|
| **FileSystemRMStateStore** | .tmp/.new 文件 + rename | 文件系统原子性 |
| **LeveldbRMStateStore** | WriteBatch | LevelDB WAL |
| **ZKRMStateStore** | SafeTransaction (Multi) | ZAB 协议 |

### 5.2 故障恢复

| 实现 | 恢复机制 | 不完整记录处理 |
|------|----------|----------------|
| **FileSystemRMStateStore** | 手动清理 .tmp 文件 | checkAndRemovePartialRecord() |
| **LeveldbRMStateStore** | 自动恢复 | LevelDB 自动应用 WAL |
| **ZKRMStateStore** | 自动恢复 | ZK 自动处理 |

### 5.3 重试机制

| 实现 | 重试方式 | 配置参数 |
|------|----------|----------|
| **FileSystemRMStateStore** | FSAction.runWithRetries() | fsNumRetries=15, fsRetryInterval=1000ms |
| **LeveldbRMStateStore** | LevelDB 内置 | 无需配置 |
| **ZKRMStateStore** | Curator 内置重试 | zkSessionTimeout, retryPolicy |

---

## 六、性能对比

### 6.1 IO 操作次数

| 操作 | FileSystemRMStateStore | LeveldbRMStateStore | ZKRMStateStore |
|------|----------------------|---------------------|----------------|
| 存储应用状态 | 2-3 次 (tmp+rename) | 1 次 (put) | 1-2 次 |
| 更新应用状态 | 3 次 (new+delete+rename) | 1 次 (put) | 1 次 |
| 删除应用 | N+1 次 | 1 次 (batch) | 多次 (递归) |
| 存储令牌+序列号 | 2-4 次 | 1 次 (batch) | 1 次 (事务) |

### 6.2 数据组织

| 特性 | FileSystemRMStateStore | LeveldbRMStateStore | ZKRMStateStore |
|------|----------------------|---------------------|----------------|
| 数据索引 | 无 (文件名) | B+Tree | Znode 路径 |
| 范围查询 | 低效 | 高效 | 中等 |
| 压缩 | 无 | Snappy | 无 |
| 缓存 | HDFS DataNode | BlockCache | ZK Client Cache |
| 节点限制 | 无 | 磁盘容量 | ZK 单节点 1MB |

---

## 七、安全性对比

### 7.1 权限控制

| 实现 | 权限机制 | 加密支持 |
|------|----------|----------|
| **FileSystemRMStateStore** | HDFS ACLs, XAttr | HDFS 加密区域 |
| **LeveldbRMStateStore** | 文件系统权限 | 无 |
| **ZKRMStateStore** | ZK ACLs (Digest/SASL) | SSL/TLS |

---

## 八、配置对比

### FileSystemRMStateStore

```xml
<property>
    <name>yarn.resourcemanager.store.class</name>
    <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.FileSystemRMStateStore</value>
</property>
<property>
    <name>yarn.resourcemanager.fs.state-store.uri</name>
    <value>hdfs://namenode:8020/rmstate</value>
</property>
<property>
    <name>yarn.resourcemanager.fs.state-store.num-retries</name>
    <value>15</value>
</property>
<property>
    <name>yarn.resourcemanager.fs.state-store.retry-interval-ms</name>
    <value>1000</value>
</property>
```

### LeveldbRMStateStore

```xml
<property>
    <name>yarn.resourcemanager.store.class</name>
    <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.LeveldbRMStateStore</value>
</property>
<property>
    <name>yarn.resourcemanager.leveldb.state-store.path</name>
    <value>/var/log/yarn/rmstate</value>
</property>
<property>
    <name>yarn.resourcemanager.leveldb.compaction-interval-secs</name>
    <value>3600</value>
</property>
```

### ZKRMStateStore

```xml
<property>
    <name>yarn.resourcemanager.store.class</name>
    <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
</property>
<property>
    <name>yarn.resourcemanager.zk.state-store.parent-path</name>
    <value>/yarn rm</value>
</property>
<property>
    <name>yarn.resourcemanager.zk.timeout-ms</name>
    <value>10000</value>
</property>
<property>
    <name>yarn.resourcemanager.zk.num-retries</name>
    <value>3</value>
</property>
<property>
    <name>yarn.resourcemanager.zk.znode-size-limit-bytes</name>
    <value>10485760</value>
</property>
```

---

## 九、优缺点总结

### FileSystemRMStateStore

| 优点 | 缺点 |
|------|------|
| ✅ 共享存储支持，适合 HA | ❌ IO 开销大 |
| ✅ 生态系统集成 | ❌ 原子性依赖外部文件系统 |
| ✅ 跨平台 (HDFS/S3/本地) | ❌ 令牌和序列号分开存储 |
| ✅ 简单直观 | ❌ 性能一般 |

### LeveldbRMStateStore

| 优点 | 缺点 |
|------|------|
| ✅ 高性能 (嵌入式) | ❌ 单机限制 |
| ✅ WriteBatch 原子性 | ❌ 无法跨机器共享 |
| ✅ 内置索引和压缩 | ❌ 不适合 HA |
| ✅ 故障自动恢复 | ❌ 容量受单机磁盘限制 |

### ZKRMStateStore

| 优点 | 缺点 |
|------|------|
| ✅ 分布式一致性 | ❌ 依赖 ZK 集群 |
| ✅ HA 成熟方案 | ❌ ZK 节点大小限制 (1MB) |
| ✅ Fencing 机制 | ❌ 性能受网络延迟影响 |
| ✅ ACL 权限控制 | ❌ 需要额外维护 ZK |

---

## 十、选择决策树

```
                        场景选择决策树
                                    │
                                    ▼
                    ┌───────────────────────────┐
                    │     是否有 HA 需求？       │
                    └───────────────────────────┘
                         /               \
                       是                 否
                       /                   \
            ┌─────────────────┐    ┌─────────────────┐
            │  是否有共享存储？ │    │  性能要求如何？  │
            └─────────────────┘    └─────────────────┘
              /           \           /           \
            是           否         高             低
            /             \         /              \
┌──────────────────┐  ┌──────────┐┌──────────┐ ┌──────────────┐
│ FileSystemRMState│  │ ZKRM    ││ Leveldb  │ │ MemoryRM     │
│ Store (HDFS)     │  │StateStore││StateStore│ │ StateStore   │
└──────────────────┘  └──────────┘└──────────┘ └──────────────┘
```

### 推荐场景

| 场景 | 推荐方案 | 原因 |
|------|----------|------|
| **生产 HA 集群** | ZKRMStateStore | 成熟 HA 方案，fencing 支持 |
| **HDFS 已有** | FileSystemRMStateStore | 共享存储利用 |
| **开发测试** | LeveldbRMStateStore | 简单高性能 |
| **小规模/边缘** | LeveldbRMStateStore | 无外部依赖 |
| **超大规模集群** | ZKRMStateStore | 分层优化 |

---

## 十一、版本演进对比

| 版本 | FileSystemRMStateStore | LeveldbRMStateStore | ZKRMStateStore |
|------|----------------------|---------------------|----------------|
| 1.0 | 初始版本 | 初始版本 | 初始版本 |
| 1.1 | - | 预留系统 | - |
| 1.2 | AMRMToken 分离 | - | AMRMToken 分离 |
| 1.3 | 预留系统 | - | 预留系统 |
| 1.4 | - | - | 应用节点分层 |
| 1.5 | - | - | 令牌节点分层 |

---

## 十二、总结

### 一句话选择

- **生产 HA**: 首选 **ZKRMStateStore**（成熟稳定，fencing 支持）
- **共享存储环境**: 使用 **FileSystemRMStateStore**（利用现有 HDFS）
- **开发测试/轻量级**: 选择 **LeveldbRMStateStore**（高性能，低依赖）

### 核心差异

| 维度 | FileSystemRMStateStore | LeveldbRMStateStore | ZKRMStateStore |
|------|----------------------|---------------------|----------------|
| **架构** | 主从 (共享存储) | 单机嵌入 | 分布式共识 |
| **一致性** | 最终一致 | 强一致 | 强一致 |
| **HA** | 依赖存储 | 不支持 | 原生支持 |
| **性能** | 中 | 高 | 中 |
| **复杂度** | 低 | 低 | 高 |
| **运维** | 简单 | 简单 | 需维护 ZK |

三种实现各有适用场景，选择时需综合考虑：
1. 是否需要 HA
2. 基础设施条件 (是否有共享存储/ZK 集群)
3. 性能要求
4. 运维复杂度容忍度
