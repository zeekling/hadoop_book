# 作业提交

## 作业提交流程

![pic](https://pan.zeekling.cn/zeekling/hadoop/hadoop_namenode_00002.png)

## 作业管理

- [container-executor](./container-executor.md)

# Yarn 调度器

## 先进先出调度器

## 容量调度器

![pic](https://pan.zeekling.cn/zeekling/hadoop/yarn_00002.png)

### 分配算法

![pic](https://pan.zeekling.cn/zeekling/hadoop/yarn_00001.png)

## 公平调度器

### 调度原理

![pic](https://pan.zeekling.cn/zeekling/hadoop/yarn_00003.png)

#### 缺额

![pic](https://pan.zeekling.cn/zeekling/hadoop/yarn_00004.png)

#### 资源分配方式

![pic](https://pan.zeekling.cn/zeekling/hadoop/yarn_00005.png)

样例 :

![pic](https://pan.zeekling.cn/zeekling/hadoop/yarn_00006.png)

DRF策略
![pic](https://pan.zeekling.cn/zeekling/hadoop/yarn_00007.png)

# 核心参数配置
![pic](https://pan.zeekling.cn/zeekling/hadoop/yarn_00008.png)

# hadoop-yarn-server-resourcemanager 模块功能点

## 核心主类 (根目录, 37个)

| 类别 | 类名 | 功能 |
|------|------|------|
| 核心 | ResourceManager | 主资源管理器入口 |
| 核心 | ClientRMService | 客户端RPC(提交应用/查询状态) |
| 核心 | ApplicationMasterService | AM通信服务 |
| 核心 | ResourceTrackerService | NM资源跟踪 |
| 核心 | AdminService | 管理员操作 |
| 核心 | RMContext/RMContextImpl | 运行时上下文 |
| 应用 | RMAppManager | 应用管理器 |
| 容器 | OpportunisticContainerAllocatorAMService | 机会性容器分配 |
| 节点 | NodesListManager | 节点列表管理 |
| 节点 | NMLivelinessMonitor | NM存活监控 |
| 节点 | DecommissioningNodesWatcher | 节点退役监控 |
| 高可用 | ActiveStandbyElectorBasedElectorService | ZK主备选举 |
| 高可用 | CuratorBasedElectorService | Curator选举 |
| 高可用 | EmbeddedElector | 嵌入式选举 |
| 安全 | RMSecretManagerService | 密钥管理服务 |
| 指标 | ClusterMetrics | 集群指标 |
| Web | IsResourceManagerActiveServlet | 活性检测Servlet |

## 子包功能详解

| 子包 | 文件数 | 功能描述 |
|------|--------|----------|
| **ahs** | 9 | 应用历史记录写入 |
| **amlauncher** | 4 | ApplicationMaster启动器 |
| **blacklist** | 3 | 节点黑名单管理(禁用/简单/智能) |
| **federation** | 4 | YARN联邦(状态存储/心跳) |
| **metrics** | 6 | Timeline指标发布(V1/V2) |
| **monitor** | 3 | 调度策略监控 |
| **nodelabels** | 8 | 节点标签/属性管理 |
| **placement** | 16 | 应用放置规则(14种规则) |
| **preprocessor** | 5 | 提交预处理(队列/标签/标签) |
| **recovery** | 27 | 状态存储(HDFS/LevelDB/Memory/ZK) |
| **reservation** | 21 | 资源预约系统 |
| **resource** | 3 | 资源配置文件管理 |
| **rmapp** | 15 | 应用生命周期/状态/事件 |
| **rmcontainer** | 13 | 容器状态/过期/回收 |
| **rmnode** | 15 | 节点状态/事件/资源更新 |
| **scheduler** | 41 | 调度器抽象/容量/公平/队列 |
| **security** | 16 | Token/ACL/授权管理 |
| **timelineservice** | 1 | Timeline收集器 |
| **volume** | 0 | CSI卷管理(空) |
| **webapp** | 38 | Web UI/REST API/DAO |

## 调度器子目录 (scheduler)

- AbstractYarnScheduler, CapacityScheduler, FairScheduler
- Queue, QueueMetrics, QueueStateManager
- SchedulerNode, SchedulerApp, SchedulerApplicationAttempt
- ResourceLimits, ResourceUsage, Allocation

## 预约系统子目录 (reservation)

- ReservationSystem, CapacityReservationSystem, FairReservationSystem
- Plan, PlanFollower, InMemoryPlan
- RLESparseResourceAllocation, ReservationAllocation

## 恢复系统子目录 (recovery)

- RMStateStore, FileSystemRMStateStore, ZKRMStateStore
- LeveldbRMStateStore, MemoryRMStateStore
- 各类事件: AppEvent, AppAttemptEvent, ReservationEvent

---

# hadoop-yarn-server-resourcemanager 详细功能点

## 一、核心服务 (Core Services)

| 功能 | 说明 |
|------|------|
| **ResourceManager** | 主入口类，负责整个集群资源管理 |
| **ClientRMService** | 客户端RPC服务，处理应用提交/查询等请求 |
| **ApplicationMasterService** | ApplicationMaster RPC服务，处理AM注册/资源请求 |
| **ResourceTrackerService** | NodeManager资源跟踪服务 |
| **AdminService** | 管理接口服务 |

## 二、应用管理 (Application Management)

| 功能 | 说明 |
|------|------|
| **RMAppManager** | 应用生命周期管理 |
| **RMAppImpl** | 应用状态机实现 |
| **RMAppAttempt** | 应用尝试管理 |
| **ApplicationMasterLauncher** | AM启动器 |
| **RMAppLogAggregation** | 应用日志聚合 |

## 三、节点管理 (Node Management)

| 功能 | 说明 |
|------|------|
| **NodesListManager** | 节点列表管理 |
| **NMLivelinessMonitor** | NM存活监控 |
| **RMNodeImpl** | 节点状态机实现 |
| **DecommissioningNodesWatcher** | 节点退役观察器 |

## 四、容器管理 (Container Management)

| 功能 | 说明 |
|------|------|
| **RMContainer** | 容器抽象 |
| **RMContainerImpl** | 容器状态机 |
| **ContainerAllocationExpirer** | 容器分配过期管理 |

## 五、调度器 (Schedulers)

### 5.1 Capacity Scheduler
- 队列容量管理、层级队列、资源分配、抢占策略、自动队列创建

### 5.2 Fair Scheduler
- 公平调度、资源分配、队列管理

### 5.3 FIFO Scheduler
- 先进先出调度

### 5.4 调度器公共组件
- ResourceScheduler接口、YarnScheduler抽象类、QueueMetrics、Allocation、AppSchedulingInfo

## 六、高可用 (High Availability)

| 功能 | 说明 |
|------|------|
| **ActiveStandbyElectorBasedElectorService** | 基于Zookeeper的主备选举 |
| **CuratorBasedElectorService** | Curator选举实现 |
| **EmbeddedElector** | 嵌入式选举器 |

## 七、状态存储与恢复 (State Store & Recovery)

| 功能 | 说明 |
|------|------|
| **RMStateStore** | 状态存储抽象 |
| **ZKRMStateStore** | Zookeeper状态存储 |
| **FileSystemRMStateStore** | HDFS状态存储 |
| **MemoryRMStateStore** | 内存状态存储 |
| **LeveldbRMStateStore** | LevelDB状态存储 |

## 八、节点标签 (Node Labels)

| 功能 | 说明 |
|------|------|
| **RMNodeLabelsManager** | 节点标签管理 |
| **FileSystemNodeAttributeStore** | 节点属性存储 |
| **RMDelegatedNodeLabelsUpdater** | 标签更新器 |

## 九、预约系统 (Reservation System)

| 功能 | 说明 |
|------|------|
| **ReservationSystem** | 预约系统抽象 |
| **CapacityReservationSystem** | 容量预约 |
| **FairReservationSystem** | 公平预约 |
| **PlanFollower** | 计划跟随器 |
| **ReservationInputValidator** | 预约验证 |

## 十、安全管理 (Security)

| 功能 | 说明 |
|------|------|
| **AMRMTokenSecretManager** | AM-RM Token密钥管理 |
| **NMTokenSecretManagerInRM** | NM Token密钥管理 |
| **RMDelegationTokenSecretManager** | 委托Token密钥管理 |
| **ClientToAMTokenSecretManagerInRM** | 客户端到AM Token |
| **QueueACLsManager** | 队列ACL管理 |
| **AppPriorityACLsManager** | 应用优先级ACL |

## 十一、Web UI & REST API

| 功能 | 说明 |
|------|------|
| **RMWebApp** | Web应用 |
| **RMWebServices** | REST API服务 |
| **DAO类** | 30+个数据传输对象用于API |

## 十二、指标发布 (Metrics Publishing)

| 功能 | 说明 |
|------|------|
| **SystemMetricsPublisher** | 系统指标发布 |
| **TimelineServiceV1Publisher** | Timeline v1发布器 |
| **TimelineServiceV2Publisher** | Timeline v2发布器 |

## 十三、其他组件

| 功能 | 说明 |
|------|------|
| **BlacklistManager** | 黑名单管理 |
| **PlacementManager** | 应用放置规则管理 |
| **SchedulerQueueManager** | 调度队列管理 |
| **ResourceProfilesManager** | 资源配置文件管理 |
| **ActivitiesManager** | 调度活动监控 |
| **SchedulingMonitor** | 调度监控 |
| **RMTimelineCollectorManager** | Timeline收集器管理 |
| **ClusterMetrics** | 集群指标 |
| **RMAuditLogger** | 审计日志 |

这是YARN ResourceManager的核心模块，完整实现了资源调度、应用管理、节点管理、高可用等企业级功能。

---

# hadoop-yarn-server-nodemanager 模块功能点

## 核心主类 (根目录, 24个)

| 类别 | 类名 | 功能 |
|------|------|------|
| 核心 | NodeManager | 主入口类，管理节点资源 |
| 核心 | ContainerManager | 容器管理服务 |
| 核心 | ContainerManagerImpl | 容器管理实现 |
| 核心 | Context | 运行时上下文 |
| 容器 | ContainerExecutor | 容器执行器抽象 |
| 容器 | DefaultContainerExecutor | 默认容器执行器(Linux) |
| 容器 | LinuxContainerExecutor | Linux容器执行器 |
| 容器 | WindowsSecureContainerExecutor | Windows安全容器执行器 |
| 本地化 | DeletionService | 文件删除服务 |
| 本地化 | DirectoryCollection | 目录集合管理 |
| 本地化 | LocalDirsHandlerService | 本地目录处理服务 |
| 资源监控 | NodeResourceMonitor | 节点资源监控 |
| 资源监控 | NodeResourceMonitorImpl | 节点资源监控实现 |
| 节点状态 | NodeStatusUpdater | 节点状态更新器 |
| 节点状态 | NodeStatusUpdaterImpl | 节点状态更新实现 |
| 审计 | NMAuditLogger | 审计日志 |
| Web | ResourceView | 资源视图 |

## 子包功能详解

| 子包 | 文件数 | 功能描述 |
|------|--------|----------|
| **amrmproxy** | 12 | AM-RM代理(联邦拦截器/秘钥管理) |
| **api** | 3 | 协议定义(LocalizationProtocol) |
| **collectormanager** | 1 | NM端收集器服务 |
| **containermanager** | 180+ | 容器管理核心 |
| **executor** | 9 | 容器执行上下文 |
| **health** | 5 | 节点健康检查 |
| **logaggregation** | 10 | 日志聚合服务 |
| **metrics** | 1 | 节点指标 |
| **nodelabels** | 9 | 节点标签/属性提供者 |
| **recovery** | 4 | 状态恢复 |
| **scheduler** | 1 | 分布式调度器 |
| **security** | 4 | NM安全(Token管理) |
| **timelineservice** | 4 | Timeline发布 |
| **util** | 6 | 工具类(Cgroups/硬件) |
| **webapp** | 17 | Web UI/REST API |

---

## 三、容器管理子模块 (containermanager)

### 3.1 应用管理 (application)
- Application, ApplicationImpl, ApplicationState
- ApplicationEvent/ApplicationEventType
- 应用容器完成/初始化/完成事件

### 3.2 容器管理 (container)
- Container, ContainerImpl, ContainerState
- 容器事件: Init/Kill/Pause/Resume/ReInit
- 资源事件: ResourceLocalized/ResourceFailed/ResourceUpdate
- 诊断/信号/容器更新

### 3.3 容器启动器 (launcher)
- ContainerLaunch - 容器启动
- ContainerRelaunch - 容器重启动
- ContainersLauncher - 启动器服务
- RecoveredContainerLaunch - 恢复启动
- SignalContainersLauncherEvent - 信号事件

### 3.4 本地化服务 (localizer)
- ResourceLocalizationService - 资源本地化服务
- ContainerLocalizer - 容器本地化器
- LocalResourcesTracker - 本地资源追踪
- LocalCacheCleaner/DirectoryManager - 缓存清理
- LocalizedResource - 本地化资源
- LocalizerContext - 本地化上下文

### 3.5 日志聚合 (logaggregation)
- LogAggregationService - 日志聚合服务
- AppLogAggregator - 应用日志聚合器
- 策略: AllContainer/AMOnly/Failed/None/Sample

### 3.6 容器监控 (monitor)
- ContainersMonitor - 容器监控服务
- ContainerMetrics - 容器指标
- 资源变更事件

### 3.7 容器调度 (scheduler)
- ContainerScheduler - 容器调度器
- OpportunisticContainersQueuePolicy - 机会性容器队列策略
- ResourceUtilizationTracker - 资源利用追踪

### 3.8 辅助服务 (AuxServices)
- AuxServices - 辅助服务管理
- 动态服务加载

### 3.9 容器运行时 (runtime)
- ContainerRuntime - 容器运行时抽象

### 3.10 资源插件 (resourceplugin)
- ResourcePluginManager - 资源插件管理
- DockerCommandPlugin - Docker命令插件
- NodeResourceUpdaterPlugin - 资源更新插件
- GPU/FPGA/Device框架支持

### 3.11 卷管理 (volume)
- CSI卷管理

---

## 四、节点健康检查 (health)

| 功能 | 说明 |
|------|------|
| NodeHealthCheckerService | 节点健康检查服务 |
| NodeHealthScriptRunner | 健康检查脚本运行器 |
| HealthReporter | 健康报告器 |
| ExceptionReporter | 异常报告 |

---

## 五、资源管理 (util)

| 功能 | 说明 |
|------|------|
| NodeManagerBuilderUtils | 构建器工具 |
| NodeManagerHardwareUtils | 硬件工具 |
| CgroupsLCEResourcesHandler | Cgroups资源处理 |
| DefaultLCEResourcesHandler | 默认资源处理 |
| ProcessIdFileReader | 进程ID文件读取 |

---

## 六、安全管理 (security)

| 功能 | 说明 |
|------|------|
| NMContainerTokenSecretManager | 容器Token管理 |
| NMTokenSecretManagerInNM | NM Token管理 |
| NMDelegationTokenManager | 委托Token管理 |

---

## 七、AM-RM代理 (amrmproxy)

| 功能 | 说明 |
|------|------|
| AMRMProxyService | AM-RM代理服务 |
| DefaultRequestInterceptor | 默认请求拦截器 |
| FederationInterceptor | 联邦拦截器 |
| AMRMProxyMetrics | 指标 |
| TokenSecretManager | Token密钥管理 |

---

## 八、状态恢复 (recovery)

| 功能 | 说明 |
|------|------|
| NMStateStoreService | 状态存储抽象 |
| NMLeveldbStateStoreService | LevelDB存储 |
| NMNullStateStoreService | 空存储 |
| RecoveryIterator | 恢复迭代器 |

---

## 九、Web服务 (webapp)

| 功能 | 说明 |
|------|------|
| NMWebApp | Web应用 |
| NMWebServices | REST API服务 |
| AggregatedLogsBlock | 聚合日志块 |
| ContainerLogsPage | 容器日志页面 |
| ApplicationPage/ContainerPage | 应用/容器页面 |
| WebServer | Web服务器 |
| ContainerShellWebSocket | WebSocket支持 |

---

## 十、节点标签 (nodelabels)

| 功能 | 说明 |
|------|------|
| NodeLabelsProvider | 节点标签提供者 |
| ScriptBasedNodeLabelsProvider | 脚本标签提供者 |
| ConfigurationNodeAttributesProvider | 配置属性提供者 |
| ScriptBasedNodeAttributesProvider | 脚本属性提供者 |
| NodeDescriptorsProvider | 节点描述符提供者 |

---

这是YARN NodeManager的核心模块，负责节点上容器的生命周期管理、本地资源分配、日志聚合、节点健康检查等核心功能。

---

# hadoop-yarn-server-router 模块功能点

## 核心主类 (根目录, 5个)

| 类别 | 类名 | 功能 |
|------|------|------|
| 核心 | Router | 主入口类，Router服务 |
| 核心 | RouterMetrics | 路由指标统计 |
| 核心 | RouterServerUtil | 工具类 |
| 审计 | RouterAuditLogger | 审计日志 |

## 子包功能详解

| 子包 | 文件数 | 功能描述 |
|------|--------|----------|
| **cleaner** | 1 | 子集群清理器 |
| **clientrm** | 10 | 客户端请求拦截/路由 |
| **rmadmin** | 7 | RM管理接口拦截 |
| **security** | 2 | Token密钥管理 |
| **webapp** | 35+ | Web UI/REST API |

---

## 三、客户端请求路由 (clientrm)

| 功能 | 说明 |
|------|------|
| **RouterClientRMService** | 客户端RPC服务入口 |
| **FederationClientInterceptor** | 联邦客户端拦截器 |
| **DefaultClientRequestInterceptor** | 默认请求拦截器 |
| **AbstractClientRequestInterceptor** | 拦截器抽象基类 |
| **ApplicationSubmissionContextInterceptor** | 应用提交上下文拦截 |
| **PassThroughClientRequestInterceptor** | 透传拦截器 |
| **ClientMethod** | 客户端方法封装 |
| **RouterYarnClientUtils** | Yarn客户端工具 |

---

## 四、RM管理接口 (rmadmin)

| 功能 | 说明 |
|------|------|
| **RouterRMAdminService** | RM管理服务入口 |
| **FederationRMAdminInterceptor** | 联邦RM管理拦截器 |
| **DefaultRMAdminRequestInterceptor** | 默认请求拦截器 |
| **AbstractRMAdminRequestInterceptor** | 拦截器抽象基类 |
| **RMAdminProtocolMethod** | RM管理协议方法 |
| **RMAdminRequestInterceptor** | 请求拦截器接口 |

---

## 五、Web服务 (webapp)

### 5.1 核心服务

| 功能 | 说明 |
|------|------|
| **RouterWebApp** | Web应用 |
| **RouterWebServices** | REST API服务 |
| **RouterController** | Web控制器 |

### 5.2 拦截器

| 功能 | 说明 |
|------|------|
| **AbstractRESTRequestInterceptor** | REST拦截器抽象 |
| **DefaultRequestInterceptorREST** | 默认REST拦截器 |
| **FederationInterceptorREST** | 联邦REST拦截器 |
| **RESTRequestInterceptor** | REST拦截器接口 |

### 5.3 页面/块

| 功能 | 说明 |
|------|------|
| **RouterView** | 路由视图 |
| **RouterBlock** | 路由信息块 |
| **FederationPage/Block** | 联邦页面/块 |
| **AppsPage/Block** | 应用页面/块 |
| **NodesPage/Block** | 节点页面/块 |
| **NodeLabelsPage/Block** | 节点标签页面/块 |

### 5.4 缓存 (cache)

| 功能 | 说明 |
|------|------|
| **RouterAppInfoCacheKey** | 应用信息缓存键 |

### 5.5 DAO数据传输对象

| 功能 | 说明 |
|------|------|
| **FederationClusterInfo** | 联邦集群信息 |
| **FederationClusterUserInfo** | 集群用户信息 |
| **FederationConfInfo** | 集群配置信息 |
| **FederationRMQueueAclInfo** | 队列ACL信息 |
| **RouterClusterMetrics** | 路由集群指标 |
| **RouterInfo** | 路由信息 |
| **RouterSchedulerMetrics** | 调度器指标 |
| **SubClusterResult** | 子集群结果 |
| **FederationBulkActivitiesInfo** | 批量活动信息 |

---

## 六、安全管理 (security)

| 功能 | 说明 |
|------|------|
| **RouterDelegationTokenSecretManager** | 委托Token密钥管理 |

---

## 七、清理服务 (cleaner)

| 功能 | 说明 |
|------|------|
| **SubClusterCleaner** | 子集群状态清理 |

---

## 八、核心功能总结

YARN Router是YARN Federation（联邦）架构中的关键组件，主要功能：

1. **请求路由**: 接收客户端请求，根据策略路由到合适的子集群RM
2. **负载均衡**: 在多个子集群间分配请求负载
3. **故障转移**: 检测子集群故障，将请求路由到健康集群
4. **统一入口**: 为 federation 集群提供统一的客户端接入点
5. **指标监控**: 收集和展示路由相关指标
6. **Web UI**: 提供Federation集群的统一Web管理界面

这是YARN Federation架构的核心模块，支持跨多个集群的统一资源调度和管理。

---

# hadoop-yarn-server-applicationhistoryservice 模块功能点

## 核心主类 (根目录, 12个)

| 类别 | 类名 | 功能 |
|------|------|------|
| 核心 | ApplicationHistoryServer | 主入口类，历史服务服务器 |
| 核心 | ApplicationHistoryManager | 历史管理器接口 |
| 核心 | ApplicationHistoryManagerImpl | 历史管理器实现 |
| 核心 | ApplicationHistoryManagerOnTimelineStore | 基于Timeline存储的实现 |
| 核心 | ApplicationHistoryClientService | 客户端RPC服务 |
| 核心 | ApplicationHistoryReader | 历史数据读取器 |
| 核心 | ApplicationHistoryWriter | 历史数据写入器 |
| 核心 | ApplicationHistoryStore | 历史存储抽象接口 |
| 存储 | FileSystemApplicationHistoryStore | HDFS文件系统存储 |
| 存储 | MemoryApplicationHistoryStore | 内存存储 |
| 存储 | NullApplicationHistoryStore | 空实现存储 |

## 子包功能详解

| 子包 | 文件数 | 功能描述 |
|------|--------|----------|
| **records** | 20 | 历史数据模型(Application/Attempt/Container) |
| **webapp** | 14 | Web UI/REST API |

---

## 三、历史数据模型 (records)

### 3.1 应用级别数据

| 数据类 | 功能 |
|--------|------|
| ApplicationHistoryData | 应用历史数据 |
| ApplicationStartData | 应用启动数据 |
| ApplicationFinishData | 应用完成数据 |

### 3.2 应用尝试级别数据

| 数据类 | 功能 |
|--------|------|
| ApplicationAttemptHistoryData | 应用尝试历史数据 |
| ApplicationAttemptStartData | 应用尝试启动数据 |
| ApplicationAttemptFinishData | 应用尝试完成数据 |

### 3.3 容器级别数据

| 数据类 | 功能 |
|--------|------|
| ContainerHistoryData | 容器历史数据 |
| ContainerStartData | 容器启动数据 |
| ContainerFinishData | 容器完成数据 |

### 3.4 Protocol Buffer实现
- ApplicationStartDataPBImpl
- ApplicationFinishDataPBImpl
- ApplicationAttemptStartDataPBImpl
- ApplicationAttemptFinishDataPBImpl
- ContainerStartDataPBImpl
- ContainerFinishDataPBImpl

---

## 四、Web服务 (webapp)

| 功能 | 说明 |
|------|------|
| **AHSWebApp** | Web应用 |
| **AHSWebServices** | REST API服务 |
| **AHSController** | Web控制器 |
| **AHSView** | 视图 |
| **AHSLogsPage** | 日志页面 |
| **AppPage** | 应用详情页面 |
| **AppAttemptPage** | 应用尝试页面 |
| **ContainerPage** | 容器页面 |
| **AboutPage** | 关于页面 |
| **NavBlock** | 导航块 |
| **JAXBContextResolver** | JAXB上下文解析器 |
| **ContextFactory** | 上下文工厂 |

---

## 五、核心功能总结

YARN ApplicationHistoryService (AHS) 是YARN的历史记录服务，主要功能：

1. **历史数据存储**: 存储应用、应用尝试、容器的历史运行数据
2. **多种存储后端**: 支持HDFS文件系统、内存存储
3. **数据查询**: 提供RPC和REST接口查询历史数据
4. **Timeline集成**: 可基于YARN Timeline Service存储历史数据
5. **Web UI**: 提供Web界面查看历史应用信息
6. **数据恢复**: 配合ResourceManager实现应用历史数据恢复

这是YARN的核心模块，解决了应用运行完成后历史数据丢失的问题，为应用调试、性能分析提供数据支撑。