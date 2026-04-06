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

---

# Hadoop YARN Router 模块功能详细列表

## 一、Router主服务

### 1.1 核心功能

- **Router**：主类，初始化ClientRM/RMAdmin/Web服务
- **安全登录**：Kerberos认证支持
- **指标系统**：JVM监控和Metrics2收集
- **命令行**：`format-state-store`、`remove-application-from-state-store`

### 1.2 Web服务

- **Jetty服务器**：嵌入式Web容器
- **CORS支持**：跨域资源共享
- **Web应用代理**：FedAppReportFetcher
- **UI2支持**：YARN Web UI2

### 1.3 清理功能

- **SubClusterCleaner**：超时子集群自动注销

---

## 二、客户端请求拦截器（ClientRM）

### 2.1 RPC服务

- **RouterClientRMService**：ApplicationClientProtocol实现

### 2.2 应用管理

| 操作 | 功能 |
|------|------|
| getNewApplication | 获取新应用ID |
| submitApplication | 提交应用到子集群 |
| getApplicationReport | 获取应用报告 |
| getApplications | 获取应用列表（并行合并） |
| forceKillApplication | 强制终止应用 |
| updateApplicationPriority | 更新优先级 |
| moveApplicationAcrossQueues | 跨队列移动 |

### 2.3 集群信息

- **getClusterMetrics**：集群度量
- **getClusterNodes**：节点信息
- **getQueueInfo/QueueUserAcls**：队列信息

### 2.4 容器管理

- **getContainer/Containers**：容器信息
- **getApplicationAttempt**：应用尝试信息

### 2.5 资源预约

- **getNewReservation/SubmitReservation**：创建/提交预约
- **updateReservation/deleteReservation**：更新/删除预约

### 2.6 委托令牌

- **getDelegationToken/Renew/Cancel**：令牌管理

### 2.7 节点标签

- **getClusterNodeLabels/LabelsToNodes**：标签管理

---

## 三、RM管理拦截器

### 3.1 RPC服务

- **RouterRMAdminService**：ResourceManagerAdministrationProtocol实现

### 3.2 队列管理

- **refreshQueues**：刷新队列配置
- **saveFederationQueuePolicy**：保存队列策略
- **batchSaveFederationQueuePolicies**：批量保存

### 3.3 节点管理

- **refreshNodes/NodesResources**：刷新节点
- **updateNodeResource**：更新节点资源
- **checkForDecommissioningNodes**：检查下线节点

### 3.4 用户组管理

- **refreshUserToGroupsMappings**：刷新用户组映射
- **refreshSuperUserGroupsConfiguration**：刷新超级用户

### 3.5 安全管理

- **refreshAdminAcls/ServiceAcls**：刷新ACL

### 3.6 节点标签

- **addToClusterNodeLabels/removeFromClusterNodeLabels**：标签操作

---

## 四、Web REST API

### 4.1 RouterWebServices

- **应用API**：createNewApplication/submit/getApplications等
- **集群API**：getClusterMetrics/Info/Nodes
- **队列API**：getQueueInfo/Acls/updateAppQueue
- **容器API**：getContainer/Containers
- **预约API**：create/submit/update/delete/listReservation
- **委托令牌API**：get/renew/cancelToken
- **调度器API**：getScheduler/Activities/Statistics

### 4.2 FederationInterceptorREST

- **REST拦截器**：联邦REST请求处理
- **缓存支持**：LRU应用信息缓存

### 4.3 Web界面

- **页面**：About/Apps/Nodes/NodeLabels/Federation
- **块**：AppsBlock/NodesBlock/FederationBlock等

---

## 五、安全功能

- **RouterDelegationTokenSecretManager**：令牌密钥管理
- **RouterPolicyProvider**：授权策略

---

## 六、缓存与指标

### 缓存

- **RouterAppInfoCacheKey**：应用信息缓存键

### 指标（RouterMetrics）

- 应用提交/创建/终止成功数
- 集群度量/节点/队列获取成功数
- 委托令牌操作成功数

### 审计（RouterAuditLogger）

- 所有客户端操作审计日志

---

## 七、拦截器管道架构

| 管道 | 最后一站 |
|------|---------|
| ClientRM | FederationClientInterceptor |
| RMAdmin | FederationRMAdminInterceptor |
| REST | FederationInterceptorREST |

---

# hadoop-yarn-server-timelineservice 模块功能点

Timeline Service V2 是 YARN 的应用历史/时间线服务，提供应用运行时信息和历史数据的收集与查询功能。

## 核心主类

| 类别 | 类名 | 功能 |
|------|------|------|
| 收集器 | TimelineCollector | 抽象时间线收集器基类，处理实体写入/聚合 |
| 收集器 | TimelineCollectorManager | 收集器生命周期管理，Writer创建/刷新任务调度 |
| 收集器 | NodeTimelineCollectorManager | NM端收集器管理，启动REST服务器 |
| 收集器 | AppLevelTimelineCollector | 应用级收集器，实体聚合状态表管理 |
| 收集器 | AppLevelTimelineCollectorWithAgg | 带聚合功能的应用级收集器 |
| 收集器 | PerNodeTimelineCollectorsAuxService | NM辅助服务，运行在NodeManager上 |
| 读取器 | TimelineReader | 时间线读取器接口 |
| 读取器 | TimelineReaderManager | 读取器管理器 |
| 读取器 | TimelineReaderWebServices | REST读取API服务端点 |
| 写入器 | TimelineWriter | 时间线写入器接口 |
| 写入器 | FileSystemTimelineWriterImpl | 基于HDFS的文件系统实现 |
| 写入器 | FileSystemTimelineReaderImpl | 基于HDFS的文件系统读取实现 |
| 写入器 | NoOpTimelineWriterImpl | 空写入实现 |
| 写入器 | NoOpTimelineReaderImpl | 空读取实现 |

## 收集器包 (collector)

| 类名 | 功能 |
|------|------|
| TimelineCollectorWebService | REST收集API服务端点，接收实体写入请求 |
| TimelineCollectorContext | 收集上下文信息(集群/用户/Flow/应用) |
| TimelineUIDConverter | UID编码/解码器(ApplicationUID, GenericEntityUID) |
| TimelineFromIdConverter | 分页ID转换器 |

## 读取器包 (reader)

