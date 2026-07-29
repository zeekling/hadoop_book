# YARN 作业状态离线分析工具

## 概述

YARN 提供了一套完整的工具链用于离线分析作业状态（即集群不运行时或在应用结束后查询）。这些工具覆盖了**实时查询 → 历史回溯 → 日志拉取 → Trace 转换 → 回放模拟 → 资源预估**全链路。

| 工具 | 核心类 | 用途 |
|------|--------|------|
| `yarn application` | `ApplicationCLI` | 查询应用/尝试/容器状态 |
| `yarn logs` | `LogsCLI` | 拉取容器日志 |
| `AppHistoryServer` | `ApplicationHistoryServer` | ATS v1 历史数据服务 |
| `TimelineReader` | `TimelineReaderServer` | ATS v2 时间轴读取服务 |
| `Rumen` | `TraceBuilder` | 历史日志转 Trace 格式 |
| `SLS` | `SLSRunner` | 离线调度回放模拟 |
| `ResourceEstimator` | `ResourceEstimatorServer` | 资源消耗预估 |

---

## 一、`yarn application` — 应用状态 CLI

**类**：`ApplicationCLI.java`
**路径**：`hadoop-yarn-project/hadoop-yarn/hadoop-yarn-client/src/main/java/org/apache/hadoop/yarn/client/cli/ApplicationCLI.java`
**注解**：`@Private @Unstable`
**基类**：`YarnCLI`

无需集群写入权限，通过 **ResourceManager RPC** 或 **HistoryServer RPC** 查询已完成/运行中的应用状态。

### 1.1 命令体系

三个子命令组：

```
yarn application [app]
yarn applicationattempt
yarn container
```

### 1.2 `yarn application` 子命令

| 子命令 | 方法 | 说明 |
|--------|------|------|
| `-status <appId>` | `executeStatusCommand()` | 获取单个应用的详细状态报告 |
| `-list [-appStates]` | `executeListCommand()` | 列出应用，可按状态过滤 |
| `-kill <appId>` | `executeKillCommand()` | 终止应用 |
| `-movetoqueue <appId> -queue <q>` | — | 迁移应用到其他队列 |
| `-changeQueue <appId> -queue <q>` | — | 更改队列 |
| `-updatePriority <appId> -priority <p>` | — | 更新优先级 |
| `-updateLifetime <appId> -lifetime <l>` | — | 更新超时时间 |

**状态过滤**（`-list -appStates`）：
```
ACCEPTED, RUNNING, FINISHED, FAILED, KILLED,
NEW, NEW_SAVING, SUBMITTED
```

### 1.3 `yarn applicationattempt` 子命令

| 子命令 | 说明 |
|--------|------|
| `-list <appId>` | 列出应用的所有尝试（attempt） |
| `-status <attemptId>` | 获取单个尝试的详情 |

### 1.4 `yarn container` 子命令

| 子命令 | 说明 |
|--------|------|
| `-list <appAttemptId>` | 列出尝试的所有容器 |
| `-status <containerId>` | 获取单个容器的详情 |

### 1.5 查询来源回退机制

```
ApplicationCLI 查询
  ├── 运行中应用 → ResourceManager (RPC)
  └── 已完成应用 → ApplicationHistoryServer (RPC, 自动回退)
```

---

## 二、`yarn logs` — 容器日志获取

**类**：`LogsCLI.java`
**路径**：`hadoop-yarn-project/hadoop-yarn/hadoop-yarn-client/src/main/java/org/apache/hadoop/yarn/client/cli/LogsCLI.java`
**注解**：`@Public @Evolving`
**基类**：`Configured implements Tool`

从 **NodeManager 本地日志** 或 **HistoryServer 聚合日志** 中离线拉取容器日志。

### 2.1 命令格式

```
yarn logs -applicationId <appId> [OPTIONS]
```

### 2.2 全部选项

