# 简介

## Namenode 和 Datanode

HDFS采用master/slave架构。一个HDFS集群是由一个Namenode和一定数目的Datanodes组成。
Namenode是一个中心服务器，负责管理文件系统的名字空间(namespace)以及客户端对文件的访问。
集群中的Datanode一般是一个节点一个，负责管理它所在节点上的存储。
HDFS暴露了文件系统的名字空间，用户能够以文件的形式在上面存储数据。
从内部看，一个文件其实被分成一个或多个数据块，这些块存储在一组Datanode上。
Namenode执行文件系统的名字空间操作，比如打开、关闭、重命名文件或目录。它也负责确定数据块到具体Datanode节点的映射。
Datanode负责处理文件系统客户端的读写请求。在Namenode的统一调度下进行数据块的创建、删除和复制。

![pic](https://hadoop.apache.org/docs/r1.0.4/cn/images/hdfsarchitecture.gif)

Namenode是所有HDFS元数据的仲裁者和管理者，这样，用户数据永远不会流过Namenode。


Namenode全权管理数据块的复制，它周期性地从集群中的每个Datanode接收心跳信号和块状态报告(Blockreport)。
接收到心跳信号意味着该Datanode节点工作正常。块状态报告包含了一个该Datanode上所有数据块的列表。

![pic](https://hadoop.apache.org/docs/r1.0.4/cn/images/hdfsdatanodes.gif)

HDFS中的文件都是一次性写入的，并且严格要求在任何时候只能有一个写入者。


Namenode上保存着HDFS的名字空间。对于任何对文件系统元数据产生修改的操作，Namenode都会使用一种称为EditLog的事务日志记录下来。
Namenode在本地操作系统的文件系统中存储这个Editlog。整个文件系统的名字空间，包括数据块到文件的映射、文件的属性等，都存储在一个称为FsImage的文件中，这个文件也是放在Namenode所在的本地文件系统上。

Namenode在内存中保存着整个文件系统的名字空间和文件数据块映射(Blockmap)的映像。

当Namenode启动时，它从硬盘中读取Editlog和FsImage，将所有Editlog中的事务作用在内存中的FsImage上，
并将这个新版本的FsImage从内存中保存到本地磁盘上，然后删除旧的Editlog，因为这个旧的Editlog的事务都已经作用在FsImage上了。

Datanode将HDFS数据以文件的形式存储在本地的文件系统中，它并不知道有关HDFS文件的信息。它把每个HDFS数据块存储在本地文件系统的一个单独的文件中。
Datanode并不在同一个目录创建所有的文件，实际上，它用试探的方法来确定每个目录的最佳文件数目，并且在适当的时候创建子目录。
在同一个目录中创建所有的本地文件并不是最优的选择，这是因为本地文件系统可能无法高效地在单个目录中支持大量的文件。
当一个Datanode启动时，它会扫描本地文件系统，产生一个这些本地文件对应的所有HDFS数据块的列表，然后作为报告发送到Namenode，这个报告就是块状态报告。


## 副本

在大多数情况下，副本系数是3，HDFS的存放策略是将一个副本存放在本地机架的节点上，一个副本放在同一机架的另一个节点上，最后一个副本放在不同机架的节点上。
这种策略减少了机架间的数据传输，这就提高了写操作的效率。机架的错误远远比节点的错误少，所以这个策略不会影响到数据的可靠性和可用性。
在这种策略下，副本并不是均匀分布在不同的机架上。三分之一的副本在一个节点上，三分之二的副本在一个机架上，其他副本均匀分布在剩下的机架中，
这一策略在不损害数据可靠性和读取性能的情况下改进了写的性能。

为了降低整体的带宽消耗和读取延时，HDFS会尽量让读取程序读取离它最近的副本。
如果在读取程序的同一个机架上有一个副本，那么就读取该副本。如果一个HDFS集群跨越多个数据中心，那么客户端也将首先读本地数据中心的副本。

## 安全模式

## Secondary NameNode 处理步骤

![pic](https://pan.zeekling.cn/zeekling/hadoop/hadoop_namenode_00001.png)

---

# Hadoop HDFS 模块功能详细列表

## 一、核心组件功能

### 1. NameNode（名称节点）

- **命名空间管理**：FSDirectory、INode管理、目录/文件/符号链接操作、权限检查、扩展属性
- **块管理**：BlockManager块映射、BlockPlacementPolicy块放置策略
- **租赁管理**：LeaseManager文件写入租约
- **快照管理**：SnapshotManager快照创建/删除/差异比较
- **配额管理**：目录空间和文件数量配额
- **镜像和编辑日志**：FSImage/FSEditLog日志存储和滚动
- **安全模式**：启动时安全模式管理
- **高可用**：HAContext/BootstrapStandby主备切换
- **加密**：FSDirEncryptionZoneOp加密区域、ReencryptionHandler重加密

### 2. DataNode（数据节点）

- **数据块存储**：FsDatasetSpi/FsVolumeImpl/BlockPoolSlice
- **数据传输**：DataXceiver/BlockSender/BlockReceiver
- **块报告**：定期/增量块报告
- **块恢复**：BlockRecoveryWorker
- **块校验**：BlockChecksumHelper
- **磁盘扫描**：DirectoryScanner/VolumeScanner损坏检测
- **磁盘平衡**：DiskBalancer
- **短路读取**：ShortCircuitRegistry
- **版本优化**：[3.3.1后优化详解](../watch/datanode优化详解.md)（细粒度锁、慢磁盘排除、动态重配置等）

### 3. 块管理（BlockManagement）

- **副本管理**：ReplicationWork/ErasureCodingWork
- **节点管理**：DatanodeManager/HeartbeatManager
- **存储策略**：BlockStoragePolicySuite（DISK/SSD/ARCHIVE）
- **慢节点检测**：SlowPeerTracker/SlowDiskTracker

### 4. 高可用功能

- **JournalNode**：QJM日志节点、RPC/HTTP服务器、JNStorage
- **故障转移**：DFSZKFailoverController/ZK自动切换
- **检查点**：SecondaryNameNode

### 5. 负载均衡

- **Balancer**：集群DataNode数据平衡、阈值/策略配置

### 6. 存储策略满足（SPS）

- **ExternalStoragePolicySatisfier**：根据策略移动块

## 二、工具功能

### 7. 管理工具

| 工具 | 功能 |
|------|------|
| DFSAdmin | 节点管理、安全模式、刷新、缓存 |
| DFSck | 文件系统检查、损坏/副本不足检测 |
| DFSHAAdmin | HA状态查看、故障转移 |
| CacheAdmin | 缓存指令/缓存池管理 |
| CryptoAdmin | 加密区域、密钥轮换 |
| ECAdmin | 纠删码策略管理 |
| StoragePolicyAdmin | 存储策略设置 |
| DiskBalancerCLI | 磁盘平衡计划/执行 |
| LsSnapshot/LsSnapshottableDir/SnapshotDiff | 快照工具 |
| OfflineImageViewer | 离线镜像查看（XML/文本/统计） |
| GetConf/JMXGet | 配置/JMX获取 |
| HDFSConcat/DebugAdmin | 文件合并/调试 |

## 三、协议与接口

### 8. 通信协议

- **ClientProtocol**：客户端-NameNode文件操作
- **DatanodeProtocol**：NameNode-DataNode块报告/心跳
- **DatanodeLifelineProtocol**：备用心跳协议
- **InterDatanodeProtocol**：DataNode间块恢复/复制
- **JournalProtocol**：编辑日志同步
- **NamenodeProtocol**：主备NameNode通信

### 9. Web接口

- **NameNode Web UI**：文件浏览、集群/节点/块状态
- **DataNode Web UI**：存储、块报告、平衡状态
- **JournalNode Web UI**：日志/同步状态
- **WebHDFS REST API**：文件操作、委托令牌

## 四、安全功能

### 10. 安全机制

- **认证**：Kerberos、委托令牌
- **授权**：FSPermissionChecker、ACL
- **加密**：透明加密、加密区域、数据传输加密
- **审计**：HdfsAuditLogger

## 五、存储功能

### 11. 快照功能

- 允许/禁止快照、创建/删除/重命名、差异报告、恢复

### 12. 存储策略

- HOT/COLD/WARM/ALL_SSD/ONE_SSD/ARCHIVE，自定义策略

### 13. 纠删码

- RS/XOR编码、EC块组管理、数据重建

## 六、网络与工具

### 14. 网络拓扑

- DFSNetworkTopology/DFSTopologyNodeImpl、机架感知、节点距离计算

### 15. 工具类

- DataTransferThrottler限流、LightWeightHashSet集合、AtomicFileOutputStream原子写入、MD5FileUtils、Diff差异计算

---

# Hadoop HDFS-Client 模块功能详细列表

## 一、核心客户端

### 1. 文件系统接口

- **DistributedFileSystem**：HDFS核心实现，文件创建/读取/写入/删除/重命名/权限管理
- **ViewDistributedFileSystem**：视图文件系统，跨集群统一访问
- **DFSClient**：底层客户端核心，管理与NameNode/DataNode通信
- **HdfsAdmin**：管理API（缓存/加密区域/快照/集群统计）

### 2. WebHDFS客户端

| 类 | 功能 |
|---|---|
| WebHdfsFileSystem | HTTP协议访问HDFS |
| SWebHdfsFileSystem | HTTPS安全访问 |
| URLConnectionFactory | HTTP连接工厂 |
| TokenAspect | 委托令牌处理 |

### 3. OAuth2认证

- OAuth2ConnectionConfigurator / AccessTokenProvider
- CredentialBasedAccessTokenProvider / ConfRefreshTokenBasedAccessTokenProvider

---

## 二、短路读取

| 类 | 功能 |
|---|---|
| ShortCircuitCache | 短路读取资源缓存 |
| ShortCircuitReplica | 本地副本信息 |
| ShortCircuitShm | 共享内存通信 |
| DfsClientShmManager | 共享内存管理 |
| DomainSocketFactory | Unix域套接字工厂 |
| ClientMmap | 内存映射读取 |

---

## 三、纠删码支持

- **DFSStripedInputStream**：纠删码文件读取
- **DFSStripedOutputStream**：纠删码文件写入
- **StripedBlockUtil**：条带块工具
- **ErasureCodingPolicy / ECPolicyLoader**：纠删码策略
- **SystemErasureCodingPolicies**：内置策略（RS等）
- **StripeReader / StatefulStripeReader**：条带读取器

---

## 四、协议与传输

### 通信协议

- **ClientProtocol**：客户端-NameNode协议（~2000行）
- **ClientDatanodeProtocol**：客户端-DataNode协议
- **DataTransferProtocol**：数据传输协议

### 数据传输

- **Op**：操作枚举（WRITE_BLOCK/READ_BLOCK等）
- **PacketReceiver / PacketHeader**：数据包处理
- **PipelineAck**：写入管道确认
- **Sender**：请求构建

### SASL安全传输

- **SaslDataTransferClient / SaslParticipant**：SASL认证
- **DataTransferSaslUtil**：SASL工具

---

## 五、网络层

- **Peer**：对等连接接口
- **BasicInetPeer**：TCP/IP连接
- **DomainPeer**：Unix域套接字
- **EncryptedPeer**：加密连接
- **PeerCache**：连接缓存

---

## 六、安全机制

### 块访问令牌

- BlockTokenIdentifier / BlockTokenSelector
- DataEncryptionKey

### 委托令牌

- DelegationTokenIdentifier / DelegationTokenSelector

### 加密

- UnknownCipherSuiteException / UnknownCryptoProtocolVersionException
- HdfsKMSUtil

---

## 七、事件通知（Inotify）

- **Event**：事件模型（Create/Close/Append/Rename/MetadataUpdate/Unlink/Truncate）
- **EventBatch / EventBatchList**：事件批次
- **DFSInotifyEventInputStream**：事件流接口

---

## 八、输入输出流

- **DFSInputStream / DFSOutputStream**：核心读写流
- **DataStreamer**：数据管道发送
- **DFSPacket**：数据包
- **HdfsDataInputStream / HdfsDataOutputStream**：HDFS特定流

---

## 九、HA与代理

- **NameNodeProxiesClient**：NameNode代理创建
- **HAUtilClient**：HA工具
- **ClientContext**：客户端共享状态

---

## 十、客户端实现

- **BlockReaderFactory / BlockReaderLocal / BlockReaderRemote**：块读取器
- **DfsClientConf / HdfsClientConfigKeys**：配置管理
- **LeaseRenewer**：租约续租

---

## 十一、工具类

| 类 | 功能 |
|---|---|
| HdfsConfiguration | HDFS配置 |
| DFSUtilClient | 静态工具方法 |
| ByteArrayManager | 字节数组池化 |
| CombinedHostsFileReader/Writer | 主机文件读写 |
| LongBitFormat | 位操作工具 |
| XAttrHelper | 扩展属性处理 |

---

## 十二、读取优化

- **ReaderStrategy**：读取策略
- **ReadStatistics**：读取统计
- **DFSHedgedReadMetrics**：对冲读取指标
- **DeadNodeDetector**：死节点检测

---

# Hadoop HttpFS 模块功能详细列表

## 一、HttpFS服务器

### 1. 主服务器

- **HttpFSServer**：JAX-RS REST API主服务器
- **HttpFSServerWebServer**：基于HttpServer2的Web服务器
- **HttpFSServerWebApp**：Web应用程序上下文

### 2. 请求处理

- **HttpFSParametersProvider**：HTTP请求参数定义
- **HttpFSExceptionProvider**：异常到HTTP状态码映射
- **FSOperations**：文件系统操作执行器

---

## 二、HttpFS客户端

| 类 | 功能 |
|---|---|
| HttpFSFileSystem | HTTP客户端FileSystem实现 |
| HttpsFSFileSystem | HTTPS安全客户端 |
| HttpFSUtils | 客户端工具类 |

---

## 三、REST API

### 读取操作（GET）

| 操作 | 功能 |
|------|------|
| OPEN | 打开读取文件 |
| GETFILESTATUS | 文件/目录状态 |
| LISTSTATUS | 目录列表 |
| GETCONTENTSUMMARY | 内容摘要 |
| GETFILECHECKSUM | 文件校验和 |
| GETACLSTATUS | ACL状态 |
| GETXATTRS/LISTXATTRS | 扩展属性 |
| GETHOMEDIRECTORY | 主目录 |
| GETSTATUS | 文件系统状态 |
| GETALLSTORAGEPOLICY | 所有存储策略 |
| GETECPOLICIES | 纠删码策略 |
| GETSNAPSHOT* | 快照相关 |
| INSTRUMENTATION | 指标信息 |

### 写入操作（PUT）

| 操作 | 功能 |
|------|------|
| CREATE | 创建文件 |
| MKDIRS | 创建目录 |
| RENAME | 重命名 |
| SETOWNER/SETPERMISSION | 权限设置 |
| SETREPLICATION | 副本数设置 |
| SETACL/MODIFYACLENTRIES | ACL管理 |
| SETXATTR | 扩展属性 |
| SETSTORAGEPOLICY | 存储策略 |
| SETECPOLICY | 纠删码策略 |
| CREATESNAPSHOT | 创建快照 |
| ALLOWSNAPSHOT/DISALLOWSNAPSHOT | 快照权限 |

### 修改操作（POST）

| 操作 | 功能 |
|------|------|
| APPEND | 追加数据 |
| CONCAT | 合并文件 |
| TRUNCATE | 截断文件 |

### 删除操作（DELETE）

| 操作 | 功能 |
|------|------|
| DELETE | 删除文件/目录 |
| DELETESNAPSHOT | 删除快照 |

---

## 四、认证与授权

- **Kerberos认证**：SPNEGO认证
- **Simple认证**：用户名认证
- **委托令牌**：支持delegation token
- **代理用户**：doas参数代理访问
- **ACL权限**：集成HDFS ACL
- **访问模式**：read-only/write-only/read-write

---

## 五、监控指标

### HttpFSServerMetrics

- **字节传输**：bytesWritten、bytesRead
- **写操作**：opsCreate、opsAppend、opsTruncate、opsDelete、opsRename、opsMkdir
- **读操作**：opsOpen、opsListing、opsStat、opsCheckAccess、opsStatus

### InstrumentationService

- **Counters**：计数器
- **Timers**：计时器
- **Variables**：变量
- **Samplers**：采样器

---

## 六、服务器框架

### 服务层

- **FileSystemAccessService**：文件系统访问服务（带缓存）
- **GroupsService**：用户组服务
- **InstrumentationService**：指标服务
- **SchedulerService**：调度服务

### Servlet过滤器

- **HttpFSAuthenticationFilter**：认证过滤
- **HttpFSReleaseFilter**：资源释放
- **MDCFilter**：日志MDC
- **HostnameFilter**：主机名过滤
- **CheckUploadContentTypeFilter**：上传类型检查

### 参数处理

- **Parameters**：参数基类
- **StringParam/IntegerParam/LongParam**：类型参数
- **JSONProvider**：JSON序列化

---

## 七、高级特性

| 功能 | 描述 |
|------|------|
| 扩展属性 | XAttr完整操作 |
| ACL管理 | 增删改查 |
| 存储策略 | HOT/COLD/WARM/SSD |
| 纠删码 | RS编码策略 |
| 快照 | 创建/删除/差异 |
| HTTP重定向 | 307数据上传优化 |
| 审计日志 | 完整操作审计 |

---

## 八、配置管理

- **Server**：通用服务器基类（配置/日志/生命周期）
- **BaseService**：服务基类
- **ConfigurationUtils**：配置工具

---

# Hadoop HDFS-Native-Client 模块功能详细列表

## 一、libhdfs - C语言原生客户端

### 1.1 连接管理API

- **hdfsConnect/hdfsConnectAsUser**：连接到HDFS（废弃）
- **hdfsBuilderConnect**：构建器模式连接（推荐）
- **hdfsNewBuilder**：创建连接构建器
- **hdfsBuilderSetNameNode/Port**：设置NameNode地址/端口
- **hdfsBuilderSetUserName/KerbTicketCache**：设置用户/Kerberos
- **hdfsDisconnect**：断开连接

### 1.2 文件操作API

| 函数 | 功能 |
|------|------|
| hdfsOpenFileBuilderAlloc/OpenFileBuilderBuild | 异步打开文件 |
| hdfsCloseFile/TruncateFile | 关闭/截断文件 |
| hdfsRead/Pread/PreadFully | 读取数据 |
| hdfsWrite/Flush/HFlush/HSync | 写入数据 |
| hdfsSeek/Tell/Available | 文件位置操作 |

### 1.3 文件系统操作

- **hdfsCopy/Move**：复制/移动文件
- **hdfsDelete/Rename**：删除/重命名
- **hdfsExists/Mkdirs**：存在检查/创建目录
- **hdfsListDirectory/GetPathInfo**：目录列表/文件信息
- **hdfsGetHosts**：获取块位置
- **hdfsChown/Chmod/Utime**：属性修改

### 1.4 零拷贝读取

- **hadoopRzOptionsAlloc**：分配零拷贝选项
- **hadoopReadZero**：mmap零拷贝读取
- **hadoopRzBufferGet/Free**：缓冲区管理

### 1.5 统计与度量

- **hdfsFileGetReadStatistics**：读取统计
- **hdfsGetHedgedReadMetrics**：对冲读取指标

---

## 二、libhdfspp - C++原生客户端

### 2.1 核心类

- **FileSystem**：文件系统操作（连接/读写/目录）
- **FileHandle**：文件句柄（位置读取/异步操作）
- **Options**：配置选项
- **Status**：状态码和错误处理

### 2.2 FileSystem方法

- **Connect/ConnectToDefaultFs**：连接集群
- **Open/GetFileInfo**：打开/获取文件信息
- **Mkdirs/Delete/Rename**：目录操作
- **GetListing/GetBlockLocations**：列表/块位置
- **GetContentSummary/FsStats**：统计信息
- **SetReplication/Times/Permission/Owner**：属性设置
- **CreateSnapshot/DeleteSnapshot/RenameSnapshot**：快照操作

### 2.3 配置选项

- **rpc_timeout**：RPC超时（默认30秒）
- **max_rpc_retries**：RPC重试次数
- **authentication**：认证方式（simple/kerberos）
- **block_size**：默认块大小

---

## 三、命令行工具

| 工具 | 功能 |
|------|------|
| hdfs-ls | 列出目录 |
| hdfs-cat | 显示文件内容 |
| hdfs-get/copy-to-local | 下载文件 |
| hdfs-mkdir | 创建目录 |
| hdfs-rm | 删除文件 |
| hdfs-rename | 重命名 |
| hdfs-chmod/chown/chgrp | 权限/所有者 |
| hdfs-du/df | 磁盘使用统计 |
| hdfs-count | 文件数量统计 |
| hdfs-find | 递归查找 |
| hdfs-setrep | 设置副本数 |
| hdfs-stat | 文件状态 |
| hdfs-tail | 查看文件末尾 |
| hdfs-create/allow/disallow/delete-snapshot | 快照管理 |

---

## 四、测试代码

### 4.1 测试用例

- **test_libhdfs_ops**：基本操作测试
- **test_libhdfs_threaded**：多线程测试
- **test_libhdfs_mini_stress**：压力测试
- **test_libhdfs_zerocopy**：零拷贝测试

### 4.2 示例代码

- **libhdfs_read.c / libhdfs_write.c**：读写示例

---

## 五、FUSE挂载驱动

### 5.1 FUSE操作

- getattr/access/open/read/write
- create/mkdir/rmdir/unlink/rename
- chmod/chown/truncate/symlink
- readdir/statfs/flush/release

### 5.2 配置选项

- **rdbuffer_size**：读取缓冲区
- **attribute_timeout**：属性缓存超时
- **entry_timeout**：目录项缓存超时
- **direct_io**：直接I/O
- **usetrash**：启用回收站
- **no_permissions**：禁用权限检查

---

## 六、技术特性

| 特性 | 描述 |
|------|------|
| 认证 | Simple/Kerberos/Token |
| 零拷贝 | mmap实现，减少CPU开销 |
| Hedged读取 | 并行对冲慢DataNode |
| 异步I/O | Asio库实现 |
| HA支持 | NameNode故障切换 |
| 重试机制 | RPC自动重试 |

---

# Hadoop HDFS-NFS 模块功能详细列表

## 一、NFSv3协议实现

### 1. 文件操作

- **GETATTR/SETATTR**：获取/设置文件属性
- **LOOKUP/ACCESS**：查找和访问权限检查
- **READLINK**：读取符号链接

### 2. 读写操作

- **READ**：从HDFS读取数据
- **WRITE**：写入数据（支持同步/异步/非稳定模式）
- **CREATE**：创建并打开文件
- **COMMIT**：刷新数据到HDFS

### 3. 目录操作

- **MKDIR/RMDIR**：创建/删除目录
- **READDIR/READDIRPLUS**：读取目录

### 4. 文件管理

- **REMOVE/RENAME**：删除/重命名
- **SYMLINK/LINK**：符号链接/硬链接

### 5. 文件系统信息

- **FSSTAT/FSINFO/PATHCONF**：文件系统状态和信息

---

## 二、Mount协议实现

### 挂载服务

- **MNT**：处理挂载请求，返回文件句柄
- **DUMP**：列出所有挂载点
- **UMNT/UMNTALL**：卸载挂载点
- **EXPORT**：列出导出路径

---

## 三、核心组件

### 写管理

- **WriteManager**：管理写操作，协调异步写入
- **OpenFileCtx**：打开文件上下文，维护写入状态
- **WriteCtx**：单个写请求上下文
- **AsyncDataService**：异步数据写回线程池

### 缓存

- **OpenFileCtxCache**：打开文件缓存（LRU，max 256）
- **DFSClientCache**：DFS客户端缓存

### 服务

- **Nfs3HttpServer**：HTTP监控界面（端口50079）
- **Nfs3Metrics**：指标监控（延迟、吞吐量）
- **Nfs3Utils**：工具类（句柄转换、权限映射）

---

## 四、访问控制

- **exports表**：基于IP的访问控制
- **只读/读写权限**
- **端口监控**
- **代理用户支持**
- **UID/GID映射**

---

## 五、配置选项

| 配置 | 默认值 | 说明 |
|------|--------|------|
| nfs.server.port | 2049 | NFS端口 |
| nfs.mountd.port | 4242 | Mount端口 |
| nfs.dump.dir | /tmp/.hdfs-nfs | 数据转储目录 |
| nfs.max.open.files | 256 | 最大打开文件数 |
| nfs.stream.timeout | 10分钟 | 流超时 |
| nfs.rtmax/wtmax | 1MB | 读/写传输大小 |

---

## 六、技术特性

- **异步写入**：顺序写直接HDFS，非顺序写缓存
- **数据转储**：内存压力时转储到磁盘
- **AIX兼容**：特殊cookieverf处理
- **ViewFS支持**：联合集群路径解析
- **指标监控**：Metrics2延迟和吞吐量统计

---

# Hadoop HDFS-RBF 模块功能详细列表

## 一、Router核心

### 1.1 主服务

- **Router**：联邦接口服务，路由客户端请求到子集群
- **RouterRpcServer**：RPC服务端，处理ClientProtocol请求
- **RouterAdminServer**：管理接口服务
- **RouterHttpServer**：HTTP/Web服务（Web UI + WebHDFS）
- **StateStoreService**：状态存储服务

### 1.2 客户端通信

- **RouterRpcClient**：到NameNode的RPC客户端，连接池管理
- **RouterClient**：管理客户端
- **ConnectionPool/ConnectionManager**：连接池管理

### 1.3 安全

- **RouterSecurityManager**：Delegation Token管理、Kerberos认证

### 1.4 异步RPC

- **RouterAsyncClientProtocol**：异步非阻塞RPC调用
- **RouterAsyncRpcClient**：异步RPC客户端

---

## 二、State Store（状态存储）

### 2.1 存储驱动

- **StateStoreFileImpl**：文件存储
- **StateStoreFileSystemImpl**：HDFS存储
- **StateStoreMySQLImpl**：MySQL存储
- **StateStoreZooKeeperImpl**：ZooKeeper存储（生产推荐）

### 2.2 数据记录

- **MountTable**：挂载表（路径映射）
- **MembershipState**：NameNode成员状态
- **RouterState**：Router状态
- **DisabledNameservice**：禁用服务

---

## 三、管理工具

### RouterAdmin命令

| 命令 | 功能 |
|------|------|
| add/update/rm | 挂载表管理 |
| ls | 列出挂载点 |
| setQuota/clrQuota | 配额管理 |
| safemode enter/leave | 安全模式 |
| disable/enable | 禁用/启用子集群 |
| dumpState | 导出状态 |

---

## 四、负载均衡

### RouterFedBalance

- **MountTableProcedure**：更新挂载表
- **RouterDistCpProcedure**：跨子集群复制

---

## 五、路径解析器

### 解析器

- **MountTableResolver**：挂载表解析
- **MultipleDestinationMountTableResolver**：多目标解析
- **MembershipNamenodeResolver**：活跃NameNode解析
- **RemoteLocation/PathLocation**：位置对象

---

## 六、指标监控

- **FederationRPCMetrics**：RPC延迟和成功率
- **StateStoreMetrics**：状态存储指标
- **NameserviceRPCMetrics**：各子集群RPC统计
- **RBFMetrics**：联邦级指标汇总

---

## 七、RPC公平性

### 策略控制器

- **StaticRouterRpcFairnessPolicyController**：静态配置
- **ProportionRouterRpcFairnessPolicyController**：比例分配
- **RouterAsyncRpcFairnessPolicyController**：异步RPC公平性

---

## 八、路由策略

| 策略 | 描述 |
|------|------|
| HASH | 哈希（基于路径） |
| LOCAL | 本地优先 |
| RANDOM | 随机选择 |
| HASH_ALL | 哈希所有副本 |
| SPACE | 空间最优 |
| LEADER_FOLLOWER | 主从优先 |

---

## 九、配置选项

| 配置 | 默认值 | 说明 |
|------|--------|------|
| dfs.router.rpc.address | 0.0.0.0:8888 | RPC端口 |
| dfs.router.http.address | 0.0.0.0:50071 | HTTP端口 |
| dfs.router.admin.address | 0.0.0.0:8811 | 管理端口 |
| dfs.router.handler.count | 10 | Handler数量 |
| dfs.router.safemode.enable | true | 安全模式 |