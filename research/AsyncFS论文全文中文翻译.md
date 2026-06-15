# AsyncFS（SwitchFS）论文全文中文翻译

## SwitchFS: Asynchronous Metadata Updates for Distributed Filesystems with In-Network Coordination

**EuroSys 2026 · 上海交通大学 IPADS 实验室（陈海波团队）**

---

# SwitchFS: Asynchronous Metadata Updates for Distributed Filesystems with In-Network Coordination 中文翻译

> 原文：SwitchFS: Asynchronous Metadata Updates for Distributed Filesystems with In-Network Coordination
> 作者：Jingwei Xu, Mingkai Dong, Qiulin Tian, Ziyi Tian, Tong Xin, Haibo Chen
> 机构：上海交通大学 IPADS 实验室
> 会议：EuroSys 2026
>
> **翻译章节**：摘要 (Abstract)、§1 引言 (Introduction)、§2 背景 (Background)、§3 动机与挑战 (Motivation and Challenges)、§4 总览 (Overview)、§5.1~§5.2 设计 (Design)

---

## 摘要 (Abstract)

分布式文件系统的元数据更新通常是同步的。这给访问效率、负载均衡和目录竞争带来了固有的挑战，尤其是在动态且偏斜的工作负载下。本文认为同步更新过于保守。我们提出了 SwitchFS，采用异步元数据更新（asynchronous metadata updates），允许操作提前返回并将目录更新推迟到读取时，从而既隐藏延迟又分摊开销。关键挑战在于高效地维护元数据更新的同步 POSIX 语义。为解决这一问题，SwitchFS 与可编程交换机（programmable switch）协同设计，利用有限的交换机资源以可忽略的开销跟踪目录状态。这使得 SwitchFS 能够在目录读取之前通过批处理（batching）和合并（consolidation）高效地聚合和应用延迟的更新。评估表明，在偏斜工作负载下，与两个最先进的分布式文件系统 Emulated-InfiniFS 和 Emulated-CFS 相比，SwitchFS 的吞吐量分别提升高达 13.34 倍和 3.85 倍，延迟分别降低 61.6% 和 57.3%。对于真实工作负载，相比 CephFS、Emulated-InfiniFS 和 Emulated-CFS，SwitchFS 的端到端吞吐量分别提升了 21.1 倍、1.1 倍和 0.3 倍。

**CCS 概念**：• 信息系统 → 分布式存储；• 社会与专业议题 → 文件系统管理；• 网络 → 网内处理（In-network processing）；可编程网络（Programmable networks）。

**关键词**：分布式文件系统，元数据管理，可编程交换机

