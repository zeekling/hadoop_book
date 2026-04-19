# HDFS DataNode 3.3.1 后优化详解

本文系统梳理 Hadoop 3.3.1 之后（3.4.0 ~ 3.5.0）对 DataNode 的全部优化，涵盖性能优化、稳定性修复、新特性、监控增强和运维改进五个维度。

## 简介

DataNode 是 HDFS 数据存储的核心组件，负责数据块的读写、校验、复制和汇报。3.3.1 之后社区对 DataNode 做了大量改进，其中最核心的是**细粒度锁机制**——将单一 FsDatasetImpl 全局锁拆分为 BlockPool → Volume → DIR 三级锁，使 DN 在高并发场景下吞吐量提升 30-50%，3.5.0 中进一步演进到基于 blockId 的更细粒度锁。此外，慢节点/慢磁盘检测、动态重配置、DirectoryScanner 优化等改进也显著提升了 DN 的可观测性和运维效率。

---

## 一、性能优化

### 1.1 细粒度锁机制（核心优化）

这是 3.3.1 之后 DN 最重要的性能优化，分三个阶段递进实现：

| 阶段 | Issue | 版本 | 锁粒度 | 说明 |
|------|-------|------|--------|------|
| Phase I | HDFS-15382 | 3.4.0 | Volume 级 | 将 FsDatasetImpl 全局锁拆为 BlockPool + Volume 两级锁 |
| Phase I 补全 | HDFS-16534 | 3.4.0 | Volume 级 | 拆分 block pool 锁到 volume 粒度（HDFS-15382 子任务） |
| Phase II | HDFS-17496 | 3.5.0 | DIR 级（基于 blockId） | 将 Volume 锁进一步拆到 `subdir[0-31]/subdir[0-31]` 目录级 |

**HDFS-15382**：拆分 FsDatasetImpl 全局锁到 Volume 级别。在 HDFS-15180（锁拆到 BlockPool 粒度）基础上，进一步将锁拆到 Volume 粒度。当一个 Volume 处于高负载时，不会阻塞同一 BlockPool 下其他 Volume 的请求，显著提升并发性能。

```java
// 3.3.1：全局单一读写锁
try (AutoCloseableLock lock = fsDatasetLock.writeLock()) {
    // 所有操作共享同一把锁
}

// 3.4.0：两级锁体系（BlockPool + Volume）
try (AutoCloseDataSetLock lock = lockManager.writeLock(
    LockLevel.BLOCK_POOL, bpid)) {
    // BlockPool 级操作
}
try (AutoCloseDataSetLock lock = lockManager.writeLock(
    LockLevel.VOLUME, volumeId)) {
    // Volume 级操作
}

// 3.5.0：三级锁体系（BlockPool + Volume + DIR）
try (AutoCloseDataSetLock lock = lockManager.writeLock(
    LockLevel.DIR, bpid, blockId)) {
    // 基于 blockId 的目录级锁
}
```

**HDFS-17496**：在 3.5.0 中引入基于 blockId 的目录级锁。在 NvmeSSD 高并发小文件场景测试中，发现 Volume 级锁仍成为瓶颈。通过 `DatanodeUtil#idToBlockDir` 将 blockId 映射到 `finalized/subdir[d1]/subdir[d2]`（共 1024 个目录），实现更细粒度的锁分区：

```java
public static File idToBlockDir(File root, long blockId) {
    int d1 = (int) ((blockId >> 16) & 0x1F);
    int d2 = (int) ((blockId >> 8) & 0x1F);
    return new File(root,
        BLOCK_SUBDIR_PREFIX + d1 + SEP + BLOCK_SUBDIR_PREFIX + d2);
}
```

性能测试（3 DN、10 客户端并发、每客户端 550 线程写入+删除）显示 DIR 级锁在 NvmeSSD 场景下相比 Volume 级锁吞吐量进一步提升。

**锁优化相关配套修复**：

