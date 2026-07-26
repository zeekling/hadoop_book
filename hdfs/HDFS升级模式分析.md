# HDFS 升级模式（Upgrade Mode）完整分析

> 基于 Apache Hadoop trunk (3.6.0-SNAPSHOT) 源码分析  
> 核心代码路径：`hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/`

---

## 一、概述

HDFS 支持两种升级方式：

| 类型 | 服务中断 | HA 支持 | 备份机制 |
|------|---------|---------|---------|
| **标准升级** | 需停止整个集群 | 有限支持 | `previous/` 目录完整备份 |
| **滚动升级** | 零停机 | 原生支持 | `fsimage_rollback*` 回滚镜像 + 标记文件 |

升级的本质是**文件系统布局版本（Layout Version）的升级**。每个 HDFS 版本定义了自己的 `layoutVersion`，升级时 NN 和 DN 的存储格式可能发生变化。

---

## 二、核心类架构

### 2.1 类层次关系

```
┌───────────────────────────────────────────────────────┐
│                  ClientProtocol (RPC 接口)              │
│  finalizeUpgrade() / upgradeStatus() / rollingUpgrade() │
└────────────────────────┬──────────────────────────────┘
                         │ 由 NameNodeRpcServer 实现
┌────────────────────────▼──────────────────────────────┐
│                  NameNodeRpcServer                      │
│  finalizeUpgrade() → namesystem.finalizeUpgrade()       │
│  upgradeStatus()   → namesystem.isUpgradeFinalized()    │
│  rollingUpgrade()  → namesystem.*RollingUpgrade()       │
└────────────────────────┬──────────────────────────────┘
                         │ 委派
┌────────────────────────▼──────────────────────────────┐
│  FSNamesystem                                           │
│  finalizeUpgrade()     ← checkSuperuserPrivilege()      │
│  startRollingUpgrade() ← 滚动升级状态机入口              │
│  finalizeRollingUpgrade()                               │
│  queryRollingUpgrade()                                  │
│  getEffectiveLayoutVersion()  ← 滚动升级 LV 锁定        │
└────────┬──────────────────────────────┬────────────────┘
         │ 启动参数                      │ 磁盘操作
┌────────▼──────────┐    ┌─────────────▼──────────────────┐
│  NameNode          │    │  FSImage                        │
│  parseArguments()  │    │  recoverTransitionRead()        │
│  → StartupOption   │    │  doUpgrade() / doRollback()     │
│  ┌ UPGRADE         │    │  finalizeUpgrade()              │
│  └ ROLLBACK        │    └──────────┬─────────────────────┘
└───────────────────┘                │ 委派
                        ┌────────────▼──────────────────┐
                        │  NNUpgradeUtil                 │
                        │  doPreUpgrade() / doUpgrade()   │
                        │  doRollBack() / doFinalize()    │
                        └───────────────────────────────┘
```

### 2.2 DN 端类层次

```
┌──────────────────────────────────────────────┐
│  DataNode                                     │
│  initStorage() → storage.recoverTransitionRead│
│  finalizeUpgradeForPool()                     │
└────────────────────┬─────────────────────────┘
                     │
┌────────────────────▼─────────────────────────┐
│  DataStorage                                  │  ← DN 级存储
│  recoverTransitionRead()                      │
│  doTransition()                  ← 状态机入口  │
│  doUpgradePreFederation() / doUpgrade()       │
│  doRollback() / doFinalize()                  │
└────────────────────┬─────────────────────────┘
                     │
┌────────────────────▼─────────────────────────┐
│  BlockPoolSliceStorage                        │  ← 块池级存储
│  doTransition()                  ← 状态机入口  │
│  doUpgrade() / doRollback() / doFinalize()    │
│  setRollingUpgradeMarkers()                   │
│  clearRollingUpgradeMarkers()                 │
│  ROLLING_UPGRADE_MARKER_FILE                  │  ← "RollingUpgradeInProgress"
└──────────────────────────────────────────────┘
```

---

## 三、存储目录状态机 (StorageState)

### 3.1 目录结构定义

文件：`Storage.java`（`hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/common/Storage.java`）

```java
public static class StorageDirectory {
    final File root;                    // 存储根目录
    final boolean isShared;             // HA 共享目录？
    FileLock lock;                      // in_use.lock 文件锁对象

    // 关键路径方法
    getCurrentDir()        → {root}/current/
    getVersionFile()       → {root}/current/VERSION
    getPreviousDir()       → {root}/previous/
    getPreviousTmp()       → {root}/previous.tmp/
    getRemovedTmp()        → {root}/removed.tmp/
    getFinalizedTmp()      → {root}/finalized.tmp/
    getLastCheckpointTmp() → {root}/lastcheckpoint.tmp/
}
```

### 3.2 StorageState 枚举（Storage.java:112）

```java
public enum StorageState {
    NON_EXISTENT,        // 存储目录不存在
    NOT_FORMATTED,       // 存在但未格式化（无 VERSION 文件）
    NORMAL,              // 正常状态
    COMPLETE_UPGRADE,    // current + previous.tmp 存在
    RECOVER_UPGRADE,     // 仅 previous.tmp 存在（current 丢失）
    COMPLETE_ROLLBACK,   // current + removed.tmp 存在
    RECOVER_ROLLBACK,    // previous + removed.tmp 存在
    COMPLETE_FINALIZE,   // finalized.tmp 存在
    COMPLETE_CHECKPOINT, // current + lastcheckpoint.tmp 存在
    RECOVER_CHECKPOINT   // 仅 lastcheckpoint.tmp 存在（current 丢失）
}
```

### 3.3 analyzeStorage() 决策树（Storage.java:659-781）

**此方法是整个升级状态机的核心**。它在 NameNode/DataNode 启动时被调用，自动检测存储目录状态：

```
analyzeStorage(startOpt, storage, checkCurrentIsEmpty)
│
├─ PROVIDED 存储类型 → NORMAL（直接跳过）
│
├─ root 不存在?
│   ├─ FORMAT/HOTSWAP? → 创建目录并继续
│   └─ 其他? → NON_EXISTENT
│
├─ root 不是目录 / 不可写 → NON_EXISTENT
│
├─ lock() → 获取 in_use.lock（FileChannel.tryLock，非阻塞排他锁）
│   ├─ 写入当前 JVM 名称到锁文件（Windows 不支持读取）
│   └─ 共享目录 (isShared=true) 跳过锁定
│
├─ FORMAT 启动? → NOT_FORMATTED
│
├─ 检查 5 个标记文件/目录的存在性：
│   hasCurrent = versionFile.exists()         // current/VERSION
│   hasPrevious                               // previous/
│   hasPreviousTmp                            // previous.tmp/
│   hasRemovedTmp                             // removed.tmp/
│   hasFinalizedTmp                           // finalized.tmp/
│   hasCheckpointTmp                          // lastcheckpoint.tmp/
│
├─ 无任何临时目录?
│   ├─ hasCurrent? → NORMAL
│   ├─ hasPrevious? → 异常（previous 无 current = 不一致）
│   └─ → NOT_FORMATTED
│
├─ 多于 1 个临时目录 → 异常
│
├─ 恰好 1 个临时目录:
│   ├─ hasCheckpointTmp → hasCurrent? COMPLETE_CHECKPOINT : RECOVER_CHECKPOINT
│   ├─ hasFinalizedTmp  → (hasPrevious 则异常) → COMPLETE_FINALIZE
│   ├─ hasPreviousTmp   → (hasPrevious 则异常) → hasCurrent? COMPLETE_UPGRADE : RECOVER_UPGRADE
│   └─ hasRemovedTmp    → (需 current XOR previous) → COMPLETE_ROLLBACK 或 RECOVER_ROLLBACK
```