| 选项 | 说明 |
|------|------|
| `-applicationId <id>` | 应用 ID（必选） |
| `-applicationAttemptId <id>` | 限定特定尝试 |
| `-containerId <id>` | 限定特定容器 |
| `-nodeAddress <addr>` | 限定特定节点 |
| `-appOwner <owner>` | 应用所有者 |
| `-am` | 只获取 AM 容器日志 |
| `-log_files <files>` | 指定日志文件名 |
| `-log_files_pattern <regex>` | 日志文件正则匹配 |
| `-list_nodes` | 列出运行过该应用的节点 |
| `-show_application_log_info` | 显示应用日志元信息 |
| `-show_container_log_info` | 显示容器日志元信息 |
| `-out <dir>` | 下载到本地目录 |
| `-size <bytes>` | 日志大小限制（bytes） |
| `-size_limit_mb <mb>` | 日志大小限制（MB） |
| `-client_max_retries <n>` | 最大重试次数 |
| `-client_retry_interval_ms <ms>` | 重试间隔 |

### 2.3 数据流

```
NodeManager 本地日志
    ↓ 日志聚合 (yarn.log-aggregation-enable=true)
  HDFS 聚合日志目录
    ↓ LogsCLI 读取
  本地文件输出
```

---

## 三、ApplicationHistoryServer — 历史数据服务（ATS v1）

**类**：`ApplicationHistoryServer.java`
**路径**：`hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-applicationhistoryservice/src/main/java/org/apache/hadoop/yarn/server/applicationhistoryservice/ApplicationHistoryServer.java`
**注解注释**：*"History server that keeps track of all types of history in the cluster. Application specific history to start with."*

独立运行的服务进程，持久化**所有已完成的 YARN 应用**的元数据。

### 3.1 启动方式

```bash
yarn historyserver
```

### 3.2 服务架构

```
ApplicationHistoryServer (CompositeService)
  ├── ApplicationHistoryClientService    ← RPC 端点
  │     (实现 ApplicationHistoryProtocol)
  │     ├── getApplicationReport()         ← 应用报告
  │     ├── getApplications()              ← 应用列表
  │     ├── getApplicationAttemptReport()  ← 尝试报告
  │     ├── getApplicationAttempts()       ← 尝试列表
  │     ├── getContainerReport()           ← 容器报告
  │     └── getContainers()                ← 容器列表
  ├── ApplicationHistoryManager            ← 管理后端存储
  ├── TimelineStore                        ← 时间轴数据存储
  │     ├── LeveldbTimelineStore           ← LevelDB 实现
  │     └── MemoryTimelineStateStore       ← 内存实现（测试用）
  ├── TimelineDataManager                  ← 时间轴数据查询
  └── TimelineV1DelegationTokenSecretManagerService
```

### 3.3 用途

- RM 重启后恢复作业元数据
- 离线查询已完成的作业状态
- 配合 `ApplicationCLI` 自动回退查询

### 3.4 存储接口继承体系

```
TimelineStateStore (抽象基类)
  ├── MemoryTimelineStateStore        (内存, 测试用)
  └── LeveldbTimelineStateStore       (LevelDB, 生产用)
```

---

## 四、TimelineReaderServer — 时间轴读取服务（ATS v2）

**类**：`TimelineReaderServer.java`
**路径**：`hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-timelineservice/src/main/java/org/apache/hadoop/yarn/server/timelineservice/reader/TimelineReaderServer.java`
**注释**：*"Main class for Timeline Reader."*

ATS v2 的读取服务端，提供 **RESTful API** 查询应用实体。

### 4.1 REST 端点

**类**：`TimelineReaderWebServices.java`（3699 行）

| 端点 | 说明 |
|------|------|
| `GET /ws/v2/timeline/apps/<appId>` | 获取应用实体 |
| `GET /ws/v2/timeline/apps/<appId>/entities/<entityType>` | 获取指定类型的实体列表 |
| `GET /ws/v2/timeline/apps/<appId>/entities/<entityType>/<entityId>` | 获取单个实体 |
| `GET /ws/v2/timeline/apps/<appId>/appattempts` | 获取应用尝试列表 |
| `GET /ws/v2/timeline/apps/<appId>/appattempts/<attemptId>/containers` | 获取容器列表 |

