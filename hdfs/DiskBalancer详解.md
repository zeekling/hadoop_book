
## HDFS DiskBalancer 详解

DiskBalancer 是 HDFS 中用于在单个 DataNode 内部不同磁盘（卷）之间均衡数据的工具。与集群级别的 Balancer（节点间均衡）互补，DiskBalancer 解决的是**节点内磁盘数据不均衡**的问题。

典型场景：当 DataNode 上新增了一块磁盘，或者某块磁盘被替换后，新磁盘使用率远低于其他磁盘，此时需要 DiskBalancer 将数据从高使用率磁盘搬运到低使用率磁盘。

---

## 一、架构概述

DiskBalancer 采用**计划-执行（Plan-Execute）**模式，分为三个阶段：

1. **Plan（规划）**：CLI 端连接 NameNode 获取节点存储快照，使用贪心算法生成 JSON 格式的搬运计划
2. **Execute（执行）**：将计划通过 RPC 提交到目标 DataNode，DataNode 在后台单线程异步执行
3. **Query/Cancel（查询/取消）**：运维人员可查询执行进度或取消运行中的计划

**关键约束：**

- 每个 DataNode 同一时间只能执行**一个**计划
- **非自动触发**，必须通过 CLI 手动操作
- 只在**同 StorageType** 的卷之间搬运数据（DISK↔DISK，SSD↔SSD）
- 跳过瞬态存储（transient）、故障卷（failed）、只读卷
- 计划执行状态是**临时的**——重启 DataNode 等效于取消计划

---

## 二、工作流程

```
运维人员                    CLI端                       NameNode                  DataNode
  |                          |                             |                        |
  |-- hdfs diskbalancer ---->|                             |                        |
  |      -plan dn1          |                             |                        |
  |                         |--获取存储报告--------------->|                        |
  |                         |<--DatanodeStorageReport-----|                        |
  |                         |--GreedyPlanner计算计划--     |                        |
  |                         |  生成dn1.plan.json          |                        |
  |<--计划文件路径----------|                             |                        |
  |                         |                             |                        |
  |-- hdfs diskbalancer ---->|                             |                        |
  |      -execute plan.json |                             |                        |
  |                         |--submitDiskBalancerPlan---->|(转发到DataNode)-------->|
  |                         |                             |                        |--验证计划
  |                         |                             |                        |--创建workMap
  |                         |                             |                        |--启动单线程执行
  |                         |                             |                        |
  |-- hdfs diskbalancer ---->|                             |                        |
  |      -query dn1         |                             |                        |
  |                         |--queryDiskBalancerPlan-----|(转发到DataNode)-------->|
  |                         |<--DiskBalancerWorkStatus---|<-----------------------|
  |<--进度信息-------------|                             |                        |
```

---

## 三、核心类及其职责

### 3.1 执行引擎（DataNode 端）

| 类 | 路径 | 职责 |
|---|------|------|
| `DiskBalancer` | `hadoop-hdfs/.../datanode/DiskBalancer.java` | 主入口，管理计划提交/验证/执行生命周期 |
| `DiskBalancerMover` | DiskBalancer 内部类 (line 768) | 实际数据搬运器，实现 `BlockMover` 接口 |
| `VolumePair` | DiskBalancer 内部类 (line 679) | 源/目标卷 UUID 对，作为 workMap 的 key |
| `DiskBalancerWorkItem` | `hadoop-hdfs-client/.../DiskBalancerWorkItem.java` | 单个卷对搬运的进度跟踪（字节/错误/带宽等） |
| `DiskBalancerWorkStatus` | `hadoop-hdfs-client/.../DiskBalancerWorkStatus.java` | 整体计划状态枚举 |

#### DiskBalancer 关键方法

