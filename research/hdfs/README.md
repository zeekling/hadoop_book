# HDFS 与小文件问题 — 近5年研究综述

> 检索时间：2026年7月 | 覆盖范围：2021-2026

## 一、问题概述

HDFS（Hadoop Distributed File System）的小文件问题是指：当存储大量远小于 HDFS 默认块大小（128MB）的文件时，每个文件都需要在 NameNode 内存中维护一份元数据（约150字节），导致 NameNode 内存耗尽、MapReduce 任务数量激增、磁盘寻道时间膨胀等性能问题。近5年围绕这一问题的研究主要集中在 **文件合并策略**、**归档文件优化**、**缓存加速**、**聚类/相关性感知合并** 等方向。

## 二、论文摘要表（按年份排序，共16篇核心论文）

### 综述类（3篇）

| # | 标题 | 年份 | Venue | 引用 | PDF | 核心贡献 |
|---|------|------|-------|:----:|:---:|---------|
| 1 | **Small files' problem in Hadoop: A systematic literature review** — Aggarwal, Verma, Siwach | 2022 | J. King Saud Univ. - Computer and Information Sciences [SCI] | ~30+ | ✓ Open Access | 最全面的 SLR，定义了 Hadoop 生态系统与小文件问题的分类体系，批判性分析了四类解决方案（合并、缓存、集群优化、Map 任务优化） |
| 2 | **Access optimization for small files in HDFS: Challenges and Solutions** — Li et al. | 2024 | ICICML 2024 [EI] | 新 | ✓ IEEE | 最新综述，总结了 HDFS 内置方案（HAR/SequenceFile）和用户自定义方案 |
| 3 | **Critical Analysis of Solutions to Hadoop Small File Problem** — IOSR | 2023 | IOSR-JCE | — | ✓ 免费 | 从文件合并、缓存、Hadoop 集群结构优化、Map 任务优化四个维度做对比分析 |

### 归档/索引优化类（3篇）

| # | 标题 | 年份 | Venue | 引用 | PDF | 核心贡献 |
|---|------|------|-------|:----:|:---:|---------|
| 4 | **Hadoop Perfect File (HPF): A fast and memory-efficient metadata access archive file** — Zhai, Tchaye-Kondi, Lin et al. | 2021 | J. Parallel and Distributed Computing [CCF-B] | **~29** | ✓ Elsevier | **代表作**。提出 HPF 归档文件，使用**双重哈希**（动态哈希+保序完美哈希）实现元数据直接访问，无需扫描整个索引文件。访问速度比原生 HDFS 快 40%+，比 HAR 快 ~11294% |
| 5 | **A Novel Technique for Handling Small File Problem of HDFS: HBAF (Hash Based Archive File)** — IOS Press | 2022 | Recent Trends in Intensive Computing | — | ✓ | 基于哈希的归档方案，无需缓存机制即可直接访问小文件元数据 |
| 6 | **Performance Study on Indexing and Accessing of Small File in HDFS** — JIKM | 2021 | J. Info. Knowledge Management | — | — | 研究了 HAR、New HAR、CombineFileInputFormat、SequenceFile 等技术的索引和访问效率 |

### 文件合并/聚类算法类（5篇）

| # | 标题 | 年份 | Venue | 引用 | PDF | 核心贡献 |
|---|------|------|-------|:----:|:---:|---------|
| 7 | **Enhanced Best Fit Algorithm for Merging Small Files** — Ali, Mirza, Ishak | 2023 | Computer Systems Science & Engineering [SCI] | — | ✓ Open | 提出**增强型最佳适应（Best Fit）合并算法**，解决小文件合并后触及块大小上限的死锁问题 |
| 8 | **Combining Similarity-Based Correlation and Hierarchical Ascending Clustering for Small Files Problem in HDFS** — Chettaoui & Hkiri | 2025 | AINA 2025 [CCF-C] | 新 | ✓ Springer | **最新研究**。基于相似度相关性 + 层次聚类合并小文件，聚类感知的文件合并策略 |
| 9 | **Consolidating industrial small files using robust graph clustering** — Wang et al. | 2023 | IEEE Trans. Network Science & Engineering [SCI] | — | ✓ IEEE | 面向工业场景的**鲁棒图聚类**合并方法 |
| 10 | **Performance Evaluation of Merging Techniques for Handling Small Size Files in HDFS** — Sharma & Barwar | 2021 | LNEDCT, Springer | 3 | ✓ Springer | 对 Sequential File、CombineFileInputFormat、HAR、Hadoop Streaming 四种合并技术做了系统性能评估，结论：HAR 和 CombineFileInputFormat 表现最稳定 |
| 11 | **Correlation aware solution for merging small files** (DM-SFS) — Springer | 2025 | J. Supercomputing | 新 | ✓ Springer | 基于分区策略（文件类型+文件大小）的 **Next-Fit 分配算法** 实现相关性感知合并 |

### 缓存/访问优化类（3篇）