**ACM 引用格式**：
Jingwei Xu, Mingkai Dong, Qiulin Tian, Ziyi Tian, Tong Xin, and Haibo Chen, Institute of Parallel and Distributed Systems, Shanghai Jiao Tong University. 2026. SwitchFS: Asynchronous Metadata Updates for Distributed Filesystems with In-Network Coordination. In *21st European Conference on Computer Systems (EUROSYS '26), April 27–30, 2026, Edinburgh, Scotland Uk*. ACM, New York, NY, USA, 20 pages. https://doi.org/10.1145/3767295.3769349

本作品采用 Creative Commons Attribution 4.0 International License 许可协议。
EUROSYS '26, Edinburgh, Scotland Uk
© 2026 Copyright held by the owner/author(s).
ACM ISBN 979-8-4007-2212-7/26/04
https://doi.org/10.1145/3767295.3769349

---

## 1 引言 (Introduction)

元数据性能是部署在现代数据中心中的分布式文件系统 (DFS) 效率的关键因素。为管理文件系统命名空间和文件属性，现代 DFS [15, 38, 45, 51, 59, 66, 75, 76] 通常采用专用的元数据服务。在访问文件数据之前，客户端必须首先联系该服务以检查权限并获取数据位置。由于数据中心中的文件通常很小 [3, 5, 17, 38, 61, 74, 75, 81, 82]，元数据操作的相对开销变得显著。例如，来自百度 AI 云的追踪数据 [75] 报告称，元数据操作占所有文件系统请求的 67%–96%，并主导了 I/O 完成时间。与数据访问带宽（随数据服务器数量近似线性扩展）不同，元数据吞吐量由于层次依赖关系 [38, 45, 59, 75] 而难以扩展。因此，元数据经常成为性能瓶颈。在深度学习模型训练 [34, 77] 和数据分析 [34] 等大规模应用中，这一瓶颈可能导致数据带宽和计算资源利用不足，最终降低端到端性能 [34, 74, 77]。

一系列研究 [32, 38, 45, 47, 51, 54, 55, 57, 59, 72, 75, 76] 致力于扩展 DFS 元数据性能，但挑战依然存在。首先，虽然 DFS 通常将目录树分区到多个元数据服务器以实现横向扩展，但选择分区粒度在负载均衡和操作开销之间存在固有的权衡，使得两者难以兼得 [46]。粗粒度分区 [15, 45, 47, 51, 59, 76] 将文件 inode 与其父目录分组，使得大目录容易成为热点，尤其是在偏斜工作负载下 [1, 74]。相比之下，细粒度分区 [14, 38, 75] 将文件 inode 均匀打散到各元数据服务器，实现了良好的负载均衡，但破坏了 inode 与父目录之间的共置关系；由于元数据分布在不同服务器上，元数据操作需要使用（昂贵的）分布式事务来一致地更新它们。其次，最先进的分布式元数据服务常常面临目录元数据上的竞争。具体而言，与父目录相关的操作（如 mkdir 和 create）在父目录上产生竞争，导致可扩展性下降 [16, 75]。

导致上述挑战的一个根本原因是同步元数据更新（synchronous metadata updates）。由于文件系统语义要求持久可见性（操作的效果对后续操作可见）和原子性（无部分更新），最先进的 DFS 通常采用同步事务。不幸的是，这种方法在关键路径上引入了跨服务器协调开销和元数据竞争，导致高延迟和吞吐量瓶颈。

在本文中，我们认为同步更新过于保守。我们观察到，目录在更新后通常不会被立即读取。这为异步更新提供了机会——将目录更新推迟到下一次读取时，从而隐藏更新延迟并对更新进行批处理。

然而，为了保持元数据更新的持久可见性，需要一种机制来识别具有待处理更新的目录，以避免读取到过期数据。一种直接的方法是在元数据操作的关键路径上专门部署一台服务器，用于跟踪目录更新并通知目录读取操作。但这会引入额外的 RTT，且该专用服务器可能成为瓶颈。相比之下，我们发现可编程交换机——自然地位于元数据操作的关键路径上，且能够提供极高的数据包处理吞吐量——是协调异步更新和读取的理想候选。

基于这些洞察，我们提出了 SwitchFS，一个与可编程交换机协同设计以实现操作协调的异步分布式文件系统。SwitchFS 符合 POSIX 规范，目标是在偏斜工作负载下同时实现良好的负载均衡、低延迟和高可扩展性。SwitchFS 具有以下设计特点。

首先，我们提出了一种高效的异步元数据更新协议，具有两大优势：(a) 它使得大多数元数据操作能够在单个 RTT 内完成，同时采用细粒度分区实现负载均衡，解决了先前 DFS 面临的权衡问题；(b) 它在应用之前合并存在竞争的元数据更新，从而缓解元数据竞争。具体而言，当一个操作更新跨多个服务器的对象（通常是文件及其父目录）时，SwitchFS 在托管文件 inode 的服务器上记录目录更新，并在可编程交换机中将该目录标记为"分散的 (scattered)"。当读取目录时，交换机会指示该目录是否处于分散状态。如果是，SwitchFS 在访问目录之前从其他服务器聚合延迟的更新。为进一步加快聚合过程并减少目录元数据上的竞争，SwitchFS 在聚合前对同一目录的元数据更新进行合并（consolidation），利用了目录属性更新的固有可交换性 [2, 7, 33, 69, 71, 75, 78]。

其次，我们利用可编程交换机在网络内高效地跟踪分散目录。我们的洞察是，得益于其高数据包处理能力和在网络中的中心位置，可编程交换机相比标准服务器是更优越的协调者。考虑到交换机上有限的内存和计算资源，我们通过以组相联方式组织交换机寄存器，设计了一个紧凑的网内脏集合（in-network dirty set），具有高内存利用率和低冲突率。通过检查数据包中的特定头部，脏集合能够以线速率插入、查询和删除目录指纹，并根据需要重定向数据包，使得 SwitchFS 能够以可忽略的开销查询和更新目录状态。

![Fig. 1: 元数据分区方法。](Fig. 1)

我们的评估表明，SwitchFS 相比 Emulated-InfiniFS [45] 吞吐量提升高达 13.34 倍，相比 Emulated-CFS [75] 延迟降低 57.3%。对于真实工作负载，相比 CephFS、Emulated-InfiniFS 和 Emulated-CFS，SwitchFS 的端到端吞吐量分别提升了 21.1 倍、1.1 倍和 0.3 倍。

总结而言，本文做出了以下贡献：

*   对扩展 DFS 元数据性能的关键挑战进行了分析（§3.2）。我们识别出命名空间分区粒度的固有权衡和目录竞争导致元数据性能次优的问题。
*   提出了一个采用异步元数据更新的分布式文件系统 SwitchFS（§5）。SwitchFS 通过异步元数据更新在不牺牲一致性的前提下解决了操作效率与负载均衡之间的权衡（§5.2），并通过变更日志压缩（change-log compaction）缓解了目录元数据上的竞争（§5.3）。
*   提出了用于跟踪目录状态的网内脏集合（§6）。据我们所知，SwitchFS 是首个将网内优化（in-network optimization）引入 DFS 的工作。

---

## 2 背景 (Background)

### 2.1 分布式元数据分区 (Distributed Metadata Partitioning)

许多 DFS 将目录树分区到一组元数据服务器上，以扩展元数据服务 [38, 45, 47, 51, 75, 76]。分区方法是一个重要的设计选择，影响操作开销和负载分布，并最终影响整体性能和可扩展性。现有方法可分为父子分组（parent-children grouping, P/C grouping）和父子分离（parent-children separation, P/C separation），取决于文件 inode 是否与其父目录共置。这些方法在 Fig. 1 中说明，并在 Tab. 1 中进行了比较。

**父子分组 (Parent-children grouping)**。P/C 分组将文件 inode 与其父目录共置，而目录 inode 可能独立分布以实现可扩展性，例如子树分区 [13, 66] 和逐目录哈希 [15, 45, 47, 51, 59]，如 Fig. 1(a) 所示。这些研究采用 P/C 分组，因为它具有良好的元数据局部性，从而降低了操作开销。由于元数据操作通常同时更新元数据对象及其父目录（例如文件创建），当父目录和子对象共置时，部分更新可在同一台服务器上执行，避免了昂贵的跨服务器协调。P/C 分组的缺点在于它将同一目录内的所有文件 inode 放置在单台服务器上，在偏斜工作负载下会导致严重的负载不均衡，从而制约可扩展性。

**父子分离 (Parent-children separation)**。P/C 分离将文件 inode 独立于其父目录的位置进行分布，例如逐文件哈希 [14, 75]，如 Fig. 1(b) 所示。通过均匀分布元数据对象，P/C 分离实现了良好的负载均衡。然而，元数据操作需要跨服务器协调以一致地更新多个对象，这会导致昂贵的开销。

### 2.2 可编程交换机 (Programmable Switch)

可编程交换机是一种配备可编程 ASIC 和带状态存储器 [6, 21, 49] 的数据包交换设备，能够实现面向分布式系统的网内协同设计（in-network co-design）。它可以在以极高速度转发数据包的同时，对数据包执行用户自定义的操作。与使用标准服务器处理数据包相比，可编程交换机具有两大优势：(a) 其在网络中的中心位置使其能够全局视角观察所有通信；(b) 基于 ASIC 的硬件以线速率处理数据包，提供高吞吐量和低延迟。我们在 §7.3.3 的评估中支持这些论断。

---

## 3 动机与挑战 (Motivation and Challenges)

### 3.1 数据中心工作负载特征 (Characteristics of Datacenter Workloads)

**目录在更新后通常不会被立即读取。** Tab. 2 展示了阿里巴巴三个已部署 PanguFS 实例 [45] 在不同工作负载（如数据处理、对象存储和块存储）下元数据操作的比例。更新目录的操作（即 create、delete、mkdir、rmdir 和 rename）占所有元数据操作的 30.76%，而读取目录的操作（即 readdir 和 statdir）仅占 4.19%。根据鸽笼原理，这一差异意味着至少 (30.76% − 4.19%) / 30.76% = 86.3% 的目录更新之后没有紧随的目录读取操作，即满足以下任一条件：(a) 该更新后的目录从未被读取过，或 (b) 该目录在被读取前至少被另一个操作更新过。因此，延迟应用目录更新有助于隐藏延迟和缓解竞争。值得注意的是，相关更新之间的偏序关系（例如同一文件的创建和删除）应得到保留，SwitchFS 保证了这一点。

**数据中心工作负载在多个维度上存在偏斜（skewed）。** 这种偏斜性给元数据管理带来了挑战，如果负载均衡不佳，可能阻碍性能扩展。

*   **不均衡的目录层次结构。** 对已部署 DFS [10, 58, 59] 的分析表明，尽管大多数目录包含少于 128 个条目，但有些目录包含超过一百万个条目。这种不均衡的层次结构使得 inode 分布和元数据负载的均衡复杂化。
*   **目录热点。** 此外，应用程序通常将相关文件分组到同一目录中，并在短时间间隔内访问它们 [38, 45, 51]。这形成了时间上的热点，进一步增加了负载均衡的难度。

### 3.2 元数据服务的挑战 (Challenges for Metadata Service)

现代数据中心应用要求 DFS 元数据操作具有高可扩展性、高吞吐量和低延迟。然而，设计中存在若干挑战。

**挑战 #1：目录树分区设计在低操作开销和负载均衡之间存在固有的权衡。**

为降低操作开销，将文件的 inode 与父目录共置很重要。元数据操作（例如 create 和 delete）通常同时更新元数据对象及其父目录的元数据。当对象共置时，操作可以在单台服务器上执行，避免了昂贵的跨服务器协调 [45]。

然而，由于同一目录中的文件通常是相关的并且在短时间内被访问，将它们分组会使文件系统容易产生热点 [38, 45, 51]。因此，为改善负载均衡，将同一目录的文件分散到不同服务器上非常重要。

不幸的是，共置与分散本质上是冲突的，使得鱼与熊掌难以兼得。

为评估这种权衡如何影响性能，我们比较了分别采用 P/C 分组和 P/C 分离的两个最先进 DFS——Emulated-InfiniFS (E-InfiniFS) [45] 和 Emulated-CFS (E-CFS) [75] 的性能。在 Fig. 2(a) 中，当对共享目录中的均匀随机文件执行 stat 操作时，E-CFS 的吞吐量线性扩展，因为文件均匀分布在多台服务器上；而 E-InfiniFS 无法扩展，因为所有文件 inode 都位于同一台服务器上，这表明 E-CFS 具有更好的负载均衡。相比之下，在 Fig. 2(b) 中，E-CFS 的 create 延迟比 E-InfiniFS 更高，主要是由于跨服务器协调带来的更高网络开销，这表明 E-InfiniFS 具有更低的操作开销。

**挑战 #2：同一目录中的并发操作由于在父目录上的竞争而导致低下的服务器间和服务器内并行度，阻碍性能扩展。**

无论采用何种分区方法，目录元数据上的竞争常常成为性能瓶颈。例如，当在单个目录中并发创建文件时（如数据准备 [77]、编译、解压等），每个 create 操作都会更新父目录的属性和条目列表。虽然单个文件的创建可以在多台服务器上并行化（在 P/C 分离下），但对父目录的更新在存储它的服务器上被串行化，导致服务器间并行度低下。此外，由于传统方法依赖事务和锁来保证元数据更新的原子性，服务器无法利用多核并行性，导致服务器内并行度低下。

在 Fig. 2(c) 和 Fig. 2(d) 中，两个 DFS 的吞吐量均不随服务器数量和每台服务器核心数的增加而扩展。这表明服务器间和服务器内的并行度都受到了父目录上竞争的限制。

---

## 4 总览 (Overview)

为克服 §3.2 中所述的固有局限性，SwitchFS 在网络中协调异步元数据操作以提升并行度并隐藏延迟。我们在本节概述设计原理和架构，然后在 §5 中介绍工作流程，在 §6 中介绍交换机数据平面。

### 4.1 设计原理 (Design Rationale)

**核心思想：异步目录更新。** 我们的核心思想是利用异步更新来隐藏跨服务器协调开销并利用元数据操作并行性。文件系统操作（除 rename 外，§5.2 讨论）最多更新两个元数据对象：目标对象及其父目录 [16]；因此，通过延迟对远程父目录的更新，SwitchFS 可以在单台服务器上本地执行操作。这种方法隐藏了跨服务器开销，并使对同一父目录的并发更新能够并行执行。

**性能使能技术：可编程交换机。** 为确保延迟的更新对后续访问可见，SwitchFS 利用集中的可编程交换机以线速率和亚 RTT 延迟存储目录状态。然而，由于片上内存和计算资源有限，为整个文件系统维护所有目录的状态并非易事。我们观察到，尽管目录的总数可能非常庞大，但在最近时间间隔内元数据已被修改的目录集合很小。例如，考虑一个能够每秒创建或删除一百万个文件的文件系统。每 100 毫秒内被修改的目录数量最多为十万个，小到足以容纳在交换机的片上内存中。因此，通过定期应用延迟更新并清除交换机上的数据结构，交换机上的状态跟踪是可行的。

### 4.2 整体架构 (Overall Architecture)

如 Fig. 4 所示，SwitchFS 由三类组件组成：客户端、元数据服务器和可编程交换机。为清晰起见，我们假设只有一个可编程交换机，关于扩展到多个交换机的讨论推迟到 §6.4。

**客户端 (Clients)。** SwitchFS 提供一个用户空间库 (LibFS) 供客户端与服务器通信。客户端维护一个元数据缓存以加速路径解析，该缓存仅缓存目录元数据。

**元数据服务器 (Metadata Servers)。** SwitchFS 使用细粒度分区将元数据对象分布到各元数据服务器上。服务器将元数据存储在一个键值存储（即 RocksDB [60]）中。服务器在按目录划分的变更日志（change-log）中跟踪对远程目录的延迟修改，并在其失效列表（invalidation list）中跟踪最近删除的目录。服务器使用预写日志 (WAL) 以实现崩溃恢复。

**网内脏集合 (In-network dirty set)。** 可编程交换机监控网络流量，维护一个集中的延迟更新脏集合。为高效利用有限的交换机内存，脏集合存储在一个多槽哈希表结构中。

### 4.3 元数据模式 (Metadata Schema)

Tab. 3 展示了 SwitchFS 的元数据模式。每个目录在创建时被分配一个唯一的 256 位标识符 id。元数据对象（即 inode 和 dentry）以键值对形式存储，其中键是父目录 id (pid) 与文件/目录名称的拼接，值是元数据属性。

SwitchFS 采用 P/C 分离方法以实现负载均衡，通过对 (pid, name) 对进行哈希来分区文件/目录元数据。目录的条目列表存储为多个 Dir Entry 键值对，所有这些键值对都存储在与该目录 inode 相同的服务器上。

虽然未在表中显示，但 SwitchFS 通过对 (pid, 目录名称) 进行哈希，为每个目录生成一个 49 位的指纹（fingerprint）。该指纹作为交换机内的目录标识符，设计为适配交换机有限的寄存器宽度。由于多个目录可能具有相同的指纹，SwitchFS 确保所有具有相同指纹的目录（称为指纹组，fingerprint group）被分区到同一台服务器，以简化冲突处理。

---

## 5 SwitchFS 设计 (SwitchFS Design)

在本节中，我们介绍 SwitchFS 的工作原理，包括异步元数据操作（§5.1 和 §5.2）、变更日志压缩（§5.3）和故障处理（§5.4）。我们在附录中讨论崩溃恢复的正确性（§A.1）和一致性属性（§A.2）。

### 5.1 目录状态与转换 (Directory State and Transition)

**状态定义。** 在 SwitchFS 中，目录处于两种状态之一：正常状态（normal state），表示所有已返回的对该目录的更新都已应用到其 inode；或分散状态（scattered state），表示在一个或多个变更日志中存在尚未应用的更新。交换机上的脏集合跟踪目录状态，元数据操作可以查询或更新它。

**状态转换。** Fig. 3 展示了目录状态之间的转换。目录在创建时处于正常状态，其 inode 是最新的。当 SwitchFS 进行异步更新时，目录转换为分散状态，指示有一个或多个延迟更新尚未应用到 inode。当 SwitchFS 读取处于分散状态的目录时，它从其他服务器聚合并应用延迟更新，将目录恢复到正常状态。

**转换粒度。** 如 §4.3 所述，交换机通过其指纹来识别目录。因此，状态转换的粒度是指纹组（fingerprint group），即具有特定指纹的所有目录。一次元数据聚合会聚合目标指纹组中的所有目录。由于 SwitchFS 将同一指纹组中的所有目录放置在同一台服务器上，这并不会使聚合复杂化。

### 5.2 异步元数据操作 (Asynchronous Metadata Operations)

**概述。** 元数据操作可根据其访问的 inode 数量分为三类 [16]。我们总结 SwitchFS 的处理方式如下：

*   **双 inode 操作（Double-inode operations）**，包括 create、delete、mkdir、rmdir，访问目标对象及其父目录的 inode。这些操作延迟对父目录的更新，并在单个元数据服务器上最终执行；我们在 §5.2.1 和 §5.2.3 中详细阐述。
*   **单 inode 操作（Single-inode operations）** 访问文件（如 open、close、stat）或目录（如 statdir、readdir）的 inode。前者像传统 DFS（如 InfiniFS [45] 和 CFS [75]）那样同步执行，为简洁起见我们省略其解释。后者读取目录属性，需要检查 inode 是否过期并可能触发聚合；我们在 §5.2.2 中讨论。
*   **重命名（Rename）** 是最复杂的元数据操作，因为它最多更新四个 inode。SwitchFS 通过分布式事务确保 rename 的一致性，并通过使用集中式重命名协调器（类似于先前的 DFS [45, 75]）避免孤立循环 [4, 45, 75]（即两个目录互为祖先）。值得注意的是，如果源是一个目录，SwitchFS 会在 rename 开始时发起一次聚合，以应用所有延迟更新。

**协议直觉。** SwitchFS 异步元数据操作的核心思想是将双 inode 操作拆分为两半：本地半部分，更新目标对象的 inode 并将对父目录的更新持久化记录到变更日志中；远程半部分，在父目录被读取时聚合并应用变更日志条目到父目录的 inode。这有效地将目录更新延迟到读取时，使操作能够提前返回并隐藏跨服务器开销。

为确保异步操作不会丢失更新，SwitchFS 使用交换机上的脏集合来协调目录的读写。我们在 §A.2 中证明元数据操作是可串行化的并保持了实时顺序。

**元数据服务器上的数据结构。** 元数据服务器维护以下数据结构：

*   **变更日志（Change-log）** 是一个按服务器、按目录维护的 FIFO 队列，记录已提交但尚未应用的对目录的异步更新。
*   **失效列表（Invalidation list）** 用于延迟失效客户端元数据缓存中的过期条目，类似于 InfiniFS [45]。当一个目录被移除、重命名或更改权限时，托管该目录 inode 的服务器向所有服务器广播一条消息，接收方将该目录的信息追加到其失效列表中。当客户端联系服务器时，它会检查失效列表并相应地使任何过期的缓存条目失效。
*   **键值存储（Key-value store）** 存储 inode 和目录条目。
*   **预写日志 (WAL)** 是一个用于崩溃恢复的持久化结构。它记录已提交操作的序列，并标记相应的异步更新是否已应用到远程服务器。

变更日志、失效列表和键值存储保存在内存中以获得性能。§5.4.2 和 §A.1 讨论了崩溃后如何恢复它们。

#### 5.2.1 mkdir、create 和 delete 的工作流程

这些操作异步更新父目录的元数据。Fig. 4 展示了 create(/a/b) 的示例。其他操作的处理方式类似。

**路径解析 (Path resolution)。** 首先，客户端在其缓存中查找每个中间目录（即 / 和 /a）(➀)。发生缓存未命中时，客户端向该目录 inode 的拥有者服务器发送查找请求，并用 inode 信息填充缓存。路径解析完成后，客户端将请求发送给服务器 B——/a/b 的 inode 的拥有者服务器（通过对 /a/b 的 pid 和 name 进行哈希确定）。

**加锁与检查 (Locking and checking)。** 服务器 B 获取目录 /a 的变更日志上的写锁和键值存储中 /a/b 的 inode 上的写锁 (➁)。然后它检查 /a/b 的任何路径组件是否出现在失效列表中，以及文件 /a/b 是否已存在 (➂)。如果检查失败，服务器 B 要求客户端使过期的缓存条目失效并重试整个操作。如果检查通过，该操作被持久记录到 WAL 并提交 (➃)。

**执行 (Execution)。** 服务器 B 将文件 /a/b 的 inode 插入键值存储，并将一个条目追加到目录 /a 的变更日志中，记录 create(/a/b) 操作 (➄)。

**脏集合更新、回复与解锁。** 服务器 B 向交换机发送一个包含父目录 /a 指纹的数据包。收到数据包后，交换机将 /a 的指纹插入脏集合 (➅)。如果插入成功，交换机会将该数据包多播给客户端和服务器 B，通知客户端操作已完成 (➆a) 并指示服务器 B 释放锁 (➆b)。如果插入失败（例如由于溢出），则操作回退到同步元数据更新：交换机将数据包转发给拥有父目录 inode 的服务器，该服务器随后同步更新父目录。

**讨论。** 值得注意的是，多台服务器可能独立地更新同一目录并记录其更新。这些更新不会冲突，因为对目录属性的更新是可交换的（commutative），并且对该目录的下一次 statdir 或 readdir 操作将聚合所有这些更新（§5.2.2 和 §5.3）。

#### 5.2.2 statdir 和 readdir 的工作流程

这些操作读取目录属性和条目列表，如果目录处于分散状态则触发聚合。Fig. 5 展示了 statdir(/a) 的示例。readdir 的处理方式类似。

**路径解析。** 客户端按照 §5.2.1 的方式进行路径解析 (➀)，然后将请求数据包发送给目录 /a 的拥有者服务器（即服务器 A）。该数据包包含 /a 的指纹。

**脏集合查询。** 当交换机转发数据包时，它会查询 /a 的状态并将结果附加到数据包上 (➁)。

**加锁与检查。** 收到数据包后，服务器 A 首先获取 /a 的 inode 上的读锁 (➂)，并按照 §5.2.1 的方式检查缓存有效性和 inode 是否存在 (➃)。然后，服务器检查脏集合查询结果。如果 /a 处于正常状态，服务器直接将目录的 inode 返回给客户端。否则，服务器 A 发起一次元数据聚合以更新目录的 inode，然后再回复客户端。

**聚合与回复。** 为发起聚合，服务器 A 阻塞 /a 的指纹组中所有目录的 statdir 和 readdir 操作，并向交换机发送一个包含 /a 指纹的聚合请求。交换机收到请求后，从脏集合中移除 /a 的指纹，并将该请求多播给所有其他服务器 (➄)。收到请求后，每个其他服务器获取 /a 的变更日志上的读锁，然后将 /a 的指纹组中的所有变更日志条目发送给服务器 A (➅)。服务器 A 收到这些变更日志条目后，将每个条目记录到 WAL (➆)，并相应地更新键值存储中 /a 的元数据 (➇)。在收到所有服务器的条目并应用完成后，服务器 A 向其他服务器多播一个确认消息，释放本地锁，并回复客户端 (➉)。其他服务器收到确认消息后，解锁变更日志 (➈a) 并在 WAL 中将变更日志条目标记为"已应用 (applied)" (➈b)。变更日志中的条目在后台回收。

#### 5.2.3 rmdir 的工作流程

Fig. 6 展示了 rmdir(/c) 的示例。前几个步骤，即路径解析 (➀)、加锁 (➁)、验证和 inode 存在性检查 (➂)，与 create 相同。在这些步骤之后，服务器 A（即 /c 的拥有者服务器）必须收集对 /c 的最新更新，以确定目录是否为空，并防止任何客户端侧对 /c 的过期缓存在未来操作中被重用。为此，服务器 A 向交换机发送一个聚合请求。交换机从脏集合中移除 /a 的指纹，并将该请求多播给所有其他服务器 (➃)。收到请求后，每个其他服务器（例如服务器 B）将 /c 插入其失效列表 (➄)，以便客户端中 /c 的过期缓存在客户端的下一次操作中被失效。然后，服务器 B 获取 /c 的变更日志上的读锁，并将条目发送给服务器 A (➅)。服务器 A 应用所有收到的变更日志条目，然后根据聚合后的元数据检查 /c 是否为空 (➆)。如果 /c 为空，服务器 A 将 rmdir(/c) 操作记录到 WAL (➇)，操作提交。否则，rmdir(/c) 失败并返回 ENOTEMPTY 错误。剩余步骤 (➈➉, ➊➀a, ➊➀b) 与 create 类似。最后，服务器 A 发送一个数据包通知其他服务器释放锁，并将它们已发送的变更日志条目在各自的 WAL 中标记为"已应用" (➊➁)。

**讨论。** SwitchFS 通过延迟失效（lazy invalidation）防止客户端访问过期的目录元数据。具体来说，rmdir(/c) 在实际删除目录 (➈) 之前，先将 /c 追加到所有服务器的失效列表中 (➄)。任何在追加之后检查路径有效性的 /c 下的操作都会检查失败并发起 lookup(/c) 请求。由于 rmdir(/c) 持有 /c 的 inode 上的写锁 (➁)，lookup 会等待直到 rmdir(/c) 完成，从而观察到最新的元数据。

### 5.3 变更日志的压缩与应用 (Change-Log Compaction and Applying)

异步元数据更新使得目录修改的并行和本地日志记录成为可能，隐藏了目录更新开销。SwitchFS 通过在将变更日志应用到目录 inode 之前对其进行压缩（compaction），进一步分摊更新开销，缓解了元数据竞争的瓶颈。

**变更日志结构。** 在 SwitchFS 中，服务器为每个分散的目录维护一个本地变更日志。一个变更日志条目表示一个延迟的目录更新，包含时间戳、操作类型（create、delete 等）和文件名，如 Fig. 7 所示。

**变更日志压缩的机会。** 对同一目录的更新是有条件可交换的，使得可以在应用前将多个更新合并为一个，从而缓解更新竞争 [2, 7, 33, 69, 71, 75, 78]。具体而言，目录更新涉及三种类型的操作：
(a) 对 size 等属性应用增量（delta），
(b) 覆盖 atime 和 mtime 等时间戳，以及
(c) 在条目列表中插入或移除一个条目。

对于类型 (a)，以任意顺序应用增量都会得到相同的最终状态。对于类型 (b)，只有最大的时间戳保留。对于类型 (c)，不同条目的插入和移除不会冲突，可以按任意顺序应用；而同一名称的重复插入和移除必须按其提交顺序应用（例如 mkdir(/a), rmdir(/a)）。在 SwitchFS 中，这一顺序由变更日志的 FIFO 结构保证，因为同一条目的插入和移除总是由同一台服务器记录。

**变更日志压缩的工作流程。** 因此，SwitchFS 提出变更日志压缩以降低目录更新的成本。当目录更新被追加到变更日志时，变更日志会保留所有条目中的最大时间戳，并存储操作类型和文件名。收到聚合请求时，服务器将相关变更日志传输给目录的拥有者服务器。拥有者服务器随后遍历操作队列，更新目录条目列表，并原子地更新 inode 的属性（例如时间戳和 size）。

变更日志压缩通过两种方式缓解目录更新竞争。首先，异步元数据更新允许所有服务器在其变更日志中本地合并对远程父目录的更新，从而增强服务器间并行度。其次，变更日志压缩预先合并时间戳和 size 增量，以最小化键值存储的 put() 操作次数。

---

# 翻译说明

**原文**: SwitchFS: Asynchronous Metadata Updates for Distributed Filesystems with In-Network Coordination（EuroSys 2026）
**作者**: 徐经纬、董明凯、田秋林、田子逸、辛童、陈海波，上海交通大学IPADS实验室
**翻译章节**: §5.3–§7.7（Part 2）

---

=== PAGE 8 ===

**主动聚合（Proactive aggregation）。** 为了防止在一系列目录更新之后首次读取目录时发生长时间聚合阻塞，服务器会在以下两种情况下主动将其变更日志（change-log）条目推送给目录的属主服务器（owner server）：(1) 当累积的条目足以填满一个最大传输单元（MTU）时；或 (2) 当在配置的时间间隔内没有新条目追加到变更日志时。收到变更日志推送后，目录的属主服务器会启动一个定时器。如果在配置的时间段内没有新的条目推送到达，属主服务器会主动发起一次聚合（aggregation）。这种主动聚合将目录恢复为正常状态，使得后续的目录读取操作无需触发聚合即可完成。

### 5.4 容错处理（Fault Tolerance）

#### 5.4.1 不可靠网络（Unreliable Network）

SwitchFS 协议基于 UDP，因此可能经历丢包和乱序。SwitchFS 通过超时和重传来处理丢包。传输中的数据包乱序不会引发问题，因为属于同一操作的数据包是按序逐个发送和接收的，而属于不同操作的数据包之间相互独立，SwitchFS 也不假定它们的顺序。

重传会导致数据包重复。服务器和交换机通过不同的机制来容忍重复包。服务器通过检查发送方附加在每个数据包上的（发送方服务器，序列号）元组来检测并丢弃重复包。交换机则处理封装在数据包中的三类脏集（dirty set）操作：指纹查询（fingerprint query）、插入（insert）和删除（remove）。交换机执行重复的查询和插入时无需特殊处理，因为它们不影响正确性。具体而言，查询无副作用，冗余的插入可能触发不必要的聚合，但不会破坏正确性。只有重复的删除操作可能引发不一致：当聚合发送的重复删除请求在聚合完成后才到达交换机时，会删除后续操作插入的指纹。为避免此问题，每个删除请求都带有一个序列号（与数据包的序列号不同），该序列号随每次新请求或重传递增。交换机记录从每个服务器收到的最高删除序列号，只有当某个删除请求的序列号超过该发送方此前所有已接收的序列号时，交换机才处理该请求。在重传删除请求后，服务器会等待最后一次重传对应的响应。通过这种方式，聚合完成后不会有重复的删除请求生效，从而确保一致性。

#### 5.4.2 崩溃恢复（Crash Recovery）

**服务器故障（Server failure）。** 服务器将数据结构维护在 DRAM 中，并使用预写式日志（WAL，write-ahead log）来在服务器故障后恢复丢失的进度，这与生产部署（例如 PanguFS 和 HDFS）的做法一致[45]。要从故障中恢复，服务器首先重做 WAL 中的操作，以重建其本地键值存储以及尚未标记为"已应用（applied）"的变更日志条目。注意，我们不需要重建已标记为"已应用"的变更日志条目，因为它们已持久化在目录 inode 所属服务器的 WAL 中。接着，服务器主动聚合其拥有的所有目录，以确保在崩溃前发出的任何中断的聚合都能执行完成。最后，服务器通过从其他服务器克隆来恢复其失效列表（invalidation list）。在恢复期间，服务器不处理正常请求。

**交换机故障（Switch failure）。** 交换机故障后，交换机中的所有状态都会丢失。SwitchFS 在交换机中初始化一个空脏集，并通知所有服务器聚合所有目录。一旦所有聚合完成，所有变更日志条目都已发送到其目录 inode 所属的服务器并被应用。因此，SwitchFS 中的所有目录都恢复到正常状态，与空脏集保持一致。在恢复期间，所有服务器停止处理正常请求。

### 5.5 讨论（Discussion）

**集群重配置（Cluster reconfiguration）。** SwitchFS 使用一致性哈希（consistent hashing）[25] 将 inode 映射到服务器。当添加或移除服务器时，需要迁移一小部分元数据。SwitchFS 以停止整个世界（stop-the-world）的方式进行迁移：所有元数据服务器停止处理请求，聚合所有目录，然后通过两阶段提交（two-phase commit）将元数据迁移到新服务器。由于哈希函数位于客户端和服务器上，交换机无需任何更改。我们在 §A.3 中讨论集群重配置的容错能力。

**硬链接支持（Support of hard links）。** 为支持硬链接，Tab. 3 中的文件元数据需要拆分为两个 KV 对象：

- **引用（reference）：** (pid, name) → (server_id, file_id)
- **属性（attributes）：** file_id → (ref_count, size, timestamps, ...)

在文件创建时，两个对象都在由哈希 (pid, name) 确定的服务器上创建。创建硬链接时，只在哈希位置创建一个新的引用，指向原始文件的属性（可能位于不同的服务器上）。属性的 ref_count 跟踪引用数量，当该计数降至零时，属性被删除。为保证一致性和原子性，元数据访问期间这两个对象受两阶段锁定（two-phase locking）保护，而跨服务器的更新则通过两阶段提交进行协调。

## 6 数据平面设计（Data Plane Design）

在本节中，我们介绍 SwitchFS 数据包的格式（§6.1）、交换机数据平面布局（§6.2）和脏集设计（§6.3）。我们还讨论了扩展到多机架（§6.4）以及可适配性问题（§6.5）。

### 6.1 SwitchFS 数据包格式（SwitchFS Packet Format）

SwitchFS 采用 UDP 作为传输层协议以实现轻量级网络通信。如图 Fig. 9 所示，UDP 载荷以一个可选头部开头，该头部封装了一个脏集操作（dirty-set operation），可由交换机解析。

=== PAGE 9 ===

Parser（解析器）→ Packet In（数据包入）→ Ingress Pipe（入站管道）→ Dirty Set（脏集）→ Egress Pipe（出站管道）→ [Packet Out / Mirror（数据包出/镜像）] → Router（路由器）[by MAC（按MAC）/ by fp.（按指纹）] → Address Rewriter（地址重写器）→ Packet Out（数据包出）

**Fig. 8: SwitchFS 交换机数据平面的逻辑视图。**

在该头部之后，载荷包含一个供服务器处理的 DFS 请求或响应。SwitchFS 为带有和不带有脏集操作头部的 SwitchFS 数据包预留了两个 UDP 端口，以便交换机区分它们。

### 6.2 数据平面布局（Data Plane Layout）

图 Fig. 8 展示了交换机数据平面的布局，包含以下组件：

**解析器（Parser）。** 解析器解析数据包头部，提取后续处理所需的字段。

**路由器（Router）。** 路由器在交换机的元数据中设置出口端口字段：(a) 对于常规数据包，按目的 MAC 地址设置；(b) 对于带有脏集操作的数据包，按指纹前缀设置。交换机据此转发数据包。

**脏集（Dirty set）。** 脏集实现为一个多槽位哈希表，支持三类操作：(1) 将指纹插入集合；(2) 查询指纹是否在集合中；(3) 从集合中删除指纹。当数据包到达时，脏集执行头部 OP 字段中指定的操作，并将结果写入 RET 字段。SEQ 是一个用于识别重复删除请求的序列号，如 §5.4.1 所述。

由于交换机可能有多个流水线（pipe），且流水线之间不共享状态，我们以无共享（shared-nothing）的方式将脏集放置在出站流水线中。每个流水线负责处理具有不同前缀的指纹，数据包据此被路由。如果某个数据包需要访问由不同于其目的端口的流水线管理的指纹，交换机会将其镜像到目标流水线，这与先前的研究一致[24, 79]。

**地址重写器（Address rewriter）。** 如果脏集插入因溢出而失败，交换机使用数据包头部中的备用 MAC 地址字段将数据包重定向以进行回退处理。地址重写器用该备用地址覆盖 L2 目的地址字段，交换机据此转发。

### 6.3 网内脏集（In-Network Dirty Set）

交换机上的脏集维护处于分散状态（scattered state）的目录集合。由于其性能和容量对 SwitchFS 至关重要，我们有以下设计目标：(1) 支持线速的指纹查询、插入和删除；(2) 最大化片上稀缺内存的利用率。

**结构（Structure）。** 由于编程限制无法实现复杂的索引结构，SwitchFS 以类似于组相联缓存（set-associative cache）的方式组织 32 位寄存器来存储指纹。如图 Fig. 9 所示，交换机数据平面包含多个阶段（stage），每个阶段包含一组寄存器数组。不同阶段中相同位置的寄存器被水平分组为一个组（set）。在我们的交换机配置中，共有十个阶段，每个阶段分配 131,072（2¹⁷）个寄存器。这种设置允许交换机存储最多 1,310,720 个指纹。我们使用指纹的高 17 位作为组索引（set index），剩余的 32 位作为标签（tag）来唯一标识每个指纹。

**脏集操作（Dirty-set operations）。** 脏集支持三种操作：插入（insert）、查询（query）和删除（remove）。每个操作在由头部 index 字段索引的寄存器上执行一系列寄存器动作。我们定义三种寄存器动作：(a) **寄存器查询（register query）** — 将寄存器的值与 tag 比较并返回结果；(b) **条件插入（conditional insert）** — 返回寄存器值是否等于零或 tag，如果旧值为零则将 tag 写入寄存器；(c) **条件删除（conditional remove）** — 如果旧值与 tag 匹配，则将零写入寄存器。

对于脏集查询，所有阶段顺序执行寄存器查询，只要任一阶段返回 true 则结果为 true。对于脏集删除，所有阶段顺序执行条件删除。对于脏集插入，阶段逐个执行条件插入，直到其中一个返回 true，后续阶段执行条件删除以确保集合中不保留重复的 tag。图 Fig. 10 演示了一个脏集插入的示例。

**属性（Properties）。** 交换机架构提供两个属性[73]：(a) **原子性（Atomicity）** — 同一阶段内的操作是原子的。(b) **有序执行（Ordered execution）** — 对于任意两个数据包 Pₐ、P_B 和两个有序阶段 Sₐ、S_B，如果 Pₐ 在 P_B 之前到达 Sₐ，则 Pₐ 在 P_B 之前到达 S_B。

=== PAGE 10 ===

**Fig. 11: 多机架部署的拓扑结构。**

基于这些属性，我们的设计确保脏集操作 (a) **幂等（idempotent）**，因为连续的重复操作与单次操作效果相同；以及 (b) **可线性化（linearizable）**，因为对同一指纹的操作由流水线串行化处理。

### 6.4 扩展到多机架（Scale to Multiple Racks）

对于单机架部署，所有元数据服务器位于同一机架中，脏集放置在架顶（ToR，top-of-rack）可编程交换机上，使其能够监控所有机架流量。对于多机架部署，SwitchFS 采用如图 Fig. 11 所示的叶脊（leaf-spine）拓扑。由于 ToR 交换机不再具有全局视图，可编程交换机部署在脊（spine）层。为进一步扩展，SwitchFS 按指纹将目录的状态管理范围分区到各台交换机上，并将涉及每个目录子集的数据包定向到相应的交换机。先前的研究[84]表明，交换机之间的额外网络跳数开销可以忽略不计，因为它比操作处理的开销小几个数量级。该方法能够适应偏斜（skewed）的工作负载，因为 (a) 由于指纹的随机性，目录在各交换机上的分布是均匀的；(b) 每台交换机可以处理元数据集群中的所有流量而不会成为瓶颈（§7.3.3）。

### 6.5 讨论（Discussion）

**可编程交换机的可适配性（Adaptability of the programmable switch）。** SwitchFS 的交换机上程序包含 847 行 P4 代码和 303 行 C 代码。它消耗 1,310,720 个 32 位片上寄存器（总计 5 MiB）。在 Tofino 交换机[21]上初始化该程序耗时不到十秒。

**局限性（Limitations）。** 鉴于 Tofino 交换机[21]片上内存有限，将 SwitchFS 与状态密集型网络功能（例如网内缓存[24, 44]）共存具有挑战性。不过，它可以与流控制和数据包过滤等轻量级功能共存。可编程交换机也构成了潜在的单点故障，我们将此留待未来工作解决。

## 7 评估（Evaluation）

### 7.1 实验设置（Experimental Setup）

**硬件配置（Hardware configuration）。** 我们使用八台服务器和三台客户端机器进行实验。Tab. 4 显示了它们的规格。所有机器通过一台配备 Tofino ASIC [21] 的 Wedge100BF-32X 可编程交换机连接。为将评估扩展到 16 台服务器，我们在服务器节点的每个 CPU 插槽上部署一个元数据服务器，利用 NUMA 本地的核心、DRAM 和 NIC。由于硬件限制，我们未使用多台可编程交换机进行评估。

| | 服务器集群 | 客户端集群 |
|---|---|---|
| **CPU** | 2× Intel Xeon Gold 5317 3.00GHz, 12 核 | 2× Intel Xeon E5-2650 v4 2.20GHz, 12 核 |
| **内存** | 16× DDR4 2933MHz 16GB | 16× DDR4 2933MHz 16GB |
| **存储** | Intel Optane 持久内存 | — |
| **网络** | 2× ConnectX-5 单端口 100GbE | 2× ConnectX-4 单端口 100GbE |

**Tab. 4: 集群硬件规格。**

**基线系统（Baseline systems）。** 我们将 SwitchFS 与 CephFS v12.2.13 [76]、IndexFS [59]、InfiniFS [45] 和 CFS [75] 进行比较。CephFS 是一个广泛使用的商用 DFS，其他系统则是针对元数据性能优化的最先进 DFS。由于 InfiniFS 和 CFS 未公开提供，我们从头实现了它们。我们根据论文[45]仔细实现以模拟 InfiniFS，并通过将 InfiniFS 的 P/C 分组方法（即按目录哈希）替换为 CFS 的 P/C 分离方法（即按文件哈希）来模拟 CFS。值得注意的是，模拟 InfiniFS（E-InfiniFS）、模拟 CFS（E-CFS）和 SwitchFS 共享相同的存储和网络框架，确保了公平比较。我们没有与 SingularFS [16] 和 DeltaFS [83] 进行比较，因为 SingularFS 专注于使用新型硬件（即持久内存和 RDMA）提升单服务器性能，这与我们的工作正交；而 DeltaFS 是为 HPC 作业设计的，其语义与 POSIX DFS 不同。

**配置细节（Configuration details）。** E-InfiniFS、E-CFS 和 SwitchFS 使用 RocksDB 的异步写入模式进行元数据存储，并采用基于协程的非阻塞 RPC 引擎（基于 DPDK 21.11.2 [53]）进行网络通信。除非另有说明，每个元数据服务器使用四个核心，客户端使用 LibFS 与服务器通信。对于 CephFS，我们部署了多个活动元数据服务器守护进程（MDS，metadata server daemon），与对象存储守护进程（OSD，object storage daemon）共存。SwitchFS 在所有实验中启用了主动变更日志聚合（proactive change-log aggregation），因此聚合开销已包含在结果中。评估中未发生脏集溢出。

### 7.2 整体性能（Overall Performance）

#### 7.2.1 吞吐量（Throughput）

我们首先评估单个操作的峰值吞吐量如何随服务器数量扩展。我们逐步增加客户端发出的并发请求数（最多 512 个），直到吞吐量不再增加。我们在两种不同的访问模式上进行实验：单个大目录和多个目录，它们分别反映 DFS 的负载均衡能力和操作开销。IndexFS 在单个大目录上的吞吐量未包含在内，因为它持续崩溃并报错。

=== PAGE 11 ===

**单个大目录（A single very large directory）。** 在此评估中，客户端对单个大目录中的 1000 万个文件执行操作，并随机选择文件进行访问。图 Fig. 12(a) 展示了随着元数据服务器增加的操作吞吐量。我们得出以下观察结果：

1) SwitchFS 的双 inode 操作（包括 create、delete、mkdir 和 rmdir）的吞吐量随服务器数量增加而良好扩展。原因是：(a) SwitchFS 的细粒度分区方法将负载均匀分布到各服务器上；(b) 异步元数据更新协议允许双 inode 操作仅访问一台服务器，避免了跨服务器协调并隐藏了目录争用；(c) Change-Log压缩（change-log compaction）合并了冲突的目录更新，消除了目录争用。需要注意的是，SwitchFS 的 rmdir 吞吐量低于 mkdir，因为 SwitchFS 在 rmdir 期间使用多播（multicast）通知目录失效。