| 方法 | 位置 | 说明 |
|------|------|------|
| `submitPlan()` | line 180 | 主入口：验证计划、创建工作项、启动执行 |
| `cancelPlan()` | line 266 | 设置退出标志取消运行中的计划 |
| `queryWorkStatus()` | line 232 | 返回当前所有卷对的搬运进度 |
| `verifyPlan()` | line 407 | 校验版本、SHA-1哈希、时间戳、节点UUID |
| `createWorkPlan()` | line 513 | 将 NodePlan Steps 转换为 ConcurrentHashMap |
| `executePlan()` | line 576 | 提交到单线程 ExecutorService 执行 |
| `shutdown()` | line 130 | 关闭磁盘均衡器，停止执行器 |

#### DiskBalancerMover 关键方法

| 方法 | 位置 | 说明 |
|------|------|------|
| `copyBlocks()` | line 1053 | 核心执行循环：遍历源卷块、拷贝到目标卷 |
| `getBlockToCopy()` | line 952 | 从迭代器中找下一个可搬运的块（first-fit策略） |
| `getNextBlock()` | line 1007 | 轮询 blockPool 迭代器获取下一个块 |
| `computeDelay()` | line 914 | 带宽节流：计算sleep时间（令牌桶算法） |
| `isCloseEnough()` | line 882 | 容差检查：是否已搬运到目标字节的容差范围内 |
| `isLessThanNeeded()` | line 855 | 判断块大小是否在剩余预算内 |

### 3.2 数据模型

| 类 | 路径 | 职责 |
|---|------|------|
| `DiskBalancerCluster` | `.../datamodel/DiskBalancerCluster.java` | 集群表示，线程池并行计算各节点计划 |
| `DiskBalancerDataNode` | `.../datamodel/DiskBalancerDataNode.java` | 单节点，含 VolumeSet（按 StorageType 分组） |
| `DiskBalancerVolumeSet` | `.../datamodel/DiskBalancerVolumeSet.java` | 同类型卷集合，计算 idealUsed 和 volumeDataDensity |
| `DiskBalancerVolume` | `.../datamodel/DiskBalancerVolume.java` | 单卷：路径/容量/已用/保留/UUID/密度值/跳过标记 |

#### 核心度量指标

**volumeDataDensity**（卷数据密度）：

```
idealUsed = totalUsed / totalEffectiveCapacity  (同一VolumeSet内所有非故障卷的汇总)
volumeDataDensity = idealUsed - (volumeUsed / volumeEffectiveCapacity)
```

- **正值** = 利用不足（需要接收数据）
- **负值** = 利用过度（需要送出数据）

**nodeDataDensity**（节点数据密度）：

```
nodeDataDensity = Σ|volumeDataDensity|  (所有卷的绝对密度之和)
```

该值越大，说明节点内磁盘数据越不均衡，用于排名哪些节点最需要均衡。

### 3.3 规划器

| 类 | 路径 | 职责 |
|---|------|------|
| `Planner` | `.../planner/Planner.java` | 接口：`NodePlan plan(DiskBalancerDataNode)` |
| `GreedyPlanner` | `.../planner/GreedyPlanner.java` | **唯一实现**，贪心算法 |
| `PlannerFactory` | `.../planner/PlannerFactory.java` | 工厂，目前只返回 GreedyPlanner |
| `NodePlan` | `.../planner/NodePlan.java` | 节点计划，含 Step 列表，可 JSON 序列化 |
| `MoveStep` | `.../planner/MoveStep.java` | 单个搬运步骤：源/目标卷 + 字节数 |

### 3.4 连接器

| 类 | 路径 | 职责 |
|---|------|------|
| `ClusterConnector` | `.../connectors/ClusterConnector.java` | 接口：读取节点数据 |
| `ConnectorFactory` | `.../connectors/ConnectorFactory.java` | URI scheme 分发：`file://` → JSON，HDFS → NameNode |
| `DBNameNodeConnector` | `.../connectors/DBNameNodeConnector.java` | 连接 NameNode 获取 DatanodeStorageReport |
| `JsonNodeConnector` | `.../connectors/JsonNodeConnector.java` | 从 JSON 文件读取（离线规划/调试用） |

