# Apache Hadoop 代码仓库模块深度分析报告

## 一、核心基础模块

---

## hadoop-common (hadoop-common-project/hadoop-common)
- **核心功能**: Hadoop 公共基础设施，提供其他所有模块依赖的核心工具类
- **关键类**: Configuration, FileSystem, FileStatus, Path, FSDataInputStream, FSDataOutputStream, UserGroupInformation, Server, Client, RPC, Writable
- **主要职责**:
  - 提供统一的配置管理系统
  - 定义抽象文件系统接口，是所有文件系统实现的基类
  - 实现RPC通信框架，支持进程间通信
  - 安全认证与授权管理
  - 序列化框架（Writable接口）和基础数据类型
  - 工具类库（字符串处理、反射、进程管理等）

---

## hadoop-annotations (hadoop-common-project/hadoop-annotations)
- **核心功能**: 提供代码注解支持
- **关键类**: InterfaceAudience, InterfaceStability
- **主要职责**:
  - 定义API的受众分类（Public、LimitedPrivate、Private）
  - 定义API的稳定性级别（Stable、Evolving、Unstable、Deprecated）
  - 用于文档生成和API兼容性管理

---

## hadoop-auth (hadoop-common-project/hadoop-auth)
- **核心功能**: HTTP身份认证支持
- **关键类**: AuthenticationFilter, KerberosAuthenticationHandler, PseudoAuthenticationHandler
- **主要职责**:
  - 提供基于Kerberos的HTTP认证
  - 支持简单的伪认证模式
  - 身份验证令牌管理

---

## hadoop-kms (hadoop-common-project/hadoop-kms)
- **核心功能**: 密钥管理服务器（Key Management Server）
- **关键类**: KMS, KMSACLs, KeyAuthorizationKeyProvider, KMSWebServer
- **主要职责**:
  - 提供加密密钥的集中管理
  - 支持密钥的生成、存储、检索和轮换
  - 提供RESTful API接口
  - 密钥访问权限控制（ACL）

---

## hadoop-nfs (hadoop-common-project/hadoop-nfs)
- **核心功能**: NFS协议支持
- **关键类**: Nfs3, Mountd
- **主要职责**:
  - 实现NFS v3协议
  - 支持通过NFS挂载访问HDFS

---

## hadoop-registry (hadoop-common-project/hadoop-registry)
- **核心功能**: 服务注册与发现
- **关键类**: RegistryOperations, RegistryOperationsClient
- **主要职责**:
  - 提供YARN服务的注册机制
  - 支持服务发现和动态服务管理

---

## hadoop-minikdc (hadoop-common-project/hadoop-minikdc)
- **核心功能**: 迷你KDC（Kerberos Distribution Center）
- **关键类**: MiniKdc
- **主要职责**:
  - 提供测试用的Kerberos环境
  - 支持安全认证的单元测试

---

## 二、HDFS 分布式文件系统模块

---

## hadoop-hdfs (hadoop-hdfs-project/hadoop-hdfs)
- **核心功能**: HDFS核心实现，包括NameNode和DataNode
- **关键类**: NameNode, NameNodeRpcServer, FSNamesystem, DataNode, BlockManager, DatanodeDescriptor, FSDirectory, INode, LeaseManager, SafeMode
- **主要职责**:
  - **NameNode**: 元数据管理、命名空间管理、块管理、心跳处理
  - **DataNode**: 数据块存储、读写处理、块复制与恢复
  - **BlockManager**: 副本放置策略、块恢复、重新复制
  - **LeaseManager**: 文件写入互斥控制（租约管理）
  - **SafeMode**: 集群启动时的安全保护机制

---

## hadoop-hdfs-client (hadoop-hdfs-project/hadoop-hdfs-client)
- **核心功能**: HDFS客户端库，提供轻量级客户端API
- **关键类**: DFSClient, DFSInputStream, DFSOutputStream, HdfsDataInputStream, HdfsDataOutputStream
- **主要职责**:
  - 提供HDFS文件读写API
  - 实现短路读（Short-circuit read）优化
  - 客户端缓存管理
  - 与NameNode/DataNode通信

---

## hadoop-hdfs-httpfs (hadoop-hdfs-project/hadoop-hdfs-httpfs)
- **核心功能**: HTTP文件系统网关
- **关键类**: HttpFSServer, FSOperations, HttpFSWebServer
- **主要职责**:
  - 提供RESTful HTTP API访问HDFS
  - 支持WebHDFS协议
  - 代理认证与授权

