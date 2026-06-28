# UpFuzz: Detecting Data Format Incompatibility Bugs during Distributed Storage System Upgrade 翻译解析

## 论文信息

| 项目 | 内容 |
|------|------|
| **标题** | UpFuzz: Detecting Data Format Incompatibility Bugs during Distributed Storage System Upgrade |
| **会议** | **NSDI '26**（USENIX Symposium on Networked Systems Design and Implementation，CCF-A 顶级会议） |
| **获奖** | **Community Award**（社区奖） |
| **作者** | Ke Han, Sruthi P C（普渡大学）；Yayu Wang（UBC）；Yaoxu Song, Bishal Basak Papan（普渡大学）；Junwen Yang（Meta）；Pedro Fonseca, Yongle Zhang（普渡大学） |
| **机构** | **普渡大学（Purdue University）** + UBC + Meta |
| **PDF** | https://www.usenix.org/system/files/nsdi26-han.pdf（Open Access，19页） |
| **代码** | https://github.com/zlab-purdue/upfuzz（开源） |

---

## 摘要翻译

**英文原文：**
> Data format incompatibility is a significant cause of cloud incidents during distributed system upgrades, often resulting in severe consequences such as data corruption and service unavailability. A majority of such bugs are only discovered post-release, largely due to the lack of automated testing techniques tailored specifically for the upgrade process. Traditional automated test generation methods face a unique challenge when applied to upgrade testing: the high cost associated with upgrading distributed storage systems due to system initialization. Therefore, the accurate selection of potential failure-inducing tests from the extensive pool of automatically generated tests becomes critical.
>
> In this work, we address this problem by proposing a novel approach to prioritize upgrade tests through analyzing data format properties over **transitively persisted states**: program states that are persisted to disk, directly or indirectly, through chains of memory copies by the old version, and eventually read by the new version after upgrade. Because data format incompatibility bugs happen due to translation errors of such states across versions, transitively persisted states satisfying unique data format properties related to changed data formats are particularly essential for testing.
>
> We build a likely invariant analysis engine that captures such properties as feedback for seed test selection in **UPFUZZ**, the automated testing engine for the distributed storage system upgrade procedure. UPFUZZ has detected 15 previously unknown upgrade failures caused by data format incompatibilities in the latest stable versions of Cassandra, HBase, and HDFS; developers have confirmed 8 of them. 7 are triggered exclusively with UPFUZZ's data format analysis. The detected bugs have severe consequences, with 6 crashing the cluster and 4 causing data loss or corruption.

**中文翻译：**

> 开源代码：https://github.com/zlab-purdue/upfuzz

数据格式不兼容是分布式系统升级过程中导致云故障的重要原因，通常会导致数据损坏和服务不可用等严重后果。这些 bug 大多数只在发布后才被发现，主要是因为缺乏专门针对升级过程的自动化测试技术。传统的自动化测试生成方法在应用于升级测试时面临一个独特的挑战：由于系统初始化的原因，升级分布式存储系统的成本极高。因此，从海量自动化生成的测试中**准确选择潜在能触发故障的测试**变得至关重要。

在本工作中，我们通过提出一种新颖的方法来解决此问题：通过分析**传递持久化状态（Transitively Persisted States，TPState）**上的数据格式属性来优先选择升级测试。TPState 是指那些被旧版本通过内存拷贝链直接或间接持久化到磁盘，最终在升级后被新版本读取的程序状态。由于数据格式不兼容 bug 正是由这类状态在跨版本转换时出错引起的，因此满足与变更的数据格式相关的特定数据格式属性的 TPState 对于测试尤为关键。

我们构建了一个**似然不变性分析引擎（likely invariant analysis engine）**，用于捕获这些属性作为 UPFUZZ 种子测试选择的反馈信号。UPFUZZ 是在分布式存储系统升级过程中进行自动化测试的引擎。UPFUZZ 已在 Cassandra、HBase 和 HDFS 的最新稳定版本中检测到 **15 个此前未知的由数据格式不兼容引发的升级故障**，其中 **8 个已得到开发者确认**。**7 个 bug 仅由 UPFUZZ 的数据格式分析技术触发**。这些 bug 后果严重：**6 个导致集群崩溃，4 个导致数据丢失或损坏**。

---

## 1. 研究动机

### 1.1 分布式系统升级中的 DFI Bug

**数据格式不兼容（Data Format Incompatibility，DFI）bug** 是分布式系统升级过程中主要的故障来源之一：