2) 尽管采用了细粒度分区，E-CFS 在 create、delete、mkdir 和 rmdir 上几乎无扩展性。这是因为这些操作按目录串行化，导致严重的争用。

3) SwitchFS 和 E-CFS 的 stat 操作都线性扩展，得益于通过细粒度分区实现的有效负载均衡。相比之下，E-InfiniFS 由于将大目录中的所有文件分配到单一服务器，遭受严重的负载不均。对于 statdir，除 CephFS 外的所有文件系统都线性扩展，因为它们以均衡方式分区目录。SwitchFS 的脏集检查给 statdir 引入了可忽略的额外开销。

4) CephFS 的吞吐量较低（低于 100 Kops/s），因其沉重的软件栈和僵化的子树分区。

**多目录（Multiple directories）。** 在此评估中，客户端随机访问均匀分布在 1024 个目录中的 1000 万个文件。图 Fig. 12(a) 展示了随元数据服务器数量增加的操作吞吐量。我们得出以下观察结果：

1) 在此设置中，操作几乎无争用，DFS 达到最优性能。SwitchFS 在所有操作中表现最佳，表明异步元数据更新在常规路径上的开销可忽略不计。

2) 对于 create 和 delete，SwitchFS 和 E-InfiniFS 性能相近且优于 E-CFS。E-InfiniFS 表现良好，因为其父子分组（parent-children grouping）使得文件和父目录元数据可以在同一台服务器上更新。SwitchFS 达到了可比性能，因为异步元数据更新隐藏了更新目录元数据的开销。相比之下，E-CFS 表现较差，因为它将文件和父目录元数据分离，需要跨服务器事务进行更新。