| Issue | 说明 |
|-------|------|
| HDFS-16600 | 修复 HDFS-16534 引入的细粒度锁死锁问题（createRbw 读锁与 evictBlocks 写锁冲突） |
| HDFS-16598 | 修复细粒度锁下 GS（GenerationStamp）校验缺失导致 pipeline recovery 失败 |
| HDFS-16785 | addVolume 操作不应持有 BP 写锁扫描新 Volume，避免长时间阻塞其他操作 |
| HDFS-16783 | 移除 deepCopyReplica 和 getFinalizedBlocks 中的冗余锁 |
| HDFS-16787 | 移除 DataSetLockManager#removeLock 中的冗余锁 |
| HDFS-16817 | 移除无效的 DN 锁相关配置（`dfs.datanode.lock.read.write.enabled` 等） |
| HDFS-16898 | 移除 processCommandFromActor 中的写锁，避免心跳延迟甚至 DN 被标记为 dead |
| HDFS-15160 | ReplicaMap、DiskBalancer、DirectoryScanner 等方法应使用读锁替代写锁 |
| HDFS-16090 | datanodeNetworkCounts 使用细粒度锁替代整个 LoadingCache 锁 |
| HDFS-17683 | 3.5.0 新增 dataset 读写锁获取的监控指标 |

### 1.2 DirectoryScanner 优化

| Issue | 版本 | 优化内容 |
|-------|------|----------|
| HDFS-15574 | 3.4.0 | 移除不必要的块列表排序（底层已用 FoldedTreeSet 有序存储） |
| HDFS-15415 | 3.4.0 | 减少 DirectoryScanner 中的锁持有时间 |
| HDFS-15934 | 3.4.0 | reconcile 块的批处理大小和批次间隔可配置 |
| HDFS-15621 | 3.4.0 | 优化 DirectoryScanner 内存使用（大 DN 场景内存消耗过高） |
| HDFS-16316 | 3.4.0 | 改进 DirectoryScanner：增加常规文件检查关联块 |
| HDFS-17346 | 3.5.0 | 修复 DirectoryScanner 错误地将正常块标记为损坏 |
| HDFS-17068 | 3.4.0 | DN 应记录上次目录扫描时间 |

```java
// HDFS-15574：移除不必要的排序
// 修改前：块存储在 FoldedTreeSet 中已经有序
final List<ReplicaInfo> bl = dataset.getFinalizedBlocks(bpid);
Collections.sort(bl); // 冗余排序，已移除

// 修改后：直接获取有序结果
final List<ReplicaInfo> bl = dataset.getSortedFinalizedBlocks(bpid);
```

### 1.3 存储与 IO 优化

| Issue | 版本 | 优化内容 |
|-------|------|----------|
| HDFS-15549 | 3.4.0 | DISK/ARCHIVE 间同文件系统挂载使用硬链接替代复制 |
| HDFS-15548 | 3.4.0 | 允许同一设备挂载上配置 DISK/ARCHIVE 存储类型 |
| HDFS-15683 | 3.4.0 | 支持为每个 Volume 单独配置 DISK/ARCHIVE 容量 |
| HDFS-17063 | 3.4.0 | 支持为每块磁盘配置不同的预留容量 |
| HDFS-15569 | 3.4.0 | 加速 DN 滚动升级中 Storage#doRecover |
| HDFS-15610 | 3.4.0 | 减少升级/硬链接线程数（默认 12→更优线程数，减少内核开销） |
| HDFS-15937 | 3.4.0 | 减少 DN 布局升级期间的内存使用（优化后 200MB/百万块 vs 之前 1.5GB/百万块） |
| HDFS-16386 | 3.4.0 | 减少 FsDatasetAsyncDiskService 工作时 DN 负载（限制总线程数） |
| HDFS-16774 | 3.4.0 | 改进异步删除副本：从 ReplicaMap 删除、磁盘删除、IBR 通知在同一线程中完成 |
| HDFS-15406 | 3.4.0 | 提升 DN 块扫描速度 |
| HDFS-15618 | 3.4.0 | 改善 DN 关闭延迟 |
| HDFS-17087 | 3.4.0 | 新增 DN 读块限流器（Throttler） |

**HDFS-15549**：当 DISK 和 ARCHIVE 存储类型在同一文件系统挂载上时，使用 `rename` 替代 `copy` 来移动副本，节省大量 IO 开销。

**HDFS-15937**：DN 布局升级（-56 到 -57）时，将 LinkArgs 对象拆分为共享路径 + 块名，并缓存重复路径对象，将内存使用从 1.5GB/百万块降至 200MB/百万块。