| 类别 | 类名 | 功能 |
|------|------|------|
| 上下文 | TimelineReaderContext | 读取查询上下文 |
| 上下文 | TimelineDataToRetrieve | 指定要检索的数据字段 |
| 上下文 | TimelineEntityFilters | 实体过滤条件 |
| Web服务 | TimelineReaderServer | 读取器Web服务器 |
| Web服务 | TimelineReaderWebServices | REST API实现(查询实体/单个实体/类型) |
| 工具 | TimelineReaderUtils | 读取器工具方法 |
| 工具 | TimelineReaderWebServicesUtils | Web服务工具方法 |
| 过滤器 | TimelineFilter | 过滤器接口 |
| 过滤器 | TimelineFilterList | 过滤器列表(AND/OR) |
| 过滤器 | TimelineKeyValueFilter | 键值过滤 |
| 过滤器 | TimelineKeyValuesFilter | 多值键过滤 |
| 过滤器 | TimelineExistsFilter | 存在性过滤 |
| 过滤器 | TimelineCompareFilter | 比较过滤(> < = !=) |
| 过滤器 | TimelinePrefixFilter | 前缀过滤 |
| 解析器 | TimelineParser | 解析器接口 |
| 解析器 | TimelineParserForRelationFilters | 关系过滤解析 |
| 解析器 | TimelineParserForKVFilters | KV过滤解析 |
| 解析器 | TimelineParserForNumericFilters | 数值过滤解析 |
| 解析器 | TimelineParserForExistFilters | 存在性过滤解析 |
| 解析器 | TimelineParserForCompareExpr | 比较表达式解析 |
| 解析器 | TimelineParserForDataToRetrieve | 数据检索解析 |
| 安全 | TimelineReaderAuthenticationFilterInitializer | 认证过滤器初始化 |
| 安全 | TimelineReaderWhitelistAuthorizationFilter | 白名单授权过滤 |
| 安全 | TimelineReaderWhitelistAuthorizationFilterInitializer | 白名单初始化 |

## 存储包 (storage)

| 类别 | 类名 | 功能 |
|------|------|------|
| 核心 | TimelineWriter | 写入接口(写实体/写域/聚合/刷新) |
| 核心 | TimelineReader | 读取接口(查实体/查实体集/查类型/健康检查) |
| 核心 | TimelineSchemaCreator | HBase表结构创建器 |
| 聚合 | TimelineAggregationTrack | 聚合轨道枚举(APP/FLOW/USER/QUEUE) |
| 聚合 | OfflineAggregationWriter | 离线聚合写入器 |
| 聚合 | OfflineAggregationInfo | 离线聚合信息 |
| 监控 | TimelineStorageMonitor | 存储监控 |
| 实现 | FileSystemTimelineWriterImpl | 基于HDFS的Writer实现 |
| 实现 | FileSystemTimelineReaderImpl | 基于HDFS的Reader实现 |

## 指标包 (metrics)

| 类名 | 功能 |
|------|------|
| TimelineReaderMetrics | 读取器指标(查询延迟/成功率) |
| PerNodeAggTimelineCollectorMetrics | NM收集器指标(PUT延迟/异步PUT延迟) |

## 安全包 (security)

| 类名 | 功能 |
|------|------|
| TimelineV2DelegationTokenSecretManagerService | V2委托令牌密钥管理服务 |
| CollectorNodemanagerSecurityInfo | NM收集器安全信息 |

## 核心功能点

### 1. 实体收集 (Timeline Collection)

- **同步写入** (putEntities): 同步写入关键实体和事件，实时性强
- **异步写入** (putEntitiesAsync): 异步写入大量实体，使用线程池缓冲
- **域管理** (putDomain): 时间线域的创建和更新
- **写入重试**: 存储失败时自动重试机制
- **刷新调度**: 定期刷新缓冲区到存储

### 2. 实体聚合 (Aggregation)

- **实时聚合**: 在收集器内进行内存实时聚合
- **聚合状态表** (AggregationStatusTable): 维护每个实体类型的聚合状态
- **聚合操作**: 支持SUM/MAX/MIN/AVG等聚合操作
- **聚合轨道**: 支持APP/FLOW/USER/QUEUE多维度聚合
- **跳过聚合**: 可配置跳过特定实体类型的聚合

### 3. 实体读取 (Timeline Reading)

- **REST API**: 完整的REST查询接口
- **多粒度查询**: 支持应用级别、Flow级别、集群级别的查询
- **字段检索**: 可指定检索的字段(事件/信息/指标/配置/关系)
- **过滤条件**: 支持时间窗口/数量限制/信息过滤/配置过滤/指标过滤/事件过滤
- **关系过滤**: 支持relatesTo/isRelatedTo关系过滤
- **分页查询**: 支持fromId分页查询
- **UID支持**: 支持复合UID查询(cluster/user/flow/run/app/entity)

### 4. 过滤解析器 (Filter Parsing)

- **关系过滤**: 解析实体间关系过滤表达式
- **KV过滤**: 解析键值对过滤表达式
- **数值过滤**: 解析数值比较过滤表达式
- **存在性过滤**: 解析字段存在性过滤
- **比较表达式**: 解析> < = !=等比较表达式
- **前缀过滤**: 支持字符串前缀匹配

### 5. 存储后端

- **文件系统存储**: 基于HDFS的文件系统实现
- **可插拔Writer**: 通过配置TimelineWriter实现类切换后端
- **健康检查**: 定期检查存储健康状态

### 6. 安全机制

- **委托令牌**: 支持Timeline delegation token
- **令牌更新**: 自动更新即将过期的令牌
- **认证过滤**: 支持Kerberos认证
- **白名单**: 支持IP白名单授权

### 7. 指标监控

- **写入延迟**: 记录同步/异步写入延迟(分位数)
- **读取延迟**: 记录查询延迟
- **成功率**: 记录操作成功率
- **Metrics2集成**: 集成Hadoop Metrics2框架

### 8. Web服务

- **Collector REST**: 接收实体写入 (PUT /ws/v2/timeline/entities)
- **Reader REST**: 提供查询API (GET /ws/v2/timeline/...)
- **健康检查**: /health 端点
- **集群上下文**: 支持多集群环境

### 9. 实体类型

- YARN_CLUSTER: 集群实体
- YARN_FLOW_RUN: Flow运行实体
- YARN_APPLICATION: 应用实体
- YARN_APPLICATION_ATTEMPT: 应用尝试实体
- YARN_CONTAINER: 容器实体
- YARN_QUEUE: 队列实体
- YARN_USER: 用户实体

### 10. 子应用支持

- **SubApplicationEntity**: 支持子应用实体
- **subappwrite参数**: 支持子应用级别写入

---

# hadoop-yarn-server-timelineservice 详细功能点

根据代码分析，`hadoop-yarn-server-timelineservice` 模块是 Apache Hadoop YARN Timeline Service (时间轴服务 v2)，主要功能点如下：

## 一、Timeline Collector（时间轴数据收集）