### 4.2 客户端 API

**类**：`TimelineReaderClient.java`
**路径**：`hadoop-yarn-project/hadoop-yarn/hadoop-yarn-common/src/main/java/org/apache/hadoop/yarn/client/api/TimelineReaderClient.java`

```java
public abstract class TimelineReaderClient extends CompositeService {
    // 获取应用实体
    getApplicationEntity(ApplicationId, String fields, Map filters);

    // 获取应用尝试实体
    getApplicationAttemptEntity(ApplicationAttemptId, String fields, Map filters);
    getApplicationAttemptEntities(ApplicationId, String fields, Map filters, long limit, String fromId);

    // 获取容器实体
    getContainerEntity(ContainerId, String fields, Map filters);
    getContainerEntities(ApplicationId, String fields, Map filters, long limit, String fromId);
}
```

工厂方法：
```java
TimelineReaderClient client = TimelineReaderClient.createTimelineReaderClient();
// 返回 TimelineReaderClientImpl 实例
```

### 4.3 ATS v1 vs v2 对比

| 对比项 | ATS v1 (AppHistoryServer) | ATS v2 (TimelineReaderServer) |
|--------|---------------------------|-------------------------------|
| 存储后端 | LevelDB | Phoenix / HBase |
| 查询接口 | RPC + REST | REST only |
| 数据模型 | 单实体类型 | 实体 + 子类型 + 指标 |
| 集群 ID | 不需要 | 需要 cluster ID |
| 流式写入 | 支持 | 支持 |
| 部署方式 | 独立服务 | 独立服务 |
| 推荐状态 | **旧版** | **当前推荐** |

---

## 五、Rumen — 历史日志转 Trace 工具

**路径**：`hadoop-tools/hadoop-rumen/`

将 JobHistory 日志转换为标准化的 **Trace JSON 格式**，供回放和模拟使用。Rumen 是 SLS 和 ResourceEstimator 的数据上游。

### 5.1 主要组件

| 类 | 路径 | 功能 |
|------|------|------|
| `TraceBuilder` | `.../rumen/TraceBuilder.java` | 主入口：JobHistory → Trace JSON |
| `HadoopLogsAnalyzer` | `.../rumen/HadoopLogsAnalyzer.java` | 解析原始 Hadoop 日志，提取作业性能数据 |
| `Folder` | `.../rumen/Folder.java` | 将多个 Trace 文件合并/分区 |
| `Anonymizer` | `.../rumen/Anonymizer.java` | 对 Trace 数据做匿名化脱敏 |

### 5.2 命令格式

```bash
bin/hadoop rumen tracebuilder \
    -config yarn-site.xml \
    -jobs /tmp/history/done/*.jhist \
    -traces /tmp/trace.json
```

### 5.3 Trace 格式

JSON 格式的作业生命周期事件，包含：
- 作业提交/启动/完成时间
- 每个 Map/Reduce Task 的时序
- 每个 Attempt 的资源使用量
- 失败/重试记录

### 5.4 典型流程

```
JobHistory 日志 (.jhist)
    ↓ TraceBuilder
  Trace JSON
    ├──→ SLS Runner (调度器模拟)
    └──→ ResourceEstimator (资源建模)
```

---

## 六、SLS — Scheduler Load Simulator

**类**：`SLSRunner.java`
**路径**：`hadoop-tools/hadoop-sls/src/main/java/org/apache/hadoop/yarn/sls/SLSRunner.java`
**注解**：`@Private @Unstable`

使用真实的历史 Trace 回放作业提交，**离线测试 YARN Scheduler 性能**，无需真实集群。

### 6.1 命令格式

```bash
bin/hadoop jar sls.jar SLSRunner \
    -tracetype RUMEN \
    -tracelocation trace.json \
    -output /tmp/sls_result
```

### 6.2 三种 Trace 来源