| 维度 | 数据 |
|------|------|
| **占比** | DFI bug 占分布式系统升级故障的 **近三分之二** |
| **后果** | 数据丢失、数据损坏、大规模服务不可用 |
| **发现时机** | 大多数发布于**发布后**才被发现 |
| **典型案例** | 2025 年 Google Cloud 故障（影响 140 万用户）：策略数据结构的格式不兼容 |
| | 2024 年 CrowdStrike 全球宕机（850 万系统崩溃）：更新过程中的格式不兼容 |

### 1.2 现有方法的局限性

| 方法 | 局限性 |
|------|--------|
| **人工编写升级测试** | 覆盖场景极为有限，手工成本高 |
| **DUPTester（先前工作）** | 依赖现有压力测试和单元测试，继承手工测试的覆盖限制 |
| **传统 Fuzzing（AFL/Syzkaller）** | 对分布式系统不适用；升级过程**耗时数十秒甚至数分钟**，与每秒千万次操作的传统操作形成巨大反差 |
| **基于覆盖率的反馈（Code Coverage）** | 无法捕获 DFI bug 的触发条件（如行末对齐、列被删除等） |
| **基于状态的反馈（State Coverage）** | 状态爆炸问题，且 DFI 条件通常涉及间接引用的变量，不在同一基本块内 |

### 1.3 核心挑战

升级操作的**极高成本**是自动化测试的核心难点：

```
一次升级操作耗时：30-145 秒（取决于系统）
而普通操作可达：数十万次/秒
```

因此，必须**精确选择**最可能触发 bug 的测试去运行升级操作，否则测试吞吐量将极低。

---

## 2. 核心思想：传递持久化状态（TPState）分析

### 2.1 TPState 定义

**传递持久化状态（Transitively Persisted State，TPState）**：是指被程序通过直接或传递性的内存拷贝链持久化到外部存储的程序状态子集，最终将在升级后被新版本读取。

```
程序状态 = 所有运行时数据
         ↓
TPState = 会被持久化到磁盘的程序状态子集
         ↓
目标状态 = TPState 中格式在版本间发生变更的状态
```

**目标状态的三层筛选（如图 1 所示）：**
1. **程序状态（Program State）**— 所有运行时数据
2. **TPState（传递持久化状态）**— 最终会写入磁盘的状态
3. **格式变更状态**— TPState 中类定义/数据成员在版本间发生变更的状态 ← **核心目标**

### 2.2 核心观察

**观察 1：** 影响反序列化过程的持久化状态的数据格式属性通常在其**内存表示**中被保留。

例如，在 Cassandra-13939 中，触发 bug 的两个条件可以通过监测表和索引的内存数据结构属性来捕获：
- 条件①：一个索引块在最后一列结束（行在块边界结束）
- 条件②：被删除的列是表中的最后一列

**观察 2：** 持久化状态内存表示**规格说明（类定义）的修改**反映了底层数据格式的相应变化。

```
旧版本 Cassandra 2.2:  class IndexBlk { Cell start; Cell end; }
新版本 Cassandra 3.0:  class IndexBlk { ClusteringPrefix start; ClusteringPrefix end; }
                                            ↑ 类型变更 = 数据格式变更
```

### 2.3 工作流程

```
测试变异 → 旧版本执行（TPState 分析）→ 升级决策（走or跳）→ 新版本执行 → 故障检测
                                                    ↑
                                             反馈信号（格式似然不变量）
```

---

## 3. 技术方法详解

### 3.1 TPState 选择性监控（Selective TPState Monitoring）

**解决两个问题：**

#### 3.1.1 如何识别 TPState

使用**过程间向后数据流分析**，从注释的磁盘 I/O 操作（如 `OutputStream.write()`）的参数出发，逆向追踪数据流：

1. 起点：磁盘 I/O 调用的参数
2. 向后追踪停止条件：
   - 初始化语句（如 `new`）
   - 非拷贝赋值语句（如 `a = b + c`）
3. 当访问的字段是缓冲区时，继续追踪被序列化到缓冲区中的状态

#### 3.1.2 在哪里监控 TPState

TPState 的生命周期：**分配 → 修改 → 序列化 → 回收**

监控点插入在**修改和序列化之间的合并点**（merge point），确保：
- 覆盖所有最终被持久化的状态
- 避免在热路径上频繁插桩
- 准确反映实际持久化的状态