| 功能类 | 说明 |
|--------|------|
| `TimelineCollector` | 抽象收集器，处理时间轴数据写入 |
| `TimelineCollectorManager` | 收集器管理服务 |
| `TimelineCollectorWebService` | 收集器REST接口 |
| `TimelineCollectorContext` | 收集上下文信息 |
| `AppLevelTimelineCollector` | 应用级时间轴收集器 |
| `AppLevelTimelineCollectorWithAgg` | 支持聚合的应用级收集器 |
| `NodeTimelineCollectorManager` | 节点级收集器管理 |
| `PerNodeTimelineCollectorsAuxService` | 节点辅助服务 |

## 二、Timeline Reader（时间轴数据读取）

| 功能类 | 说明 |
|--------|------|
| `TimelineReaderWebServices` | REST读取接口 (`/ws/v2/timeline`) |
| `TimelineReaderManager` | 读取管理器 |
| `TimelineReaderServer` | 读取服务主类 |
| `TimelineReaderContext` | 读取上下文 |
| `TimelineReaderUtils` | 工具类 |

**过滤器功能：**

- `TimelineFilter` - 过滤器基类
- `TimelineKeyValueFilter` - 键值过滤
- `TimelineKeyValuesFilter` - 多键值过滤
- `TimelinePrefixFilter` - 前缀过滤
- `TimelineExistsFilter` - 存在性过滤
- `TimelineCompareFilter` - 比较过滤
- `TimelineFilterList` - 过滤器列表
- `TimelineEntityFilters` - 实体过滤器
- `TimelineDataToRetrieve` - 数据检索配置

**解析器：**

- `TimelineParser` - 解析器接口
- `TimelineParserForKVFilters` - KV过滤器解析
- `TimelineParserForNumericFilters` - 数值过滤解析
- `TimelineParserForRelationFilters` - 关系过滤解析
- `TimelineParserForExistFilters` - 存在性过滤解析
- `TimelineParserForCompareExpr` - 比较表达式解析

## 三、Timeline Storage（时间轴存储）

| 功能类 | 说明 |
|--------|------|
| `TimelineWriter` | 写入接口 |
| `TimelineReader` | 读取接口 |
| `FileSystemTimelineWriterImpl` | HDFS文件系统存储实现 |
| `FileSystemTimelineReaderImpl` | HDFS文件系统读取实现 |
| `NoOpTimelineWriterImpl` | 空实现 |
| `NoOpTimelineReaderImpl` | 空读取实现 |
| `TimelineSchemaCreator` | 数据库Schema创建工具 |
| `TimelineStorageMonitor` | 存储健康监控 |
| `OfflineAggregationWriter` | 离线聚合写入 |
| `TimelineAggregationTrack` | 聚合跟踪 |

## 四、Security（安全认证）

| 功能类 | 说明 |
|--------|------|
| `TimelineV2DelegationTokenSecretManagerService` | Delegation Token管理 |
| `TimelineReaderAuthenticationFilterInitializer` | 认证过滤器初始化 |
| `TimelineReaderWhitelistAuthorizationFilter` | 白名单授权过滤 |
| `TimelineReaderWhitelistAuthorizationFilterInitializer` | 授权初始化 |

## 五、Metrics（性能指标）

| 功能类 | 说明 |
|--------|------|
| `TimelineReaderMetrics` | 读取器指标 |
| `PerNodeAggTimelineCollectorMetrics` | 节点聚合收集器指标 |

## 六、核心功能总结

1. **数据收集** - 收集YARN应用/容器的运行事件和指标
2. **数据存储** - 支持HDFS文件系统存储和可扩展的存储后端
3. **数据读取** - 提供REST API查询时间轴数据
4. **数据聚合** - 支持应用级和节点级数据聚合
5. **过滤查询** - 支持多种过滤条件（键值、前缀、比较、存在性）
6. **安全认证** - Kerberos认证和委托令牌支持
7. **监控指标** - 收集器和读取器的性能指标

---

# hadoop-yarn-server-sharedcachemanager 模块功能点

## 一、核心主类

| 类别 | 类名 | 功能 |
|------|------|------|
| 核心 | SharedCacheManager | 主入口服务，管理元数据、处理RPC调用、持久化存储、清理任务 |
| 上传 | SharedCacheUploaderService | 处理NodeManager上传请求的RPC服务 |
| 清理 | CleanerService | 定期清理过期缓存条目 |
| 客户端 | ClientProtocolService | 客户端协议服务 |
| 管理 | SCMAdminProtocolService | 管理协议服务 |
| 检查 | RemoteAppChecker | 远程应用检查器 |

## 二、存储层 (store)

| 类名 | 功能 |
|------|------|
| SCMStore | 抽象数据存储基类，线程安全 |
| InMemorySCMStore | 内存存储实现 |
| SharedCacheResource | 缓存资源模型 |
| SharedCacheResourceReference | 资源引用模型 |

## 三、Web服务 (webapp)

| 类名 | 功能 |
|------|------|
| SCMWebServer | Web服务器 |
| SCMController | Web控制器 |
| SCMOverviewPage | 概览页面 |
| SCMMetricsInfo | 指标信息 |

## 四、指标 (metrics)

| 类名 | 功能 |
|------|------|
| SharedCacheUploaderMetrics | 上传者指标 |
| ClientSCMMetrics | 客户端指标 |
| CleanerMetrics | 清理器指标 |

## 五、核心功能

1. **缓存资源管理**
   - 资源声明 (claimResource)
   - 资源释放 (releaseResource)
   - 资源查询 (getResource)

2. **缓存上传协议**
   - canUpload: 检查是否可以上传
   - notify: 通知缓存管理器

3. **缓存清理**
   - 定期清理过期条目
   - 全局清理锁机制
   - 并发清理任务调度

4. **应用检查**
   - 检查应用是否仍在运行
   - 清理无引用的缓存

5. **安全机制**
   - ACL访问控制
   - Kerberos认证支持

6. **Web UI**
   - 缓存使用统计
   - 缓存文件列表
   - 清理任务状态

## 六、核心特点

- **分布式缓存**: 多个NodeManager共享Jar包/字典等公共资源
- **节省磁盘**: 相同资源只存储一份
- **自动清理**: 定期清理无引用的过期缓存
- **指标监控**: 上传/清理操作指标统计
- **可插拔存储**: 支持内存/HDFS等多种存储后端

---

# hadoop-yarn-server-web-proxy 模块功能点

## 一、核心主类