```java
// HDFS-15937：路径对象复用
// 修改前：每个块存储完整路径字符串（大量重复前缀）
LinkArgs(src="/data/dn/.../subdir0/subdir0/blk_123", dest="/data/dn/.../subdir0/subdir10/blk_123")

// 修改后：路径前缀复用
LinkArgs(srcDir=cache.get("/data/dn/.../subdir0/subdir0"),
         destDir=cache.get("/data/dn/.../subdir0/subdir10"),
         blockFile="blk_123")
```

**HDFS-16774**：优化异步删除流程。原流程中从 ReplicaMap 移除后异步删磁盘和通知 NN，可能导致客户端读到已删除副本（ReplicaNotFoundException）。优化后将三步操作合并到同一线程执行：

```java
// HDFS-16774：优化前（三步分离）
// 1. 主线程：removeFromReplicaMap(replica)
// 2. 线程池A：deleteFileOnDisk(replica)      -- 延迟删除
// 3. 线程池B：notifyNamenodeViaIBR(replica)  -- 延迟通知

// 优化后（同一线程串行）
asyncDeleteThread.execute(() -> {
    removeFromReplicaMap(replica);
    deleteFileOnDisk(replica);
    notifyNamenodeViaIBR(replica);
});
```

### 1.4 网络与心跳优化

| Issue | 版本 | 优化内容 |
|-------|------|----------|
| HDFS-16898 | 3.4.0 | 移除 processCommandFromActor 写锁，避免阻塞心跳更新导致 DN 被标记 dead |
| HDFS-16402 | 3.4.0 | 改进 HeartbeatManager 逻辑避免统计错误（stats.subtract/add 事务性保证） |
| HDFS-16735 | 3.4.0 | 减少 HeartbeatManager 循环次数 |
| HDFS-16332 | 3.4.0 | 修复过期 BlockToken 在 SASL 握手中未处理导致慢读 |
| HDFS-16099 | 3.4.0 | bpServiceToActive 声明为 volatile，避免并发可见性问题 |
| HDFS-16139 | 3.4.0 | 原子更新 BPServiceActor Scheduler 的 nextBlockReportTime |
| HDFS-16426 | 3.4.0 | 修复强制触发全量块报告时 nextBlockReportTime 计算错误 |
| HDFS-15629 | 3.4.0 | BlockReceiver 慢镜像/慢磁盘告警增加 seqno 信息 |
| HDFS-16907 | 3.4.0 | BPServiceActor 增加 LastHeartbeatResponseTime 指标 |

**HDFS-16898** 是关键修复：`processCommandFromActor` 方法持有写锁处理命令，如果命令处理耗时长，会阻塞 `updateActorStatesFromHeartbeat`，导致 lastContact 持续增大，超过 600s 阈值后 DN 被标记为 dead。

---

## 二、慢节点/慢磁盘检测与处理

3.4.0 引入了完整的慢节点/慢磁盘检测和处理机制：

### 2.1 检测增强

| Issue | 版本 | 内容 |
|-------|------|------|
| HDFS-16311 | 3.4.0 | 修复 DataNodeVolumeMetrics 中 metadataOperationRate 计算错误（影响 NN 侧慢磁盘指标） |
| HDFS-16315 | 3.4.0 | 增加 Transfer 和 NativeCopy 相关指标 |
| HDFS-16526 | 3.4.0 | 新增慢 DN 指标（FlushOrSync、PacketResponder send ACK） |
| HDFS-16521 | 3.4.0 | DFS API 获取慢节点列表（支持 `dfsadmin -report` 展示慢节点） |
| HDFS-16320 | 3.4.0 | DN 从 NN 获取慢节点信息，以便采取本地优化动作 |
| HDFS-15745 | 3.4.0 | DataNodePeerMetrics 低阈值和最小异常检测节点数可配置 |
| HDFS-16203 | 3.4.0 | 通过标准差发现 Volume 使用不均衡的 DN |

### 2.2 处理策略