### 3.5 CLI 命令

| 类 | 路径 | 职责 |
|---|------|------|
| `DiskBalancerCLI` | `.../tools/DiskBalancerCLI.java` | 顶层 CLI 入口，分发到子命令 |
| `PlanCommand` | `.../command/PlanCommand.java` | 生成计划，写入 `.before.json` + `.plan.json` |
| `ExecuteCommand` | `.../command/ExecuteCommand.java` | 读取计划文件，SHA-1 校验，提交 RPC |
| `QueryCommand` | `.../command/QueryCommand.java` | 查询 DataNode 当前计划状态 |
| `CancelCommand` | `.../command/CancelCommand.java` | 通过计划文件或 planID+node 取消 |
| `ReportCommand` | `.../command/ReportCommand.java` | 报告卷信息或 Top N 不均衡节点 |
| `HelpCommand` | `.../command/HelpCommand.java` | 打印帮助信息 |

### 3.6 异常与常量

**DiskBalancerException.Result** 枚举：

| 值 | 说明 |
|----|------|
| DISK_BALANCER_NOT_ENABLED | 磁盘均衡未启用 |
| INVALID_PLAN_VERSION | 计划版本无效 |
| INVALID_PLAN | 计划无效 |
| INVALID_PLAN_HASH | 计划哈希校验失败 |
| OLD_PLAN_SUBMITTED | 计划已过期 |
| DATANODE_ID_MISMATCH | 节点UUID不匹配 |
| MALFORMED_PLAN | 计划格式错误 |
| PLAN_ALREADY_IN_PROGRESS | 已有计划在执行 |
| INVALID_VOLUME | 无效卷 |
| INVALID_MOVE | 无效搬运步骤 |

---

## 四、配置项

| 配置键 | 默认值 | 说明 |
|--------|--------|------|
| `dfs.disk.balancer.enabled` | `true` | 是否启用 DiskBalancer，禁用后所有 execute 命令被拒绝 |
| `dfs.disk.balancer.max.disk.throughputInMBperSec` | `10` | 搬运时最大磁盘带宽 (MB/s) |
| `dfs.disk.balancer.max.disk.errors` | `5` | 每步最大容忍错误数，超过则放弃该步 |
| `dfs.disk.balancer.plan.valid.interval` | `1d` | 计划有效期（支持 ms/s/m/h/d 后缀），过期计划被拒绝 |
| `dfs.disk.balancer.block.tolerance.percent` | `10` | 搬运完成容差百分比，在此范围内视为完成 |
| `dfs.disk.balancer.plan.threshold.percent` | `10` | 规划阈值——卷密度绝对值超过此百分比才触发均衡 |

---

## 五、协议接口

### 5.1 Protobuf RPC

定义在 `ClientDatanodeProtocol.proto` 中：

| RPC 方法 | 请求 | 响应 | 说明 |
|----------|------|------|------|
| `submitDiskBalancerPlan` | planID, plan, planVersion, ignoreDateCheck, planFile | 空 | 提交计划 |
| `cancelDiskBalancerPlan` | planID | 空 | 取消计划 |
| `queryDiskBalancerPlan` | 空 | result, planID, currentStatus, planFile | 查询状态 |
| `getDiskBalancerSetting` | key | value | 获取配置值 |

### 5.2 Java 接口

`ClientDatanodeProtocol.java`（line 176-201）：

```java
void submitDiskBalancerPlan(String planID, long planVersion,
    String planFile, String planData, boolean skipDateCheck) throws IOException;

void cancelDiskBalancePlan(String planID) throws IOException;

DiskBalancerWorkStatus queryDiskBalancerPlan() throws IOException;

String getDiskBalancerSetting(String key) throws IOException;
```

### 5.3 DataNode 端实现