| 类别 | 类名 | 功能 |
|------|------|------|
| 核心 | WebAppProxyServer | 主入口服务器，管理代理服务生命周期 |
| 核心 | WebAppProxy | 代理核心服务，启动HTTP服务器 |
| 核心 | WebAppProxyServlet | Servlet实现，处理代理请求转发 |
| 工具 | ProxyUriUtils | URI解析/构建工具 |
| 工具 | ProxyUtils | 代理通用工具方法 |
| 工具 | ProxyCA | 代理证书管理 |

## 二、应用报告获取 (AppReportFetcher)

| 类名 | 功能 |
|------|------|
| AppReportFetcher | 应用报告获取抽象类 |
| DefaultAppReportFetcher | 默认实现，单RM环境 |
| FedAppReportFetcher | 联邦环境多RM获取实现 |

## 三、AM Filter (amfilter)

| 类名 | 功能 |
|------|------|
| AmIpFilter | AM IP过滤器，验证请求来源IP |
| AmIpServletRequestWrapper | Servlet请求包装器 |
| AmIpPrincipal | IP主体信息 |
| AmFilterInitializer | Filter初始化器 |

## 四、核心功能

### 1. 请求代理

- **URL重写**: 解析应用URL并转发到AM Web界面
- **Cookie传递**: 透明传递Cookie到后端AM
- **HTTP方法转发**: 支持GET/POST/PUT等方法
- **响应代理**: 将AM的HTTP响应返回给用户

### 2. 安全机制

- **Kerberos认证**: 支持安全模式下的认证
- **IP白名单**: 限制可访问的代理地址
- **ACL访问控制**: 配置访问控制列表
- **证书管理**: ProxyCA用于HTTPS支持

### 3. 应用发现

- **RM查询**: 从ResourceManager获取应用信息
- **追踪URI**: 支持自定义追踪URI
- **联邦支持**: Federation环境下多RM查询

### 4. Filter链

- **请求验证**: 验证请求是否来自合法RM/NM
- **IP过滤**: 基于IP的访问控制
- **安全传递**: 安全传递用户身份

### 5. 配置选项

- **代理地址**: 可配置监听地址和端口
- **超时设置**: 代理连接超时配置
- ** ACL配置**: 访问控制列表

## 五、工作流程

1. 用户通过代理访问 `http://proxy-host:port/proxy/application_123_456/`
2. ProxyServlet 从RM获取应用信息
3. 获取AM的追踪URL
4. 将请求转发到AM Web界面
5. 响应返回给用户

## 六、核心特点

- **透明访问**: 用户无需知道AM的实际地址
- **安全防护**: IP过滤和ACL保障安全
- **联邦支持**: 支持YARN Federation环境
- **单点入口**: 提供统一的AM访问入口
- **Kerberos集成**: 与Hadoop安全框架集成

---

# hadoop-yarn-applications 模块功能点

## 一、子模块概览

| 子模块 | 说明 |
|--------|------|
| distributedshell | 分布式Shell应用示例 |
| unmanaged-am-launcher | 非托管AM启动器 |
| yarn-services | YARN服务框架(Slider) |
| yarn-applications-catalog | 应用目录 |
| yarn-applications-mawo | 未托管应用工作负载编排器 |

---

## 二、DistributedShell (分布式Shell)

| 类别 | 类名 | 功能 |
|------|------|------|
| 客户端 | Client | 提交分布式Shell应用到集群 |
| AM | ApplicationMaster | 管理容器执行Shell命令 |
| 插件 | DistributedShellTimelinePlugin | Timeline发布插件 |
| 配置 | PlacementSpec | 容器放置规则 |
| 常量 | DSConstants | 常量定义 |

**核心功能：**
- 在多容器中执行Shell命令
- 支持容器数量/资源/优先级配置
- Timeline指标发布
- 容器放置规则

---

## 三、UnmanagedAMLauncher (非托管AM启动器)

| 类名 | 功能 |
|------|------|
| UnmanagedAMLauncher | 启动非托管的ApplicationMaster |

**核心功能：**
- 启动用户自己的AM（非YARN管理）
- 支持Kerberos认证
- 用于迁移现有应用到YARN

---

## 四、YARN Services (Slider)

### 4.1 核心组件

| 类别 | 类名 | 功能 |
|------|------|------|
| 主入口 | ServiceMaster | AM主类，管理服务生命周期 |
| 管理 | ServiceManager | 服务状态/组件管理 |
| 调度 | ServiceScheduler | 容器调度 |
| 客户端 | Client | 服务提交/管理CLI |
| AM服务 | ClientAMService | AM与客户端通信 |

### 4.2 组件管理 (component)

| 类名 | 功能 |
|------|------|
| Component | 组件定义(多个实例) |
| ComponentInstance | 组件实例 |
| ComponentRestartPolicy | 重启策略(Always/Never/OnFailure) |

### 4.3 容器启动 (containerlaunch)

| 类名 | 功能 |
|------|------|
| ContainerLaunchService | 容器启动服务 |
| CommandLineBuilder | 命令行构建 |
| ClasspathConstructor | 类路径构造 |

### 4.4 健康监控 (monitor)

| 类别 | 类名 | 功能 |
|------|------|------|
| 监控 | ServiceMonitor | 服务健康监控 |
| 探测器 | PortProbe | 端口探测 |
| 探测器 | HttpProbe | HTTP探测 |
| 探测器 | DefaultProbe | 默认探测 |

### 4.5 资源提供者 (provider)

| 类别 | 类名 | 功能 |
|------|------|------|
| 抽象 | AbstractProviderService | 抽象基类 |
| Docker | DockerProviderService | Docker容器支持 |
| Tarball | TarballProviderService | Tarball分发支持 |
| Default | DefaultProviderService | 默认实现 |

### 4.6 工具类 (utils)

| 类名 | 功能 |
|------|------|
| ServiceApiUtil | API工具 |
| SliderFileSystem | HDFS操作封装 |
| JsonSerDeser | JSON序列化 |
| ConfigHelper | 配置帮助 |

### 4.7 Timeline发布

| 类名 | 功能 |
|------|------|
| ServiceTimelinePublisher | Timeline数据发布 |
| ServiceTimelineEvent | 事件定义 |
| ServiceMetricsSink | 指标接收器 |

**核心功能：**
- 多组件服务部署
- Docker/Tarball/Archive支持
- 服务健康检查与自动恢复
- 滚动升级
- 依赖管理
- 亲和性/反亲和性放置

---

## 五、Application Catalog (应用目录)

| 类别 | 类名 | 功能 |
|------|------|------|
| 核心 | AppCatalog | 应用目录服务 |
| 客户端 | YarnServiceClient | YARN服务客户端 |
| 模型 | Application | 应用模型 |
| 模型 | AppDetails | 应用详情 |
| 模型 | AppEntry | 应用条目 |
| 控制器 | AppStoreController | 应用商店控制器 |
| 控制器 | AppDetailsController | 详情控制器 |
| 控制器 | AppListController | 列表控制器 |
| Solr | AppCatalogSolrClient | Solr搜索客户端 |