3) 对于 mkdir，SwitchFS 优于 IndexFS、E-InfiniFS 和 E-CFS，因为异步更新隐藏了跨服务器协调的开销（更新位于不同服务器上的两个目录的元数据），而其他 DFS 需要使用分布式事务。对于 rmdir，SwitchFS 的性能与 E-InfiniFS 和 E-CFS 相近，因为多播开销抵消了异步更新的收益。IndexFS 的 rmdir 实现不完整，因此省略其结果。

#### 7.2.2 延迟（Latency）

我们通过使用单个客户端逐个发出请求来评估操作的延迟。DFS 使用八台服务器。图 Fig. 13 显示了平均延迟。

1) SwitchFS 显著降低了 create、delete、mkdir 和 rmdir 的延迟，相比其他 DFS 更为出色，因为其异步元数据更新隐藏了更新父目录的开销。

=== PAGE 12 ===

2) SwitchFS 的 statdir 延迟比 E-InfiniFS 和 E-CFS 高 28.6%，这是因为异步元数据更新为了正确性引入了额外的检查。

### 7.3 技术贡献分解（Contribution Breakdown）

#### 7.3.1 各技术的贡献（Contribution of Techniques）

在此评估中，我们分析每个设计特性对 SwitchFS 吞吐量和延迟的贡献。客户端在单个目录中创建 1000 万个文件，DFS 使用八台服务器。在评估吞吐量时，我们进一步改变每台服务器的核心数，以展示 SwitchFS 的服务器内并行性。图 Fig. 14 展示了结果。

**Baseline（基线）** 是 SwitchFS 的框架，采用细粒度分区和同步操作。由于跨服务器协调导致高延迟，父目录争用导致低吞吐量。

**+Async** 采用异步元数据更新但不进行 Change-Log压缩。与 Baseline 相比，平均延迟降低了 34.7%，而吞吐量保持不变。这是因为异步元数据更新隐藏了更新父目录的延迟，但并未减少更新开销。在聚合期间，对父目录元数据的更新由于争用而被键值存储串行化，导致并行性差。

**+Compaction** 是完整的 SwitchFS 设计，同时包含异步元数据更新和 Change-Log压缩优化。与之前的设置相比，吞吐量提升了最多 2.48 倍，并随每台服务器核心数增加而扩展。这是因为 Change-Log压缩将多个对目录属性（例如时间戳）的更新合并为一次更新，同时目录项的插入或删除可以并行执行。与 +Async 相比，平均延迟降低了 18.2%，p99 延迟从 173 μs 降至 22 μs。

#### 7.3.2 脏集溢出的影响（Impact of Dirty-Set Overflow）