| # | 标题 | 年份 | Venue | 引用 | PDF | 核心贡献 |
|---|------|------|-------|:----:|:---:|---------|
| 12 | **Small files access efficiency in HDFS: a case study on British Library text files (VFS-HDFS)** — Alange & Sagar | 2023 | Cluster Computing [CCF-C] | ~4 | ✓ Springer | 引入**虚拟文件系统层** + 缓存 + 分类分桶元数据表，在不改 HDFS 架构前提下提升访问效率。以大英图书馆数据集验证 |
| 13 | **Small Files in HDFS and Their Impact on Hadoop Performance** — Koren et al. | 2021 | iiWAS2021 [EI] | ~1 | ✓ ACM | 实验分析碎片化对批处理性能的影响，指出通过应用设计和配置优化可在不大幅改动 HDFS 的情况下规避多数问题 |
| 14 | **Merging and cooperative caching based HDFS small files access optimization** — IEEE | 2024 | IEEE Conf. | 新 | ✓ IEEE | 合并 + 协同缓存的双重策略优化小文件访问性能 |

### 其他相关（2篇）

| # | 标题 | 年份 | Venue | 引用 | PDF | 核心贡献 |
|---|------|------|-------|:----:|:---:|---------|
| 15 | **Optimizing HDFS Storage and Managing TTL for Unused Hive Tables** — Mantri | 2023 | JAIMLD | ~1 | ✓ | 压缩技术 + 文件格式优化 + 分区策略，LinkedIn/Spotify/Netflix 案例经验 |
| 16 | **Chaotic Leader Particle Swarm Optimization (CL-PSO) for efficient data storage in cloud-based HDFS** — 2024 | 2024 | ResearchGate | 新 | — | 基于粒子群优化算法，让一个 block 存储多个小文件并让 DataNode 保存部分元数据 |

## 三、研究趋势分析

### 技术演进路线（2021→2025）

```
2021                   2022                  2023                  2024-2025
├─ HPF (双重哈希归档)    ├─ SLR (Aggarwal)     ├─ VFS-HDFS (虚拟FS层)  ├─ 聚类合并 (AINA 2025)
├─ 性能评估 (Sharma)    ├─ HBAF (哈希归档)     ├─ Best Fit 合并         ├─ CL-PSO 优化
├─ iiWAS 实验分析       └─ 索引研究 (JIKM)     ├─ 图聚类合并 (IEEE)    ├─ 协同缓存 + 合并
                                              ├─ DM-SFS 相关性合并   └─ Li 综述 (ICICML)
                                              └─ 关键分析 (IOSR)
```

### 研究方法分布

| 方法类别 | 论文数 | 代表工作 |
|---------|:------:|---------|
| **归档/索引优化** | 3 | HPF (2021)，HBAF (2022)，JIKM (2021) |
| **文件合并算法** | 5 | Best Fit (2023)，AINA 聚类 (2025)，图聚类 (2023)，DM-SFS (2025)，性能评估 (2021) |
| **缓存/虚拟化** | 2 | VFS-HDFS (2023)，协同缓存 (2024) |
| **智能优化** | 1 | CL-PSO (2024) |
| **综述** | 3 | Aggarwal SLR (2022)，Li 综述 (2024)，IOSR (2023) |

### 关键发现

1. **HPF（2021）是目前引用最高的单篇工作（~29 引）**，其双重哈希索引设计被后续多篇论文引用，是该领域的标杆
2. **2023 年是小文件研究的爆发年**：Best Fit、VFS-HDFS、图聚类、Cluster Computing 等多篇高质量工作集中发表
3. **2024-2025 年趋势**：从单一合并策略向 **聚类/机器学习辅助合并** 演进（AINA 2025 的层次聚类、CL-PSO 优化、DM-SFS 相关性感知），表明 AI 驱动的存储优化正在渗透该领域
4. **综述类持续更新**（2022→2024 两篇 SLR），说明该问题仍在活跃研究中，远未完全解决
5. **Hadoop 3.x 的纠删码和联邦架构**未直接解决小文件问题，这是独立研究方向

### 必读 Top 5 推荐

| 排名 | 论文 | 理由 |
|:---:|------|------|
| 1️⃣ | **HPF (Zhai et al., 2021)** — JPDC | 该领域最高引代表作，哈希归档的最新技术水平 |
| 2️⃣ | **SLR (Aggarwal et al., 2022)** — JKSU-CIS | 最全面的系统文献综述，含完整分类体系 |
| 3️⃣ | **Enhanced Best Fit (Ali et al., 2023)** — CSSE | 解决了合并触及块大小上限的实际工程问题 |
| 4️⃣ | **AINA 2025 Clustering (Chettaoui, 2025)** | 反映最新聚类+合并趋势 |
| 5️⃣ | **VFS-HDFS (Alange & Sagar, 2023)** — Cluster Computing | 缓存+虚拟 FS 层的代表性工程方案 |

---

**说明**：引用数基于搜索时的观测值（Semantic Scholar 因限流无法获取精确值），标注 `~` 的为近似估算值。部分 2024-2025 新论文引用数较低属正常现象。