### 3.4 doRecover() 配对恢复（StorageDirectory 内部方法）

每个 `StorageState` 都有对应的恢复逻辑：

| StorageState | doRecover 操作 | 说明 |
|-------------|---------------|------|
| `COMPLETE_UPGRADE` | `rename(previous.tmp → previous)` | 完成升级收尾 |
| `RECOVER_UPGRADE` | `delete(current) + rename(previous.tmp → current)` | 升级中断恢复 |
| `COMPLETE_ROLLBACK` | `deleteDir(removed.tmp)` | 完成回滚收尾 |
| `RECOVER_ROLLBACK` | `rename(removed.tmp → current)` | 回滚中断恢复 |
| `COMPLETE_FINALIZE` | `deleteAsync(finalized.tmp)` | 完成定稿收尾 |
| `COMPLETE_CHECKPOINT` | 处理检查点 | 检查点收尾 |
| `RECOVER_CHECKPOINT` | `rename(lastcheckpoint.tmp → current)` | 检查点中断恢复 |

**设计关键**：在任何步骤崩溃，`analyzeStorage` 都能自动检测出当前状态，`doRecover` 负责修复。这就是 HDFS 存储的"崩溃安全"（Crash Safety）设计。

### 3.5 目录锁机制（in_use.lock）

```java
// StorageDirectory.tryLock() — 第 931 行
File lockF = new File(root, "in_use.lock");
RandomAccessFile file = new RandomAccessFile(lockF, "rws");
// 写入当前 JVM 名称用于诊断
file.write(jvmName.getBytes(StandardCharsets.UTF_8));
res = file.getChannel().tryLock();  // 非阻塞，失败即返回
```

- 使用 `FileChannel.tryLock()` 获取**排他文件锁**
- `deleteOnExit()` 确保 JVM 正常退出时清理
- HDFS HA 共享存储目录 (`isShared=true`) 跳过锁定
- NFS 环境可能锁不可靠

---

## 四、NNUpgradeUtil 四个核心方法

文件：`hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/namenode/NNUpgradeUtil.java`

### 4.1 doPreUpgrade() — 预升级（第 116-147 行）

将 current 迁移到 previous.tmp，为新数据腾出空间：

```
步骤:
1. renameCurToTmp(sd):
   ├── 前置条件: current/存在, previous/不存在, previous.tmp/不存在
   ├── NNStorage.rename(current, previous.tmp)     ← 原子重命名
   └── curDir.mkdir()                              ← 创建空 current/

2. 硬链接编辑日志文件:
   ├── Files.walkFileTree(previous.tmp, depth=1)
   ├── 遍历 previous.tmp/ 顶层所有文件
   ├── 过滤: 必须是 edits_* 文件（非 fsimage、VERSION）
   └── Files.createLink(current/edits_X, previous.tmp/edits_X)
                     ↑ 硬链接 API     ↑ 源文件
```

**硬链接设计的精妙之处**：
- `current/edits_*` 和 `previous.tmp/edits_*` 指向同一 inode
- 升级过程中新的 edits 事务追加时，两个路径都可见
- `previous.tmp → previous` 重命名后，`previous/edits_*` 仍有完整事务记录
- 回滚时不丢失任何编辑日志

### 4.2 doUpgrade() — 完成升级（第 183-204 行）

```
步骤:
1. storage.writeProperties(sd)   ← 向 current/ 写新 VERSION 文件
   (含 layoutVersion, clusterID, namespaceID, blockpoolID, cTime)
2. 前置条件: previous/不存在, previous.tmp/存在
3. NNStorage.rename(previous.tmp, previous)  ← 升级完成！
```

**升级后的目录状态**：`current/`（新数据）+ `previous/`（旧数据备份，可用于回滚）

### 4.3 doRollBack() — 回滚（第 213-236 行）

```
步骤:
1. 前置条件: removed.tmp/不存在, current/存在
2. NNStorage.rename(current, removed.tmp)     ← current 暂存（后续删除）
3. NNStorage.rename(previous, current)         ← previous 恢复为 current
4. NNStorage.deleteDir(removed.tmp)            ← 删除新数据
```

**安全设计**：先移走 current 再移入 previous。如果在 2-3 之间崩溃，`removed.tmp` 保留，可手动恢复。

### 4.4 doFinalize() — 定稿（第 88-103 章）

```
步骤:
1. 如果 previous/ 不存在 → return（无需定稿）
2. NNStorage.rename(previous, finalized.tmp)  ← 先移出
3. NNStorage.deleteDir(finalized.tmp)         ← 删除备份
4. isUpgradeFinalized = true                  ← 不可再回滚
```

**定稿不可逆**：定稿后 `previous/` 被彻底删除，不能再回滚。

### 4.5 canRollBack() — 回滚可行性判断（第 55-79 行）

```java
static boolean canRollBack(StorageDirectory sd, StorageInfo storage,
    StorageInfo prevStorage, int targetLayoutVersion) throws IOException {
    // previous/ 不存在 → 不可回滚（但读取当前 VERSION 做一致性校验）
    if (!sd.getPreviousDir().exists()) { return false; }
    // 读取 previous/VERSION
    prevStorage.readPreviousVersionProperties(sd);
    // 校验布局版本是否匹配
    if (prevStorage.getLayoutVersion() != targetLayoutVersion) {
        throw IOException("版本不匹配，需用旧版本 HDFS 回滚");
    }
    return true;
}
```

| 条件 | 结果 |
|------|------|
| `previous/` 不存在 | `return false`，不可回滚 |
| `previous/` 存在，`LV == targetLV` | `return true`，可以回滚 |
| `previous/` 存在，`LV != targetLV` | **抛出 IOException**（不是返回 false） |

---

## 五、NameNode 升级/回滚/定稿完整流程

### 5.1 FSImage.recoverTransitionRead()（FSImage.java:226-342）

NameNode 启动时的核心恢复方法，6 个阶段：

```
recoverTransitionRead(startOpt, target, recovery)
│
├── 阶段1: 目录可用性检查（行 232-239）
│   └── imageDirs 和 editsDirs 不能都为空
│
├── 阶段2: 存储目录状态分析（行 243-256）
│   ├── recoverStorageDirs() → 遍历所有存储目录
│   │   └── sd.analyzeStorage(startOpt, storage) → curState
│   │       ├── NON_EXISTENT → 异常
│   │       ├── NOT_FORMATTED → 跳过（后续格式化）
│   │       ├── NORMAL → 读取 VERSION
│   │       └── 其他 → sd.doRecover(curState) ← 自动恢复
│   └── 读取 VERSION 验证一致性（storage.readProperties）
│
├── 阶段3: 布局版本检查（行 259-285）
│   ├── LV < LAST_PRE_UPGRADE_LAYOUT_VERSION(-3) ?
│   │   ├── checkVersionUpgradable() ← 校验是否可升级
│   │   └── 非 UPGRADE/UPGRADEONLY/ROLLINGUPGRADE 启动？
│   │       └── LV != serviceLV → 强制要求升级！
│
├── 阶段4: 处理升级的 clusterId/blockpoolId（行 287）
│   └── storage.processStartupOptionsForUpgrade()
│       ├── 从预Federation版本升级？→ 生成新 clusterID + blockpoolID
│       └── 从支持Federation版本升级？→ 验证 clusterID 一致性
│
├── 阶段5: 格式化未格式化目录（行 290-322）
│
└── 阶段6: StartupOption 分发（行 325-341）
    ├── UPGRADE / UPGRADEONLY → doUpgrade(target) ← 执行升级
    ├── IMPORT → doImportCheckpoint()
    ├── ROLLBACK → 现在不经过这里（独立命令）
    └── REGULAR → loadFSImage() ← 正常加载
```