DataNode.java 中的 RPC 实现：

| 方法 | 位置 | 说明 |
|------|------|------|
| `submitDiskBalancerPlan()` | line 4231 | 检查超级用户权限 + DataNode 状态为 REGULAR，委托给 DiskBalancer |
| `cancelDiskBalancePlan()` | line 4251 | 检查权限后委托给 DiskBalancer |
| `queryDiskBalancerPlan()` | line 4263 | 委托给 DiskBalancer |
| `getDiskBalancerSetting()` | line 4277 | 支持 `DiskBalancerVolumeName` 和 `DiskBalancerBandwidth` 两个 key |
| `initDiskBalancer()` | line 1640 | 创建 DiskBalancerMover 和 DiskBalancer 实例 |

---

## 六、CLI 使用

### 6.1 生成计划

```bash
hdfs diskbalancer -plan <hostname> \
  [-bandwidth <MB/s>] \
  [-thresholdPercentage <pct>] \
  [-maxerror <n>] \
  [-out <output_path>] \
  [-v]
```

- `-bandwidth`：每步搬运带宽限制（MB/s），默认使用配置值
- `-thresholdPercentage`：卷密度阈值，超过才规划搬运，默认 10%
- `-maxerror`：每步最大错误数，默认 5
- `-out`：计划文件输出路径
- `-v`：详细输出

输出文件：
- `<hostname>.before.json` — 节点当前存储快照
- `<hostname>.plan.json` — 生成的搬运计划

### 6.2 执行计划

```bash
hdfs diskbalancer -execute <plan_file_path> [-skipDateCheck]
```

- `-skipDateCheck`：跳过计划有效期检查（适用于旧计划）

### 6.3 查询状态

```bash
hdfs diskbalancer -query <hostname:port> [-v]
```

返回结果状态：

| 状态 | 说明 |
|------|------|
| NO_PLAN | 无计划 |
| PLAN_UNDER_PROGRESS | 计划执行中 |
| PLAN_DONE | 计划已完成 |
| PLAN_CANCELLED | 计划已取消 |

### 6.4 取消计划

```bash
# 通过计划文件取消
hdfs diskbalancer -cancel <plan_file_path>

# 通过 planID 和节点取消
hdfs diskbalancer -cancel <planID> -node <hostname:port>
```

### 6.5 查看报告

```bash
# 查看前 N 个最不均衡的节点
hdfs diskbalancer -report -top <n>

# 查看指定节点的卷详情
hdfs diskbalancer -report -node <hostname1,hostname2,...>
```

### 6.6 帮助

```bash
hdfs diskbalancer -help [plan|execute|query|cancel|report]
```

---

## 七、GreedyPlanner 算法详解

GreedyPlanner 是目前唯一的规划算法实现，位于 `GreedyPlanner.java`。

### 7.1 算法流程

```
对每个 DiskBalancerVolumeSet（同 StorageType 的卷集合）：

1. 计算 idealUsed = totalUsed / totalEffectiveCapacity
2. 对每个卷计算 volumeDataDensity = idealUsed - (used / effectiveCapacity)
3. 将卷按 volumeDataDensity 排序放入 TreeSet（MinHeap）
4. 循环直到所有卷的 |volumeDataDensity| < threshold：
   a. 取 first()（最不足/密度最高）= 接收端
   b. 取 last()（最过度/密度最低）= 发送端
   c. 计算搬运量：
      maxLowCanReceive  = (idealUsed × lowCapacity) - lowUsed
      maxHighCanGive    = highUsed - (idealUsed × highCapacity)
      bytesToMove       = min(maxLowCanReceive, maxHighCanGive)
   d. 若任一端无法继续，标记该卷 skip
   e. 创建 MoveStep(高卷→低卷, bytesToMove)
   f. 模拟执行：更新 used 值，重算 volumeDataDensity
5. 返回 NodePlan
```