**核心功能：**
- 应用元数据管理
- 应用搜索发现
- REST API接口
- Solr搜索引擎集成

---

## 六、MAWO (未托管应用工作负载编排器)

包含核心模块用于编排未托管的YARN应用。

---

## 七、核心特点总结
1. **DistributedShell**: YARN入门示例，展示如何在多容器中执行命令
2. **UnmanagedAMLauncher**: 支持非YARN托管的AM，利于应用迁移
3. **YARN Services**: 企业级服务部署框架，支持Docker/滚动升级/健康检查
4. **Application Catalog**: 应用发现与目录管理
5. **多运行时支持**: Docker/Tarball/Java进程多种容器类型
6. **服务治理**: 依赖管理、亲和性、故障恢复

---

# hadoop-yarn-client 模块功能点

## 一、核心Client API

### 1. YarnClient (应用客户端API)

#### 应用管理
| 方法 | 功能 |
|------|------|
| `createApplication()` | 创建新的应用程序，获取ApplicationSubmissionContext |
| `submitApplication()` | 提交应用程序到YARN（阻塞调用） |
| `killApplication()` | 终止运行中的应用 |
| `failApplicationAttempt()` | 使应用尝试失败 |

#### 应用查询
| 方法 | 功能 |
|------|------|
| `getApplicationReport()` | 获取应用报告 |
| `getApplications()` | 获取所有应用（支持按类型/状态/标签/用户/队列过滤） |
| `getApplicationAttempts()` | 获取应用的所有尝试 |
| `getApplicationAttemptReport()` | 获取应用尝试报告 |
| `getContainerReport()` | 获取容器报告 |
| `getContainers()` | 获取应用尝试的所有容器 |

#### 集群信息
| 方法 | 功能 |
|------|------|
| `getYarnClusterMetrics()` | 获取集群指标（节点数、应用数等） |
| `getNodeReports()` | 获取节点报告（可按状态过滤） |

#### 队列管理
| 方法 | 功能 |
|------|------|
| `getQueueInfo()` | 获取队列信息 |
| `getAllQueues()` | 获取所有队列 |
| `getRootQueueInfos()` | 获取根队列信息 |
| `getChildQueueInfos()` | 获取子队列信息 |
| `getQueueAclsInfo()` | 获取用户队列ACL信息 |
| `moveApplicationAcrossQueues()` | 移动应用到其他队列 |

#### 节点标签
| 方法 | 功能 |
|------|------|
| `getNodeToLabels()` | 获取节点到标签的映射 |
| `getLabelsToNodes()` | 获取标签到节点的映射 |
| `getClusterNodeLabels()` | 获取集群节点标签集合 |

#### 节点属性
| 方法 | 功能 |
|------|------|
| `getClusterAttributes()` | 获取集群节点属性 |
| `getAttributesToNodes()` | 获取属性到节点的映射 |
| `getNodeToAttributes()` | 获取节点到属性的映射 |

#### 资源管理
| 方法 | 功能 |
|------|------|
| `getResourceProfiles()` | 获取可用的资源配置文件 |
| `getResourceProfile()` | 获取特定资源配置文件详情 |
| `getResourceTypeInfo()` | 获取支持的资源类型信息 |

#### 预约管理
| 方法 | 功能 |
|------|------|
| `createReservation()` | 创建新的预约 |
| `submitReservation()` | 提交预约请求 |
| `updateReservation()` | 更新现有预约 |
| `deleteReservation()` | 删除预约 |
| `listReservations()` | 列出预约 |

#### 应用优先级与超时
| 方法 | 功能 |
|------|------|
| `updateApplicationPriority()` | 更新应用优先级 |
| `updateApplicationTimeouts()` | 更新应用超时时间 |
| `getAMRMToken()` | 获取AMRM令牌（用于非托管AM） |

#### 容器操作
| 方法 | 功能 |
|------|------|
| `signalToContainer()` | 向容器发送信号 |
| `shellToContainer()` | 获取容器的shell访问 |
| `getRMDelegationToken()` | 获取RM委托令牌 |

---

### 2. AMRMClient (Application Master资源请求客户端)

#### AM注册
| 方法 | 功能 |
|------|------|
| `registerApplicationMaster()` | 向RM注册ApplicationMaster |
| `unregisterApplicationMaster()` | 注销AM |

#### 容器请求
| 方法 | 功能 |
|------|------|
| `addContainerRequest()` | 添加容器请求 |
| `removeContainerRequest()` | 移除容器请求 |
| `getMatchingRequests()` | 获取匹配的容器请求 |
| `requestContainerUpdate()` | 请求更新容器资源 |
| `requestContainerResourceChange()` | (已废弃)请求容器资源变更 |

#### 容器释放与分配
| 方法 | 功能 |
|------|------|
| `releaseAssignedContainer()` | 释放已分配的容器 |
| `allocate()` | 请求额外容器并接收分配（心跳） |

#### 集群信息
| 方法 | 功能 |
|------|------|
| `getAvailableResources()` | 获取集群可用资源 |
| `getClusterNodeCount()` | 获取集群节点数 |

#### 黑名单管理
| 方法 | 功能 |
|------|------|
| `updateBlacklist()` | 更新应用程序黑名单 |

#### 调度请求
| 方法 | 功能 |
|------|------|
| `addSchedulingRequests()` | 批量添加调度请求 |

**核心功能：**
- **放置约束**：支持Placement Constraints用于复杂的放置策略
- **执行类型**：支持Guaranteed和Opportunistic两种执行类型
- **资源配置文件**：支持通过资源配置文件请求容器
- **Timeline V2集成**：支持注册Timeline V2客户端

---

### 3. NMClient (Node Manager客户端)

#### 容器生命周期管理
| 方法 | 功能 |
|------|------|
| `startContainer()` | 启动分配的容器 |
| `stopContainer()` | 停止容器 |
| `getContainerStatus()` | 查询容器状态 |

#### 容器资源更新
| 方法 | 功能 |
|------|------|
| `updateContainerResource()` | 更新容器资源 |
| `increaseContainerResource()` | (已废弃)增加容器资源 |

#### 容器升级操作
| 方法 | 功能 |
|------|------|
| `reInitializeContainer()` | 重新初始化容器 |
| `restartContainer()` | 重启容器 |
| `rollbackLastReInitialization()` | 回滚重初始化 |
| `commitLastReInitialization()` | 提交重初始化 |