---

## hadoop-hdfs-nfs (hadoop-hdfs-project/hadoop-hdfs-nfs)
- **核心功能**: HDFS的NFS网关
- **关键类**: Nfs3, HdfsNfsGateway
- **主要职责**:
  - 将HDFS导出为NFS文件系统
  - 支持标准NFS客户端访问

---

## hadoop-hdfs-rbf (hadoop-hdfs-project/hadoop-hdfs-rbf)
- **核心功能**: HDFS路由联邦（Router-Based Federation）
- **关键类**: Router, RouterRpcServer, RouterClient, MountTable, ConnectionManager
- **主要职责**:
  - 实现HDFS联邦架构的路由层
  - 多命名空间的透明访问
  - 挂载表管理
  - 跨子集群操作支持

---

## hadoop-hdfs-native-client (hadoop-hdfs-project/hadoop-hdfs-native-client)
- **核心功能**: HDFS原生C/C++客户端库
- **关键类**: hdfsClient, hdfsStream
- **主要职责**:
  - 提供C语言版本的HDFS客户端API
  - 支持libhdfs库

---

## 三、YARN 资源调度模块

---

## hadoop-yarn-api (hadoop-yarn-project/hadoop-yarn/hadoop-yarn-api)
- **核心功能**: YARN核心API定义
- **关键类**: ApplicationId, ContainerId, ApplicationSubmissionContext, ContainerLaunchContext, Resource, ResourceRequest, NodeReport, ApplicationReport, Priority
- **主要职责**:
  - 定义YARN核心数据模型
  - 定义客户端与服务端通信协议
  - 资源请求与分配模型
  - 应用程序状态定义

---

## hadoop-yarn-common (hadoop-yarn-project/hadoop-yarn/hadoop-yarn-common)
- **核心功能**: YARN公共工具类
- **关键类**: YarnConfiguration, ResourceManager, NodeManager, ContainerLogAppender
- **主要职责**:
  - 配置管理
  - 日志处理
  - 公共工具类

---

## hadoop-yarn-client (hadoop-yarn-project/hadoop-yarn/hadoop-yarn-client)
- **核心功能**: YARN客户端API
- **关键类**: YarnClient, AMRMClient, NMClient, YarnClientApplication
- **主要职责**:
  - **YarnClient**: 应用程序提交与管理
  - **AMRMClient**: ApplicationMaster与ResourceManager通信
  - **NMClient**: 与NodeManager通信，管理容器生命周期

---

## hadoop-yarn-server-resourcemanager (hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-resourcemanager)
- **核心功能**: YARN资源管理器核心
- **关键类**: ResourceManager, ClientRMService, ApplicationMasterService, ResourceTrackerService, RMAppManager, RMContext, NodesListManager
- **主要职责**:
  - 集群资源管理
  - 应用程序生命周期管理
  - 接受NodeManager心跳，跟踪节点状态
  - 调度器协调
  - 客户端请求处理
  - ApplicationMaster请求处理

---

## hadoop-yarn-server-nodemanager (hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-nodemanager)
- **核心功能**: YARN节点管理器
- **关键类**: NodeManager, ContainerManagerImpl, ContainerExecutor, NodeStatusUpdater, LocalDirsHandlerService, DeletionService
- **主要职责**:
  - 单节点上的容器管理
  - 容器启动、监控、停止
  - 本地资源本地化（Localization）
  - 节点健康状态监控
  - 与ResourceManager心跳通信

---

## hadoop-yarn-server-common (hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-common)
- **核心功能**: YARN服务端公共组件
- **关键类**: NMProxy, RMProxy
- **主要职责**:
  - 提供ResourceManager和NodeManager的公共功能
  - 代理和通信工具

---

## hadoop-yarn-server-web-proxy (hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-web-proxy)
- **核心功能**: YARN Web代理服务
- **关键类**: WebAppProxy, WebAppProxyServlet
- **主要职责**:
  - 提供ApplicationMaster Web UI的代理访问
  - 安全认证转发

---

## hadoop-yarn-server-applicationhistoryservice (hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-applicationhistoryservice)
- **核心功能**: 应用程序历史服务
- **关键类**: ApplicationHistoryManager, ApplicationHistoryWriter
- **主要职责**:
  - 记录应用程序历史信息
  - 支持历史数据查询