### 7.2 算法特点

- **贪心策略**：每次选择最不均衡的一对（最过度→最不足）
- **迭代收敛**：每步后重新计算密度，后续步骤考虑已搬运量
- **按存储类型隔离**：只在同 StorageType 卷间搬运
- **阈值门控**：仅当 `|volumeDataDensity| > threshold/100` 时才生成搬运步骤
- **确定性**：相同快照产生相同计划

### 7.3 示例

假设一个 DataNode 有 3 块 DISK，容量均为 1TB：

| 卷 | 已用 | 使用率 |
|----|------|--------|
| /disk1 | 800GB | 80% |
| /disk2 | 500GB | 50% |
| /disk3 | 200GB | 20% |

totalUsed = 1500GB，totalCapacity = 3000GB，idealUsed = 50%

volumeDataDensity：
- /disk1: 0.5 - 0.8 = **-0.3**（过度使用）
- /disk2: 0.5 - 0.5 = **0**（理想）
- /disk3: 0.5 - 0.2 = **+0.3**（使用不足）

第一步：/disk1(过度) → /disk3(不足)
- maxGive = 800 - 500 = 300GB
- maxReceive = 500 - 200 = 300GB
- bytesToMove = 300GB

模拟后：/disk1=500GB, /disk3=500GB, 全部达到 idealUsed，均衡完成。

---

## 八、执行机制详解

### 8.1 计划提交流程

```
CLI ExecuteCommand
  → 读取 plan.json 文件
  → 计算 SHA-1 哈希作为 planID
  → 创建 ClientDatanodeProtocol 代理
  → 调用 submitDiskBalancerPlan(planHash, version=1, planFile, planData, skipDateCheck)
  → DataNode 检查超级用户权限 + REGULAR 状态
  → DiskBalancer.submitPlan():
      1. 检查 enabled
      2. 检查无其他计划在运行（future.isDone()）
      3. verifyPlan(): 版本(1)、SHA-1、时间戳、节点UUID
      4. createWorkPlan(): Steps → ConcurrentHashMap<VolumePair, WorkItem>
         - 相同 VolumePair 的多个 Step 合并字节数
      5. executePlan(): 单线程 ExecutorService 提交 Runnable
```

### 8.2 块拷贝执行循环

```
对每个 VolumePair in workMap:
  1. 通过 UUID 查找源卷和目标卷 FsVolumeSpi
  2. 拒绝瞬态存储
  3. 在源卷上打开 blockPool 迭代器
  4. while(shouldRun()):
     a. errorCount > maxDiskErrors → 放弃此步
     b. isCloseEnough() → 容差范围内，标记完成
     c. getNextBlock() → 轮询 blockPool，first-fit 选择块
     d. 无合适块 → 退出
     e. 检查目标卷剩余空间 > bytesToCopy
     f. dataset.moveBlockAcrossVolumes(block, dest) → 本地磁盘拷贝
     g. computeDelay() → Thread.sleep() → 带宽节流
     h. 更新 bytesCopied, blocksCopied, secondsElapsed
  5. finally: 关闭 blockPool 迭代器
```

### 8.3 计划验证

| 验证项 | 规则 |
|--------|------|
| 版本检查 | planVersion 必须为 1（MIN_VERSION=MAX_VERSION=1） |
| 哈希校验 | SHA-1(planData) 必须等于提交的 planID |
| 时间戳检查 | planTimestamp + planValidityInterval >= 当前时间（除非 skipDateCheck） |
| 节点UUID检查 | plan 中的 nodeUUID 必须匹配目标 DataNode 的 UUID |

---

## 九、节流与带宽控制

### 9.1 带宽节流机制

`DiskBalancerMover.computeDelay()`（line 914）实现类令牌桶算法：