| Issue | 版本 | 内容 |
|-------|------|------|
| HDFS-16371 | 3.4.0 | Volume 选择时排除慢磁盘，防止拖低整体吞吐量 |
| HDFS-16348 | 3.4.0 | 将慢节点标记为坏节点以恢复 pipeline |
| HDFS-16111 | 3.4.0 | RoundRobinVolumeChoosingPolicy 增加避免失败 Volume 的配置 |
| HDFS-16858 | 3.4.0 | 动态调整最大排除慢磁盘数 |
| HDFS-16974 | 3.4.0 | 选择目标 DN 时考虑各 Volume 平均负载 |
| HDFS-16287 | 3.4.0 | `dfs.namenode.avoid.read.slow.datanode` 支持动态重配置 |
| HDFS-15285 | 3.4.0 | 同距离同负载 DN 节点随机选择（避免集中写入） |

**HDFS-16371**：Volume 选择时排除慢磁盘。DN 已能检测慢磁盘（HDFS-11461），HDFS-16311 修复了慢磁盘指标计算错误，本 issue 实现根据规则在 Volume 选择时排除慢磁盘，防止个别慢盘拖低整个 DN 吞吐量。

**HDFS-16521**：提供 DFS API 获取慢节点列表。目的有二：1）运维人员通过 `dfsadmin -report` 查看慢节点；2）下游系统（如 HBase 的 FanOutOneBlockAsyncDFSOutput）可利用此 API 排除慢节点，避免自行实现慢节点检测逻辑。

---

## 三、动态重配置

3.4.0 大幅增强了 DN 运行时动态重配置能力，减少滚动重启需求：

| 配置参数 | Issue | 说明 |
|----------|-------|------|
| `dfs.blockreport.intervalMsec` | HDFS-16331 | 块报告间隔（冷数据 EC 集群需增大间隔减轻 NN 压力） |
| 缓存报告参数 | HDFS-16399 | 缓存报告间隔和参数 |
| DataXceiver 参数 | HDFS-16400 | 数据传输线程相关参数 |
| 慢磁盘参数 | HDFS-16397 | 慢磁盘检测阈值等 |
| 慢节点参数 | HDFS-16396 | 慢节点检测阈值等 |
| DFS 使用量参数 | HDFS-16413 | DN 磁盘使用量计算参数 |
| 传输/写带宽 | HDFS-17113 | `dfs.datanode.data.transfer.bandwidthPerSec` 和写带宽 |
| 磁盘平衡器参数 | HDFS-17075 | `dfs.disk.balancer.enabled` 和 plan 有效期 |
| 退役 Backoff 参数 | HDFS-16811 | DecommissionBackoffMonitor 参数 |
| SlowIoWarningThreshold | HDFS-17282 | 慢 IO 告警阈值 |
| 块报告参数 | HDFS-16398 | 块报告相关配置 |

```bash
# 动态重配置示例
hdfs dfsadmin -reconfig datanode <host:port> start
hdfs dfsadmin -reconfig datanode <host:port> status
```

---

## 四、监控指标增强

| Issue | 版本 | 新增指标 |
|-------|------|----------|
| HDFS-15242 | 3.4.0 | FsDatasetImpl 操作持锁时间指标 |
| HDFS-16315 | 3.4.0 | Transfer 和 NativeCopy 操作指标 |
| HDFS-16352 | 3.4.0 | getDatanodeStorageReport 返回真实 numBlocks |
| HDFS-17301 | 3.4.0 | 读写 DataXceiver 线程数指标 |
| HDFS-17208 | 3.4.0 | PendingAsyncDiskOperations 指标 |
| HDFS-16526 | 3.4.0 | 慢 DN 指标（FlushOrSync、PacketResponder ACK） |
| HDFS-15754 | 3.4.0 | DN Packet 指标 |
| HDFS-16917 | 3.4.0 | DN 读传输速率分位指标 |
| HDFS-17312 | 3.4.0 | packetsReceived 指标忽略心跳包 |
| HDFS-15781 | 3.4.0 | replaceBlock 块移动指标 |
| HDFS-17345 | 3.5.0 | 块报告生成耗时指标 |
| HDFS-17683 | 3.5.0 | dataset 读写锁获取指标 |
| HDFS-16593 | 3.4.0 | 修正 BlocksRemoved 指标不准确问题 |
| HDFS-17291 | 3.5.0 | 修复 bytesWritten 指标某些场景不准确 |
| HDFS-17725 | 3.5.0 | DataNodeVolumeMetrics 和 BalancerMetrics 增加 MetricTag |