---

## hadoop-yarn-server-timelineservice (hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-timelineservice)
- **核心功能**: 时间线服务v2（Timeline Service v2）
- **关键类**: TimelineCollector, TimelineWriter, TimelineReader
- **主要职责**:
  - 收集应用程序运行时数据
  - 存储和查询时间线事件
  - 支持聚合指标

---

## hadoop-yarn-server-router (hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-router)
- **核心功能**: YARN路由器
- **关键类**: Router, RouterServer
- **主要职责**:
  - 多ResourceManager联邦的路由层
  - 请求路由与负载均衡

---

## hadoop-yarn-server-sharedcachemanager (hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-sharedcachemanager)
- **核心功能**: 共享缓存管理器
- **关键类**: SharedCacheManager, SharedCacheManagerService
- **主要职责**:
  - 管理应用程序间的共享缓存资源
  - 减少重复资源上传

---

## hadoop-yarn-registry (hadoop-yarn-project/hadoop-yarn/hadoop-yarn-registry)
- **核心功能**: YARN服务注册
- **关键类**: RegistryOperations
- **主要职责**:
  - 服务注册与发现

---

## hadoop-yarn-applications-distributedshell (hadoop-yarn-project/hadoop-yarn/hadoop-yarn-applications/hadoop-yarn-applications-distributedshell)
- **核心功能**: 分布式Shell示例应用
- **关键类**: ApplicationMaster, Client
- **主要职责**:
  - 提供YARN应用程序开发示例
  - 在集群中执行Shell命令

---

## hadoop-yarn-services-core (hadoop-yarn-project/hadoop-yarn/hadoop-yarn-applications/hadoop-yarn-services/hadoop-yarn-services-core)
- **核心功能**: YARN服务框架
- **关键类**: ServiceMaster, ServiceScheduler, ServiceManager, Component
- **主要职责**:
  - 支持长时间运行的服务
  - 服务编排与管理
  - 组件生命周期管理
  - 容器故障恢复

---

## YARN调度器详解

---

### Capacity Scheduler (hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-resourcemanager/scheduler/capacity)
- **核心功能**: 容量调度器
- **关键类**: CapacityScheduler, LeafQueue, ParentQueue, CSQueue, CapacitySchedulerConfiguration
- **主要职责**:
  - 多租户资源分配
  - 层级队列管理
  - 容量保证与弹性扩展
  - 用户资源限制

---

### Fair Scheduler (hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-resourcemanager/scheduler/fair)
- **核心功能**: 公平调度器
- **关键类**: FairScheduler, FSLeafQueue, FSParentQueue, FSQueue, SchedulingPolicy
- **主要职责**:
  - 公平资源分配
  - 动态队列创建
  - 多种调度策略（FIFO、Fair、DRF）

---

### FIFO Scheduler (hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-resourcemanager/scheduler/fifo)
- **核心功能**: 先进先出调度器
- **关键类**: FifoScheduler
- **主要职责**:
  - 简单的FIFO调度
  - 适用于小规模集群

---

## 四、MapReduce 计算框架模块

---

## hadoop-mapreduce-client-core (hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-core)
- **核心功能**: MapReduce核心API
- **关键类**: Job, Mapper, Reducer, InputFormat, OutputFormat, RecordReader, RecordWriter, Partitioner, Counter, JobContext, TaskAttemptContext
- **主要职责**:
  - 定义MapReduce编程模型接口
  - 作业配置与提交
  - 输入输出格式抽象
  - 计数器机制

---

## hadoop-mapreduce-client-app (hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-app)
- **核心功能**: MapReduce ApplicationMaster
- **关键类**: MRAppMaster, TaskAttemptListener, TaskHeartbeatHandler, JobImpl, TaskImpl
- **主要职责**:
  - MapReduce作业的ApplicationMaster
  - 任务调度与跟踪
  - 任务失败重试
  - 与YARN交互

---

## hadoop-mapreduce-client-jobclient (hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-jobclient)
- **核心功能**: MapReduce作业客户端
- **关键类**: JobClient, ResourceRequestDelegate
- **主要职责**:
  - 作业提交
  - 作业状态查询
  - 与ResourceManager通信

---