```
修改点（Modification Site）    序列化点（Serialization Site）
      │                              │
      └──────── 监控点 ──────────────┘
              (控制流合并点)
```

### 3.2 选择性属性推断（Selective Property Inference）

#### 3.2.1 数据格式属性分类

基于 12 个真实 DFI bug 的研究，论文归纳出 **5 类属性模板**：

| 属性类型 | 描述 | 示例 | 占比 |
|---------|------|------|------|
| **边界（Boundary）** | 偏移量是否达到边界，边界项是否在序列边界 | `x.offset ≥ b`，`seq.first/last` | 23% |
| **特殊值（Special Value）** | 检查 0、null、长度等特殊值 | `x = 0`, `x = null`, `x.length() = 1` | 37% |
| **等价（Equivalence）** | 两个变量是否相等 | `Objects.equals(x, y)` | 17% |
| **类型（Type）** | 是否出现新类型 | `x.getClass() ∈ {τ₁, τ₂, ...}` | 31% |
| **合取（Conjunction）** | 多个属性同时成立 | `INV₁(x.f₁) ∧ INV₂(x.f₂)` | 34% |

> 占比总和超过 100% 是因为一个属性可能同时属于多个类型。合取类型的占比表示需要与其他属性合取才能表达 bug 条件的属性比例。

#### 3.2.2 与版本差异关联

UPFUZZ 通过**引用距离（Reference Distance）**将属性与版本差异关联：

```
引用距离 = TPState 中监测状态到最近的版本差异的引用层数
```

使用指数衰减概率模型选择测试：
```
P(N) = c · e^(-k·N)
```

其中 `c = 0.8`，`k = -0.5`，`N` 为引用距离。距离越小，测试被选中执行升级的概率越高。

**示例：** 在 Cassandra-13939 中，`IndexBlk.end` 的类型从 `Cell` 变为 `ClusteringPrefix`，引用距离为 0，因此能被优先选中。

### 3.3 高效属性推断与选择性子结构合取

#### 3.3.1 似然不变量推断

对每个 TPState，构建**对象引用图（Object Reference Graph，ORG）**，推断似然不变量：

| 不变量 | 含义 | 示例 |
|--------|------|------|
| **Unary Likely Inv** | 单一属性不变量 | `val != null`（所有索引块都有 start） |
| **Binary Likely Inv** | 二元属性不变量 | `IndexBlk.end == Column.last` |

**二进制似然不变量推断的时空局部性约束：**
- **时间局部性**：仅在同一个序列化窗口内的对象间推断（同一函数调用内）
- **空间局部性**：仅在有限深度（默认 5 层）内的数据成员间推断

#### 3.3.2 选择性子结构合取

**为什么需要子结构合取？** DFI bug 通常需要多个属性在**同一个数据实例**上同时成立。

例如，Cassandra-13939 仅当**同一个表**同时满足两个条件时才触发：
- 一个索引块在行边界结束
- 最后一列被删除

**频率基合取缩减（Frequency-based Conjunction Reduction）：**
1. 测量每个似然不变量的违反频率
2. 仅对**低频率**的不变量执行合取操作
3. 仅当合取包含排名前 5 的不变量时才报告

这避免了合取带来的指数级状态空间增长（"语料库爆炸"）。

### 3.4 升级跳过策略

UPFUZZ 使用概率模型决定是否跳过升级：

```
P_drop(d) = c · e^(-k·d)
```
其中 `c = 0.4`，`k = 0.3`，`d` 为变异深度

- 随机生成的测试（深度 0）有 40% 的概率跳过升级
- 深层变异的测试更可能被执行升级（更难生成，更有价值）
- **始终**对触发新反馈的测试执行升级

### 3.5 测试语料管理

划分三个子语料库，按概率选择种子：

| 语料库 | 选择概率 | 含义 |
|--------|---------|------|
| **Version Diff Corpus** | **70%** | 触发与版本差异相关不变量的测试 |
| **Format Corpus** | 20% | 触发新数据格式似然不变量的测试 |
| **Branch Coverage Corpus** | 10% | 触发新分支覆盖率的测试 |

> **关键设计**：优先探索与版本差异相关的数据格式空间（70%），而非新代码覆盖率（10%）。

---

## 4. 实验评估

### 4.1 实验设置