| TraceType | 枚举值 | 来源说明 |
|-----------|--------|----------|
| `RUMEN` | `TraceType.RUMEN` | Rumen 工具从 JobHistory 生成的 Trace |
| `SLS` | `TraceType.SLS` | 原生 SLS JSON 格式（人工构造） |
| `SYNTH` | `TraceType.SYNTH` | 按参数随机合成的负载 |

### 6.3 组件架构

```
SLSRunner (主控)
  ├── RMRunner           ← 模拟 ResourceManager
  ├── NMRunner           ← 模拟 NodeManager
  ├── AMRunner           ← 模拟 ApplicationMaster
  ├── AMSimulator        ← 模拟 AM 行为（心跳/申请/释放）
  ├── RumenToSLSConverter ← Rumen Trace → SLS 格式转换
  ├── SLSUtils            ← 工具方法
  └── SLSWebApp           ← Web UI 查看结果
```

### 6.4 分析输出

```
SLSWebApp 提供：
  - 集群利用率时间线
  - 作业延迟分布图
  - 队列公平性分析
  - 调度器吞吐量统计
  - 各作业的详细时序
```

---

## 七、ResourceEstimator — 资源预估服务

**类**：`ResourceEstimatorServer.java`
**路径**：`hadoop-tools/hadoop-resourceestimator/src/main/java/org/apache/hadoop/resourceestimator/service/ResourceEstimatorServer.java`

分析历史作业执行数据，构建资源消耗模型，为新作业提供资源量建议。

### 7.1 命令格式

```bash
bin/hadoop estimator \
    -p pipeline \
    -d input_dir \
    -o output_dir
```

### 7.2 架构

```
ResourceEstimatorServer (CompositeService)
  ├── ResourceEstimatorService   ← REST 服务端点
  ├── BaseSolver / LpSolver       ← 线性规划求解器
  ├── BaseLogParser               ← 日志解析器
  │     ├── RmSingleLineParser    ← RM 日志单行解析
  │     └── NativeSingleLineParser ← 原生格式解析
  └── InMemoryStore / ...         ← SkylineStore 存储

REST API:
  POST /resourceestimator/estimator/estimate  ← 提交预估请求
  GET  /resourceestimator/estimator/estimate  ← 查询预估结果
```

### 7.3 处理流程

```
JobHistory 日志 (经过 Rumen 处理)
    ↓ BaseLogParser
  作业执行事件
    ↓ LpSolver (线性规划)
  资源消耗模型
    ↓ SkylineStore
  REST API 查询
```

### 7.4 使用限制

- 需要先通过 Rumen 处理历史日志
- 仅支持特定类型的作业
- 准确度依赖历史数据量

---

## 八、StateStore — 状态存储体系

YARN 的作业状态在运行时持久化到多种 StateStore，此供恢复和查询使用。

### 8.1 RMStateStore 继承体系

**路径**：`hadoop-yarn-server-resourcemanager/src/main/java/org/apache/hadoop/yarn/server/resourcemanager/recovery/`

```
RMStateStore (抽象基类, 97行)
  ├── MemoryRMStateStore         (内存, 测试用)
  ├── LeveldbRMStateStore        (LevelDB, 生产)
  ├── FileSystemRMStateStore     (HDFS, 生产)
  └── ZKRMStateStore             (ZooKeeper, HA 生产)
```

### 8.2 NMStateStore 继承体系

**路径**：`.../recovery/NMStateStoreService.java`

```
NMStateStoreService (抽象基类, 53行)
  ├── NMNullStateStoreService    (空实现)
  ├── NMMemoryStateStoreService  (内存, 测试)
  └── NMLeveldbStateStoreService (LevelDB, 生产)
```

### 8.3 FederationStateStore

```
FederationStateStore (接口)
  ├── MemoryFederationStateStore
  ├── SQLFederationStateStore
  └── ZookeeperFederationStateStore
```

---

## 九、工具关系全景图