## hadoop-mapreduce-client-shuffle (hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-shuffle)
- **核心功能**: MapReduce Shuffle服务
- **关键类**: ShuffleHandler
- **主要职责**:
  - Reduce任务的中间数据拉取
  - 运行在NodeManager上的辅助服务

---

## hadoop-mapreduce-client-hs (hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-hs)
- **核心功能**: MapReduce历史服务器
- **关键类**: JobHistoryServer, HistoryClientService
- **主要职责**:
  - 保存作业历史信息
  - 提供历史作业查询

---

## hadoop-mapreduce-client-common (hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-common)
- **核心功能**: MapReduce公共组件
- **关键类**: MRConstants, TaskType
- **主要职责**:
  - 提供MapReduce客户端的公共功能

---

## hadoop-mapreduce-examples (hadoop-mapreduce-project/hadoop-mapreduce-examples)
- **核心功能**: MapReduce示例程序
- **关键类**: WordCount, Sort, Grep, SecondarySort, Join, TeraSort
- **主要职责**:
  - 提供MapReduce学习示例
  - 基准测试程序

---

## 五、工具模块（Hadoop Tools）

---

## hadoop-distcp (hadoop-tools/hadoop-distcp)
- **核心功能**: 分布式复制工具
- **关键类**: DistCp, CopyListing, DistCpOptions, CopyMapper
- **主要职责**:
  - 大规模集群间数据复制
  - 增量复制
  - 支持MapReduce并行复制

---

## hadoop-streaming (hadoop-tools/hadoop-streaming)
- **核心功能**: Hadoop Streaming
- **关键类**: HadoopStreaming, StreamJob, PipeMapper, PipeReducer
- **主要职责**:
  - 支持非Java语言的MapReduce开发
  - 通过标准输入输出与脚本交互

---

## hadoop-archives (hadoop-tools/hadoop-archives)
- **核心功能**: Hadoop归档工具
- **关键类**: HadoopArchives, HarFileSystem
- **主要职责**:
  - 将小文件打包成HAR格式
  - 减少NameNode内存占用

---

## hadoop-aws (hadoop-tools/hadoop-aws)
- **核心功能**: AWS S3文件系统支持
- **关键类**: S3AFileSystem, S3AInputStream, S3ABlockOutputStream, S3AUtils
- **主要职责**:
  - 实现S3A文件系统
  - 支持S3作为HDFS的替代存储
  - 支持S3加密、多部分上传

---

## hadoop-azure (hadoop-tools/hadoop-azure)
- **核心功能**: Azure Blob存储支持
- **关键类**: NativeAzureFileSystem, AzureNativeFileSystemStore, BlockBlobAppendStream
- **主要职责**:
  - 实现WASB文件系统
  - 支持Azure Blob Storage访问

---

## hadoop-azure-datalake (hadoop-tools/hadoop-azure-datalake)
- **核心功能**: Azure Data Lake Storage支持
- **关键类**: AdlFileSystem
- **主要职责**:
  - 实现ADLS文件系统
  - 支持Azure Data Lake Storage访问

---

## hadoop-gcp (hadoop-tools/hadoop-gcp)
- **核心功能**: Google Cloud Storage支持
- **关键类**: GoogleHadoopFileSystem, GCSFileSystem
- **主要职责**:
  - 实现GCS文件系统
  - 支持Google Cloud Storage访问

---

## hadoop-gridmix (hadoop-tools/hadoop-gridmix)
- **核心功能**: 集群压力测试工具
- **关键类**: Gridmix, GridmixJob
- **主要职责**:
  - 模拟真实工作负载
  - 集群性能基准测试

---

## hadoop-rumen (hadoop-tools/hadoop-rumen)
- **核心功能**: 作业日志分析工具
- **关键类**: JobTrace, TraceBuilder
- **主要职责**:
  - 解析作业历史日志
  - 生成工作负载描述

---

## hadoop-sls (hadoop-tools/hadoop-sls)
- **核心功能**: 调度器负载模拟器（Scheduler Load Simulator）
- **关键类**: SLSRunner, SchedulerWrapper
- **主要职责**:
  - 模拟YARN调度器负载
  - 调度器性能测试

---

## hadoop-datajoin (hadoop-tools/hadoop-datajoin)
- **核心功能**: 数据连接工具
- **关键类**: DataJoinMapperBase, DataJoinReducerBase
- **主要职责**:
  - 支持MapReduce中的数据连接操作