| 配置 | 详情 |
|------|------|
| **测试对象** | Cassandra（P2P 数据库）、HDFS（分布式文件系统）、HBase（主从数据库） |
| **升级模式** | 全停止升级（Full-stop upgrade） |
| **硬件** | x86_64, 192GB ECC DDR4, 2× Intel Xeon Silver（10核, 2.20GHz） |
| **部署** | Docker 容器集群：Cassandra 1 节点，HDFS/HBase 3 节点 |
| **运行时间** | 每次实验 24 小时，重复 3 次 |
| **基线** | UPFUZZ Baseline（传统代码覆盖率 + 先进 fuzzing 技术） |
| **消融维度** | DF（数据格式反馈）、VD（版本差异引导）、S（升级跳过） |

### 4.2 发现的 Bug

#### 15 个新 DFI Bug 概述

| 系统 | Bug 数量 | 后果 | 状态 |
|------|---------|------|------|
| **Cassandra** | 9 | 3 个集群崩溃，2 个数据损坏，2 个读取错误，2 个读取不一致 | 5 已确认，1 已修复，3 已报告 |
| **HDFS** | 3 | 1 个数据丢失，2 个读取不一致 | 2 已确认，1 已报告 |
| **HBase** | 3 | 3 个集群崩溃 | 1 已确认，2 已修复 |
| **总计** | **15** | **6 个集群崩溃，4 个数据丢失/损坏** | **8 已确认，3 已修复** |

#### 关键 Bug 案例

| Bug ID | 摘要 | 后果 | 触发时间 |
|--------|------|------|---------|
| CS-18105 | 截断的数据在升级后恢复 | 数据损坏 | 8.35h（全配置） |
| CS-18108 | 升级后数据丢失 | 数据损坏 | 5.14h（DF+VD+S） |
| CS-19590 | 升级时反序列化变异错误 | 崩溃 | 6.93h（DF+VD+S） |
| HD-16984 | 目录时间戳在升级后丢失 | 数据丢失 | 0.06h |
| HD-17219 | 升级后计数结果错误 | 读取不一致 | 4.82h（DF+VD+S） |
| HB-29021 | 升级后旧数据读取错误 | 读取不一致 | 0.33h（DF+VD+S） |

#### 其他额外发现的 Bug

| 类型 | 数量 | 说明 |
|------|------|------|
| 非 DFI 升级 bug | 7 | 逻辑错误或并发 bug |
| 单版本 bug | 16 | 仅在单版本中出现的 bug |
| **总计** | **38** | 其中 18 个已确认 |

### 4.3 Bug 检测效率

#### 与基线对比

| 指标 | Baseline | DF+S | DF+VD+S | 提升 |
|------|---------|------|---------|------|
| **触发 73% DFI bug** | 慢 | 较快 | **最快** | 全配置最优 |
| **仅由 TPState 触发** | — | — | **7/15 (47%)** | 基线完全无法发现 |
| **4.7× 加速** | — | — | ✓ | 达到相同状态数 |

#### 升级跳过效果（消融实验）

| 配置 | DFI bug 检测 | 说明 |
|------|-------------|------|
| Baseline（传统覆盖率） | 差 | 难以检测 DFI bug |
| Baseline + S（跳过） | 更差 | 无 TPState 引导时，跳过降低效率 |
| DF（数据格式反馈） | 较好 | 但状态空间大 |
| DF + VD（版本差异） | 更好 | 缩小焦点 |
| **DF + VD + S（全配置）** | **最佳** | 73% bug 检测加速 |

### 4.4 状态探索效率

| 指标 | Baseline | DF+S | DF+VD+S |
|------|---------|------|---------|
| **Cassandra 不同违规数** | 基准 | +9.1% | **+13.5%** |
| **HDFS 不同违规数** | 基准 | +7.6% | **+13.1%** |
| **HBase 不同违规数** | 基准 | +2.4% | **+6.1%** |
| **达 750 违规（Cassandra）** | ~24h | ~17.9h | **~4.8h（5× 加速）** |
| **达 650 违规（HDFS）** | ~24h | ~23.8h | **~5.6h（4.3× 加速）** |
| **达 1940 违规（HBase）** | ~24h | ~22.4h | **~5.0h（4.8× 加速）** |

### 4.5 性能开销

| 开销类型 | Cassandra | HDFS | HBase |
|---------|-----------|------|-------|
| **启动开销** | 0.08% | 11.47% | 0.08% |
| **命令执行开销** | 10.79% | 10.26% | **22.03%** |

> 端到端开销适中，因为命令执行仅占运行时的一小部分（大部分时间为系统启动和升级过程）。

