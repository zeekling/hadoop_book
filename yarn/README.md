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