### 5.2 FSImage.doUpgrade()（FSImage.java:455-519）

升级操作的两阶段设计，确保原子性：

```
doUpgrade(target)
│
├── 1. checkUpgrade() ← 确保无 previous/ 残留（否则必须先 finalize 或 rollback）
│
├── 2. loadFSImage(target, UPGRADE, null) ← 加载最新的镜像+编辑日志
│
├── 3. 更新版本信息:
│   ├── storage.cTime = now()                    ← 新时间戳
│   └── storage.layoutVersion = serviceLV        ← 当前软件版本 LV
│
├── 4. [第一阶段] 每目录 doPreUpgrade():
│   ├── NNUpgradeUtil.doPreUpgrade(conf, sd)
│   │   ├── current/ → previous.tmp/
│   │   ├── 创建空 current/
│   │   └── 硬链接 edits_* 文件
│   └── [HA] editLog.doPreUpgradeOfSharedLog()
│
├── 5. saveFSImageInAllDirs(target, lastTxId) ← 将所有新数据写入 current/
│
├── 6. [第二阶段] 每目录 doUpgrade():
│   ├── NNUpgradeUtil.doUpgrade(sd, storage)
│   │   ├── storage.writeProperties(sd)           ← 写 VERSION
│   │   └── previous.tmp/ → previous/             ← 完成
│   └── [HA] editLog.doUpgradeOfSharedLog()
│
└── 7. isUpgradeFinalized = false ← 可回滚状态
```

**两阶段设计的意义**：第一阶段移走旧数据、创建新目录，第二阶段写入新数据。如果在阶段 1 完成后崩溃，重启时 `analyzeStorage` 检测到 `COMPLETE_UPGRADE`，`doRecover` 完成 `previous.tmp → previous`。

### 5.3 FSImage.doRollback()（FSImage.java:521-572）

```
doRollback(fsns)
│
├── 1. 先检查可行性（不直接操作磁盘）
│   ├── 每目录 NNUpgradeUtil.canRollBack()
│   └── [HA] editLog.canRollBackSharedLog()
│   → 至少一个目录可回滚，否则 throw IOException
│
├── 2. 执行回滚:
│   ├── 每目录 NNUpgradeUtil.doRollBack()
│   │   ├── current → removed.tmp/
│   │   ├── previous → current/
│   │   └── 删除 removed.tmp/
│   └── [HA] editLog.doRollback()
│
└── 3. isUpgradeFinalized = true
```

**回滚的两次遍历**：先 check 再 execute——要么全部回滚要么不回滚，保证事务性。

### 5.4 FSImage.finalizeUpgrade()（FSImage.java:616-632）

```java
void finalizeUpgrade(boolean finalizeEditLog) {
    for (StorageDirectory sd : storage.dirIterator(false)) {
        NNUpgradeUtil.doFinalize(sd);          // 删除 previous/
    }
    if (finalizeEditLog) {
        editLog.doFinalizeOfSharedLog();        // HA+Active 时处理共享日志
    }
    isUpgradeFinalized = true;
}
```

`finalizeEditLog` 参数仅在 `HA && Active` 时才为 `true`（Standby 不处理共享日志）。

### 5.5 升级过程中目录状态变化全景

```
                    ┌──────────┐
                    │  NORMAL  │ 只有 current/（含 VERSION）
                    └────┬─────┘
                         │
              doPreUpgrade()
              (current → previous.tmp, 创建新 current/)
                         │
               ┌─────────▼──────────┐
               │  previous.tmp/ 存在 │  ← COMPLETE_UPGRADE / RECOVER_UPGRADE
               │  + current/ (空)   │
               └─────────┬──────────┘
                         │
               doUpgrade()
               (写 VERSION, previous.tmp → previous/)
                         │
               ┌─────────▼──────────┐
               │  previous/ 存在     │  ← 可回滚或定稿
               │  + current/ (新)   │
               └─────────┬──────────┘
                       ╱    ╲
                      ╱      ╲
              doFinalize     doRollBack
                    ╲          ╱
                     ╲        ╱
               ┌──────▼──────────┐
               │    NORMAL       │  only current/
               └─────────────────┘
```

---

## 六、参数与命令

### 6.1 NameNode 启动命令

| 命令 | StartupOption | 效果 |
|------|-------------|------|
| `hdfs namenode -upgrade` | `UPGRADE` | 升级并启动 NN（进入 Active） |
| `hdfs namenode -upgradeOnly` | `UPGRADEONLY` | 仅升级文件系统，不启动 NN 服务后即 `terminate(0)` |
| `hdfs namenode -rollingUpgrade started` | `ROLLINGUPGRADE(STARTED)` | 滚动升级启动 |
| `hdfs namenode -rollingUpgrade rollback` | `ROLLINGUPGRADE(ROLLBACK)` | 滚动升级回滚 |
| `hdfs namenode -rollback` | `ROLLBACK` | 回滚标准升级 |
| `hdfs namenode -regular` | `REGULAR` | 正常启动（默认） |

### 6.2 DataNode 启动命令

| 命令 | StartupOption | 效果 |
|------|-------------|------|
| `hdfs datanode -regular` | `REGULAR` | 正常启动（默认） |
| `hdfs datanode -rollback` | `ROLLBACK` | 回滚 |

### 6.3 管理命令（RPC 调用）

| 命令 | 调用 RPC | 权限 |
|------|---------|------|
| `hdfs dfsadmin -finalizeUpgrade` | `ClientProtocol.finalizeUpgrade()` | Superuser |
| `hdfs dfsadmin -upgrade query` | `ClientProtocol.upgradeStatus()` | **无**（任何用户可查） |
| `hdfs dfsadmin -upgrade finalize` | 同 `-finalizeUpgrade` | Superuser |
| `hdfs dfsadmin -rollingUpgrade query` | `ClientProtocol.rollingUpgrade(QUERY)` | Superuser |
| `hdfs dfsadmin -rollingUpgrade prepare` | `ClientProtocol.rollingUpgrade(PREPARE)` | Superuser |
| `hdfs dfsadmin -rollingUpgrade finalize` | `ClientProtocol.rollingUpgrade(FINALIZE)` | Superuser |

### 6.4 超级用户权限判定

```java
// FSPermissionChecker.java 第 286 行
isSuperUser = callerUgi.getShortUserName().equals(fsOwner)     // NN 启动用户
    || callerUgi.getGroups().contains(supergroup);             // 属于 supergroup
```

受 `dfs.permissions.enabled` 控制（默认 true）。若禁用则任何用户可执行升级操作。

---

## 七、滚动升级（RollingUpgrade）

### 7.1 完整状态机

```
                        ┌────────┐
                        │  IDLE  │  rollingUpgradeInfo == null
                        └───┬────┘
                            │ startRollingUpgrade() ← superuser + WRITE
                            v
                  ┌─────────┴──────────┐
                  │ 非 HA 模式          │ HA 模式
                  │ SafeMode 必须开启   │ SafeMode 必须关闭
                  │ saveNamespace       │ startRollingUpgradeInternal()
                  │ (IMAGE_ROLLBACK)    │ logStartRollingUpgrade()
                  │ → SafeMode 关闭     │ rollEditLog() (Standby tail)
                  └─────────┬──────────┘
                            │
                            v
                  ┌─────────┴──────────┐
                  │ ROLLING UPGRADE     │ rollingUpgradeInfo != null
                  │ IN PROGRESS         │ && !isFinalized()
                  │                     │
                  │ LV 锁定到旧版本      │ ★ 关键行为
                  │ 新功能受限           │ ★ requireEffectiveLayoutVersionForFeature()
                  └─────────┬──────────┘
                          ╱ ╲
                         ╱   ╲
                  finalize   rollback
                       ╲     ╱
                  ┌─────┴──┴──────┐
                  │    IDLE        │
                  └───────────────┘
```

