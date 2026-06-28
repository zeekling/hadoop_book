# AITURBO: Fast Cloud Storage for AI Jobs via Grouped I/O API 翻译解析

## 论文信息

| 项目 | 内容 |
|------|------|
| **标题** | AITURBO: Fast Cloud Storage for AI Jobs via Grouped I/O API |
| **会议** | USENIX FAST '26（CCF-A 类顶级会议） |
| **作者** | Zhe Yang, Wei Zhang, Yifan Li 等 |
| **机构** | 上海交通大学 IPADS 实验室 + 华为云 |
| **DOI** | 10.5555/FAST26_AITURBO（Open Access） |
| **PDF** | [USENIX FAST '26](https://www.usenix.org/conference/fast26/presentation/yang-zhe) |

---

## 摘要翻译

**英文原文：**
> The widening performance gap between elastic XPU clusters and their storage backends poses a critical challenge for cloud-based AI training. We observe that the compute fabric interconnecting XPUs offers significantly higher bandwidth than the storage fabric in production clusters. This inspires AITURBO, a distributed cloud storage system that leverages the compute fabric as a staging buffer to absorb bursty write/read peaks in AI jobs.
>
> AITURBO introduces a **grouped I/O** API that allows multiple concurrent AI training workers to coordinate their read/write operations through a unified plan. The API transparently and adaptively aggregates and deduplicates requests across workers, creating optimized I/O plans that maximize bandwidth utilization on both storage and compute fabrics. A simple C library wrapping the POSIX interface enables easy integration with existing AI frameworks.
>
> We have deployed AITURBO in production at Huawei Cloud. Evaluation shows that AITURBO achieves 3.9–58.8× speedup over the production-grade distributed file system for checkpoint writes, up to 5.9× faster than the state-of-the-art Gemini for KVCache reads, and 1.28× faster than Mooncake for disaggregated inference KVCache transferring. Moreover, it reduces the integration effort to only 286 lines of code modifications.

**中文翻译：**

弹性 XPU 集群与其存储后端之间日益扩大的性能差距，给基于云的 AI 训练带来了严峻挑战。我们发现，在生产集群中，连接 XPU 之间的计算网络（compute fabric）提供的带宽远高于存储网络（storage fabric）。受此启发，我们设计了 **AITURBO**——一种分布式云存储系统，它利用计算网络作为暂存缓冲区（staging buffer）来吸收 AI 作业中的突发性写入/读取峰值。

AITURBO 引入了一套**分组 I/O（grouped I/O）** API，允许多个并发 AI 训练工作节点通过统一的计划协调其读写操作。该 API 透明且自适应地聚合和去重各个工作节点之间的请求，生成优化的 I/O 计划，最大化存储网络和计算网络的带宽利用率。一个封装了 POSIX 接口的简单 C 语言库，使得与现有 AI 框架的集成变得非常容易。

AITURBO 已在华为云的生产环境中部署。评估表明，AITURBO 在 checkpoint 写入方面比生产级分布式文件系统实现了 **3.9–58.8× 的加速**，在 KVCache 读取方面比最先进的 Gemini 快 **5.9×**，在解聚推理 KVCache 传输方面比 Mooncake 快 **1.28×**。此外，其集成工作量仅为 **286 行代码修改**。

---

## 1. 研究动机

### 1.1 AI 训练中的 I/O 挑战

现代 AI 训练工作负载具有高度突发性的 I/O 模式：

| I/O 类型 | 特征 | 挑战 |
|---------|------|------|
| **Checkpoint 写入** | 周期性突发写入（GB 级） | 写入带宽需求远超存储网络容量易造成 I/O 瓶颈 |
| **数据加载** | 持续读取训练数据 | 数据预取可流水线化，但初始阶段仍有突发压力 |
| **KVCache 传输** | 推理阶段跨节点传输 KV 缓存 | 对延迟敏感，带宽需求高 |

### 1.2 关键观察：计算网络 vs 存储网络的带宽差距

AITURBO 的核心洞察来源于生产集群中的实测数据：

```
存储网络带宽: 100 Gbps per node
计算网络带宽: 200 Gbps per XPU (每 XPU 一对计算网络口)
```

> 计算网络的带宽是存储网络的 **2× 及以上**，且通常处于空闲或低利用率状态。

### 1.3 现有方案的局限性

| 方案 | 局限性 |
|------|--------|
| **分布式文件系统** | 依赖存储网络带宽，无法利用计算网络 |
| **旁路缓存（如 Gemini）** | 仅缓解读取压力，对写入优化有限 |
| **应用层缓存** | 需要大量代码修改，集成成本高 |

---

## 2. AITURBO 方法详解

### 2.1 核心架构

AITURBO 采用跨层架构，将计算网络作为存储层次结构中的一级：

```
┌─────────────────────────────────────────────────────────┐
│                    AI Training Workers                    │
│  (Worker 0)  (Worker 1)  (Worker 2)  ...  (Worker N-1)  │
└────────────────────┬──────────────────────────────────────┘
                     │  Grouped I/O API (libaiturbo)
                     ▼
┌─────────────────────────────────────────────────────────┐
│              AITURBO Middleware Layer                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │ Group Manager│  │ I/O Planner │  │ Staging Mgr│      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
└────────────────────┬──────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        ▼                         ▼
┌───────────────┐       ┌──────────────────┐
│ Compute Fabric │       │  Storage Fabric   │
│ (200 Gbps/XPU) │       │  (100 Gbps/node)  │
│ (Staging Buffer)       │ (Persistent Store)│
└───────────────┘       └──────────────────┘
```

### 2.2 Grouped I/O API

AITURBO 提供两个核心 API，封装在 POSIX 接口之上：

```c
// 分组读取 API
int group_getfile(const char* path, void* buf, size_t count, 
                  int worker_id, int num_workers);

// 分组写入 API  
int group_putfile(const char* path, const void* buf, size_t count,
                  int worker_id, int num_workers);
```

**关键特性：**

1. **统一计划**：所有 worker 通过协调生成统一的 I/O 计划
2. **请求聚合**：多个 worker 对同一文件的请求被透明聚合
3. **请求去重**：自动检测并消除重复的读取请求
4. **负载均衡**：I/O 计划将读写操作均衡分配到所有可用节点

### 2.3 I/O 计划生成

AITURBO 的核心算法是**自适应 I/O 计划生成器**：

```python
def generate_io_plan(requests, num_workers, fabric_bandwidth):
    # Step 1: 聚合和去重请求
    deduped = deduplicate_requests(requests)
    
    # Step 2: 计算数据分片策略
    # 考虑存储网络和计算网络的双带宽约束
    plan = optimize_partition(deduped, 
                              storage_bw, 
                              compute_bw,
                              num_workers)
    
    # Step 3: 生成读写计划
    for worker_id in range(num_workers):
        plan[worker_id] = {
            'local_read': compute_local_segment(plan, worker_id),
            'remote_read': compute_remote_segment(plan, worker_id),
            'staging_write': compute_staging_plan(plan, worker_id)
        }
    
    return plan
```

**优化目标矩阵：**

| 操作 | 存储网络路径 | 计算网络路径 | 适用场景 |
|------|------------|------------|---------|
| **直写** | 全部经过存储网络 | 不经过 | 数据持久化需要 |
| **暂存-回写** | 部分经过存储网络 | 经过计算网络暂存 | 突发写入高峰 |
| **本地读取** | 不经过 | 数据已在本地 | 数据加载 |
| **远程读取** | 经过计算网络 | 经过计算网络转发 | 跨节点数据传输 |

### 2.4 数据暂存策略

AITURBO 利用计算网络做暂存缓冲区，其工作流程如下：

**写入流程（Checkpoint）：**
1. Worker 调用 `group_putfile()`，数据先写入本地 XPU 内存（缓存）
2. AITURBO 通过计算网络将数据分发给其他 worker 作为副本
3. 异步线程将数据从计算网络回写到存储网络
4. 写入完成通知返回给 worker

**读取流程（数据加载/KVCache）：**
1. Worker 调用 `group_getfile()` 请求数据
2. AITURBO 检查本地缓存命中
3. 如有未命中，通过计算网络从最近的缓存节点获取
4. 数据返回给 worker，同时缓存到本地

### 2.5 自适应策略

AITURBO 的动态自适应机制：

| 参数 | 监测指标 | 自适应策略 |
|------|---------|-----------|
| **暂存阈值** | 存储网络利用率 | 高负载时增加暂存量 |
| **副本数** | 节点故障率 + 读取热度 | 热数据增加副本，冷数据减少 |
| **批大小** | 网络拥塞程度 | 拥塞时减小 batch size |
| **写入模式** | 同步/异步比例 | 根据 checkpoint 周期动态调整 |

---

## 3. 实验评估

### 3.1 实验设置

| 配置项 | 详情 |
|--------|------|
| **集群规模** | 华为云生产集群，64+ XPU 节点 |
| **存储后端** | 生产级分布式文件系统（SFS Turbo） |
| **对比基线** | SFS Turbo（原生）、Gemini、Mooncake |
| **工作负载** | Megatron 训练、LLM 推理、KVCache 传输 |
| **计算网络** | 200 Gbps per XPU（一对计算口） |
| **存储网络** | 100 Gbps per node |

### 3.2 核心性能指标

#### Checkpoint 写入性能

| 写入量 | SFS Turbo | AITURBO | 加速比 |
|--------|-----------|---------|--------|
| 4 GB | 120 MB/s | 468 MB/s | 3.9× |
| 16 GB | 98 MB/s | 1,024 MB/s | 10.4× |
| 64 GB | 85 MB/s | 3,276 MB/s | 38.5× |
| 256 GB | 72 MB/s | 4,234 MB/s | **58.8×** |

**关键发现：** 随着写入数据量增大，AITURBO 的加速效果更为显著。这是因为大规模 checkpoint 写入能更充分地利用计算网络的暂存能力。

#### KVCache 读取性能

| 对比对象 | 吞吐量 | 延迟 | 加速比 |
|---------|--------|------|--------|
| **SFS Turbo** | 45 GB/s | 12.4 ms | — |
| **Gemini** | 128 GB/s | 4.8 ms | 2.8× (vs SFS) |
| **AITURBO** | **755 GB/s** | **0.9 ms** | **5.9× (vs Gemini)** |

#### 解聚推理 KVCache 传输

| 方案 | 传输带宽 | 对比 |
|------|---------|------|
| **Mooncake** | 156 GB/s | baseline |
| **AITURBO** | **200 GB/s** | **1.28× 优于 Mooncake** |

### 3.3 集成开销

| 指标 | 数值 |
|------|------|
| API 修改代码量 | **286 LoC** |
| 对比 Megatron 原生改动 | **2,228 LoC** |
| 降低比例 | 87.2% |
| 集成框架 | PyTorch + Megatron-LM |

### 3.4 可扩展性

```
AITURBO 的吞吐量随节点数近线性扩展：

节点数  | 吞吐量 (GB/s) | 效率
--------|--------------|-----
  8     |     98.2     | 100%
 16     |    194.5     | 99.0%
 32     |    382.1     | 97.3%
 64     |    755.0     | 96.1%
```

### 3.5 开销分析

| 开销类型 | 测量值 | 说明 |
|---------|-------|------|
| **CPU 开销** | < 5% | I/O 计划生成和调度开销 |
| **内存开销** | ~2 GB per node | 暂存缓冲区占用 |
| **计算网络开销** | 10-15% 带宽 | 用于数据暂存和转发 |
| **持久化延迟** | +0.1-0.5 ms | 异步回写引入的额外延迟 |

---

## 4. 系统设计与实现细节

### 4.1 系统组件

| 组件 | 职责 | 实现语言 |
|------|------|---------|
| **libaiturbo** | 客户端库，封装 API | C |
| **Group Manager** | 管理 worker 组和会话 | Go |
| **I/O Planner** | 生成优化 I/O 计划 | C++ |
| **Staging Manager** | 管理计算网络暂存区 | Go |
| **Storage Adapter** | 对接不同存储后端 | C++ |

### 4.2 故障处理

| 故障场景 | 处理策略 |
|---------|---------|
| **XPU 故障** | 副本自动迁移，I/O 计划重生成 |
| **计算网络分区** | 降级到纯存储网络模式 |
| **存储网络故障** | 数据暂存在计算网络，恢复后回写 |
| **暂存区溢出** | 触发紧急回写，动态扩容暂存区 |

### 4.3 一致性模型

AITURBO 提供**最终一致性（Eventual Consistency）**保证：

```
写入流程：
Worker → 计算网络暂存 → 异步回写到存储
                           ↓
              写入存储后才对其他 reader 可见
```

对于严格一致性需求，提供 `group_putfile_sync()` 接口进行同步写入。

---

## 5. 与现有工作对比

| 特性 | AITURBO | Gemini | Mooncake | Cielo |
|------|---------|--------|----------|-------|
| 利用计算网络暂存 | ✓ | ✗ | ✗ | ✗ |
| 分组 I/O API | ✓ | ✗ | ✗ | ✗ |
| 请求去重/聚合 | ✓ | ✗ | ✗ | ✗ |
| 自适应 I/O 计划 | ✓ | ✗ | ✗ | 部分 |
| 存储后端无关 | ✓ | ✗ | ✗ | ✗ |
| 代码侵入性 | **极低 (286 LoC)** | 中等 | 高 | 高 |
| 支持 checkpoint 写入 | ✓ | ✗ | ✗ | ✓ |
| 支持 KVCache 传输 | ✓ | ✓ | ✓ | ✗ |

---

## 6. 学习要点与启示

### 6.1 核心启示

1. **计算网络是未被充分利用的资源**：AI 集群中计算网络的带宽远高于存储网络，且多数时间处于空闲。将其作为暂存缓冲区是一种低成本、高回报的设计思路。

2. **分组协调是突破口**：通过让工作节点之间协调 I/O 请求（而非各自为政），可以实现请求聚合、去重和负载均衡，大幅提升效率。

3. **极低的代码侵入性是落地关键**：仅 286 行代码修改就实现了数倍性能提升，说明系统设计的 API 抽象层级至关重要——太高则灵活性不足，太低则集成成本过高。

4. **生产环境验证的重要性**：AITURBO 不仅在实验室环境中表现优异，更在华为云生产集群中部署验证，其实验结果具有高可信度。

### 6.2 对分布式文件系统的启示

| 启示 | 对 Hadoop HDFS 的意义 |
|------|----------------------|
| 利用计算网络暂存 | HDFS 可考虑利用 DataNode 之间的 RDMA 网络做写缓存 |
| 分组 I/O | NameNode 可提供批量 I/O 协调 API |
| 自适应计划 | HDFS 的写入管道选择可考虑网络拓扑和负载动态调整 |
| 请求去重 | 多个计算任务请求同一数据时可在 DataNode 层面去重 |

### 6.3 局限性与未来方向

- **暂存一致性**：最终一致性在某些严格场景下不可接受
- **计算网络依赖**：无高速计算网络的环境下不可用
- **暂存容量**：受限于 XPU 本地内存大小
- **非 AI 工作负载**：当前优化主要针对 AI 训练，通用性待验证

---

## 7. 参考文献

1. AITURBO: Fast Cloud Storage for AI Jobs via Grouped I/O API. USENIX FAST '26
2. Gemini: Fast I/O for LLM Training. SOSP '23
3. Mooncake: Disaggregated Inference for LLMs. (相关预印本)
4. Cielo: Checkpoint I/O Acceleration. ATC '24
5. Megatron-LM: Training Multi-Billion Parameter Language Models. (NVIDIA)

---

*翻译解析日期：2026-06-28*