### 4.6 误报率

| 指标 | 值 |
|------|----|
| **误报率** | **< 1%** |
| 主要原因 | 非确定性输出（如 SCAN 的执行时间）、版本相关输出格式 |
| 缓解措施 | 输出归一化（掩盖非确定性字段、标准化版本特定格式） |

### 4.7 与其他工具对比

| 对比对象 | DFI bug 检测能力 | 说明 |
|---------|----------------|------|
| **DUPTester (SOTA)** | 差 | 仅检测 4 个已知 DFI bug，无法发现任何新 bug |
| **UPFUZZ Baseline** | 有限 | 仅检测部分 DFI bug |
| **UPFUZZ 全配置** | **15 个新 DFI bug** | 8 已确认，3 已修复 |

---

## 5. 系统实现

### 5.1 架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                     UPFUZZ 架构                              │
├─────────────────────────────────────────────────────────────┤
│  测试生成阶段 (~9k LoC)                                      │
│  ├─ 系统配置生成器                                           │
│  └─ 集成测试输入生成器（语法感知变异）                        │
├─────────────────────────────────────────────────────────────┤
│  旧版本执行阶段 (~7k LoC)                                    │
│  ├─ TPState 分析引擎                                         │
│  │  ├─ 选择性监控点插桩                                      │
│  │  ├─ 似然不变量推断                                       │
│  │  └─ 结构合取分析                                         │
│  └─ 反馈收集（分支覆盖率 + 格式不变量）                       │
├─────────────────────────────────────────────────────────────┤
│  升级决策                                                   │
│  └─ 概率跳过模型（基于变异深度）                              │
├─────────────────────────────────────────────────────────────┤
│  升级测试执行阶段 (~2k LoC)                                  │
│  └─ 故障检测                                                │
│     ├─ 崩溃/异常/错误日志捕获                                │
│     ├─ 差分 Oracle（版本间结果对比）                          │
│     └─ 故障分组（相似日志聚类）                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 关键技术实现

| 组件 | 技术 | 代码量 |
|------|------|--------|
| **测试生成** | 语法感知输入变异 + 配置变异 + 有状态 fuzzing | ~9k LoC |
| **TPState 分析** | 基于 Soot 编译框架的静态数据流分析 | ~7k LoC |
| **升级执行** | 集群容器化部署 + 升级脚本 | ~2k LoC |
| **故障检测** | 差分 Oracle + 错误日志聚类 | 内置 |

### 5.3 应用扩展到新系统

将 UPFUZZ 应用于新系统需要：

1. **开发系统特定的输入/配置生成器和升级脚本**（与 Syzkaller 等 fuzzer 类似）
2. **适配 UPFUZZ 的数据流分析**：对使用反射或动态类加载的系统，需补充轻量级注解：
   - 系统特定的 I/O sink API
   - 少量难以解析的调用图边

> 典型集成工作量：**一名研究生约两周**。

---

## 6. 学习要点与启示

### 6.1 核心启示

1. **DFI bug 是分布式系统升级中最隐蔽但最危险的 bug 类**：占升级故障的近三分之二，后果严重（数据丢失、集群崩溃），但传统测试方法几乎无法发现。

2. **TPState 分析是有效的突破口**：通过关注那些"格式在版本间发生变更且最终会被持久化"的状态，UPFUZZ 将巨大的状态空间压缩到可控范围。

3. **版本差异引导 + 数据格式反馈 + 升级跳过"三合一"效果最佳**：消融实验证明三者缺一不可，全配置可获得 4.7× 加速。

4. **似然不变量推断经过精心裁剪才能实际可用**：选择性监控、选择性属性推断、频率基合取缩减三项设计共同控制了状态爆炸。

5. **差分 Oracle 在存储系统测试中极为有效**：< 1% 的误报率说明，跨版本行为保留是存储系统的核心语义，版本间行为差异几乎一定是 bug。

### 6.2 对分布式文件系统的启示（HDFS 相关）

| 启示 | 对 HDFS 的意义 |
|------|---------------|
| **升级测试自动化** | HDFS 从 2.10 升级到 3.3 需 145 秒，UPFUZZ 发现 3 个新 DFI bug（HD-16984, HD-17219, HD-17686） |
| **TPState 分析** | HDFS 的 FsImage、EditLog、DataNode 存储等持久化状态是 DFI bug 的高发区 |
| **版本差异关注点** | HDFS 类的成员变更（如 Directory 的 timestamp 字段）可能导致升级后数据丢失 |
| **差分 Oracle** | HDFS 的读写结果在版本间应保持一致，差异即 bug |
| **升级跳过策略** | HDFS 升级耗时最长（145s），更需要精确的测试选择 |