### 7.2 FSNamesystem 核心方法

#### 7.2.1 startRollingUpgrade()（第 7656-7686 行）

```java
RollingUpgradeInfo startRollingUpgrade() {
    checkSuperuserPrivilege("startRollingUpgrade");   // 超级用户
    checkOperation(OperationCategory.WRITE);            // 仅 Active NN
    writeLock();

    if (!haEnabled) {
        // 非 HA：要求 SafeMode 开启
        if (!isInSafeMode()) { throw IOException("SafeMode 必须开启"); }
        getFSImage().saveNamespace(this, NameNodeFile.IMAGE_ROLLBACK, null);
        setSafeMode(SafeModeAction.SAFEMODE_LEAVE);     // 自动离开
        setRollingUpgradeInfo(true, startTime);         // createdRollbackImages=true
    } else {
        checkNameNodeSafeMode("Failed");                 // HA：要求 SafeMode 关闭
        setRollingUpgradeInfo(false, startTime);         // createdRollbackImages=false
    }

    getEditLog().logStartRollingUpgrade(startTime);
    if (haEnabled) { getFSImage().rollEditLog(getEffectiveLayoutVersion()); }
}
```

#### 7.2.2 finalizeRollingUpgrade()（第 7847-7878 行）

```java
RollingUpgradeInfo finalizeRollingUpgrade() {
    checkSuperuserPrivilege("finalizeRollingUpgrade");
    checkOperation(OperationCategory.WRITE);
    checkNameNodeSafeMode("Failed");                      // 不能在 SafeMode

    rollingUpgradeInfo.finalize(now());                    // 设置 finalizeTime
    getEditLog().logFinalizeRollingUpgrade(...);
    getFSImage().updateStorageVersion();                   // ★ LV 提升到当前版本
    getFSImage().renameCheckpoint(IMAGE_ROLLBACK → IMAGE); // 回滚镜像 → 普通镜像
}
```

#### 7.2.3 queryRollingUpgrade()（第 7636-7654 行）

```java
RollingUpgradeInfo queryRollingUpgrade() {
    checkSuperuserPrivilege("queryRollingUpgrade");
    checkOperation(OperationCategory.READ);
    // 返回 info（含 startTime, finalizeTime, createdRollbackImages）
    boolean hasRollbackImage = getFSImage().hasRollbackFSImage();
    rollingUpgradeInfo.setCreatedRollbackImages(hasRollbackImage);
    return rollingUpgradeInfo;
}
```

### 7.3 布局版本锁定机制

#### getEffectiveLayoutVersion()（FSNamesystem.java:7791）

滚动升级期间锁定 LV 的关键：

```java
public int getEffectiveLayoutVersion() {
    if (isRollingUpgrade() && storageLV <= minCompatLV) {
        // 滚动升级中且旧 LV 满足最低兼容 → 保持旧 LV
        return storageLV;
    }
    return currentLV;
}
```

**效果**：NN 用旧布局版本写入 fsimage/edits，确保降级到旧 HDFS 时可读。

#### requireEffectiveLayoutVersionForFeature()（FSNamesystem.java:7828）

阻止使用需要新 LV 的功能：

```java
private void requireEffectiveLayoutVersionForFeature(Feature f) {
    if (!NameNodeLayoutVersion.supports(f, getEffectiveLayoutVersion())) {
        throw HadoopIllegalArgumentException(
            "Feature " + f + " unsupported at LV " + getEffectiveLayoutVersion() +
            ".  If rolling upgrade is in progress, finalize first.");
    }
}
```

### 7.4 DN 端标记文件管理（BlockPoolSliceStorage.java）

DN 不靠 NN 显式通知滚动升级终结，而是通过**心跳 + 标记文件**：

```
标记文件: <bpRoot>/RollingUpgradeInProgress
```

| DN 心跳收到状态 | 动作 |
|---------------|------|
| `!isFinalized()` | `enableTrash(bpid)` + `setRollingUpgradeMarker()` → 创建标记文件 |
| `isFinalized()` | `clearTrash(bpid)` + `clearRollingUpgradeMarker()` → 删除标记 + `doFinalize()` |

实现代码（BlockPoolSliceStorage.java）：

**setRollingUpgradeMarkers()**（第 863 行）：
```java
for (StorageDirectory sd : dnStorageDirs) {
    File bpRoot = getBpRoot(blockpoolID, sd.getCurrentDir());
    File markerFile = new File(bpRoot, "RollingUpgradeInProgress");
    if (!markerFile.exists() && markerFile.createNewFile()) { ... }
}
```

**clearRollingUpgradeMarkers()**（第 889 行）：
```java
for (StorageDirectory sd : dnStorageDirs) {
    if (markerFile.exists()) {
        doFinalize(sd.getCurrentDir());   // ★ 终结 layout upgrade
        markerFile.delete();
    }
}
```

关键：`clearRollingUpgradeMarkers()` 不仅删除标记文件，还会调用 `doFinalize()` 清理 `previous/` 目录。

### 7.5 心跳处理链路

```
NN → HeartbeatResponse(rollingUpgradeStatus)
  → BPServiceActor.handleRollingUpgradeStatus()   [行 656]
  → BPOfferService.signalRollingUpgrade()          [行 554]
  → dn.getFSDataset().setRollingUpgradeMarker() / clearRollingUpgradeMarker()
  → BlockPoolSliceStorage.setRollingUpgradeMarkers() / clearRollingUpgradeMarkers()
```

仅在 NN 状态为 **ACTIVE** 时才处理滚动升级状态信息。

---

## 八、DataNode 升级流程

### 8.1 DN 升级启动流程

```
DataNode.initStorage()
  → storage.recoverTransitionRead(this, nsInfo, dataDirs, startOpt)
     [DataStorage.java:564]
```

#### 第一阶段：DataStorage 层级（DN 级）

```java
loadStorageDirectory() {
    sd.analyzeStorage(startOpt, this, true) → curState;
    doTransition(sd, nsInfo, startOpt);        // ★ 核心状态机
    writeProperties(sd);                       // 写 VERSION
}
```

**DataStorage.doTransition()**（第 719-780 行）：

```java
boolean doTransition(sd, nsInfo, startOpt) {
    // 1. 优先处理 ROLLBACK
    if (startOpt == ROLLBACK) { doRollback(sd, nsInfo); }
    // 2. 读取 VERSION
    readProperties(sd);
    checkVersionUpgradable(layoutVersion);
    // 3. 判断是否需要升级
    if (storedLV == currentLV) { return false; }        // 正常，无需升级
    if (storedLV > currentLV) {                          // 需要升级
        if (federationSupported) { upgradeProperties(); } // 仅写 VERSION
        else { doUpgradePreFederation(); }               // 完整升级
        return true;
    }
    throw IOException("stored LV newer than supported");  // BUG
}
```

**doUpgradePreFederation()**（第 804-850 行）：
```
1. 删除 <SD>/previous/       ← 清理旧备份
2. rename(current → previous.tmp)  ← 移走旧数据
3. 格式化 BP 目录
4. linkAllBlocks()            ← 硬链接块文件
5. storage.writeProperties()  ← 写 VERSION
6. rename(previous.tmp → previous) ← 完成
```

#### 第二阶段：BlockPoolSliceStorage 层级（块池级）

```java
loadStorageDirectory() {
    sd.analyzeStorage() → curState;
    doTransition(sd, nsInfo, startOpt);    // ★ BP 级状态机
    writeProperties(sd);
}
```

**BlockPoolSliceStorage.doTransition()**（第 369-425 行）—— 独特逻辑：