---

## hadoop-extras (hadoop-tools/hadoop-extras)
- **核心功能**: 额外工具集
- **关键类**: IdentityMapper, IdentityReducer
- **主要职责**:
  - 提供额外的MapReduce工具类

---

## hadoop-pipes (hadoop-tools/hadoop-pipes)
- **核心功能**: Hadoop Pipes（C++ MapReduce支持）
- **关键类**: PipesMapper, PipesReducer
- **主要职责**:
  - 支持C++编写MapReduce程序

---

## hadoop-archive-logs (hadoop-tools/hadoop-archive-logs)
- **核心功能**: 日志归档工具
- **关键类**: ArchiveLogs
- **主要职责**:
  - 聚合和归档容器日志

---

## hadoop-federation-balance (hadoop-tools/hadoop-federation-balance)
- **核心功能**: 联邦数据均衡
- **关键类**: FederationBalance, MountTableProcedure
- **主要职责**:
  - HDFS联邦环境下的数据迁移与均衡

---

## hadoop-resourceestimator (hadoop-tools/hadoop-resourceestimator)
- **核心功能**: 资源估算工具
- **关键类**: ResourceEstimator, SkylineFilter
- **主要职责**:
  - 预测作业资源需求

---

## hadoop-dynamometer (hadoop-tools/hadoop-dynamometer)
- **核心功能**: 集群基准测试工具
- **关键类**: Dynamometer, DynoInfraCLI
- **主要职责**:
  - 模拟NameNode负载
  - 集群性能测试

---

## hadoop-kafka (hadoop-tools/hadoop-kafka)
- **核心功能**: Kafka支持
- **关键类**: KafkaInputFormat, KafkaOutputFormat
- **主要职责**:
  - 支持Kafka作为MapReduce数据源

---

## hadoop-openstack (hadoop-tools/hadoop-openstack)
- **核心功能**: OpenStack Swift存储支持
- **关键类**: SwiftFileSystem
- **主要职责**:
  - 实现Swift文件系统

---

## hadoop-aliyun (hadoop-tools/hadoop-aliyun)
- **核心功能**: 阿里云OSS存储支持
- **关键类**: AliyunOSSFileSystem
- **主要职责**:
  - 实现阿里云对象存储访问

---

## hadoop-benchmark (hadoop-tools/hadoop-benchmark)
- **核心功能**: 基准测试工具
- **关键类**: BenchmarkJob
- **主要职责**:
  - 提供性能基准测试

---

## hadoop-fs2img (hadoop-tools/hadoop-fs2img)
- **核心功能**: 文件系统镜像工具
- **主要职责**:
  - 生成HDFS镜像文件

---

## 六、客户端模块（Hadoop Client Modules）

---

## hadoop-client (hadoop-client-modules/hadoop-client)
- **核心功能**: Hadoop客户端依赖聚合
- **关键类**: 无（仅依赖管理）
- **主要职责**:
  - 聚合所有客户端依赖
  - 提供一站式客户端库

---

## hadoop-client-api (hadoop-client-modules/hadoop-client-api)
- **核心功能**: 客户端API定义
- **关键类**: 仅包含接口定义
- **主要职责**:
  - 定义客户端公共API

---

## hadoop-client-runtime (hadoop-client-modules/hadoop-client-runtime)
- **核心功能**: 客户端运行时依赖
- **关键类**: 无
- **主要职责**:
  - 提供运行时依赖

---

## hadoop-client-minicluster (hadoop-client-modules/hadoop-client-minicluster)
- **核心功能**: 迷你集群客户端
- **关键类**: 无
- **主要职责**:
  - 提供迷你集群测试支持

---

## hadoop-minicluster (hadoop-minicluster)
- **核心功能**: 迷你集群测试框架
- **关键类**: MiniDFSCluster, MiniYARNCluster, MiniMRCluster
- **主要职责**:
  - 提供单进程测试环境
  - 支持集成测试

---

## 七、云存储模块（Hadoop Cloud Storage Project）

---

## hadoop-cloud-storage (hadoop-cloud-storage-project/hadoop-cloud-storage)
- **核心功能**: 云存储公共抽象
- **关键类**: 抽象接口
- **主要职责**:
  - 云存储的公共接口定义