#### 本地化
| 方法 | 功能 |
|------|------|
| `localize()` | 本地化资源 |
| `getLocalizationStatuses()` | 获取本地化状态 |

#### 容器清理
| 方法 | 功能 |
|------|------|
| `cleanupRunningContainersOnStop()` | 停止时清理运行中的容器 |

---

### 4. AHSClient (Application History Server客户端)

| 方法 | 功能 |
|------|------|
| `getApplicationReport()` | 获取历史应用报告 |
| `getApplications()` | 获取所有历史应用 |
| `getApplicationAttemptReport()` | 获取历史应用尝试报告 |
| `getApplicationAttempts()` | 获取历史应用的所有尝试 |
| `getContainerReport()` | 获取历史容器报告 |
| `getContainers()` | 获取历史容器列表 |

---

### 5. SharedCacheClient (共享缓存客户端)

| 方法 | 功能 |
|------|------|
| `use()` | 声明使用共享缓存中的资源 |
| `release()` | 释放共享缓存资源 |
| `getFileChecksum()` | 计算文件校验和 |

---

## 二、异步客户端API

### 1. AMRMClientAsync
- 基于回调的异步AMRM客户端
- 支持事件处理器处理容器分配、节点健康更新等

### 2. NMClientAsync
- 基于回调的异步NM客户端
- 支持ContainerEventCallback处理容器事件

---

## 三、CLI命令行工具

### 1. ApplicationCLI (应用管理命令)
```bash
yarn app [OPTIONS] COMMAND
```
- `yarn app -list` - 列出应用
- `yarn app -status <app_id>` - 查看应用状态
- `yarn app -submit <app_definition>` - 提交应用
- `yarn app -kill <app_id>` - 终止应用
- `yarn app -move <app_id> -targetQueue <queue>` - 移动应用到队列
- `yarn app -updatePriority <app_id>` - 更新应用优先级
- `yarn app -updateLifetime <app_id>` - 更新应用生命周期
- `yarn app -changeQueue <app_id> -targetQueue <queue>` - 改变应用队列

### 2. ApplicationAttemptCLI (应用尝试命令)
```bash
yarn applicationattempt [OPTIONS] COMMAND
```
- `yarn applicationattempt -list <app_id>` - 列出应用尝试
- `yarn applicationattempt -status <attempt_id>` - 查看尝试状态

### 3. ContainerCLI (容器命令)
```bash
yarn container [OPTIONS] COMMAND
```
- `yarn container -list <attempt_id>` - 列出容器
- `yarn container -status <container_id>` - 查看容器状态
- `yarn container -signal <container_id> <command>` - 信号容器

### 4. NodeCLI (节点管理命令)
```bash
yarn node [OPTIONS] COMMAND
```
- `yarn node -list` - 列出节点
- `yarn node -status <node_id>` - 查看节点状态

### 5. QueueCLI (队列管理命令)
```bash
yarn queue [OPTIONS] COMMAND
```
- `yarn queue -status <queue_name>` - 查看队列状态

### 6. RMAdminCLI (RM管理命令)
```bash
yarn rmadmin [OPTIONS] COMMAND
```
- 刷新队列配置、刷新ACL、刷新节点、获取应用/队列报告

### 7. LogsCLI (日志命令)
```bash
yarn logs [OPTIONS]
```
- 获取应用日志

### 8. ClusterCLI (集群信息命令)
```bash
yarn cluster [OPTIONS] COMMAND
```
- `yarn cluster -list` - 列出集群信息

### 9. SchedConfCLI (调度器配置命令)
```bash
yarn schedconf [OPTIONS] COMMAND
```
- 查看和修改调度器配置

### 10. RouterCLI (路由器命令)
```bash
yarn router [OPTIONS] COMMAND
```
- 管理Federation Router

### 11. NodeAttributesCLI (节点属性命令)
```bash
yarn nodeattributes [OPTIONS] COMMAND
```
- 管理节点属性

### 12. TopCLI (Top命令)
```bash
yarn top [OPTIONS]
```
- 查看集群和队列资源使用情况

### 13. GpgCLI (GPG命令)
```bash
yarn gpg [OPTIONS]
```
- GPG加密相关操作

---

## 四、工具类

| 类名 | 功能 |
|------|------|
| `YarnClientUtils` | YARN客户端通用工具方法 |
| `MemoryPageUtils` | 内存页相关工具方法 |
| `FormattingCLIUtils` | CLI格式化工具 |
| `NMTokenCache` | NM令牌缓存管理 |
| `ContainerShellWebSocket` | 容器Shell WebSocket支持 |

---

## 五、代理与高可用

### ContainerManagementProtocolProxy
- 容器管理协议代理
- 支持连接池和超时控制

### HA支持
- Federation RMFailoverProxyProvider
- HedgingRequestRMFailoverProxyProvider
- 支持RM高可用和联邦

---

## 六、包结构

```
org.apache.hadoop.yarn.client
├── api/                    # 客户端API接口
│   ├── YarnClient.java     # 应用客户端API
│   ├── AMRMClient.java     # AM资源管理客户端
│   ├── NMClient.java       # 节点管理客户端
│   ├── AHSClient.java      # 应用历史客户端
│   ├── SharedCacheClient.java  # 共享缓存客户端
│   ├── async/              # 异步客户端
│   └── impl/               # API实现
├── cli/                    # CLI命令行工具
│   ├── ApplicationCLI.java # 应用命令
│   ├── NodeCLI.java        # 节点命令
│   ├── QueueCLI.java       # 队列命令
│   ├── RMAdminCLI.java     # RM管理命令
│   ├── LogsCLI.java        # 日志命令
│   └── ...                 # 其他CLI
└── util/                   # 工具类
```

---

## 七、核心特点总结

1. **完整的客户端API**：提供YarnClient、AMRMClient、NMClient、AHSClient等完整API
2. **丰富的CLI工具**：支持应用、节点、队列、容器等全方位管理命令
3. **异步支持**：提供AMRMClientAsync和NMClientAsync异步客户端
4. **高可用支持**：支持RM Federation和HA场景
5. **资源管理**：支持资源配置文件、节点标签、节点属性
6. **预约系统**：支持资源预约和调度
7. **容器操作**：支持容器启动、停止、更新、信号、shell访问
8. **Timeline集成**：支持Timeline V2指标发布

---

# hadoop-yarn-common 模块功能点

## 一、包结构概览