```
                    ┌──────────────────────────────────┐
                    │       YARN 集群运行时               │
                    │  RM / NM / AppMaster              │
                    └──────┬───────────────────────────┘
                           │ 写
                           ▼
                    ┌──────────────────────────────────┐
                    │          状态持久化层              │
                    ├──────────────────────────────────┤
                    │ RMStateStore  (ZK/LevelDB/FS)     │  ← RM 故障恢复
                    │ NMStateStore (LevelDB)            │  ← NM 故障恢复
                    │ Aggregated Logs (HDFS)            │  ← 容器日志聚合
                    │ TimelineStore (LevelDB/HBase)     │  ← ATS 历史数据
                    └──────────────────────────────────┘
                           │ 读
        ┌──────────────────┼──────────────────────┐
        │                  │                      │
        ▼                  ▼                      ▼
 ┌──────────────┐  ┌────────────────┐  ┌──────────────────┐
 │ yarn app     │  │ AppHistory     │  │ yarn logs        │
 │ status/list  │  │ Server         │  │                  │
 ├──────────────┤  ├────────────────┤  ├──────────────────┤
 │ 实时查询 RM  │  │ ATS v1 历史     │  │ 拉取容器日志      │
 │ 完成→HS 回退 │  │ RPC + REST     │  │ HDFS 聚合目录    │
 └──────────────┘  └────────────────┘  └──────────────────┘
                           │
                           ▼
                    ┌──────────────────┐
                    │ TimelineReader   │── REST JSON
                    │ Server (ATS v2)  │
                    └──────────────────┘
                           │
                           ▼
                    ┌──────────────────┐
                    │  Rumen           │──→ Trace JSON
                    │  TraceBuilder    │
                    └───────┬──────────┘
                            │
               ┌────────────┼────────────┐
               │            │            │
               ▼            ▼            ▼
        ┌──────────┐ ┌────────────┐ ┌────────────────┐
        │ SLS      │ │ Folder/    │ │ Resource       │
        │ Runner   │ │ Anonymizer │ │ Estimator      │
        ├──────────┤ ├────────────┤ ├────────────────┤
        │ 调度器    │ │ 合并/脱敏   │ │ 资源消耗建模    │
        │ 离线回放  │ │ Trace 处理  │ │ 容量预估       │
        └──────────┘ └────────────┘ └────────────────┘
```

---

## 十、常用命令速查

```bash
# ========== 应用状态查询 ==========

# 查看所有运行中的应用
yarn application -list -appStates RUNNING

# 查看已完成的应用（带所有者筛选）
yarn application -list -appStates FINISHED,FAILED,KILLED -appOwner hadoop

# 查看应用详细状态
yarn application -status application_123_456789

# 查看应用的所有尝试
yarn applicationattempt -list application_123_456789

# 查看尝试的所有容器
yarn container -list appattempt_123_456789_0001


# ========== 日志拉取 ==========

# 拉取应用所有容器日志
yarn logs -applicationId application_123_456789 -out ./logs

# 只拉取 AM 日志
yarn logs -applicationId application_123_456789 -am -out ./logs

# 拉取特定容器的日志
yarn logs -applicationId application_123_456789 \
    -containerId container_123_456789_01_000001


# ========== Trace 转换（Rumen） ==========

# 从 JobHistory 生成 Trace JSON
hadoop rumen tracebuilder \
    -config $HADOOP_CONF_DIR/yarn-site.xml \
    -jobs /tmp/history/done/中间结果/*.jhist \
    -traces /tmp/trace_output.json


# ========== 调度器模拟（SLS） ==========

# 使用 Rumen Trace 运行模拟
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-sls-*.jar \
    SLSRunner \
    -tracetype RUMEN \
    -tracelocation /tmp/trace_output.json \
    -output /tmp/sls_result \
    -printsimulation

# 使用 SLS 原生格式
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-sls-*.jar \
    SLSRunner \
    -tracetype SLS \
    -tracelocation /tmp/sls_trace.json


# ========== 资源预估 ==========

# 运行资源预估流水线
hadoop estimator \
    -p pipeline \
    -d /tmp/history_input \
    -o /tmp/estimator_output
```