如果由于溢出导致插入可编程交换机的脏集失败，该操作会回退到服务器端同步更新（§5.2.1）。我们强制脏集插入始终失败，并使用 §7.3.1 中描述的设置评估 create 的性能。与插入成功的情况相比，吞吐量下降了 69.7%，平均延迟上升了 0.85 倍，与 §7.3.1 中 Baseline 的性能高度吻合。

**Fig. 16: 目录状态由属主服务器跟踪时 create 操作的延迟。每个盒子的上下边缘分别代表 p75 和 p25，盒子中的线条代表中位数。**

#### 7.3.3 可编程交换机的贡献（Contribution of the Programmable Switch）

SwitchFS 中的异步更新协议与可编程交换机并非紧耦合。本评估考察了使用可编程交换机与两种替代方案的权衡：(a) 使用专用服务器维护脏集；(b) 让每个目录的属主服务器跟踪其自身的脏状态。

**用专用服务器跟踪目录状态（Tracking directory state with a dedicated server）。** 使用专用服务器而非可编程交换机跟踪目录状态有两个缺点。首先，如图 Fig. 15(a) 所示，涉及脏集的操作会增加一个额外的 RTT（约 3 μs），使 create 和 statdir 的平均延迟分别上升 24.1% 和 13.1%。其次，服务器无法达到可编程交换机的吞吐量。在图 Fig. 15(b) 中，我们在 100 万个目录上执行 statdir 来测量吞吐量，每台服务器使用 12 个核心，专用服务器利用 DPDK [53] 进行包处理。专用服务器达到 11 Mops/s 的吞吐量上限，而可编程交换机的吞吐量随元数据服务器数量线性扩展。理论上，可编程交换机最高可处理 4.8 Bops/s [22]，远超元数据集群的需求。因此，单个可编程交换机足以服务整个集群，而专用服务器则需要将指纹分片到多台机器才能满足吞吐量需求。

**用属主服务器跟踪目录状态（Tracking directory state with the owner server）。** 在属主服务器上维护目录状态而非使用可编程交换机，会使 mkdir 和 create 等双 inode 操作的峰值吞吐量降低约 10%，因为每个操作需要处理的数据包数量翻倍，消耗了宝贵的 CPU 资源。由于元数据性能受 CPU 限制，这会降低总体吞吐量。

此外，在属主服务器上跟踪目录状态会显著增加操作延迟。图 Fig. 16 展示了在中等和高负载下 create 操作的延迟分布。与原版 SwitchFS 相比，该变体在中等负载下中位数、p90 和 p99 延迟分别增加 23.2%、24.9% 和 107.4%；在高负载下分别增加 36.9%、80.8% 和 36.4%。目录更新关键路径上的额外服务器引入了额外的排队和队头阻塞（head-of-line blocking），放大了延迟——尤其是在高负载下。

=== PAGE 13 ===

### 7.4 操作突发性能（Operation Burst Performance）

在此评估中，我们评估 SwitchFS 如何处理时间性负载不均（temporal load imbalance），即应用程序在短时间内执行的一组空间相关的操作。我们将时间性负载不均建模为操作突发（operation burst），定义为同一目录中连续的文件创建操作。连续的操作突发在不同的目录中创建文件。SwitchFS 使用八台服务器，客户端发出 32 或 256 个在途请求以对服务器施加压力。我们观察改变突发大小时吞吐量的变化。

如图 Fig. 17 所示，E-InfiniFS 和 E-CFS 都对操作突发敏感。在 32 个在途请求的情况下（Fig. 17(a)），与突发大小 10 相比，E-InfiniFS 和 E-CFS 在突发大小为 50 时吞吐量分别下降 53.7% 和 47.1%，在突发大小为 1000 时分别下降 71.9% 和 66.1%。相比之下，SwitchFS 随着突发大小增加表现出稳定的性能，因为它将突发缓冲在变更日志中并在之后应用。图 Fig. 17(b) 展示了 256 个在途请求的结果，这对服务器施加了更大的压力。即使在这种场景下，SwitchFS 依然保持稳定的性能。这些实验证明了 SwitchFS 容忍时间性负载不均并维持稳定吞吐量的能力。

### 7.5 目录聚合开销（Directory Aggregation Overhead）

在 SwitchFS 中读取分散目录需要进行元数据聚合。为评估其开销，我们测量了在一系列文件创建操作之后执行 statdir 操作的延迟。在 statdir 操作期间，由之前 create 操作生成的变更日志条目被聚合（aggregated）并应用（applied）到目录。我们同时改变服务器数量和前置 create 操作的数量。

图 Fig. 18(a) 展示了 statdir 延迟如何随着前置 create 数量的增加而增长，最终收敛在大约 500 μs。原因是随着前置 create 数量的增加，聚合期间必须处理更多的变更日志条目。然而，每台服务器上的条目数量是有限的（在我们的实现中为 29），因为服务器一旦本地变更日志条目可以填满一个 MTU，就会主动将其推送给 inode。

图 Fig. 18(b) 展示了平均 statdir 延迟随服务器数量增加而增长。随着服务器增加，在触发主动聚合之前变更日志中可以保留更多条目，导致 statdir 期间聚合的条目更多。


| 工作负载 | 操作比例 |
|---|---|
| 数据中心服务 [45] | 52.6% open/close, 12.4% stat, 9.58% create, 11.9% delete, 9.3% file rename, 0.1% chmod, 3.9% readdir, 0.2% statdir |
| CNN 训练 | 42.8% open/close, 21.4% stat, 14.2% read, 7.1% write, 7.1% create, 7.1% delete, 0.1% mkdir, 0.1% rmdir, 0.1% statdir, 0.1% readdir |
| 缩略图 | 43.9% open/close, 21.9% stat, 12.2% read, 10.9% write, 10.9% create, 0.1% mkdir, 0.1% statdir, 0.1% readdir |

**Tab. 5: 真实工作负载追踪中的元数据操作比例。**

### 7.6 端到端性能（End-to-End Performance）

我们基于真实操作比例的综合工作负载和两个真实追踪（包含数据访问）评估端到端性能。本实验使用八台元数据服务器和八台数据节点，客户端发出 256 个在途请求。本评估中未发生脏集溢出。Tab. 5 总结了各工作负载的操作比例。

综合工作负载源自阿里巴巴部署的 PanguFS 追踪的操作比例，PanguFS 服务于多种数据中心服务，包括数据处理、对象存储和块存储[45]。基于这些比例，我们在 1024 个目录中执行 1000 万次操作，其中 80% 的操作集中在 20% 的目录上，以模拟工作负载偏斜。综合工作负载中不包含数据访问，因为相应的操作比例不可用。

CV Training 和 Thumbnail 工作负载是典型的多小文件工作负载。CV Training 工作负载来自在 ImageNet 数据集[20]上训练 ALEXNET 模型[27, 28]的追踪，包含 128 万个文件（大部分小于 256KB），分组到 1000 个目录中。该追踪捕获了数据集的完整生命周期，包括下载、访问和删除。Thumbnail 工作负载记录了访问 100 万张图像（大部分小于 256KB）并生成缩略图的过程。我们在启用和禁用数据访问的情况下重放追踪。

如图 Fig. 19 所示，SwitchFS 在元数据吞吐量上的提升转化为端到端吞吐量的显著提升。与 CephFS、E-InfiniFS 和 E-CFS 相比，SwitchFS 在元数据吞吐量上分别实现了最高 76.3 倍、2.1 倍和 0.7 倍的加速，在端到端性能上分别实现了最高 21.1 倍、1.1 倍和 0.3 倍的加速。

### 7.7 崩溃恢复时间（Crash Recovery Time）

**Fig. 19：端到端工作负载的吞吐量（Throughput of end-to-end workloads）。**

服务器需要 5.77 秒来恢复约 125 万个索引节点（inode）到其键值存储（key-value store）以及约 125 万条尚未应用的变更日志条目（change-log entries）。交换机故障后，所有服务器需要 3.82 秒来刷新尚未应用的变更日志条目，以便在交换机重启后将 SwitchFS 恢复为一致状态。注意，恢复时间与需要恢复的操作数量成正比，并且可以通过使用检查点（checkpointing）大幅缩短。在恢复过程中，端到端吞吐量为零。

# 8 相关工作（Related Work）

**元数据分区（Metadata partitioning）。** 早期的分布式文件系统如 GFS [13]、HDFS [66]、Farsite [11] 和 QFS [50] 在单一元数据服务器上管理所有元数据，导致可扩展性较差。较新的分布式文件系统将命名空间（namespace）划分为多个元数据服务器，以实现可扩展的元数据管理。一些分布式文件系统以子树（subtree）粒度划分目录树，例如 AFS [19] 和 CephFS [76]。子树划分保留了元数据局部性（locality），但容易出现负载不均衡 [64, 74]。另一些分布式文件系统使用每目录（per-directory）划分，例如 BeeGFS [15]、IndexFS [52, 57, 59]、HopsFS [47]、Tectonic [51] 和 InfiniFS [45]。由于它们仍将文件的 inode 与其父目录分组，因此在少数目录上密集执行操作时也会出现负载不均衡。CFS [75] 采用每文件（per-file）划分，实现了良好的负载均衡，但双 inode 操作（double-inode operations）会引入跨服务器协调开销。

以往的分布式文件系统在选择划分粒度时，不得不在负载均衡和操作开销之间进行权衡。相比之下，SwitchFS 以每文件粒度进行划分，同时通过异步元数据更新（asynchronous metadata updates）隐藏跨服务器协调开销，从而实现了良好的负载均衡。通过这种方式，SwitchFS 兼得了两者的优势。

**目录争用缓解（Directory contention mitigation）。** 先前的分布式文件系统探索了如何缓解目录争用。LocoFS [38] 放宽了 POSIX 语义，避免在双 inode 操作中更新目录元数据。DeltaFS [83] 不提供全局共享的命名空间，而是要求客户端显式管理同步范围，从而缩小争用域。CFS [75] 要求双 inode 操作进行跨服务器协调，同时使用有序更新和数据库原语（ordered updates and database primitives）来缩短目录更新的临界区（critical section）。SingularFS [16] 将元数据存储在单台服务器的非易失性主内存（NVMM）上，并使用原子指令（atomic instructions）更新目录以减少争用。SwitchFS 提供符合 POSIX 标准的全局命名空间，且不依赖 NVMM。SwitchFS 的变更日志压缩（change-log compaction）合并了并发目录更新，消除了目录更新的单点串行化，并且可以与先前的技术结合使用。

**网内加速（In-network acceleration）。** 此前已有研究探索了分布式系统中的网内协调（in-network coordination），包括键值缓存（key-value cache）[12, 23, 24, 31, 37, 39, 42, 44, 65, 67, 85]、网内计算（in-network computation）[18, 29, 40, 43, 62, 63]、分布式内存（distributed memory）[30, 73]、分布式锁（distributed locks）[79, 80]、共识与并发控制（consensus and concurrency control）[8, 9, 26, 35, 36, 68, 84] 以及调度（scheduling）[56, 70]。SwitchFS 的网内结构与 Harmonia [84] 类似。然而，Harmonia 针对快速键值存储复制的网内冲突检测，而 SwitchFS 采用网内状态追踪（in-network state tracking）以实现零开销的异步分布式文件系统操作。据我们所知，SwitchFS 是首个将网内优化引入分布式文件系统的方案。

**异步更新（Asynchronous update）。** 此前有研究 [41, 48, 83] 采用了数据和元数据的异步更新，SwitchFS 与它们有所不同。XSYNCFS [48] 和 POSIX-data-write [41] 采用了异步数据更新，但其方法不适用于元数据，因为它们允许崩溃后更新丢失，而 POSIX 语义禁止任何元数据更新的丢失。相比之下，SwitchFS 的异步协议是持久且一致的。DeltaFS [83] 探索了客户端之间的异步元数据同步。然而，它使用显式 API 来"获取"（get）和"发布"（publish）元数据更新，将同步负担留给了用户。相比之下，通过引入精心设计的协议和网内协调，SwitchFS 的异步元数据操作对用户透明且与 POSIX 兼容。

---

# 9 结论（Conclusion）

本文提出了 SwitchFS，一个采用异步元数据更新的分布式文件系统，以解决操作效率与负载均衡之间的权衡问题。SwitchFS 利用可编程交换机（programmable switch）在网络中追踪目录状态，并采用变更日志压缩（change-log compaction）来缓解目录元数据的争用。评估结果表明，SwitchFS 的性能优于现有最先进的系统。

---

# 致谢（Acknowledgements）

我们衷心感谢导师 Marc Shapiro 以及匿名审稿人给予的建设性意见和富有洞察力的建议。本工作部分受以下项目资助：国家重点研发计划（2024YFB4506200）、国家自然科学基金（No. 62132014）、中央高校基本科研业务费专项资金、教育部基础与交叉学科突破计划（JYB2025XDXM 113）以及华为技术有限公司。通讯作者为董明凯（mingkaidong@sjtu.edu.cn）。

---

# 参考文献（References）