| 子包 | 功能描述 |
|------|----------|
| **api** | Protobuf转换器、放置约束转换 |
| **client** | 客户端代理、HA支持、Timeline客户端 |
| **event** | 事件分发框架（同步/异步） |
| **factories** | RPC工厂模式实现 |
| **factory** | RPC工厂提供者 |
| **ipc** | RPC通信框架 |
| **logaggregation** | 日志聚合工具 |
| **metrics** | 指标收集框架 |
| **nodelabels** | 节点标签/属性管理 |
| **security** | 安全认证与授权 |
| **state** | 状态机框架 |
| **util** | 通用工具类 |
| **webapp** | Web应用框架 |
| **sharedcache** | 共享缓存校验和 |

---

## 二、核心组件详解

### 1. 客户端代理 (client)

| 类名 | 功能 |
|------|------|
| `RMProxy` | ResourceManager代理基类，支持HA |
| `NMProxy` | NodeManager代理 |
| `ClientRMProxy` | 客户端到RM的RPC代理 |
| `AHSProxy` | ApplicationHistoryServer代理 |
| `RMFailoverProxyProvider` | RM故障转移代理提供器 |
| `DefaultNoHARMFailoverProxyProvider` | 非HA环境代理 |
| `ConfiguredRMFailoverProxyProvider` | 配置的HA代理 |
| `AutoRefreshRMFailoverProxyProvider` | 自动刷新HA代理 |
| `RequestHedgingRMFailoverProxyProvider` | 请求hedging HA代理 |
| `TimelineClient` | Timeline服务客户端 |
| `TimelineV2Client` | Timeline V2客户端 |
| `TimelineReaderClient` | Timeline读取客户端 |

### 2. 事件框架 (event)

| 类名 | 功能 |
|------|------|
| `Event` | 事件基类 |
| `EventHandler` | 事件处理器接口 |
| `Dispatcher` | 事件分发器接口 |
| `AsyncDispatcher` | 异步事件分发器 |
| `EventDispatcher` | 事件分发实现 |
| `AbstractEvent` | 抽象事件基类 |

### 3. 状态机框架 (state)

| 类名 | 功能 |
|------|------|
| `StateMachine` | 状态机接口 |
| `StateMachineFactory` | 状态机工厂 |
| `SingleArcTransition` | 单弧转换接口 |
| `MultipleArcTransition` | 多弧转换接口 |
| `StateTransitionListener` | 状态转换监听器 |
| `MultiStateTransitionListener` | 多状态转换监听器 |
| `InvalidStateTransitionException` | 无效状态转换异常 |
| `VisualizeStateMachine` | 状态机可视化 |

### 4. 安全框架 (security)

| 类别 | 类名 | 功能 |
|------|------|------|
| Token | `ContainerTokenIdentifier` | 容器Token标识符 |
| Token | `NMTokenIdentifier` | NM Token标识符 |
| Token | `AMRMTokenIdentifier` | AM-RM Token标识符 |
| Token | `DockerCredentialTokenIdentifier` | Docker凭证Token |
| Selector | `ContainerTokenSelector` | 容器Token选择器 |
| Selector | `NMTokenSelector` | NM Token选择器 |
| Selector | `AMRMTokenSelector` | AM-RM Token选择器 |
| ACL | `AdminACLsManager` | 管理ACL管理 |
| AuthZ | `YarnAuthorizationProvider` | YARN授权提供者 |
| AuthZ | `ConfiguredYarnAuthorizer` | 配置的授权器 |
| Other | `AccessType` | 访问类型枚举 |
| Other | `AccessRequest` | 访问请求 |
| Other | `Permission` | 权限 |
| Other | `PrivilegedEntity` | 特权实体 |
| Other | `SchedulerSecurityInfo` | 调度器安全信息 |
| Other | `ContainerManagerSecurityInfo` | 容器管理器安全信息 |

### 5. 节点标签管理 (nodelabels)

| 类名 | 功能 |
|------|------|
| `CommonNodeLabelsManager` | 通用节点标签管理器 |
| `NodeLabelUtil` | 节点标签工具 |
| `RMNodeLabel` | RM节点标签 |
| `RMNodeAttribute` | RM节点属性 |
| `NodeLabelStore` | 节点标签存储接口 |
| `FileSystemNodeLabelsStore` | 文件系统存储 |
| `NodeAttributesManager` | 节点属性管理器 |
| `NodeAttributeStore` | 节点属性存储 |
| `AbstractLabel` | 抽象标签 |
| `AttributeValue` | 属性值 |
| `StringAttributeValue` | 字符串属性值 |
| `AttributeExpressionOperation` | 属性表达式操作 |
| `NonAppendableFSNodeLabelStore` | 不可追加文件系统存储 |

### 6. 工具类 (util)

| 类别 | 类名 | 功能 |
|------|------|------|
| 应用 | `Apps` | 应用相关工具（解析ApplicationId等） |
| 时间 | `Times` | 时间格式化工具 |
| 时间 | `UTCClock` | UTC时钟 |
| 时间 | `SystemClock` | 系统时钟 |
| 时间 | `MonotonicClock` | 单调时钟 |
| 转换 | `ConverterUtils` | 类型转换工具 |
| 资源 | `Resources` | 资源操作工具 |
| 资源 | `ResourceCalculator` | 资源计算器接口 |
| 资源 | `DefaultResourceCalculator` | 默认计算器 |
| 资源 | `DominantResourceCalculator` | 主导资源计算器 |
| 进程树 | `ProcfsBasedProcessTree` | 基于procfs的进程树 |
| 进程树 | `WindowsBasedProcessTree` | Windows进程树 |
| 进程树 | `ResourceCalculatorProcessTree` | 资源计算进程树 |
| 进程树 | `ResourceCalculatorPlugin` | 资源计算插件 |
| 进程树 | `LinuxResourceCalculatorPlugin` | Linux资源计算插件 |
| 进程树 | `WindowsResourceCalculatorPlugin` | Windows资源计算插件 |
| 网络 | `RackResolver` | 机架解析器 |
| 配置 | `RMHAUtils` | RM HA工具 |
| 版本 | `YarnVersionInfo` | YARN版本信息 |
| 类加载 | `ApplicationClassLoader` | 应用类加载器 |
| 下载 | `FSDownload` | 文件系统下载 |
| Docker | `DockerClientConfigHandler` | Docker客户端配置处理 |
| 其他 | `StringHelper` | 字符串辅助工具 |
| 其他 | `LRUCache` | LRU缓存 |
| 其他 | `LRUCacheHashMap` | LRU缓存HashMap |
| 其他 | `BoundedAppender` | 有界追加器 |
| 其他 | `AbstractLivelinessMonitor` | 抽象存活监控器 |
| 其他 | `CacheNode` | 缓存节点 |
| 其他 | `AuxiliaryServiceHelper` | 辅助服务帮助类 |
| 其他 | `TrackingUriPlugin` | 追踪URI插件 |
| 其他 | `Log4jWarningErrorMetricsAppender` | Log4j指标追加器 |
| Timeline | `TimelineUtils` | Timeline工具 |
| Timeline | `TimelineEntityV2Converter` | V2实体转换器 |