| 条件 | 动作 |
|------|------|
| `ROLLBACK + previous 存在` | `doRollback()` 标准回滚 |
| `ROLLBACK + previous 不存在` | `restoreBlockFilesFromTrash()` 从回收站恢复 |
| `LV > currentLV` | 先 `restoreBlockFilesFromTrash()`，再 `doUpgrade()` |
| `cTime > nnCTime` | **抛出异常** — DN 比 NN 还新，不能启动 |

**BP 级 doUpgrade()**（第 445-519 行）：
```
1. 删除 DN 级 previous/   ← 先清父级
2. 删除 BP 级 previous/
3. rename(current → previous.tmp)
4. linkAllBlocks()        ← 硬链接所有块文件
5. writeProperties(LV + cTime)
6. rename(previous.tmp → previous)
```

**BP 级 doRollback()**（第 603-645 行）：
```java
void doRollback(bpSd, nsInfo) {
    File prevDir = bpSd.getPreviousDir();
    // 校验：prevLV >= currentLV && prevCTime <= nnCTime
    // 1. rename current → removed.tmp
    // 2. rename previous → current
    // 3. delete removed.tmp
}
```

### 8.2 DN 升级相关类汇总

| 类 | 文件路径 | 主要职责 |
|------|---------|---------|
| `DataStorage` | `.../datanode/DataStorage.java` | DN 级存储升级/回滚/定稿 |
| `BlockPoolSliceStorage` | `.../datanode/BlockPoolSliceStorage.java` | BP 级存储升级 + 滚动升级标记文件 |
| `DataNode` | `.../datanode/DataNode.java` | 启动参数解析、`finalizeUpgradeForPool()` |
| `FinalizeCommand` | `.../protocol/FinalizeCommand.java` | `DNA_FINALIZE` 命令结构 |
| `BPOfferService` | `.../datanode/BPOfferService.java` | 心跳中协调滚动升级标记 |

---

## 九、安全与权限

### 9.1 各类升级操作权限要求

| 操作 | 权限 | SafeMode 要求 | HA 允许 Standby？ |
|------|------|--------------|-----------------|
| `finalizeUpgrade()` | Superuser | 无限制 | **允许**（`UNCHECKED`） |
| `upgradeStatus()` | **无权限限制** | 无限制 | 允许 |
| `rollingUpgrade(QUERY)` | Superuser | 无限制 | 否（READ，仅 Active） |
| `rollingUpgrade(PREPARE)` | Superuser | HA：必须关闭 / 非HA：必须开启 | 否（WRITE，仅 Active） |
| `rollingUpgrade(FINALIZE)` | Superuser | 不能处于 SafeMode | 否（WRITE，仅 Active） |
| `isUpgradeFinalized()` (NamenodeProtocol) | Superuser | 无限制 | **允许** |

### 9.2 安全注意事项

1. `upgradeStatus()` 无权限检查——任何用户可查升级是否已定稿
2. `dfs.permissions.enabled=false` 时所有升级操作不受限
3. `finalizeUpgrade()` 使用 `OperationCategory.UNCHECKED`，Standby NN 也可执行
4. `DFSAdmin` 在 HA+逻辑 URI 时向所有 NN（含 Standby）发送 `finalizeUpgrade`
5. 滚动升级中 `PREPARE` 和 `FINALIZE` 使用 `WRITE` 类别，仅 Active NN 可执行

---

## 十、布局版本管理

### 10.1 关键版本常量

| 常量 | 值 | 位置 |
|------|-----|------|
| `LAST_PRE_UPGRADE_LAYOUT_VERSION` | `-3` | `Storage.java:88` |
| `CURRENT_LAYOUT_VERSION` | 动态计算 | `NameNodeLayoutVersion.java:36` |
| `MINIMUM_COMPATIBLE_LAYOUT_VERSION` | 动态计算 | `NameNodeLayoutVersion.java:38` |

### 10.2 升级时的 clusterId/blockpoolId 处理

```java
// NNStorage.java 第 918 行
if (startOpt == UPGRADE || startOpt == UPGRADEONLY) {
    if (!NameNodeLayoutVersion.supports(FEDERATION, oldLV)) {
        // 从预Federation版本升级（LV < -45）
        startOpt.setClusterId(newClusterID());
        setBlockPoolID(newBlockPoolID());
    } else {
        // 从支持Federation版本升级，验证 clusterId 一致性
    }
}
```

---

## 十一、核心设计原则总结

### 11.1 崩溃安全（Crash Safety）

通过 `*.tmp` 临时目录实现：

```
升级前: [current]
  → rename current → previous.tmp       ← 原子操作
  → 创建新的 current/                    ← 空目录
  → 写入新数据到 current/                 ← 产生新内容
  → 写 VERSION                           ← 标记新版本
  → rename previous.tmp → previous       ← 原子操作，升级完成
升级后: [current (new)] + [previous (old)]
```

在任何步骤中断，重启后 `analyzeStorage()` 自动检测并恢复。

### 11.2 硬链接（Hardlink）优化

| 场景 | 文件类型 | 目的 |
|------|---------|------|
| NN 升级 | `edits_*` 编辑日志 | 回滚时不丢事务，避免数据复制 |
| DN 升级 | 所有块文件（`rbw/`, `finalized/` 中的） | 避免复制海量块数据 |

### 11.3 标准升级 vs 滚动升级对比

| 对比维度 | 标准升级 | 滚动升级 |
|---------|---------|---------|
| 服务影响 | 全部停止，有停机窗口 | 零停机，逐台滚动 |
| 备份方式 | `previous/` 完整目录 | `fsimage_rollback*` 回滚镜像 |
| 回滚方式 | `-rollback` 重启 NN+DN | `-rollingUpgrade rollback` 逐一回滚 |
| 布局版本切换 | 立即提升 | 锁定旧版，定稿后才提升 |
| HA 支持 | 有限（共享 edits 处理复杂） | 原生支持 |
| 新功能可用性 | 立即可用 | 定稿后才可用 |

### 11.4 DN 回滚校验条件

```java
prevInfo.getLayoutVersion() >= DataNodeLayoutVersion.getCurrentLayoutVersion()
    && prevInfo.getCTime() <= nsInfo.getCTime()
```

- previous 的 LV 不能比当前版本低
- previous 的 cTime 不能比 NN 的 cTime 新

---

## 十二、关键文件索引

