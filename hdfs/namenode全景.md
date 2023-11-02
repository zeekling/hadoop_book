
## 简介

NameNode管理着整个HDFS文件系统的元数据。
从架构设计上看，元数据大致分成两个层次：Namespace管理层，负责管理文件系统中的树状目录结构以及文件与数据块的映射关系；
块管理层，负责管理文件系统中文件的物理块与实际存储位置的映射关系BlocksMap，如图1所示。
Namespace管理的元数据除内存常驻外，也会周期Flush到持久化设备上FsImage文件；BlocksMap元数据只在内存中存在；
当NameNode发生重启，首先从持久化设备中读取FsImage构建Namespace，之后根据DataNode的汇报信息重新构造BlocksMap。
这两部分数据结构是占据了NameNode大部分JVM Heap空间。

![pic](https://pan.zeekling.cn/zeekling/hadoop/nn_0001.png)

除了对文件系统本身元数据的管理之外，NameNode还需要维护整个集群的机架及DataNode的信息、Lease管理以及集中式缓存引入的缓存管理等等。
这几部分数据结构空间占用相对固定，且占用较小。


## 内存全景

NameNode整个内存结构大致可以分成四大部分：Namespace、BlocksMap、NetworkTopology及其它

![pic](https://pan.zeekling.cn/zeekling/hadoop/nn_0002.png)

- Namespace：维护整个文件系统的目录树结构及目录树上的状态变化；
- BlockManager：维护整个文件系统中与数据块相关的信息及数据块的状态变化；
- NetworkTopology：维护机架拓扑及DataNode信息，机架感知的基础；
- 其他:
  - LeaseManager：读写的互斥同步就是靠Lease实现，支持HDFS的Write-Once-Read-Many的核心数据结构；
  - CacheManager：Hadoop 2.3.0引入的集中式缓存新特性，支持集中式缓存的管理，实现memory-locality提升读性能；
  - SnapshotManager：Hadoop 2.1.0引入的Snapshot新特性，用于数据备份、回滚，以防止因用户误操作导致集群出现数据问题；
  - DelegationTokenSecretManager：管理HDFS的安全访问； 另外还有临时数据信息、统计信息metrics等等

NameNode常驻内存主要被Namespace和BlockManager使用，二者使用占比分别接近50%。其它部分内存开销较小且相对固定，与Namespace和BlockManager相比基本可以忽略。


## 内存分析

HDFS对文件系统的目录结构也是按照树状结构维护，Namespace保存了目录树及每个目录/文件节点的属性。
除在内存常驻外，这部分数据会定期flush到持久化设备上，生成一个新的FsImage文件，方便NameNode发生重启时，从FsImage及时恢复整个Namespace。
下图所示为Namespace内存结构。前述集群中目录和文件总量即整个Namespace目录树中包含的节点总数，可见Namespace本身其实是一棵非常巨大的树。

![pic](https://pan.zeekling.cn/zeekling/hadoop/nn_0003.png)


在整个Namespace目录树中存在两种不同类型的INode数据结构：INodeDirectory和INodeFile。其中INodeDirectory标识的是目录树中的目录，INodeFile标识的是目录树中的文件。
由于二者均继承自INode，所以具备大部分相同的公共信息INodeWithAdditionalFields，除常用基础属性外，其中还提供了扩展属性features，
如Quota、Snapshot等均通过Feature增加，如果以后出现新属性也可通过Feature方便扩展。
不同的是，INodeFile特有的标识副本数和数据块大小组合的header（2.6.1之后又新增了标识存储策略ID的信息）及该文件包含的有序Blocks数组；
INodeDirectory则特有子节点的列表children。
这里需要特别说明children是默认大小为5的ArrayList，按照子节点name有序存储，虽然在插入时会损失一部分写性能，但是可以方便后续快速二分查找提高读性能，
对一般存储系统，读操作比写操作占比要高。具体的继承关系见下图。


![pic](https://pan.zeekling.cn/zeekling/hadoop/nn_0004.png)


## BlockManager 

BlocksMap在NameNode内存空间占据很大比例，由BlockManager统一管理，相比Namespace，BlockManager管理的这部分数据要复杂的多。
Namespace与BlockManager之间通过前面提到的INodeFile有序Blocks数组关联到一起。图5所示BlockManager管理的内存结构。

![pic](https://pan.zeekling.cn/zeekling/hadoop/nn_0005.png)

每一个INodeFile都会包含数量不等的Block，具体数量由文件大小及每一个Block大小（默认为64M）比值决定，这些Block按照所在文件的先后顺序组成BlockInfo数组，
如上图所示的BlockInfo[A~K]，BlockInfo维护的是Block的元数据，结构如下图所示，数据本身是由DataNode管理，所以BlockInfo需要包含实际数据到底由哪些DataNode管理的信息，
这里的核心是名为triplets的Object数组，大小为`3*replicas`，其中replicas是Block副本数量。triplets包含的信息：

- triplets[i]：Block所在的DataNode；
- triplets[i+1]：该DataNode上前一个Block；
- triplets[i+2]：该DataNode上后一个Block；

其中i表示的是Block的第i个副本，i取值[0,replicas)。

![pic](https://pan.zeekling.cn/zeekling/hadoop/nn_0006.png)

为了快速通过blockid快速定位Block，引入了BlocksMap。

BlocksMap底层通过LightWeightGSet实现，本质是一个链式解决冲突的哈希表。

为了避免rehash过程带来的性能开销，初始化时，索引空间直接给到了整个JVM可用内存的2%，并且不再变化。


NameNode内存中所有数据都要随读写情况发生变化，BlockManager当然也需要管理这部分动态数据。
当Block发生变化不符合预期时需要及时调整Blocks的分布。这里涉及几个核心的数据结构：
- excessReplicateMap: 某个Block实际存储的副本数多于预设副本数，这时候需要删除多余副本，这里多余副本会被置于excessReplicateMap中。
  excessReplicateMap是从DataNode的StorageID到Block集合的映射集。
- neededReplications: 若某个Block实际存储的副本数少于预设副本数，这时候需要补充缺少副本，这里哪些Block缺少多少个副本都统一存在neededReplications里，
  本质上neededReplications是一个优先级队列，缺少副本数越多的Block之后越会被优先处理。
- invalidateBlocks: 若某个Block即将被删除，会被置于invalidateBlocks中。
  invalidateBlocks是从DataNode的StorageID到Block集合的映射集。如某个文件被客户端执行了删除操作，该文件所属的所有Block会先被置于invalidateBlocks中。
- corruptReplicas:有些场景Block由于时间戳/长度不匹配等等造成Block不可用，会被暂存在corruptReplicas中，之后再做处理。


BlockManager内部的ReplicationMonitor线程会持续从其中取出数据并通过逻辑处理后分发给具体的DatanodeDescriptor对应数据结构,
当对应DataNode的心跳过来之后，NameNode会遍历DatanodeDescriptor里暂存的数据，将其转换成对应指令返回给DataNode，DataNode收到任务并执行完成后再反馈回NameNode,
之后DatanodeDescriptor里对应信息被清除。

![pic](https://pan.zeekling.cn/zeekling/hadoop/nn_0007.png)

## NetworkTopology 

NameNode不仅需要管理所有DataNode，由于数据写入前需要确定数据块写入位置，NameNode还维护着整个机架拓扑NetworkTopology。

![pic](https://pan.zeekling.cn/zeekling/hadoop/nn_0008.png)

包含两个部分：机架拓扑结构NetworkTopology和DataNode节点信息。
其中树状的机架拓扑是根据机架感知（一般都是外部脚本计算得到）在集群启动完成后建立起来，整个机架的拓扑结构在NameNode的生命周期内一般不会发生变化；
另一部分是比较关键的DataNode信息，BlockManager已经提到每一个DataNode上的Blocks集合都会形成一个双向链表，
更准确的应该是DataNode的每一个存储单元DatanodeStorageInfo上的所有Blocks集合会形成一个双向链表，这个链表的入口就是机架拓扑结构叶子节点即DataNode管理的DatanodeStorageInfo。


## LeaseManager
Lease 机制是重要的分布式协议，广泛应用于各种实际的分布式系统中。HDFS支持Write-Once-Read-Many，对文件写操作的互斥同步靠Lease实现。
Lease实际上是时间约束锁，其主要特点是排他性。客户端写文件时需要先申请一个Lease，一旦有客户端持有了某个文件的Lease，其它客户端就不可能再申请到该文件的Lease,
这就保证了同一时刻对一个文件的写操作只能发生在一个客户端。NameNode的LeaseManager是Lease机制的核心，
维护了文件与Lease、客户端与Lease的对应关系，这类信息会随写数据的变化实时发生对应改变。

![pic](https://pan.zeekling.cn/zeekling/hadoop/nn_0009.png)

上图为为LeaseManager内存结构，包括以下三个主要核心数据结构：

- sortedLeases：Lease集合，按照时间先后有序组织，便于检查Lease是否超时；
- leases：客户端到Lease的映射关系；
- sortedLeasesByPath：文件路径到Lease的映射关系；

其中每一个写数据的客户端会对应一个Lease，每个Lease里包含至少一个标识文件路径的Path。
Lease本身已经维护了其持有者（客户端）及该Lease正在操作的文件路径集合，之所以增加了leases和sortedLeasesByPath为提高通过Lease持有者或文件路径快速索引到Lease的性能。

当Lease发生超时后需要强制回收，内存中与该Lease相关的内容要被及时清除。超时检查及超时后的处理逻辑由LeaseManager.Monitor统一执行。
LeaseManager中维护了两个与Lease相关的超时时间：软超时（softLimit）和硬超时（hardLimit），使用场景稍有不同。

客户端向集群写文件前需要向NameNode的LeaseManager申请Lease；写文件过程中定期更新Lease时间，以防Lease过期，周期与softLimit相关；写完数据后申请释放Lease。

整个过程可能发生两类问题：
（1）写文件过程中客户端没有及时更新Lease时间；
（2）写完文件后没有成功释放Lease。

两个问题分别对应为softLimit和hardLimit。两种场景都会触发LeaseManager对Lease超时强制回收。
如果客户端写文件过程中没有及时更新Lease超过softLimit时间后，另一客户端尝试对同一文件进行写操作时触发Lease软超时强制回收；
如果客户端写文件完成但是没有成功释放Lease，则会由LeaseManager的后台线程LeaseManager.Monitor检查是否硬超时后统一触发超时回收。


转自：https://tech.meituan.com/2016/08/26/namenode.html

