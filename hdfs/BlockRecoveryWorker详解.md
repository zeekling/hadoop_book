# BlockRecoveryWorker 详解

## 1. 核心定位

DataNode 侧处理块恢复的执行器。NameNode 检测到块处于 `UNDER_RECOVERY` 状态后，下发 `BlockRecoveryCommand`，由 BlockRecoveryWorker 协调该块所有副本同步到一致状态。

## 2. 类结构

| 类/接口 | 职责 |
|---------|------|
| `BlockRecoveryWorker` | 主类，接收恢复命令，调度执行 |
| `RecoveryTaskContiguous` | 连续块（副本）恢复 |
| `RecoveryTaskStriped` | 纠删码块恢复 |
| `BlockRecord` | 封装 DataNode 副本信息（id + proxy + ReplicaRecoveryInfo）|

## 3. 核心流程

```
NN 下发 BlockRecoveryCommand
    ↓
BlockRecoveryWorker.recoverBlocks()
    → 创建 Daemon 线程异步执行
    ↓
对每个 RecoveringBlock:
    ├─ isStriped? → RecoveryTaskStriped.recover() [EC块]
    │   ├─ 为每个 internal block 获取副本信息
    │   ├─ 计算 safeLength（所有 data block 的最小长度）
    │   └─ truncatePartialBlock() 截断到 safeLength
    │
    └─ else → RecoveryTaskContiguous.recover() [连续块]
        ├─ 遍历所有 DN，调用 initReplicaRecovery 获取副本
        ├─ 验证：genStamp ≥ block.genStamp && length > 0
        ├─ 筛选状态为 RWR 或更好的副本
        └─ syncBlock() 同步
            ├─ 根据最佳状态计算新长度
            ├─ 对每个有效副本调用 updateReplicaUnderRecovery
            └─ 通知 NN commitBlockSynchronization
```

## 4. 副本状态选择（syncBlock:212-275）

| 最佳状态 | 新长度计算 |
|---------|-----------|
| `FINALIZED` | 取所有 FINALIZED 副本的共同长度；若存在 RBW 且长度=finalizedLength 也加入 |
| `RBW` / `RWR` | 取所有同状态副本的**最小长度** |

## 5. 关键约束

- 恢复命令来自 NN，通过 `BPOfferService.recoverBlocks()` 触发
- 每个恢复任务在独立 Daemon 线程中执行
- 指标 `DataNodeBlockRecoveryWorkerCount` 追踪活跃恢复数
- 纠删码恢复：缺失 block 数 ≤ parity block 数才能恢复，否则删块

## 6. BPOfferService 块恢复调用链

### 完整调用链路

```
NameNode
  │
  │ 1. 检测到块处于 UNDER_RECOVERY 状态
  │ 2. 下发 BlockRecoveryCommand (DNA_RECOVERBLOCK)
  │
  ▼
BPServiceActor.sendHeartbeat() ──→ HeartbeatResponse.getCommands()
  │                                         │
  │                                         ▼
  │                              commandProcessingThread.enqueue(commands)
  │                                         │
  ▼                                         ▼
BPServiceActor.processCommand()  ←── 从队列取出命令
  │
  ▼
BPOfferService.processCommandFromActor()
  │
  ├─ 判断来自 Active NN 还是 Standby NN
  │
  ▼
processCommandFromActive() / processCommandFromStandby()
  │
  ▼
case DNA_RECOVERBLOCK:
  dn.getBlockRecoveryWorker().recoverBlocks(who,
    ((BlockRecoveryCommand)cmd).getRecoveringBlocks())
```

### 关键代码点

**BPOfferService.java:783-787**
```java
case DatanodeProtocol.DNA_RECOVERBLOCK:
  String who = "NameNode at " + nnSocketAddress;
  dn.getBlockRecoveryWorker().recoverBlocks(who,
    ((BlockRecoveryCommand)cmd).getRecoveringBlocks());
  break;
```

### 流程要点

| 阶段 | 组件 | 关键操作 |
|------|------|---------|
| 下发 | NameNode | 块状态为 UNDER_RECOVERY 时构造 BlockRecoveryCommand |
| 心跳 | BPServiceActor | 定时 sendHeartbeat，接收包含命令的 HeartbeatResponse |
| 分发 | commandProcessingThread | 异步队列处理，调用 processCommand() |
| 路由 | BPOfferService | 根据命令类型 DNA_RECOVERBLOCK 路由 |
| 执行 | BlockRecoveryWorker | 协调多 DN 副本同步 |

### 触发条件

NameNode 在以下场景标记块为 `UNDER_RECOVERY`：
1. **客户端 lease 过期** - 写入未完成但客户端失联
2. **写入中断** - 客户端调用 abandonBlock
3. **Pipeline 恢复** - 数据传输失败触发恢复

## 7. NameNode 侧块恢复触发流程

### 完整调用链路