---

## 五、稳定性修复

### 5.1 关键 Bug 修复

| Issue | 版本 | 优先级 | 修复内容 |
|-------|------|--------|----------|
| HDFS-15240 | 3.4.0 | **Blocker** | EC：脏缓冲区导致重建块错误 |
| HDFS-15963 | 3.4.0 | **Critical** | 未释放的 Volume 引用导致无限循环 |
| HDFS-15641 | 3.4.0 | **Critical** | DN 调用 refreshNameNode 可能死锁 |
| HDFS-16600 | 3.4.0 | Major | 细粒度锁死锁修复 |
| HDFS-16598 | 3.4.0 | Major | 细粒度锁 GS 校验缺失修复 |
| HDFS-16586 | 3.4.0 | Major | 清除 FsDatasetAsyncDiskService 线程组（导致 BPServiceActor IllegalThreadStateException） |
| HDFS-16985 | 3.4.0 | Major | 修复删除本地块文件时的数据丢失问题 |
| HDFS-17342 | 3.5.0 | Major | 修复 DN 可能误删正常块导致 missing block |
| HDFS-16303 | 3.4.0 | Major | 超过 100 个 DN 处于 decommissioning 状态时阻塞所有退役 |
| HDFS-16583 | 3.4.0 | Major | DatanodeAdminDefaultMonitor 无限循环修复 |
| HDFS-16704 | 3.4.0 | Major | DN 重启期间 GetVolumeInfo 返回空响应而非 NPE |
| HDFS-16623 | 3.4.0 | Major | LifelineSender 中 IllegalArgumentException 修复 |
| HDFS-15651 | 3.4.0 | Major | CommandProcessingThread 退出时客户端无法获取块 |
| HDFS-15644 | 3.4.0 | Major | 失败的 Volume 可能导致 DN 停止块报告 |
| HDFS-15386 | 3.4.0 | Major | 删除多个数据目录后 ReplicaNotFoundException 持续发生 |
| HDFS-15403 | 3.4.0 | Major | FileIoProvider#transferToSocketFully 中 NPE |

### 5.2 EC 相关修复

| Issue | 版本 | 修复内容 |
|-------|------|----------|
| HDFS-15759 | 3.4.0 | **EC 数据完整性**：DN 端验证 EC 重建正确性，防止数据损坏 |
| HDFS-15795 | 3.4.0 | EC 重建异常时校验和错误修复 |
| HDFS-15709 | 3.4.0 | StripedBlockChecksumReconstructor 中 Socket 文件描述符泄漏 |
| HDFS-14353 | 3.4.0 | EC xmitsInProgress 指标变为负数 |

**HDFS-15759** 是重要的 EC 数据完整性增强：EC 重建曾在 HDFS-14768、HDFS-15186、HDFS-15240 中导致数据损坏，且损坏不易被发现和恢复。该特性在每次 EC 重建时验证输出正确性：解码输入与原始输入比较，失败则任务重试。

---

## 六、安全与新特性

### 6.1 安全增强

| Issue | 版本 | 内容 |
|-------|------|------|
| HDFS-15353 | 3.4.0 | 安全 DN 使用 sudo 替代 su 允许 nologin 用户 |
| HDFS-15344 | 3.4.0 | DataNode#checkSuperuserPrivilege 使用 UGI#getGroups |

### 6.2 新特性

| Issue | 版本 | 内容 |
|-------|------|----------|
| HDFS-15025 | 3.4.0 | NVDIMM 存储类型支持 |
| HDFS-17087 | 3.4.0 | DN 读块限流器 |
| HDFS-17184 | 3.4.0 | BlockReceiver 初始化时正确抛出 DiskOutOfSpaceException |
| HDFS-15785 | 3.4.0 | DN 支持通过 DNS 解析 NameService 地址 |
| HDFS-15764 | 3.4.0 | 磁盘上块缺失或新增时尽快通知 NN |
| HDFS-16519 | 3.4.0 | EC 重建增加限流器 |
| HDFS-16180 | 3.4.0 | FsVolumeImpl.nextBlock 处理块元数据文件已删除情况 |
| HDFS-17526 | 3.5.0 | Windows 平台 getMetadataInputStream 使用共享删除文件输入流 |