### 7. Web应用框架 (webapp)

| 类别 | 类名 | 功能 |
|------|------|------|
| 核心 | `WebApp` | Web应用抽象基类 |
| 核心 | `WebApps` | Web应用构建器 |
| 核心 | `Controller` | 控制器基类 |
| 核心 | `Dispatcher` | 请求分发器 |
| 核心 | `Router` | 路由 |
| 视图 | `View` | 视图接口 |
| 视图 | `SubView` | 子视图 |
| 视图 | `HtmlPage` | HTML页面 |
| 视图 | `HtmlBlock` | HTML块 |
| 视图 | `Html` | HTML帮助类 |
| 视图 | `JQueryUI` | jQuery UI |
| 视图 | `TwoColumnLayout` | 两列布局 |
| 视图 | `TwoColumnCssLayout` | 两列CSS布局 |
| 视图 | `TextPage` | 文本页面 |
| 视图 | `TextView` | 文本视图 |
| 视图 | `ErrorPage` | 错误页面 |
| 视图 | `DefaultPage` | 默认页面 |
| 视图 | `NavBlock` | 导航块 |
| 视图 | `HeaderBlock` | 头部块 |
| 视图 | `FooterBlock` | 底部块 |
| 视图 | `InfoBlock` | 信息块 |
| 视图 | `LipsumBlock` | 占位块 |
| 异常 | `WebAppException` | Web应用异常 |
| 异常 | `BadRequestException` | 错误请求异常 |
| 异常 | `NotFoundException` | 未找到异常 |
| 异常 | `ForbiddenException` | 禁止异常 |
| 异常 | `ConflictException` | 冲突异常 |
| 异常 | `GenericExceptionHandler` | 通用异常处理 |
| 工具 | `WebAppUtils` | Web应用工具 |
| 工具 | `WebServiceClient` | Web服务客户端 |
| 工具 | `YarnWebServiceUtils` | YARN Web服务工具 |
| 参数 | `Params` | 参数 |
| 参数 | `YarnWebParams` | YARN Web参数 |
| 日志 | `AggregatedLogsPage` | 聚合日志页面 |
| 日志 | `AggregatedLogsBlock` | 聚合日志块 |
| 日志 | `AggregatedLogsNavBlock` | 聚合日志导航块 |
| DAO | `ConfInfo` | 配置信息 |
| DAO | `QueueConfigInfo` | 队列配置信息 |
| DAO | `SchedConfUpdateInfo` | 调度配置更新信息 |
| Hamlet | `Hamlet` | HTML生成库 |
| Hamlet | `HamletSpec` | HTML规范 |
| Hamlet | `HamletGen` | HTML生成器 |
| Hamlet | `HamletImpl` | HTML实现 |
| 其他 | `ResponseInfo` | 响应信息 |
| 其他 | `RemoteExceptionData` | 远程异常数据 |
| 其他 | `MimeType` | MIME类型 |
| 其他 | `ToJSON` | JSON转换 |
| 其他 | `YarnJacksonJaxbJsonProvider` | JSON提供者 |
| 其他 | `DefaultWrapperServlet` | 默认包装Servlet |

### 8. 日志聚合 (logaggregation)

| 类名 | 功能 |
|------|------|
| `AggregatedLogFormat` | 聚合日志格式 |
| `LogAggregationUtils` | 日志聚合工具 |
| `LogAggregationMetaCollector` | 日志元数据收集器 |
| `LogAggregationWebUtils` | Web日志工具 |
| `LogCLIHelpers` | CLI帮助类 |
| `LogToolUtils` | 日志工具 |
| `AggregatedLogDeletionService` | 聚合日志删除服务 |
| `ContainerLogMeta` | 容器日志元数据 |
| `ContainerLogFileInfo` | 容器日志文件信息 |
| `ContainerLogAggregationType` | 容器日志聚合类型 |
| `ContainerLogsRequest` | 容器日志请求 |
| `ExtendedLogMetaRequest` | 扩展日志元请求 |

### 9. 指标 (metrics)

| 类名 | 功能 |
|------|------|
| `EventTypeMetrics` | 事件类型指标 |
| `GenericEventTypeMetrics` | 通用事件类型指标 |
| `CustomResourceMetrics` | 自定义资源指标 |
| `CustomResourceMetricValue` | 自定义资源指标值 |

### 10. 共享缓存 (sharedcache)

| 类名 | 功能 |
|------|------|
| `SharedCacheChecksum` | 共享缓存校验和接口 |
| `SharedCacheChecksumFactory` | 校验和工厂 |
| `ChecksumSHA256Impl` | SHA256校验和实现 |

### 11. RPC通信 (ipc)

| 类名 | 功能 |
|------|------|
| `YarnRPC` | YARN RPC抽象 |
| `RPCUtil` | RPC工具 |
| `HadoopYarnProtoRPC` | Hadoop YARN Proto RPC |

### 12. 工厂模式 (factories/factory)

| 类名 | 功能 |
|------|------|
| `RpcClientFactory` | RPC客户端工厂 |
| `RpcServerFactory` | RPC服务器工厂 |
| `RpcFactoryProvider` | RPC工厂提供者 |

### 13. 配置与日志

| 类名 | 功能 |
|------|------|
| `YarnUncaughtExceptionHandler` | 未捕获异常处理器 |
| `FileSystemBasedConfigurationProvider` | 基于文件系统的配置提供者 |
| `ContainerLogAppender` | 容器日志追加器 |
| `ContainerRollingLogAppender` | 容器滚动日志追加器 |

---

## 三、核心功能总结

1. **事件驱动框架**: 提供同步/异步事件分发，支持状态机实现
2. **状态机框架**: 通用状态机工厂，支持单弧和多弧转换
3. **客户端代理**: 支持RM、NM、AHS的RPC代理，提供HA支持
4. **安全框架**: Token管理、ACL授权、认证机制
5. **节点标签管理**: 节点标签和属性的存储、查询、管理
6. **Web框架**: 完整的MVC Web框架，支持REST API
7. **工具库**: 时间、转换、资源、进程树等通用工具
8. **日志聚合**: 容器日志聚合、查看、删除工具
9. **指标系统**: 事件类型和自定义资源指标
10. **RPC通信**: 统一的RPC通信抽象
11. **共享缓存**: 校验和计算

---

**这是YARN的核心公共模块，为所有YARN组件提供基础框架和通用工具，是YARN基础设施的核心依赖。**