```
1. Lease 过期触发
   └─→ LeaseManager.checkLeases()
       └─→ FSNamesystem.internalReleaseLease()

2. 初始化恢复
   └─→ FSNamesystem.internalReleaseLease():3881-3889
       ├─ blockManager.addBlockRecoveryAttempt(lastBlock)
       ├─ nextGenerationStamp() // 生成新 gen stamp
       └─ uc.initializeBlockRecovery(lastBlock, blockRecoveryId, true)

3. 加入恢复队列
   └─→ BlockUnderConstructionFeature.initializeBlockRecovery():238-290
       ├─ setBlockUCState(UNDER_RECOVERY)
       ├─ 选择主恢复节点（最新活跃的）
       └─ primaryNode.addBlockToBeRecovered(blockInfo) // 加入 DatanodeDescriptor.recoverBlocks 队列

4. 心跳响应下发命令
   └─→ DatanodeManager.handleHeartbeat():1853
       ├─ nodeinfo.getLeaseRecoveryCommand() // 从队列取块
       ├─ getBlockRecoveryCommand() // 构建 BlockRecoveryCommand
       └─ 返回给 DataNode
```

### 关键代码点

| 阶段 | 文件:行号 | 关键操作 |
|------|----------|---------|
| Lease检查 | `LeaseManager.java:614` | `fsnamesystem.internalReleaseLease()` |
| 初始化恢复 | `FSNamesystem.java:3881-3889` | `blockManager.addBlockRecoveryAttempt()` + `initializeBlockRecovery()` |
| 入队 | `BlockUnderConstructionFeature.java:286` | `addBlockToBeRecovered(blockInfo)` |
| 下发命令 | `DatanodeManager.java:1886` | `getBlockRecoveryCommand()` |

### 触发场景

1. **Lease 硬限过期** - 客户端写入超时未完成
2. **截断恢复** - 客户端调用 truncate 后需要同步
3. **写入中断** - Pipeline 失败后的恢复

### 待恢复块存储

- `DatanodeDescriptor.recoverBlocks` (类型: `BlockQueue<BlockInfo>`)
- 每个 DataNode 独立队列，心跳时按需下发

## 8. 端到端完整流程

```
客户端写入中断/Lease过期
    ↓
LeaseManager.checkLeases()
    ↓
FSNamesystem.internalReleaseLease()
    ↓
BlockUnderConstructionFeature.initializeBlockRecovery()
    ├─ 设置块状态为 UNDER_RECOVERY
    ├─ 生成新的 generation stamp
    └─ 将块加入主恢复节点的 recoverBlocks 队列
    ↓
DataNode 心跳
    ↓
DatanodeManager.handleHeartbeat()
    ├─ 从 recoverBlocks 队列取出待恢复块
    ├─ 构建 BlockRecoveryCommand
    └─ 通过 HeartbeatResponse 返回
    ↓
BPServiceActor.processCommand()
    ↓
BPOfferService.processCommandFromActor()
    ↓
BlockRecoveryWorker.recoverBlocks()
    ↓
RecoveryTaskContiguous.recover() / RecoveryTaskStriped.recover()
    ├─ 协调所有副本同步
    └─ 通知 NameNode commitBlockSynchronization
    ↓
块恢复完成，文件可关闭
```

## 9. 关键数据结构

### BlockRecoveryCommand

```java
public class BlockRecoveryCommand extends DatanodeCommand {
  private final Collection<RecoveringBlock> recoveringBlocks;

  public static class RecoveringBlock extends LocatedBlock {
    private final long newGenerationStamp;
    private final Block newBlock; // for truncate recovery
  }

  public static class RecoveringStripedBlock extends RecoveringBlock {
    private final byte[] blockIndices;
    private final ErasureCodingPolicy ecPolicy;
  }
}
```

### BlockRecord

```java
static class BlockRecord {
  private final DatanodeID id;
  private final InterDatanodeProtocol datanode;
  private final ReplicaRecoveryInfo rInfo;
  private String storageID;

  void updateReplicaUnderRecovery(String bpid, long recoveryId,
      long newBlockId, long newLength) throws IOException;
}
```

## 10. 副本状态机

```
TEMPORARY
    ↓ (写入开始)
RUR (Replica Under Recovery)
    ↓ (恢复完成)
RBW (Replica Being Written)
    ↓ (写入完成)
RWR (Replica Waiting Recovery)
    ↓ (恢复完成)
FINALIZED
```

## 11. 相关配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `dfs.heartbeat.interval` | 3s | DataNode 心跳间隔 |
| `dfs.block.recovery.timeout` | 10min | 块恢复超时时间 |
| `dfs.block.size` | 128MB | 默认块大小 |

## 12. 常见问题

### Q1: 为什么需要块恢复？

当客户端写入过程中断（如崩溃、网络故障），块可能处于不一致状态（不同副本长度不同）。块恢复确保所有副本同步到一致状态。

### Q2: 如何选择主恢复节点？

选择最近活跃的 DataNode（`getLastUpdateMonotonic` 最大），该节点负责协调恢复。

### Q3: 纠删码块恢复与普通块恢复的区别？

- **普通块**：选择最佳状态副本，同步长度
- **纠删码块**：计算 safeLength（所有 data block 的最小长度），截断到 stripe 边界

### Q4: 恢复失败会怎样？

如果所有副本都失败，块会被删除。如果部分失败，恢复会重试（通过 `addBlockRecoveryAttempt` 检查）。