| 功能 | 文件路径 | 类 |
|------|---------|-----|
| 存储状态机、StorageDirectory | `hadoop-hdfs/.../server/common/Storage.java` | `Storage` |
| 存储版本信息基类 | `hadoop-hdfs/.../server/common/StorageInfo.java` | `StorageInfo` |
| 启动选项枚举 | `hadoop-hdfs/.../server/common/HdfsServerConstants.java` | `StartupOption` |
| NN 升级工具类 | `hadoop-hdfs/.../server/namenode/NNUpgradeUtil.java` | `NNUpgradeUtil` |
| NN FSImage 升级 | `hadoop-hdfs/.../server/namenode/FSImage.java` | `FSImage` |
| NN 存储管理 | `hadoop-hdfs/.../server/namenode/NNStorage.java` | `NNStorage` |
| 滚动升级状态机 | `hadoop-hdfs/.../server/namenode/FSNamesystem.java` | `FSNamesystem` |
| NameNode 启动入口 | `hadoop-hdfs/.../server/namenode/NameNode.java` | `NameNode` |
| NN RPC 服务 | `hadoop-hdfs/.../server/namenode/NameNodeRpcServer.java` | `NameNodeRpcServer` |
| NN 权限检查器 | `hadoop-hdfs/.../server/namenode/FSPermissionChecker.java` | `FSPermissionChecker` |
| DN 存储升级 | `hadoop-hdfs/.../server/datanode/DataStorage.java` | `DataStorage` |
| DN 块池升级 | `hadoop-hdfs/.../server/datanode/BlockPoolSliceStorage.java` | `BlockPoolSliceStorage` |
| DN 主类 | `hadoop-hdfs/.../server/datanode/DataNode.java` | `DataNode` |
| DN 心跳协调 | `hadoop-hdfs/.../server/datanode/BPOfferService.java` | `BPOfferService` |
| DN Finalize 命令 | `hadoop-hdfs/.../server/protocol/FinalizeCommand.java` | `FinalizeCommand` |
| RPC 协议接口 | `hadoop-hdfs-client/.../protocol/ClientProtocol.java` | `ClientProtocol` |
| 滚动升级 Info | `hadoop-hdfs-client/.../protocol/RollingUpgradeInfo.java` | `RollingUpgradeInfo` |
| 滚动升级状态 | `hadoop-hdfs-client/.../protocol/RollingUpgradeStatus.java` | `RollingUpgradeStatus` |
| HDFS 常量 | `hadoop-hdfs-client/.../protocol/HdfsConstants.java` | `HdfsConstants` |
| 布局版本定义 | `hadoop-hdfs/.../protocol/LayoutVersion.java` | `LayoutVersion` |
| NN 布局版本 | `hadoop-hdfs/.../server/namenode/NameNodeLayoutVersion.java` | `NameNodeLayoutVersion` |
| DN 布局版本 | `hadoop-hdfs/.../server/datanode/DataNodeLayoutVersion.java` | `DataNodeLayoutVersion` |
| 管理命令 CLI | `hadoop-hdfs/.../tools/DFSAdmin.java` | `DFSAdmin` |

---

## 十三、重要测试文件

| 测试文件 | 路径 | 测试内容 |
|---------|------|---------|
| `TestDFSUpgrade.java` | `hadoop-hdfs/.../hdfs/TestDFSUpgrade.java` | 标准升级完整流程 |
| `TestDFSUpgradeFromImage.java` | `hadoop-hdfs/.../hdfs/TestDFSUpgradeFromImage.java` | 从旧镜像升级 |
| `TestDFSRollback.java` | `hadoop-hdfs/.../hdfs/TestDFSRollback.java` | 回滚测试 |
| `TestDFSFinalize.java` | `hadoop-hdfs/.../hdfs/TestDFSFinalize.java` | 定稿测试 |
| `TestRollingUpgradeRollback.java` | `hadoop-hdfs/.../hdfs/TestRollingUpgradeRollback.java` | 滚动升级回滚 |
| `TestDFSUpgradeWithHA.java` | `hadoop-hdfs/.../namenode/ha/TestDFSUpgradeWithHA.java` | HA 升级测试 |

---

## 十四、滚动升级期间文件写入行为差异

滚动升级的核心挑战是：**集群中同时运行新旧两个版本的 HDFS 进程**。新版本的软件可能使用新的布局版本（`layoutVersion`），但旧版本无法识别新格式。因此，滚动升级期间必须确保所有持久化数据（fsimage、edit log、DataNode 块文件）都以**所有节点都能识别的旧格式**写入。

本章详细分析开启滚动升级后，文件写入路径与正常模式的具体差异。

### 14.1 有效布局版本锁定机制

#### 14.1.1 `getEffectiveLayoutVersion()`

**文件**：`FSNamesystem.java:7791-7812`

滚动升级期间锁定布局版本的核心方法：

```java
public int getEffectiveLayoutVersion() {
    return getEffectiveLayoutVersion(isRollingUpgrade(),
        fsImage.getStorage().getLayoutVersion(),
        NameNodeLayoutVersion.MINIMUM_COMPATIBLE_LAYOUT_VERSION,
        NameNodeLayoutVersion.CURRENT_LAYOUT_VERSION);
}

static int getEffectiveLayoutVersion(boolean isRollingUpgrade, int storageLV,
    int minCompatLV, int currentLV) {
    if (isRollingUpgrade) {
        if (storageLV <= minCompatLV) {
            // 存储的旧布局版本满足最低兼容性 → 锁定旧 LV
            return storageLV;
        }
    }
    // 正常模式：使用当前软件版本的 LV
    return currentLV;
}
```

**锁定条件**：`storageLV <= minCompatLV`（LV 是负数，所以这是"版本号 ≥ minCompat"的比较）

**场景示例**：

| 场景 | storageLV | minCompatLV | 是否锁定 | 有效 LV |
|------|-----------|-------------|---------|---------|
| 正常模式（LV=-65，minCompat=-60） | -65 | -60 | 否 | -65 |
| 滚动升级且旧 LV 在兼容范围内（旧=-55，minCompat=-60） | -55 | -60 | 是 ( -55 ≤ -60 ) | -55（锁定！） |
| 滚动升级但旧 LV 不兼容（旧=-70，minCompat=-60） | -70 | -60 | 否 ( -70 > -60 ) | -65（不锁定） |

> **注意**：LV 是负数，所以"更小"=版本号更新。`≤` 意味着新/相同版本满足兼容要求。

#### 14.1.2 被限制的新功能

**文件**：`FSNamesystem.java:7828-7837`

```java
private void requireEffectiveLayoutVersionForFeature(Feature f)
    throws HadoopIllegalArgumentException {
    int lv = getEffectiveLayoutVersion();
    if (!NameNodeLayoutVersion.supports(f, lv)) {
        throw new HadoopIllegalArgumentException(String.format(
            "Feature %s unsupported at NameNode layout version %d. ...",
            f, lv));
    }
}
```

**所有调用点及被限制的功能**：

| 调用点（文件:行号） | 触发条件 | 被阻塞的功能 |
|---------------------|----------|-------------|
| `FSDirTruncateOp.java:42` | `Truncate` 操作 | `TRUNCATE` |
| `FSDirAppendOp.java:266` | Append 写新 Block | `APPEND_NEW_BLOCK` |
| `NvdimmMain.java:150` | NVDIMM 操作 | `NVDIMM_SUPPORT` |
| `FSDirSnapshotOp.java:133` | 快照 Diff 操作 | `SNAPSHOT`（检查 `FSIMAGE_NAME_OPTIMIZATION`） |
| `FSDirectory.java:1037` | 获取存储类型配额 | `QUOTA_BY_STORAGE_TYPE` |
| `FSDirectory.java:1136` | 设置存储类型配额 | `QUOTA_BY_STORAGE_TYPE` |

**典型例子**——滚动升级期间 Truncate 操作被拒绝：
```
org.apache.hadoop.util.HadoopIllegalArgumentException:
Feature TRUNCATE unsupported at NameNode layout version -55.
If a rolling upgrade is in progress, then it must be finalized before using this feature.
```

### 14.2 编辑日志（Edit Log）序列化差异

#### 14.2.1 编辑日志段文件头写入锁定后的旧 LV

**文件**：`EditLogFileOutputStream.java`

**`create()` 阶段**（定长 header + 变长 extendedHeader）：
```java
// 正常启动时：写入 CURRENT_LAYOUT_VERSION（最新）
// 滚动升级启动时：写入 getEffectiveLayoutVersion() 锁定的旧 LV
```

**`writeHeader()`** 写入的 `logVersion` 被锁定到旧 LV。这个版本号对所有后续 FSEditLogOp 序列化分支有决定性影响。

#### 14.2.2 FSEditLogOp 序列化中的条件分支

**文件**：`FSEditLogOp.java`（约 3000 行）

编辑日志操作（op）的序列化/反序列化中穿插了大量 `NameNodeLayoutVersion.supports(feature, logVersion)` 条件判断。当 `logVersion` 是旧版本时，新功能的字段**不被写入**，读取时也不被期望。