```
每拷完一个块后：
1. 记录 bytesCopied 和 timeUsed（实际耗时）
2. 将 bytesCopied 转换为 MB
3. 若 < 1MB，不延迟（小块跳过）
4. 计算期望耗时 = bytesInMB / bandwidth(MB/ms)
5. 计算延迟 = 期望耗时 - 实际耗时
6. 若延迟 > 0，Thread.sleep(delay)
```

特点：
- **突发+均摊**模式：全速拷贝，然后 sleep 均摊
- **单线程顺序执行**：带宽限制是每个搬运步骤独立计算
- 带宽可通过 CLI `-bandwidth` 参数逐步骤指定，或使用全局配置

### 9.2 错误容忍

- **maxDiskErrors**（默认5）：每个搬运步骤独立计数，超过则放弃该步，继续下一步
- **tolerancePercent**（默认10%）：`bytesCopied + (bytesCopied × 10%) >= bytesToCopy` 时视为完成

### 9.3 计划有效期

- **plan.valid.interval**（默认1天）：防止执行过时的计划，因为磁盘使用情况可能已变化
- 可通过 `-skipDateCheck` 跳过此检查

---

## 十、与集群级 Balancer 对比

| 维度 | DiskBalancer（节点内） | Balancer（集群级） |
|------|----------------------|-------------------|
| **作用域** | 同一 DataNode 内磁盘间 | DataNode 间跨网络 |
| **数据搬运** | 本地 `moveBlockAcrossVolumes()` | 网络传输到远程 DataNode |
| **触发方式** | CLI 手动，需逐节点操作 | CLI 可持续运行 |
| **执行位置** | DataNode 端（后台线程） | Client 端（直接编排） |
| **并发模型** | 单线程顺序执行搬运步骤 | 多线程并发搬运 |
| **存储类型** | 仅同 StorageType 间搬运 | 可跨类型（配合存储策略） |
| **节流方式** | 磁盘带宽 MB/s | 网络带宽 `dfs.datanode.balance.bandwidthPerSec` |
| **计划持久化** | JSON 写入 HDFS `/system/diskbalancer/` | 无显式持久化 |
| **自动运行** | 不支持，需手动执行 | 支持持续运行 |

**两者互补**：Balancer 保证集群级数据均匀分布，DiskBalancer 保证节点内磁盘级均匀分布。两者可以同时运行，互不干扰——DiskBalancer 使用本地磁盘 I/O 带宽，Balancer 使用网络带宽。

---

## 十一、源码文件清单

### 主要源码

| 文件 | 路径 |
|------|------|
| DiskBalancer.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/datanode/DiskBalancer.java` |
| DiskBalancerWorkItem.java | `hadoop-hdfs-project/hadoop-hdfs-client/src/main/java/org/apache/hadoop/hdfs/server/datanode/DiskBalancerWorkItem.java` |
| DiskBalancerWorkStatus.java | `hadoop-hdfs-project/hadoop-hdfs-client/src/main/java/org/apache/hadoop/hdfs/server/datanode/DiskBalancerWorkStatus.java` |
| DiskBalancerConstants.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/DiskBalancerConstants.java` |
| DiskBalancerException.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/DiskBalancerException.java` |
| DiskBalancerCLI.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/tools/DiskBalancerCLI.java` |
| Command.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/command/Command.java` |
| PlanCommand.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/command/PlanCommand.java` |
| ExecuteCommand.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/command/ExecuteCommand.java` |
| QueryCommand.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/command/QueryCommand.java` |
| CancelCommand.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/command/CancelCommand.java` |
| ReportCommand.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/command/ReportCommand.java` |
| HelpCommand.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/command/HelpCommand.java` |
| DiskBalancerCluster.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/datamodel/DiskBalancerCluster.java` |
| DiskBalancerDataNode.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/datamodel/DiskBalancerDataNode.java` |
| DiskBalancerVolume.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/datamodel/DiskBalancerVolume.java` |
| DiskBalancerVolumeSet.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/datamodel/DiskBalancerVolumeSet.java` |
| Planner.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/planner/Planner.java` |
| GreedyPlanner.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/planner/GreedyPlanner.java` |
| PlannerFactory.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/planner/PlannerFactory.java` |
| NodePlan.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/planner/NodePlan.java` |
| Step.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/planner/Step.java` |
| MoveStep.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/planner/MoveStep.java` |
| ClusterConnector.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/connectors/ClusterConnector.java` |
| ConnectorFactory.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/connectors/ConnectorFactory.java` |
| DBNameNodeConnector.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/connectors/DBNameNodeConnector.java` |
| JsonNodeConnector.java | `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/diskbalancer/connectors/JsonNodeConnector.java` |