---

## 七、版本优化对比

### 3.4.0 核心优化

| 类别 | 关键 Issue | 影响 |
|------|-----------|------|
| **锁优化** | HDFS-15382, HDFS-16534 | 吞吐量提升 30-50%（联邦场景） |
| **慢磁盘** | HDFS-16371, HDFS-16320, HDFS-16348 | 慢磁盘自动排除，pipeline 自动恢复 |
| **动态重配置** | HDFS-16331 等 11 项 | 减少滚动重启，运维效率大幅提升 |
| **EC 完整性** | HDFS-15759 | DN 端验证 EC 重建正确性，防止静默数据损坏 |
| **升级优化** | HDFS-15937, HDFS-15610 | 布局升级内存降低 7.5 倍，线程开销减少 |
| **删除优化** | HDFS-16774 | 异步删除流程优化，避免 ReplicaNotFoundException |

### 3.5.0 核心优化

| 类别 | 关键 Issue | 影响 |
|------|-----------|------|
| **锁优化 Phase II** | HDFS-17496 | 基于 blockId 的 DIR 级锁，NvmeSSD 高并发场景进一步提升 |
| **锁监控** | HDFS-17683 | dataset 读写锁获取监控，便于定位锁瓶颈 |
| **DirectoryScanner** | HDFS-17346 | 修复误标记正常块为损坏的严重问题 |
| **块误删** | HDFS-17342 | 修复 DN 可能误删正常块 |
| **块报告** | HDFS-17345 | 块报告生成耗时指标 |

---

## 八、优化演进时间线

```
3.3.1                          3.4.0 (2024.3)                    3.5.0 (2026.4)
  │                               │                                  │
  │  全局 FsDatasetImpl 锁         │  Volume 级细粒度锁               │  DIR 级细粒度锁
  │  无慢磁盘排除                  │  慢磁盘检测+排除                 │  锁获取监控指标
  │  需重启修改配置                │  11+ 参数动态重配置              │  块报告生成耗时
  │  EC 重建无验证                 │  EC 重建正确性验证               │  DN 误删块修复
  │  升级内存开销大                │  升级内存降 7.5x                 │  DirectoryScanner 修复
  │  异步删除可能数据不一致         │  异步删除流程优化                │
  │                               │  DN 读块限流器                   │
```

---

## 九、升级建议

### 9.1 推荐升级场景

| 场景 | 推荐版本 | 原因 |
|------|---------|------|
| 联邦部署/高并发 | 3.4.0+ | Volume 级锁吞吐量提升 30-50% |
| NvmeSSD + 小文件高并发 | 3.5.0 | DIR 级锁进一步提升并发性能 |
| EC 集群 | 3.4.0+ | 重建正确性验证防止静默数据损坏 |
| 大规模集群（滚动重启困难） | 3.4.0+ | 11+ 参数动态重配置减少重启 |
| 多磁盘 DN | 3.4.0+ | 慢磁盘自动排除、Volume 选择优化 |

### 9.2 升级注意事项

1. **锁机制变更**：3.4.0 默认启用 Volume 级锁，相关旧配置（`dfs.datanode.lock.read.write.enabled`）已移除
2. **HDFS-16600 死锁**：3.4.0 早期可能存在细粒度锁死锁问题，确保升级到包含修复的版本
3. **HDFS-17342 块误删**：3.5.0 修复了 DN 可能误删正常块的问题，建议 3.4.0 用户关注此修复
4. **EC 验证**：3.4.0 启用 EC 重建正确性验证会增加少量 CPU 开销，但可防止数据损坏

---

## 推荐阅读

- [HDFS DataNode启动过程](./dataNode启动过程.md)
- [HDFS NameNode全景](./namenode全景.md)
- [HDFS BPServiceActor详解](./BPServiceActor详解.md)
- [3.3.1-3.4.1兼容性分析](../watch/3.3.1-3.4.1兼容性分析.md)
- [值得关注的特性](../watch/值得关注的特性.md)