**核心模式**：

```java
// 序列化（write）时：如果 logVersion 不支持该 feature，跳过写
if (NameNodeLayoutVersion.supports(Feature.ERASURE_CODING, logVersion)) {
    out.writeInt(numECs);
    // ...写 EC 字段
}

// 反序列化（readFields）时：如果 logVersion 不支持，使用默认值
if (NameNodeLayoutVersion.supports(Feature.ERASURE_CODING, logVersion)) {
    ecPolicy = ECSchema.fromInt(out.readInt());
} else {
    ecPolicy = null; // 默认值
}
```

**按 op 类型分类的条件分支**：

| EditLogOp 类型 | Feature 条件 | 受影响字段 |
|----------------|-------------|-----------|
| **OP_ADD / OP_ADD_BLOCK / OP_CLOSE** | `ERASURE_CODING` | EC 策略、ECSchema |
| | `STORAGE_GROUPS` | 存储组信息 |
| | `SNAPSHOT` | 快照 ID |
| | `BLOCKS_WITH_XATTRS` | Block xattrs |
| **OP_APPEND** | `APPEND_NEW_BLOCK` | AppendBlockInfo |
| **OP_RENAME** | `RENAME_BATCH_1` / `RENAME_BATCH_2` | 时间戳、可选字段 |
| **OP_MKDIR** | `OPTIONAL_CREATE_PARENT` | 是否创建父目录 |
| **OP_SET_OWNER** | 无 | 无条件 |
| **OP_SET_PERMISSION** | 无 | 无条件 |
| **OP_SET_REPLICATION** | 无 | 无条件 |
| **OP_CONCAT_DELETE** | `CONCAT` | 目标 block 列表 |
| **OP_TRUNCATE** | `TRUNCATE` | Truncate 长度/时间（实际上在调用层被 `requireEffectiveLayoutVersionForFeature` 阻塞） |
| **OP_ROLLING_UPGRADE_START / FINALIZE** | 无（升级相关 op 无条件） | 时间戳 |

**ERASURE_CODING 字段写入的例子**（OP_ADD）：
```java
// 正常模式 (LV=-65): supports(ERASURE_CODING, -65) = true → 写入 EC 字段
// 滚动升级模式 (LV=-55): supports(ERASURE_CODING, -55) = false → 跳过 EC 字段
```

**效果**：滚动升级期间写入的 edit log 完全向前兼容——旧版本的 NN（无论是否降级）都能正确回放这些编辑日志。

#### 14.2.3 文件追加（Append）的 op 选择差异

**文件**：`FSDirAppendOp.java:260-275`

```java
private OpInstanceCache.OpInstance getOp(...) {
    return OpInstanceCache.get(
        // 根据有效 LV 选择 op 类型
        NameNodeLayoutVersion.supports(Feature.APPEND_NEW_BLOCK,
            getEffectiveLayoutVersion())
        ? OpType.LOG_APPEND_FILE    // 新版本：logAppendFile
        : OpType.LOG_OPEN_FILE);    // 旧版本：logOpenFile
}
```

| 模式 | 有效 LV 支持 APPEND_NEW_BLOCK? | 使用的 op | 含义 |
|------|-------------------------------|-----------|------|
| 正常 | 是（LV=-65 支持） | `logAppendFile` | 追加文件作为独立 op |
| 滚动升级 | 否（旧 LV=-55 不支持） | `logOpenFile` | 退化为"打开文件"op，保持旧格式兼容 |

**影响**：滚动升级期间追加文件使用 `logOpenFile`（写完整的新 block 信息）而非 `logAppendFile`（只写追加的 block）。旧版本 NN 无法识别 `logAppendFile`，因此必须使用 `logOpenFile`。

### 14.3 FSImage 序列化差异

#### 14.3.1 FileSummary 中的 layoutVersion

**文件**：`FSImageFormatProtobuf.java`

```java
// saveInternal 中使用 getEffectiveLayoutVersion()
builder.setLayoutVersion(getEffectiveLayoutVersion());
FileSummary summary = builder.build();
```

| 模式 | summary.layoutVersion | 含义 |
|------|----------------------|------|
| 正常 | `CURRENT_LAYOUT_VERSION` (如 -65) | 所有 section 被保存 |
| 滚动升级 | 锁定后的旧 LV (如 -55) | 新功能 section 被跳过 |

#### 14.3.2 Section 级别的条件保存

`saveInternal()` 遍历所有 `Section` 保存时，会检查：

```java
// 使用 summary 中的 layoutVersion 判断
if (NameNodeLayoutVersion.supports(Feature.ERASURE_CODING, summary.getLayoutVersion())) {
    saveErasureCodingSection(...);
}
if (NameNodeLayoutVersion.supports(Feature.SNAPSHOT, summary.getLayoutVersion())) {
    saveSnapshotSection(...);
}
```

**被跳过的 section**（当 LV 被锁定到旧版本时）：

| Section | Feature 条件 |
|---------|-------------|
| Erasure Coding | `ERASURE_CODING` |
| SBN SPS | `ROLLING_UPGRADE_SBN_SPS` |
| 其他新功能 section | 对应 feature |

#### 14.3.3 FSImage.save 跳过 updateStorageVersion

**文件**：`FSImage.java`

```java
void save(FSNamesystem source, NameNodeFile nnf, ...) throws IOException {
    // ... 正常保存流程 ...
    if (!isRollingUpgrade()) {
        updateStorageVersion();     // ← 滚动升级时跳过！
    }
}
```

**`updateStorageVersion()`** 将存储中的 layoutVersion 更新到当前软件版本：
```java
void updateStorageVersion() {
    // 写入所有存储目录的 VERSION 文件，将 layoutVersion 设为 CURRENT_LAYOUT_VERSION
    storage.setLayoutVersion(NameNodeLayoutVersion.CURRENT_LAYOUT_VERSION);
    // 写所有 sd 的 VERSION
}
```

**为什么滚动升级时跳过**：如果 `save()` 后升级了存储 LV，一旦降级回旧版本，旧 NN 读取 VERSION 文件发现 LV 比它支持的新，会拒绝启动。必须等到 `finalizeRollingUpgrade()` 显式调用 `updateStorageVersion()` 时才更新。

**时间线**：
```
滚动升级 PREPARE
    → 创建回滚镜像 (saveNamespace with IMAGE_ROLLBACK)
        → 使用有效 LV = 旧 LV
        → 跳过 updateStorageVersion()
    → 滚动重启 NN，旧格式正常运行
滚动升级 FINALIZE
    → updateStorageVersion()    ← 此时才正式升级存储 LV
    → renameCheckpoint(IMAGE_ROLLBACK → IMAGE)
    → 后续所有镜像/edits 都使用最新 LV
```

### 14.4 DataNode 块文件操作差异

#### 14.4.1 块删除走 Trash 路径

**文件**：`BPOfferService.java:554-567`，`FsDatasetImpl.java`

**正常模式**：删除块时，块文件被立即删除，释放磁盘空间。

**滚动升级模式**：NN 心跳中的 `rollingUpgradeStatus`（`!isFinalized()`）触发：

```java
void signalRollingUpgrade(RollingUpgradeStatus status) throws IOException {
    if (status == null) return;
    String bpid = getBlockPoolId();
    if (!status.isFinalized()) {
        dn.getFSDataset().enableTrash(bpid);      // 启动 Trash
        dn.getFSDataset().setRollingUpgradeMarker(bpid);  // 创建标记文件
    } else {
        dn.getFSDataset().clearTrash(bpid);       // 清理 Trash
        dn.getFSDataset().clearRollingUpgradeMarker(bpid);  // 删除标记 + 终结
    }
}
```

**Trash 机制实现**（`FsDatasetImpl` 的 block 删除方法）：