[1] Cristina L. Abad, Huong Luu, Nathan Roberts, Kihwal Lee, Yi Lu, and Roy H. Campbell. 2012. Metadata Traces and Workload Models for Evaluating Big Storage Systems. In *Proceedings of the 2012 IEEE/ACM Fifth International Conference on Utility and Cloud Computing (UCC '12)*. IEEE Computer Society, Chicago, Illinois, USA, 125–132. doi:10.1109/UCC.2012.27

[2] Mehdi Ahmed-Nacer, Stéphane Martin, and Pascal Urso. 2012. File system on CRDT. arXiv:1207.5990 http://arxiv.org/abs/1207.5990

[3] Doug Beaver, Sanjeev Kumar, Harry C. Li, Jason Sobel, and Peter Vajgel. 2010. Finding a Needle in Haystack: Facebook's Photo Storage. In *Proceedings of the 9th USENIX Conference on Operating Systems Design and Implementation (OSDI'10)*. USENIX Association, Vancouver, BC, Canada, 47–60.

[4] Nikolaj Bjørner. 2007. Models and Software Model Checking of a Distributed File Replication System. Springer Berlin Heidelberg, Berlin, Heidelberg, 1–23. doi:10.1007/978-3-540-75221-9_1

[5] Philip Carns, Sam Lang, Robert Ross, Murali Vilayannur, Julian Kunkel, and Thomas Ludwig. 2009. Small-file access in parallel file systems. In *Proceedings of the 2009 IEEE International Symposium on Parallel&Distributed Processing (IPDPS '09)*. IEEE Computer Society, Rome, Italy, 1–11. doi:10.1109/IPDPS.2009.5161029

[6] Cisco. 2020. Cisco Nexus 34180YC and 3464C Programmable Switches Data Sheet.

[7] Austin T. Clements, M. Frans Kaashoek, Eddie Kohler, Robert T. Morris, and Nickolai Zeldovich. 2017. The scalable commutativity rule: designing scalable software for multicore processors. *Commun. ACM* 60, 8 (July 2017), 83–90. doi:10.1145/3068914

[8] Huynh Tu Dang, Pietro Bressana, Han Wang, Ki Suh Lee, Noa Zilberman, Hakim Weatherspoon, Marco Canini, Fernando Pedone, and Robert Soulé. 2020. P4xos: Consensus as a Network Service. *IEEE/ACM Transactions on Networking* 28, 4 (2020), 1726–1738. doi:10.1109/TNET.2020.2992106

[9] Huynh Tu Dang, Daniele Sciascia, Marco Canini, Fernando Pedone, and Robert Soulé. 2015. NetPaxos: consensus at network speed. In *Proceedings of the 1st ACM SIGCOMM Symposium on Software Defined Networking Research (SOSR '15)*. Association for Computing Machinery, Santa Clara, California, Article 5, 7 pages. doi:10.1145/2774993.2774999

[10] Shobhit Dayal. 2008. Characterizing HEC storage systems at rest. *Parallel Data Lab, CMU* (2008).

[11] John R. Douceur and Jon Howell. 2006. Distributed directory service in the Farsite file system. In *Proceedings of the 7th Symposium on Operating Systems Design and Implementation (OSDI '06)*. USENIX Association, Seattle, Washington, 321–334.

[12] Roy Friedman, Or Goaz, and Dor Hovav. 2023. PKache: A Generic Framework for Data Plane Caching. In *Proceedings of the 38th ACM/SIGAPP Symposium on Applied Computing (SAC '23)*. Association for Computing Machinery, Tallinn, Estonia, 1268–1276. doi:10.1145/3555776.3590826

[13] Sanjay Ghemawat, Howard Gobioff, and Shun-Tak Leung. 2003. The Google File System. In *Proceedings of the Nineteenth ACM Symposium on Operating Systems Principles (SOSP '03)*. Association for Computing Machinery, Bolton Landing, NY, USA, 29–43. doi:10.1145/945445.945450

[14] Gluster. 2019. Storage for Your Cloud. https://www.gluster.org/. Accessed: April 18, 2023.

[15] ThinkParQ GmbH. 2023. BeeGFS Documentation 7.4.2. https://doc.beegfs.io/latest/index.html. Accessed: November 17, 2023.

[16] Hao Guo, Youyou Lu, Wenhao Lv, Xiaojian Liao, Shaoxun Zeng, and Jiwu Shu. 2023. SingularFS: A Billion-Scale Distributed File System Using a Single Metadata Server. In *2023 USENIX Annual Technical Conference (USENIX ATC 23)*. USENIX Association, Boston, MA, 915–928. https://www.usenix.org/conference/atc23/presentation/guo

[17] Tyler Harter, Dhruba Borthakur, Siying Dong, Amitanand Aiyer, Liyin Tang, Andrea C. Arpaci-Dusseau, and Remzi H. Arpaci-Dusseau. 2014. Analysis of HDFS Under HBase: A Facebook Messages Case Study. In *Proceedings of the 12th USENIX Conference on File and Storage Technologies (FAST'14)*. USENIX Association, Santa Clara, CA, 199–212.

[18] Yongchao He, Wenfei Wu, Yanfang Le, Ming Liu, and ChonLam Lao. 2023. A Generic Service to Provide In-Network Aggregation for Key-Value Streams. In *Proceedings of the 28th ACM International Conference on Architectural Support for Programming Languages and Operating Systems, Volume 2 (ASPLOS 2023)*. Association for Computing Machinery, Vancouver, BC, Canada, 33–47. doi:10.1145/3575693.3575708

[19] J. Howard, M. Kazar, S. Menees, D. Nichols, M. Satyanarayanan, Robert N. Sidebotham, and M. West. 1987. Scale and Performance in a Distributed File System. *SIGOPS Oper. Syst. Rev.* 21, 5 (nov 1987), 1–2. doi:10.1145/37499.37500

[20] ImageNet. 2020. ImageNet Object Localization Challenge. https://www.kaggle.com/competitions/imagenet-object-localization-challenge/data. Accessed: November 19, 2023.

[21] Intel. 2023. Intel Tofino Series. https://www.intel.com/content/www/us/en/products/details/network-io/intelligent-fabric-processors/tofino.html. Accessed: September 5, 2023.

[22] Intel. 2023. Intel® Tofino 6.4 Tbps, 4 pipelines. https://www.intel.com/content/www/us/en/products/sku/218643/intel-tofino-6-4-tbps-4-pipelines/specifications.html. Accessed: November 19, 2023.

[23] Xin Jin, Xiaozhou Li, Haoyu Zhang, Nate Foster, Jeongkeun Lee, Robert Soulé, Changhoon Kim, and Ion Stoica. 2018. NetChain: Scale-Free Sub-RTT Coordination. In *15th USENIX Symposium on Networked Systems Design and Implementation (NSDI 18)*. USENIX Association, Renton, WA, 35–49. https://www.usenix.org/conference/nsdi18/presentation/jin

[24] Xin Jin, Xiaozhou Li, Haoyu Zhang, Robert Soulé, Jeongkeun Lee, Nate Foster, Changhoon Kim, and Ion Stoica. 2017. NetCache: Balancing Key-Value Stores with Fast In-Network Caching. In *Proceedings of the 26th Symposium on Operating Systems Principles (SOSP '17)*. Association for Computing Machinery, Shanghai, China, 121–136. doi:10.1145/3132747.3132764

[25] David Karger, Eric Lehman, Tom Leighton, Rina Panigrahy, Matthew Levine, and Daniel Lewin. 1997. Consistent hashing and random trees: distributed caching protocols for relieving hot spots on the World Wide Web. In *Proceedings of the Twenty-Ninth Annual ACM Symposium on Theory of Computing (STOC '97)*. Association for Computing Machinery, El Paso, Texas, USA, 654–663. doi:10.1145/258533.258660

[26] Gyuyeong Kim and Wonjun Lee. 2022. In-Network Leaderless Replication for Distributed Data Stores. *Proc. VLDB Endow.* 15, 7 (mar 2022), 1337–1349. doi:10.14778/3523210.3523213

[27] Alex Krizhevsky. 2014. One weird trick for parallelizing convolutional neural networks. *CoRR* abs/1404.5997 (2014). arXiv:1404.5997 http://arxiv.org/abs/1404.5997

[28] Alex Krizhevsky, Ilya Sutskever, and Geoffrey E. Hinton. 2017. ImageNet classification with deep convolutional neural networks. *Commun. ACM* 60, 6, 84–90. doi:10.1145/3065386

[29] ChonLam Lao, Yanfang Le, Kshiteej Mahajan, Yixi Chen, Wenfei Wu, Aditya Akella, and Michael Swift. 2021. ATP: In-network Aggregation for Multi-tenant Learning. In *18th USENIX Symposium on Networked Systems Design and Implementation (NSDI 21)*. USENIX Association, Virtual Event, USA, 741–761. https://www.usenix.org/conference/nsdi21/presentation/lao

[30] Seung-seob Lee, Yanpeng Yu, Yupeng Tang, Anurag Khandelwal, Lin Zhong, and Abhishek Bhattacharjee. 2021. MIND: In-Network Memory Management for Disaggregated Data Centers. In *Proceedings of the ACM SIGOPS 28th Symposium on Operating Systems Principles (SOSP '21)*. Association for Computing Machinery, Virtual Event, Germany, 488–504. doi:10.1145/3477132.3483561

[31] Jason Lei and Vishal Shrivastav. 2024. Seer: Enabling Future-Aware Online Caching in Networked Systems. In *21st USENIX Symposium on Networked Systems Design and Implementation (NSDI 24)*. USENIX Association, Santa Clara, CA, 635–649. https://www.usenix.org/conference/nsdi24/presentation/lei

[32] Paul Lensing, Dirk Meister, and André Brinkmann. 2010. hashFS: Applying Hashing to Optimize File Systems for Small File Reads. In *2010 International Workshop on Storage Network Architecture and Parallel I/Os*. IEEE Computer Society, Los Alamitos, CA, USA, 33–42. doi:10.1109/SNAPI.2010.12

[33] Cheng Li, Daniel Porto, Allen Clement, Johannes Gehrke, Nuno Preguiça, and Rodrigo Rodrigues. 2012. Making Geo-Replicated Systems Fast as Possible, Consistent when Necessary. In *10th USENIX Symposium on Operating Systems Design and Implementation (OSDI 12)*. USENIX Association, Hollywood, CA, 265–278. https://www.usenix.org/conference/osdi12/technical-sessions/presentation/li

[34] Jiahao Li, Biao Cao, Jielong Jian, Cheng Li, Sen Han, Yiduo Wang, Yufei Wu, Kang Chen, Zhihui Yin, Qiushi Chen, Jiwei Xiong, Jie Zhao, Fengyuan Liu, Yan Xing, Liguo Duan, Miao Yu, Ran Zheng, Feng Wu, and Xianjun Meng. 2025. Mantle: Efficient Hierarchical Metadata Management for Cloud Object Storage Services. In *Proceedings of the ACM SIGOPS 31st Symposium on Operating Systems Principles (SOSP '25)*. Association for Computing Machinery, Seoul, South Korea.

[35] Jialin Li, Ellis Michael, and Dan R. K. Ports. 2017. Eris: Coordination-Free Consistent Transactions Using In-Network Concurrency Control. In *Proceedings of the 26th Symposium on Operating Systems Principles (SOSP '17)*. Association for Computing Machinery, Shanghai, China, 104–120. doi:10.1145/3132747.3132751

[36] Jialin Li, Ellis Michael, Naveen Kr. Sharma, Adriana Szekeres, and Dan R. K. Ports. 2016. Just Say NO to Paxos Overhead: Replacing Consensus with Network Ordering. In *12th USENIX Symposium on Operating Systems Design and Implementation (OSDI 16)*. USENIX Association, Savannah, GA, 467–483. https://www.usenix.org/conference/osdi16/technical-sessions/presentation/li

[37] Jialin Li, Jacob Nelson, Ellis Michael, Xin Jin, and Dan R. K. Ports. 2020. Pegasus: Tolerating skewed workloads in distributed storage with in-network coherence directories. In *Proceedings of the 14th USENIX Conference on Operating Systems Design and Implementation (OSDI'20)*. USENIX Association, Virtual Event, USA, Article 22, 20 pages.

[38] Siyang Li, Youyou Lu, Jiwu Shu, Yang Hu, and Tao Li. 2017. LocoFS: A Loosely-Coupled Metadata Service for Distributed File Systems. In *Proceedings of the International Conference for High Performance Computing, Networking, Storage and Analysis (SC '17)*. Association for Computing Machinery, Denver, Colorado, Article 4, 12 pages. doi:10.1145/3126908.3126928

[39] Xiaozhou Li, Raghav Sethi, Michael Kaminsky, David G. Andersen, and Michael J. Freedman. 2016. Be Fast, Cheap and in Control with SwitchKV. In *13th USENIX Symposium on Networked Systems Design and Implementation (NSDI 16)*. USENIX Association, Santa Clara, CA, 31–44. https://www.usenix.org/conference/nsdi16/technical-sessions/presentation/li-xiaozhou

[40] Zhaoyi Li, Jiawei Huang, Yijun Li, Aikun Xu, Shengwen Zhou, Jingling Liu, and Jianxin Wang. 2023. A2TP: Aggregator-aware In-network Aggregation for Multi-tenant Learning. In *Proceedings of the Eighteenth European Conference on Computer Systems (EuroSys '23)*. Association for Computing Machinery, Rome, Italy, 639–653. doi:10.1145/3552326.3587436

[41] Linux. 2025. openat(2) - Linux manual page 6.15. Linux man-pages project. https://man7.org/linux/man-pages/man2/openat.2.html Accessed: 2025-9-12.

[42] Ming Liu, Liang Luo, Jacob Nelson, Luis Ceze, Arvind Krishnamurthy, and Kishore Atreya. 2017. IncBricks: Toward In-Network Computation with an In-Network Cache. In *Proceedings of the Twenty-Second International Conference on Architectural Support for Programming Languages and Operating Systems (ASPLOS '17)*. Association for Computing Machinery, Xi'an, China, 795–809. doi:10.1145/3037697.3037731

[43] Shuo Liu, Qiaoling Wang, Junyi Zhang, Wenfei Wu, Qinliang Lin, Yao Liu, Meng Xu, Marco Canini, Ray C. C. Cheung, and Jianfei He. 2023. In-Network Aggregation with Transport Transparency for Distributed Training. In *Proceedings of the 28th ACM International Conference on Architectural Support for Programming Languages and Operating Systems, Volume 3 (ASPLOS 2023)*. Association for Computing Machinery, Vancouver, BC, Canada, 376–391. doi:10.1145/3582016.3582037

[44] Zaoxing Liu, Zhihao Bai, Zhenming Liu, Xiaozhou Li, Changhoon Kim, Vladimir Braverman, Xin Jin, and Ion Stoica. 2019. DistCache: Provable Load Balancing for Large-Scale Storage Systems with Distributed Caching. In *17th USENIX Conference on File and Storage Technologies (FAST 19)*. USENIX Association, Boston, MA, 143–157. https://www.usenix.org/conference/fast19/presentation/liu

[45] Wenhao Lv, Youyou Lu, Yiming Zhang, Peile Duan, and Jiwu Shu. 2022. InfiniFS: An Efficient Metadata Service for Large-Scale Distributed Filesystems. In *20th USENIX Conference on File and Storage Technologies (FAST 22)*. USENIX Association, Santa Clara, CA, 313–328. https://www.usenix.org/conference/fast22/presentation/lv

[46] Peter Macko and Jason Hennessey. 2022. Survey of Distributed File System Design Choices. *ACM Trans. Storage* 18, 1, Article 4 (mar 2022), 34 pages. doi:10.1145/3465405

[47] Salman Niazi, Mahmoud Ismail, Seif Haridi, Jim Dowling, Steffen Grohsschmiedt, and Mikael Ronström. 2017. HopsFS: Scaling Hierarchical File System Metadata Using NewSQL Databases. In *15th USENIX Conference on File and Storage Technologies (FAST 17)*. USENIX Association, Santa Clara, CA, 89–104. https://www.usenix.org/conference/fast17/technical-sessions/presentation/niazi

[48] Edmund B. Nightingale, Kaushik Veeraraghavan, Peter M. Chen, and Jason Flinn. 2008. Rethink the sync. *ACM Trans. Comput. Syst.* 26, 3, Article 6 (Sept. 2008), 26 pages. doi:10.1145/1394441.1394442

[49] OpenSwitch. 2019. Cavium XPliant. https://www.openswitch.net/cavium/. Accessed: September 19, 2023.

[50] Michael Ovsiannikov, Silvius Rus, Damian Reeves, Paul Sutter, Sriram Rao, and Jim Kelly. 2013. The quantcast file system. *Proc. VLDB Endow.* 6, 11 (aug 2013), 1092–1101. doi:10.14778/2536222.2536234

[51] Satadru Pan, Theano Stavrinos, Yunqiao Zhang, Atul Sikaria, Pavel Zakharov, Abhinav Sharma, Shiva Shankar P, Mike Shuey, Richard Wareing, Monika Gangapuram, Guanglei Cao, Christian Preseau, Pratap Singh, Kestutis Patiejunas, JR Tipton, Ethan Katz-Bassett, and Wyatt Lloyd. 2021. Facebook's Tectonic Filesystem: Efficiency from Exascale. In *19th USENIX Conference on File and Storage Technologies (FAST 21)*. USENIX Association, Virtual Event, USA, 217–231. https://www.usenix.org/conference/fast21/presentation/pan

[52] Swapnil Patil and Garth Gibson. 2011. Scale and concurrency of GIGA+: file system directories with millions of files. In *Proceedings of the 9th USENIX Conference on File and Stroage Technologies (FAST'11)*. USENIX Association, San Jose, California, 13.

[53] DPDK Project. 2023. The Open Source Data Plane Development Kit Accelerating Network Performance. https://www.dpdk.org/. Accessed: November 19, 2023.

[54] Yingjin Qian, Wen Cheng, Lingfang Zeng, Xi Li, Marc-André Vef, Andreas Dilger, Siyao Lai, Shuichi Ihara, Yong Fan, and André Brinkmann. 2023. Xfast: Extreme File Attribute Stat Acceleration for Lustre. In *Proceedings of the International Conference for High Performance Computing, Networking, Storage and Analysis (SC '23)*. Association for Computing Machinery, Denver, CO, USA, Article 96, 12 pages. doi:10.1145/3581784.3607080

[55] Yingjin Qian, Wen Cheng, Lingfang Zeng, Marc-André Vef, Oleg Drokin, Andreas Dilger, Shuichi Ihara, Wusheng Zhang, Yang Wang, and André Brinkmann. 2022. MetaWBC: POSIX-compliant metadata write-back caching for distributed file systems. In *Proceedings of the International Conference on High Performance Computing, Networking, Storage and Analysis (SC '22)*. IEEE Press, Dallas, Texas, Article 56, 20 pages.

[56] Benjamin Reidys, Yuqi Xue, Daixuan Li, Bharat Sukhwani, Wen-Mei Hwu, Deming Chen, Sameh Asaad, and Jian Huang. 2023. RackBlox: A Software-Defined Rack-Scale Storage System with Network-Storage Co-Design. In *Proceedings of the 29th Symposium on Operating Systems Principles (SOSP '23)*. Association for Computing Machinery, Koblenz, Germany, 182–199. doi:10.1145/3600006.3613170

[57] Kai Ren and Garth Gibson. 2013. TABLEFS: Enhancing Metadata Efficiency in the Local File System. In *2013 USENIX Annual Technical Conference (USENIX ATC 13)*. USENIX Association, San Jose, CA, 145–156. https://www.usenix.org/conference/atc13/technical-sessions/presentation/ren

[58] Kai Ren, YongChul Kwon, Magdalena Balazinska, and Bill Howe. 2013. Hadoop's adolescence: an analysis of Hadoop usage in scientific workloads. *Proc. VLDB Endow.* 6, 10 (Aug. 2013), 853–864. doi:10.14778/2536206.2536213

[59] Kai Ren, Qing Zheng, Swapnil Patil, and Garth Gibson. 2014. IndexFS: scaling file system metadata performance with stateless caching and bulk insertion. In *Proceedings of the International Conference for High Performance Computing, Networking, Storage and Analysis (SC '14)*. IEEE Press, New Orleans, Louisana, 237–248. doi:10.1109/SC.2014.25

[60] RocksDB. 2021. A persistent key-value store for fast storage environments. http://rocksdb.org. Accessed: April 18, 2023.

[61] Drew Roselli, Jacob R. Lorch, and Thomas E. Anderson. 2000. A comparison of file system workloads. In *Proceedings of the Annual Conference on USENIX Annual Technical Conference (ATEC '00)*. USENIX Association, San Diego, California, 4.

[62] Amedeo Sapio, Ibrahim Abdelaziz, Abdulla Aldilaijan, Marco Canini, and Panos Kalnis. 2017. In-Network Computation is a Dumb Idea Whose Time Has Come. In *Proceedings of the 16th ACM Workshop on Hot Topics in Networks (HotNets '17)*. Association for Computing Machinery, Palo Alto, CA, USA, 150–156. doi:10.1145/3152434.3152461

[63] Amedeo Sapio, Marco Canini, Chen-Yu Ho, Jacob Nelson, Panos Kalnis, Changhoon Kim, Arvind Krishnamurthy, Masoud Moshref, Dan Ports, and Peter Richtarik. 2021. Scaling Distributed Machine Learning with In-Network Aggregation. In *18th USENIX Symposium on Networked Systems Design and Implementation (NSDI 21)*. USENIX Association, 785–808. https://www.usenix.org/conference/nsdi21/presentation/sapio

[64] Michael A. Sevilla, Noah Watkins, Carlos Maltzahn, Ike Nassi, Scott A. Brandt, Sage A. Weil, Greg Farnum, and Sam Fineberg. 2015. Mantle: a programmable metadata load balancer for the ceph file system. In *Proceedings of the International Conference for High Performance Computing, Networking, Storage and Analysis (SC '15)*. Association for Computing Machinery, Austin, Texas, Article 21, 12 pages. doi:10.1145/2807591.2807607

[65] Siyuan Sheng, Huancheng Puyang, Qun Huang, Lu Tang, and Patrick P. C. Lee. 2023. FarReach: Write-back Caching in Programmable Switches. In *2023 USENIX Annual Technical Conference (USENIX ATC 23)*. USENIX Association, Boston, MA, 571–584. https://www.usenix.org/conference/atc23/presentation/sheng

[66] Konstantin Shvachko, Hairong Kuang, Sanjay Radia, and Robert Chansler. 2010. The Hadoop Distributed File System. In *2010 IEEE 26th Symposium on Mass Storage Systems and Technologies (MSST)*. IEEE, Incline Village, NV, USA, 1–10. doi:10.1109/MSST.2010.5496972

[67] Y. Su, D. Feng, Y. Hua, Z. Shi, and T. Zhu. 2018. NetRS: Cutting Response Latency in Distributed Key-Value Stores with In-Network Replica Selection. In *2018 IEEE 38th International Conference on Distributed Computing Systems (ICDCS)*. IEEE Computer Society, Los Alamitos, CA, USA, 143–153. doi:10.1109/ICDCS.2018.00024

[68] Hatem Takruri, Ibrahim Kettaneh, Ahmed Alquraan, and Samer Al-Kiswany. 2020. FLAIR: Accelerating Reads with Consistency-Aware Network Routing. In *17th USENIX Symposium on Networked Systems Design and Implementation (NSDI 20)*. USENIX Association, Santa Clara, CA, 723–737. https://www.usenix.org/conference/nsdi20/presentation/takruri

[69] Vinh Tao, Marc Shapiro, and Vianney Rancurel. 2015. Merging semantics for conflict updates in geo-distributed file systems. In *Proceedings of the 8th ACM International Systems and Storage Conference (SYSTOR '15)*. Association for Computing Machinery, Haifa, Israel, Article 10, 12 pages. doi:10.1145/2757667.2757683

[70] Sreeharsha Udayashankar, Ashraf Abdel-Hadi, Ali Mashtizadeh, and Samer Al-Kiswany. 2024. Draconis: Network-Accelerated Scheduling for Microsecond-Scale Workloads. In *Proceedings of the Nineteenth European Conference on Computer Systems (EuroSys '24)*. Association for Computing Machinery, Athens, Greece, 333–348. doi:10.1145/3627703.3650060

[71] Romain Vaillant, Dimitrios Vasilas, Marc Shapiro, and Thuy Linh Nguyen. 2021. CRDTs for truly concurrent file systems. In *Proceedings of the 13th ACM Workshop on Hot Topics in Storage and File Systems (HotStorage '21)*. Association for Computing Machinery, Virtual, USA, 35–41. doi:10.1145/3465332.3470872

[72] Marc-André Vef, Rebecca Steiner, Reza Salkhordeh, Jörg Steinkamp, Florent Vennetier, Jean-François Smigielski, and André Brinkmann. 2020. DelveFS - An Event-Driven Semantic File System for Object Stores. In *2020 IEEE International Conference on Cluster Computing (CLUSTER)*. IEEE, Kobe, Japan, 35–46. doi:10.1109/CLUSTER49012.2020.00014

[73] Qing Wang, Youyou Lu, Erci Xu, Junru Li, Youmin Chen, and Jiwu Shu. 2021. Concordia: Distributed Shared Memory with In-Network Cache Coherence. In *19th USENIX Conference on File and Storage Technologies (FAST 21)*. USENIX Association, 277–292. https://www.usenix.org/conference/fast21/presentation/wang

[74] Yiduo Wang, Cheng Li, Xinyang Shao, Youxu Chen, Feng Yan, and Yinlong Xu. 2021. Lunule: An Agile and Judicious Metadata Load Balancer for CephFS. In *Proceedings of the International Conference for High Performance Computing, Networking, Storage and Analysis (SC '21)*. Association for Computing Machinery, St. Louis, Missouri, Article 47, 16 pages. doi:10.1145/3458817.3476196

[75] Yiduo Wang, Yufei Wu, Cheng Li, Pengfei Zheng, Biao Cao, Yan Sun, Fei Zhou, Yinlong Xu, Yao Wang, and Guangjun Xie. 2023. CFS: Scaling Metadata Service for Distributed File System via Pruned Scope of Critical Sections. In *Proceedings of the Eighteenth European Conference on Computer Systems (EuroSys '23)*. Association for Computing Machinery, Rome, Italy, 331–346. doi:10.1145/3552326.3587443

[76] Sage A. Weil, Scott A. Brandt, Ethan L. Miller, Darrell D. E. Long, and Carlos Maltzahn. 2006. Ceph: A Scalable, High-Performance Distributed File System. In *Proceedings of the 7th Symposium on Operating Systems Design and Implementation (OSDI '06)*. USENIX Association, Seattle, Washington, 307–320.

[77] Jingwei Xu, Junbin Kang, Mingkai Dong, Mingyu Liu, Lu Zhang, Shaohong Guo, Ziyan Qiu, Mingzhen You, Ziyi Tian, Anqi Yu, Tianhong Ding, Xinwei Hu, and Haibo Chen. 2025. FalconFS: Distributed File System for Large-Scale Deep Learning Pipeline. arXiv:2507.10367 [cs.DC] https://arxiv.org/abs/2507.10367

[78] Elena Yanakieva, Michael Youssef, Ahmad Hussein Rezae, and Annette Bieniusa. 2021. Access Control Conflict Resolution in Distributed File Systems using CRDTs. In *Proceedings of the 8th Workshop on Principles and Practice of Consistency for Distributed Data (PaPoC '21)*. Association for Computing Machinery, Virtual Event, United Kingdom, Article 1, 3 pages. doi:10.1145/3447865.3457970

[79] Zhuolong Yu, Yiwen Zhang, Vladimir Braverman, Mosharaf Chowdhury, and Xin Jin. 2020. NetLock: Fast, Centralized Lock Management Using Programmable Switches. In *Proceedings of the Annual Conference of the ACM Special Interest Group on Data Communication on the Applications, Technologies, Architectures, and Protocols for Computer Communication (SIGCOMM '20)*. Association for Computing Machinery, Virtual Event, USA, 126–138. doi:10.1145/3387514.3405857

[80] Hanze Zhang, Ke Cheng, Rong Chen, and Haibo Chen. 2024. Fast and Scalable In-network Lock Management Using Lock Fission. In *18th USENIX Symposium on Operating Systems Design and Implementation (OSDI 24)*. USENIX Association, Santa Clara, CA, 251–268. https://www.usenix.org/conference/osdi24/presentation/zhang-hanze

[81] Shuanglong Zhang, Robert Roy, Leah Rumancik, and An-I Andy Wang. 2020. The Composite-File File System: Decoupling One-to-One Mapping of Files and Metadata for Better Performance. *ACM Trans. Storage* 16, 1, Article 5 (March 2020), 18 pages. doi:10.1145/3366684

[82] Nannan Zhao, Vasily Tarasov, Hadeel Albahar, Ali Anwar, Lukas Rupprecht, Dimitrios Skourtis, Arnab K. Paul, Keren Chen, and Ali R. Butt. 2021. Large-Scale Analysis of Docker Images and Performance Implications for Container Storage Systems. *IEEE Trans. Parallel Distrib. Syst.* 32, 4 (April 2021), 918–930. doi:10.1109/TPDS.2020.3034517

[83] Qing Zheng, Charles D. Cranor, Gregory R. Ganger, Garth A. Gibson, George Amvrosiadis, Bradley W. Settlemyer, and Gary A. Grider. 2021. DeltaFS: a scalable no-ground-truth filesystem for massively-parallel computing. In *Proceedings of the International Conference for High Performance Computing, Networking, Storage and Analysis (SC '21)*. Association for Computing Machinery, St. Louis, Missouri, Article 48, 15 pages. doi:10.1145/3458817.3476148

[84] Hang Zhu, Zhihao Bai, Jialin Li, Ellis Michael, Dan R. K. Ports, Ion Stoica, and Xin Jin. 2019. Harmonia: near-linear scalability for replicated storage with in-network conflict detection. *Proc. VLDB Endow.* 13, 3 (nov 2019), 376–389. doi:10.1145/3368289.3368301

[85] Zeying Zhu, Yibo Zhao, and Zaoxing Liu. 2024. In-Memory Key-Value Store Live Migration with NetMigrate. In *22nd USENIX Conference on File and Storage Technologies (FAST 24)*. USENIX Association, Santa Clara, CA, 209–224. https://www.usenix.org/conference/fast24/presentation/zhu

---

# 附录 A（Appendix A）

在本附录中，我们详细讨论 SwitchFS 的正确性和一致性（correctness and consistency）。

## A.1 崩溃后的正确性（Correctness After a Crash）

在本小节中，我们讨论 SwitchFS 如何在服务器崩溃、交换机崩溃或两者同时崩溃时保持一致性，并提供一个服务器在聚合（aggregation）过程中发生故障的具体示例。

**交换机故障（Switch failure）。** 当交换机从故障中恢复时，它会初始化一个空的脏集合（dirty set），各服务器聚合所有目录。由此，SwitchFS 恢复为一致状态。

**服务器故障（Server failure）。** 当服务器从故障中恢复时，它按以下方式恢复其易失状态（volatile states）：从其本地 WAL 中恢复键值存储和变更日志，并通过克隆其他服务器上的失效列表（invalidation）来恢复失效信息。注意，每个尚未聚合的目录更新都存储在变更日志所有者服务器或目录所有者服务器的 WAL 中，因此不会丢失任何更新。该服务器还会主动聚合其拥有的所有目录，以确保其在崩溃前发出的任何被中断的聚合（其对应的指纹可能在交换机上已被移除）能够执行完成，从而使交换机上的脏集合正确反映目录的状态。

**聚合过程中服务器故障的示例（Example of server failure during aggregation）。** 让我们考虑一个聚合过程中服务器故障的示例，涉及服务器 A（托管目录 D 的 inode）和服务器 B（托管目录 D 的一个变更日志）。

考虑在某一时刻一台或两台服务器崩溃时，某些变更日志条目可能已写入服务器 A 的 WAL，其中一部分在服务器 B 的 WAL 中被标记为"已应用"（applied）。

如果服务器 A 崩溃，它将已写入服务器 A WAL 中的变更日志条目应用于目录 D 的 inode。然后，服务器 A 主动聚合其拥有的所有目录（包括 D），从服务器 B 检索 D 剩余的变更日志条目。

如果服务器 B 崩溃，服务器 A 的聚合会超时（因为未收到服务器 B 关于所有变更日志条目已检索完毕的信号）并开始重试。服务器 B 随后从其 WAL 中恢复所有未标记为"已应用"的变更日志条目，服务器 A 在重试时聚合这些条目。值得注意的是，服务器 A 可能收到重复的条目，它利用条目 ID（entry IDs）确保每个条目只被处理一次。

如果两台服务器都崩溃，服务器 A 将已写入服务器 A WAL 中的变更日志条目应用于目录 D 的 inode，服务器 B 从服务器 B 的 WAL 中恢复剩余的变更日志条目。然后，服务器 A 主动聚合其拥有的所有目录（包括 D），检索 D 的所有变更日志条目。

**恢复的幂等性（Idempotence of recovery）。** 上述描述的恢复过程是幂等的（idempotent）。首先，易失状态的恢复（即重建键值存储、变更日志和失效列表）是幂等的，因为它总是在崩溃后从干净状态开始。其次，聚合是幂等的，因为：(a) 目录所有者服务器在发送方服务器将其 WAL 中的变更日志条目标记为"已应用"之前，就已将该条目持久化到自己的 WAL 中，确保不会丢失任何变更日志条目；(b) 相同变更日志条目的重复聚合不会损害正确性，因为服务器会检查条目 ID，确保每个条目只被处理一次。因此，嵌套崩溃（nested crashes）不会损害正确性。

## A.2 操作一致性与性质（Operation Consistency and Properties）

在本小节中，我们首先证明 SwitchFS 在无崩溃环境下是可串行化（serializable）的，并保持了实时顺序（real-time order）。然后，我们展示 SwitchFS 在存在崩溃的情况下仍能保持可串行化。

### A.2.1 无崩溃环境下的一致性（Consistency in a Crash-Free Environment）

我们证明 SwitchFS 确保以下两个性质（当无崩溃发生时）：

- 元数据操作是可串行化的。
- 任何在操作 α 返回之后发出的元数据操作 β 都能看到 α 所做的更改。

**性质 1：元数据操作是可串行化的。**

在 SwitchFS 中，同步操作（如 open、read、stat）通过加锁（locking）进行串行化（与传统分布式文件系统一样），而两个异步更新操作（如 mkdir、create）根据它们获取父目录变更日志写锁的顺序进行串行化。下面我们讨论异步更新和目录读取是如何串行化的。

考虑两个操作 α 和 β：α 对一个目录执行异步更新，β 读取同一目录。不失一般性，设 α 为 `create(/a/b)`，β 为 `readdir(/a)`。假设服务器 A 托管目录 /a 的 inode，服务器 B 托管文件 /a/b 的 inode。

α 和 β 的执行可能有多种交织方式。我们根据 β 在交换机上的脏集合中观察到目录 /a 为 **scattered**（分散）或 **normal**（正常）来对这些交织进行分类。

**情况 1：β 在查询脏集合时发现 /a 被标记为 scattered。** 根据 α 在 β 获取锁之前还是之后更新变更日志，又分为两个子情况。Fig. 20 展示了它们各自的工作流程。

- **情况 1.a：** α 在 β 获取锁（➂）之前更新变更日志（➀），如 Fig. 20(a) 所示。在这种情况下，β 在聚合期间获取了 α 的更新，从而观察到其效果，将 α 串行化在 β 之前。
- **情况 1.b：** α 在 β 获取锁（➂）之后更新变更日志（➃），如 Fig. 20(b) 所示。在这种情况下，β 在聚合期间未获取 α 的更新，因此未观察到其效果，将 α 串行化在 β 之后。值得注意的是，在此交织中存在一个 happens-before 关系链：(a) α 在追加变更日志（➃）之后将 /a 的指纹插入脏集合（➄）；(b) β 在获取变更日志锁（➂）之前将指纹从脏集合中移除（➁）。因此，α 的指纹插入（➄）严格发生在 β 的指纹移除（➁）之后。结果是，后续的目录读取将检测到 α 插入的指纹并启动聚合，从而获取 α 的更新，使 α 的效果可见。

**情况 2：β 在查询脏集合时发现 /a 被标记为 normal。** 根据 β 是否观察到 α 的效果，又分为两个子情况。Fig. 21 展示了它们各自的工作流程。

- **情况 2.a：** β 读取服务器 A 上目录 /a 的 inode 并返回，未观察到 α 的更新，如 Fig. 21(a) 所示。在这种情况下，α 被串行化在 β 之后。值得注意的是，由于 β 在脏集合中发现 /a 被标记为"normal"，β 一定在 α 插入 /a 的指纹（➁）之前查询了脏集合（➀）。
- **情况 2.b：** 另一个操作 γ 在 α 更新变更日志之后、β 读取 inode 之前触发了一次聚合，如 Fig. 21(b) 所示。因此，β 观察到了 α 的效果，α 被串行化在 β 之前。具体来说，β 在 α 插入 /a 的指纹（➁）之前查询脏集合（➀）。随后，γ 查询脏集合（➂）并对目录 /a 启动聚合（➃），在此过程中 /a 的 inode 被锁定。β 的请求在 γ 获取锁之后到达服务器 A，等待聚合完成（➄、➅、➆），然后读取 /a 的 inode 并返回（➇）。这种情况之所以可能，是因为网络中的数据包可能存在任意重排序。

在上述所有场景中，操作都被正确串行化，消除了任何检查-使用时间差（Time-Of-Check-To-Time-Of-Use, TOCTTOU）问题。

**性质 2：任何在操作 α 返回之后发出的元数据操作 β 都能看到 α 所做的更改。**

在 SwitchFS 中，同步操作在返回前对对象进行原地更新（in-place updates），因此其效果对后续操作可见，与传统分布式文件系统一样。下面我们解释异步更新和目录读取如何保持这一性质。

在 Fig. 22 中，操作 α 对目录 /a 执行异步目录更新，操作 β 和 γ 读取目录 /a。我们将 α 返回之后发出的目录读取分为两类——基于它们在脏集合中观察到目录为 scattered 或 normal——操作 β 和 γ 是其中的示例。下面我们说明为什么这两类操作都必然观察到 α 所做的更改。

- 如果目录读取（即操作 β）在查询脏集合时（➄）发现目录被标记为 scattered，它会触发聚合，获取 /a 的更新。根据 Fig. 22 中标记的事件因果顺序，β 必定在 α 更新变更日志（➀）之后才获取变更日志上的锁（➆）。因此，如性质 1 的情况 1.a 所述，β 保证能看到 α 的效果。
- 如果目录读取（即操作 γ）在查询脏集合时发现目录被标记为 normal，则表明该目录的指纹已被之前的聚合（➅）从脏集合中移除。由于聚合在完成之前会阻塞目录读取，γ 将看到聚合获取的目录更新，包括 α 的更新。

### A.2.2 存在崩溃时的一致性（Consistency in the Presence of Crashes）

我们证明在存在崩溃的情况下操作仍然是可串行化的，如下所述。

当崩溃发生时，被中断的操作不会返回，客户端在恢复后重试它们。被中断的元数据读取和未提交的更新对客户端不可见，且不留下副作用，因此不影响操作的可串行化。我们只需要考虑恢复后已提交的元数据更新操作的顺序。

§A.1 中的恢复过程按照其原始顺序恢复操作。

---

## A.3 集群重配置期间的容错

> 原文结尾处 A.3 部分内容未在提供的文件中完整出现。上述翻译覆盖了文件中存在的全部内容。