### 6.3 发现的典型 HDFS DFI Bug

| Bug | 后果 | 描述 |
|-----|------|------|
| **HD-16984** | 数据丢失 | 目录时间戳在升级后丢失 |
| **HD-17219** | 读取不一致 | 升级后计数结果错误 |
| **HD-17686** | 数据丢失 | 升级后 stat 结果错误 |

### 6.4 局限性与未来方向

- **适用范围**：当前主要针对 Java 实现的分布式存储系统（Cassandra、HBase、HDFS）
- **动态特性**：对动态类加载和反射的支持需要补充注解
- **升级模式**：目前仅支持全停止升级，滚动升级（rolling upgrade）待扩展
- **假阴性**：并非所有 DFI bug 都能被捕获（部分 bug 需确定性序列化条件）

---

## 7. 结论

UpFuzz 是**首个专门针对分布式存储系统升级过程中数据格式不兼容 bug 的自动化 fuzzing 工具**。其核心创新在于：

1. **TPState 分析**：将巨大的状态空间聚焦到"格式在版本间变更且会被持久化"的状态子集
2. **选择性属性推断**：基于真实 bug 研究推导的 5 类数据格式属性模板
3. **频率基合取缩减**：使结构合取在工程上可行

在实际系统中发现了 **15 个新 DFI bug**（8 已确认）、**7 个额外升级 bug** 和 **16 个单版本 bug**，合计 **38 个新 bug**，获 **NSDI '26 Community Award**。

UPFUZZ 已在 GitHub 完全开源：**https://github.com/zlab-purdue/upfuzz**

---

## 8. 参考文献

1. UpFuzz: Detecting Data Format Incompatibility Bugs during Distributed Storage System Upgrade. *NSDI '26*
2. DUPTester - 先前工作，分布式系统升级测试（触发约 1/3 的已知升级故障）
3. AFL / Syzkaller - 传统 fuzzing 工具
4. Cassandra-13939 - 论文的贯穿案例
5. Google Cloud 2025 故障（影响 140 万用户）
6. CrowdStrike 2024 全球宕机（850 万系统崩溃）

---

---

## 附录：开源代码

UPFUZZ 是完全开源的，代码仓库托管在 GitHub：

> **https://github.com/zlab-purdue/upfuzz**

该仓库包含：
- 完整的 UPFUZZ fuzzing 引擎源代码（~18k LoC）
- 测试脚本和配置示例（Cassandra、HDFS、HBase）
- 论文实验的可复现数据
- 发现的 bug 清单与对应 JIRA 链接

*翻译解析日期：2026-06-28*

---

### 附：Cassandra-13939 贯穿示例详解

该 bug 是论文用于说明 TPState 分析效果的贯穿案例：

1. **测试场景**：在 Cassandra 2.2 中创建一个表（复合主键），插入两行（同一 k1），删除最后一列，读取第一行
2. **升级**：升级到 Cassandra 3.0（行式布局 → 列式布局）
3. **Bug 触发**：由于 `last_ptr` 未在删除列时更新，第一行后的块边界检查失败，导致错误地继续读取第二行
4. **根因**：Cassandra 3.0 反序列化代码在遇到被删除的 cell 时跳过了 `last_ptr` 的更新
5. **修复**：明确在计算 `last_ptr` 时考虑被删除的 cell

```
Table 内存表示:

旧版本磁盘格式 (Cell-based):
┌─────┬─────┬─────┬─────┐
│k1,v1│k1,v2│k2,v1│k2,v2│  ← 每个 cell 独立存储
└─────┴─────┴─────┴─────┘

新版本期望格式 (Row-based):
┌─────────┬─────────┐
│ k1,v1   │ k2,v1   │  ← 每行统一存储
│ k1,v2   │ k2,v2   │
└─────────┴─────────┘

Bug 条件:
① 行在索引块边界结束
② 最后一列被删除
→ 反序列化时 last_ptr 未更新
→ 下一行被错误包含
```

这一示例完美展示了为什么传统的分支覆盖率和状态覆盖率都无法捕获 DFI bug——触发条件涉及跨数据结构的引用关系，不被任何基本块或变量直接体现。