```java
if (trashEnabled) {
    // 块文件移动到 trash 子目录，不是直接删除
    // 记录到 TrashMonitor 的待删除队列
    trash(blockFile);
    trash(metaFile);
} else {
    // 正常模式直接删除
    blockFile.delete();
    metaFile.delete();
}
```

**设计意图**：滚动升级期间，旧版本 DN 可能尚未感知升级完成，新 DN 写入的 block 可能在旧 DN 上被认为是"多余"而被删除。Trash 路径确保即使误删，也可以从 trash 恢复。

#### 14.4.2 标记文件驱动的生命周期

```
滚动升级 PREPARE
    → NN 心跳返回 rollingUpgradeStatus(!isFinalized)
    → BPServiceActor.handleRollingUpgradeStatus()
    → BPOfferService.signalRollingUpgrade()
        → enableTrash(bpid)               ← 开启块删除 Trash
        → setRollingUpgradeMarkers()       ← 创建 <bproot>/RollingUpgradeInProgress

滚动升级进行中
    → 块删除 → Trash 目录
    → 所有 DN 保持标记文件存在

滚动升级 FINALIZE
    → NN 心跳返回 rollingUpgradeStatus(isFinalized=true), 或 null
    → BPOfferService.signalRollingUpgrade()
        → clearTrash(bpid)                 ← 清理 Trash
        → clearRollingUpgradeMarkers()     ← 删除标记 + doFinalize()

滚动升级回滚
    → BlockPoolSliceStorage.doTransition(ROLLBACK + 无 previous)
    → restoreBlockFilesFromTrash()         ← 从 Trash 恢复块文件
```

#### 14.4.3 从 trash 恢复块文件

**文件**：`BlockPoolSliceStorage.java:550-589`

```java
private int restoreBlockFilesFromTrash(File trashRoot) throws IOException {
    // 递归遍历 trash 目录
    int filesRestored = 0;
    // 将文件重命名回 current 对应位置
    // 删除 trash 目录
    FileUtil.fullyDelete(trashRoot);
    return filesRestored;
}
```

滚动升级回滚时，`BlockPoolSliceStorage.doTransition()` 检测到 `ROLLBACK + no previous` 时执行此方法：

```java
if (startOpt == StartupOption.ROLLBACK && !sd.getPreviousDir().exists()) {
    // 滚动升级回滚的特殊路径：从 trash 恢复
    int restored = restoreBlockFilesFromTrash(getTrashRootDir(sd));
    LOG.info("Restored {} block files from trash.", restored);
}
```

### 14.5 完整文件写入链路的对比总结

#### 14.5.1 创建新文件路径

```
                      正常模式                           滚动升级模式
                      ────────                          ────────
Client 请求           dfs.create()                      dfs.create()

FSNamesystem          startFile()                       startFile()
                      (no effective LV check)           (no effective LV check)

FSDirWriteFileOp     ADD_FILE op 写入 edit log         ADD_FILE op 写入 edit log
                       LV = CURRENT_LAYOUT_VERSION       LV = 锁定后的旧 LV
                     所有字段完整写入                   新功能字段跳过(如 EC 字段)

FSImage.save         save():                              save():
                      layoutVersion = CURRENT_LV          layoutVersion = 旧 LV
                     所有 section 保存                   新 section 跳过
                     updateStorageVersion()              跳过 updateStorageVersion()

EditLog roll         新段写入 HEADER                     新段写入 HEADER
                       logVersion = CURRENT_LV             logVersion = 旧 LV

DN 写 block          normal write                        normal write (block 格式不变)
                     delete = 立即删除                   delete = 移动到 trash
```

#### 14.5.2 追加(Append)文件路径

```
                      正常模式                           滚动升级模式
                      ────────                          ────────
Client 请求           dfs.append()                      dfs.append()

FSDirAppendOp         requireEffectiveLayoutVersionForFeature
                        APPEND_NEW_BLOCK                   ← 通过 ↓
                      使用 logAppendFile op               使用 logOpenFile op
                      （追加专用 op）                     （退化为打开文件 op

EditLog 序列化        OP_APPEND 写入                     OP_ADD/OP_CLOSE_ADD
                      APPEND_NEW_BLOCK 字段写入           旧格式字段写入

DN 写 block          正常追加写入                        正常追加写入
```

#### 14.5.3 Truncate 文件路径

```
                      正常模式                           滚动升级模式
                      ────────                          ────────
Client 请求           dfs.truncate()                    dfs.truncate()

FSDirTruncateOp       requireEffectiveLayoutVersionForFeature
                        TRUNCATE                          ← 抛出异常!
                      执行 truncate                      操作被拒绝
                      OP_TRUNCATE 写入                    (不执行)
```

#### 14.5.4 升级定稿后的转变

```java
// finalizeRollingUpgrade() 执行后：
FSNamesystem.finalizeRollingUpgrade()
    → finalizeRollingUpgradeInternal(now())
        → rollingUpgradeInfo.finalize(finalizeTime)  // isRollingUpgrade() 返回 false
    → getFSImage().updateStorageVersion()             // 正式升级存储 LV
    → getFSImage().renameCheckpoint(IMAGE_ROLLBACK → IMAGE) // 回滚镜像转正
```

定稿后：
- `getEffectiveLayoutVersion()` 返回 `CURRENT_LAYOUT_VERSION`
- `isRollingUpgrade()` 返回 `false`
- 后续 edit log 和 fsimage 使用最新 LV 格式
- DN 清理所有标记文件，关闭 Trash 路径
- 所有被限制的功能（Truncate、Append New Block、NVDIMM、QUOTA_BY_STORAGE_TYPE）恢复正常

### 14.6 关键代码文件索引

| 文件 | 行号 | 内容 |
|------|------|------|
| `FSNamesystem.java` | 7791-7812 | `getEffectiveLayoutVersion()` — 有效 LV 锁定 |
| `FSNamesystem.java` | 7828-7837 | `requireEffectiveLayoutVersionForFeature()` — 功能限制 |
| `FSDirTruncateOp.java` | 42 | TRUNCATE 功能阻塞 |
| `FSDirAppendOp.java` | 260-275 | Append 的 op 选择（logOpenFile vs logAppendFile） |
| `FSDirAppendOp.java` | 266 | APPEND_NEW_BLOCK 功能阻塞 |
| `FSImage.java` | 保存路径 | save() 中跳过 updateStorageVersion 的判断 |
| `FSImageFormatProtobuf.java` | saveInternal | FileSummary.layoutVersion 设为有效 LV |
| `FSEditLogOp.java` | 各 op write/readFields | Conditional 序列化基于 logVersion |
| `EditLogFileOutputStream.java` | writeHeader | 写入锁定后的 logVersion |
| `BPOfferService.java` | 554-567 | `signalRollingUpgrade()` — Trash + 标记文件 |
| `BlockPoolSliceStorage.java` | 82 | `ROLLING_UPGRADE_MARKER_FILE` 常量 |
| `BlockPoolSliceStorage.java` | 550-589 | `restoreBlockFilesFromTrash()` — Trash 恢复 |
| `BlockPoolSliceStorage.java` | 863-881 | `setRollingUpgradeMarkers()` — 创建标记 |
| `BlockPoolSliceStorage.java` | 889-909 | `clearRollingUpgradeMarkers()` — 清理标记 + doFinalize |
| `NvdimmMain.java` | 150 | NVDIMM 功能阻塞 |
| `FSDirSnapshotOp.java` | 133 | SNAPSHOT 功能阻塞（FSIMAGE_NAME_OPTIMIZATION） |
| `FSDirectory.java` | 1037, 1136 | QUOTA_BY_STORAGE_TYPE 功能阻塞 |