---

## 十一、完整文件清单

| 文件路径 | 类名 | 职责 |
|----------|------|------|
| `hadoop-yarn-client/.../cli/ApplicationCLI.java` | `ApplicationCLI` | `yarn application` 命令实现 |
| `hadoop-yarn-client/.../cli/LogsCLI.java` | `LogsCLI` | `yarn logs` 命令实现 |
| `.../applicationhistoryservice/ApplicationHistoryServer.java` | `ApplicationHistoryServer` | ATS v1 服务端 |
| `.../applicationhistoryservice/ApplicationHistoryClientService.java` | `ApplicationHistoryClientService` | ATS v1 RPC 服务 |
| `.../timelineservice/reader/TimelineReaderServer.java` | `TimelineReaderServer` | ATS v2 服务端 |
| `.../timelineservice/reader/TimelineReaderWebServices.java` | `TimelineReaderWebServices` | ATS v2 REST 端点 |
| `hadoop-yarn-common/.../api/TimelineReaderClient.java` | `TimelineReaderClient` | ATS v2 读取客户端 |
| `hadoop-yarn-common/.../api/TimelineClient.java` | `TimelineClient` | ATS v1 写入客户端 |
| `.../timeline/recovery/LeveldbTimelineStateStore.java` | `LeveldbTimelineStateStore` | LevelDB 存储实现 |
| `.../resourcemanager/recovery/RMStateStore.java` | `RMStateStore` | RM 状态存储抽象 |
| `.../resourcemanager/recovery/LeveldbRMStateStore.java` | `LeveldbRMStateStore` | RM LevelDB 存储 |
| `.../resourcemanager/recovery/ZKRMStateStore.java` | `ZKRMStateStore` | RM ZooKeeper 存储 |
| `.../resourcemanager/recovery/FileSystemRMStateStore.java` | `FileSystemRMStateStore` | RM HDFS 存储 |
| `.../nodemanager/recovery/NMStateStoreService.java` | `NMStateStoreService` | NM 状态存储抽象 |
| `.../nodemanager/recovery/NMLeveldbStateStoreService.java` | `NMLeveldbStateStoreService` | NM LevelDB 存储 |
| `hadoop-tools/hadoop-rumen/.../TraceBuilder.java` | `TraceBuilder` | JobHistory → Trace |
| `hadoop-tools/hadoop-rumen/.../HadoopLogsAnalyzer.java` | `HadoopLogsAnalyzer` | 日志分析 |
| `hadoop-tools/hadoop-rumen/.../Folder.java` | `Folder` | Trace 合并/分区 |
| `hadoop-tools/hadoop-rumen/.../Anonymizer.java` | `Anonymizer` | Trace 脱敏 |
| `hadoop-tools/hadoop-sls/.../SLSRunner.java` | `SLSRunner` | 调度器模拟主控 |
| `hadoop-tools/hadoop-sls/.../RMRunner.java` | `RMRunner` | 模拟 RM |
| `hadoop-tools/hadoop-sls/.../NMRunner.java` | `NMRunner` | 模拟 NM |
| `hadoop-tools/hadoop-sls/.../AMRunner.java` | `AMRunner` | 模拟 AM |
| `hadoop-tools/hadoop-sls/.../AMSimulator.java` | `AMSimulator` | AM 行为模拟 |
| `hadoop-tools/hadoop-sls/.../web/SLSWebApp.java` | `SLSWebApp` | SLS Web UI |
| `hadoop-tools/hadoop-resourceestimator/.../ResourceEstimatorServer.java` | `ResourceEstimatorServer` | 资源预估服务 |
| `hadoop-tools/hadoop-resourceestimator/.../BaseLogParser.java` | `BaseLogParser` | 日志解析抽象 |
| `hadoop-tools/hadoop-resourceestimator/.../BaseSolver.java` | `BaseSolver` | 求解器抽象 |
| `hadoop-tools/hadoop-resourceestimator/.../LpSolver.java` | `LpSolver` | 线性规划求解 |