### 测试文件

| 文件 | 路径 |
|------|------|
| TestDiskBalancer.java | `hadoop-hdfs-project/hadoop-hdfs/src/test/java/org/apache/hadoop/hdfs/server/diskbalancer/TestDiskBalancer.java` |
| TestDiskBalancerWithMockMover.java | `hadoop-hdfs-project/hadoop-hdfs/src/test/java/org/apache/hadoop/hdfs/server/diskbalancer/TestDiskBalancerWithMockMover.java` |
| TestDiskBalancerRPC.java | `hadoop-hdfs-project/hadoop-hdfs/src/test/java/org/apache/hadoop/hdfs/server/diskbalancer/TestDiskBalancerRPC.java` |
| TestPlanner.java | `hadoop-hdfs-project/hadoop-hdfs/src/test/java/org/apache/hadoop/hdfs/server/diskbalancer/TestPlanner.java` |
| TestDataModels.java | `hadoop-hdfs-project/hadoop-hdfs/src/test/java/org/apache/hadoop/hdfs/server/diskbalancer/TestDataModels.java` |
| TestConnectors.java | `hadoop-hdfs-project/hadoop-hdfs/src/test/java/org/apache/hadoop/hdfs/server/diskbalancer/TestConnectors.java` |
| TestDiskBalancerCommand.java | `hadoop-hdfs-project/hadoop-hdfs/src/test/java/org/apache/hadoop/hdfs/server/diskbalancer/command/TestDiskBalancerCommand.java` |
| DiskBalancerTestUtil.java | `hadoop-hdfs-project/hadoop-hdfs/src/test/java/org/apache/hadoop/hdfs/server/diskbalancer/DiskBalancerTestUtil.java` |

### 文档

| 文件 | 路径 |
|------|------|
| HDFSDiskbalancer.md | `hadoop-hdfs-project/hadoop-hdfs/src/site/markdown/HDFSDiskbalancer.md` |

---

## 十二、常见运维场景

### 场景1：新增磁盘后均衡

```bash
# 1. 查看哪些节点最不均衡
hdfs diskbalancer -report -top 5

# 2. 为目标节点生成计划
hdfs diskbalancer -plan datanode1.example.com -bandwidth 20 -thresholdPercentage 5

# 3. 执行计划
hdfs diskbalancer -execute /system/diskbalancer/datanode1.plan.json

# 4. 查看进度
hdfs diskbalancer -query datanode1.example.com:9867 -v
```

### 场景2：磁盘使用率告警时均衡

```bash
# 1. 查看特定节点的卷详情
hdfs diskbalancer -report -node datanode2.example.com

# 2. 生成计划（使用较低带宽避免影响业务）
hdfs diskbalancer -plan datanode2.example.com -bandwidth 5

# 3. 执行并持续监控
hdfs diskbalancer -execute /system/diskbalancer/datanode2.plan.json
hdfs diskbalancer -query datanode2.example.com:9867
```

### 场景3：取消运行中的计划

```bash
# 方式1：通过计划文件
hdfs diskbalancer -cancel /system/diskbalancer/datanode1.plan.json

# 方式2：通过planID和节点
hdfs diskbalancer -cancel <planID> -node datanode1.example.com:9867
```