---

## hadoop-cos (hadoop-cloud-storage-project/hadoop-cos)
- **核心功能**: 腾讯云COS支持
- **关键类**: CosFileSystem
- **主要职责**:
  - 实现腾讯云对象存储访问

---

## hadoop-huaweicloud (hadoop-cloud-storage-project/hadoop-huaweicloud)
- **核心功能**: 华为云OBS支持
- **关键类**: OBSFileSystem
- **主要职责**:
  - 实现华为云对象存储访问

---

## hadoop-tos (hadoop-cloud-storage-project/hadoop-tos)
- **核心功能**: 字节跳动TOS支持
- **关键类**: TosFileSystem
- **主要职责**:
  - 实现字节跳动对象存储访问

---

## 八、构建与分发模块

---

## hadoop-project (hadoop-project)
- **核心功能**: 项目依赖管理
- **主要职责**:
  - 定义所有子模块的公共依赖版本
  - Maven POM模板

---

## hadoop-project-dist (hadoop-project-dist)
- **核心功能**: 项目分发配置
- **主要职责**:
  - 定义发布包结构

---

## hadoop-assemblies (hadoop-assemblies)
- **核心功能**: 打包配置
- **主要职责**:
  - 定义发布包结构
  - Maven Assembly配置

---

## hadoop-dist (hadoop-dist)
- **核心功能**: 分发包构建
- **主要职责**:
  - 构建最终的发布包

---

## hadoop-build-tools (hadoop-build-tools)
- **核心功能**: 构建工具
- **主要职责**:
  - Checkstyle配置
  - 构建脚本

---

## hadoop-maven-plugins (hadoop-maven-plugins)
- **核心功能**: 自定义Maven插件
- **主要职责**:
  - 提供Hadoop特定的Maven插件

---

## 九、其他重要模块

---

## hadoop-yarn-ui (hadoop-yarn-project/hadoop-yarn/hadoop-yarn-ui)
- **核心功能**: YARN Web UI
- **主要职责**:
  - 提供YARN的Web界面
  - 展示集群状态和应用程序信息

---

## hadoop-yarn-csi (hadoop-yarn-project/hadoop-yarn/hadoop-yarn-csi)
- **核心功能**: CSI（Container Storage Interface）支持
- **主要职责**:
  - 支持容器存储接口
  - 与Kubernetes CSI集成

---

## hadoop-yarn-catalog (hadoop-yarn-project/hadoop-yarn/hadoop-yarn-applications/hadoop-yarn-applications-catalog)
- **核心功能**: YARN应用程序目录
- **主要职责**:
  - 应用程序模板管理
  - 应用程序目录服务

---

## hadoop-yarn-applications-unmanaged-am-launcher (hadoop-yarn-project/hadoop-yarn/hadoop-yarn-applications/hadoop-yarn-applications-unmanaged-am-launcher)
- **核心功能**: 非托管ApplicationMaster启动器
- **关键类**: UnmanagedAMLauncher
- **主要职责**:
  - 在YARN外部启动ApplicationMaster
  - 用于调试和开发

---

## 总结

Apache Hadoop 是一个完整的大数据基础平台，主要包含三大核心组件：

### 1. HDFS (Hadoop Distributed File System)
- 分布式文件系统，提供高可靠性、高吞吐量的数据存储
- 核心角色：NameNode（元数据管理）、DataNode（数据存储）
- 支持联邦架构（RBF）和HA高可用

### 2. YARN (Yet Another Resource Negotiator)
- 资源管理平台，负责集群资源调度和应用程序管理
- 核心角色：ResourceManager（全局资源管理）、NodeManager（单节点容器管理）
- 支持多种调度器：Capacity、Fair、FIFO

### 3. MapReduce
- 分布式计算框架，提供批处理计算能力
- 编程模型：Map（映射）、Shuffle（洗牌）、Reduce（归约）
- 支持非Java语言开发（Streaming、Pipes）

此外，Hadoop 还提供了丰富的生态系统支持：
- 多种云存储后端支持（AWS S3、Azure Blob、Google Cloud Storage、阿里云OSS等）
- 完善的客户端API和工具库
- 安全认证与密钥管理（KMS）
- 联邦架构支持（RBF）
- 丰富的运维和测试工具（DistCp、Gridmix、SLS、Dynamometer等）
