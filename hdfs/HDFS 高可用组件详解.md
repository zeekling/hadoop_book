# HDFS 高可用组件详解

## 一、概述

HDFS 高可用（High Availability, HA）是 HDFS 的关键特性，通过 ZKFailoverController（ZKFC）、JournalNode 和 Router 等组件实现 NameNode 的主备切换和故障自动转移，确保 HDFS 集群的高可用性。

**核心目标**：
- 实现 NameNode 的主备切换
- 支持故障自动转移
- 提供共享编辑日志
- 实现联邦架构的统一访问

**位置**: 
- ZKFC: `hadoop-common-project/hadoop-common/src/main/java/org/apache/hadoop/ha/ZKFailoverController.java`
- JournalNode: `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/qjournal/server/JournalNode.java`
- Router: `hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-router/src/main/java/org/apache/hadoop/yarn/server/router/Router.java`

---

## 二、核心架构

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                        客户端                                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Router (路由器)                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  RouterClientRMService (客户端 RPC 服务)              │   │
│  │  - ApplicationClientProtocol 实现                     │   │
│  │  - 请求拦截器链 (RequestInterceptorChain)            │   │
│  │  - 用户管道缓存 (LRUCacheHashMap)                     │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  RouterRMAdminService (管理 RPC 服务)                 │   │
│  │  - ResourceManagerAdminProtocol 实现                 │   │
│  │  - 集群管理操作                                       │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  RouterWebApp (Web 服务)                             │   │
│  │  - REST API                                          │   │
│  │  - Web UI                                            │   │
│  │  - WebAppProxy (应用代理)                            │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  SubClusterCleaner (子集群清理器)                     │   │
│  │  - 定期清理失效的子集群                               │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              ZKFailoverController (ZK 故障转移控制器)         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  HealthMonitor (健康监控)                            │   │
│  │  - 监控 NameNode 健康状态                             │   │
│  │  - 检测 NameNode 故障                                 │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  ActiveStandbyElector (主备选举器)                  │   │
│  │  - ZooKeeper 选举                                    │   │
│  │  - 主备切换                                          │   │
│  │  - 故障转移                                          │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  ZKFCRpcServer (RPC 服务)                            │   │
│  │  - 处理 ZKFC RPC 请求                                │   │
│  │  - 提供管理接口                                      │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              JournalNode (日志节点)                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  JournalNodeRpcServer (RPC 服务)                     │   │
│  │  - 处理 JournalNode RPC 请求                         │   │
│  │  - 提供编辑日志服务                                  │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  JournalNodeHttpServer (HTTP 服务)                    │   │
│  │  - 提供 HTTP 接口                                    │   │
│  │  - Web UI                                            │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Journal (编辑日志)                                   │   │
│  │  - 存储编辑日志                                      │   │
│  │  - 提供日志服务                                      │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  JournalNodeSyncer (日志同步器)                       │   │
│  │  - 同步编辑日志                                      │   │
│  │  - 确保数据一致性                                    │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              NameNode (主节点/备节点)                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  ActiveState (活跃状态)                              │   │
│  │  - 处理客户端请求                                    │   │
│  │  - 写入编辑日志                                      │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  StandbyState (待机状态)                             │   │
│  │  - 读取编辑日志                                      │   │
│  │  - 同步编辑日志                                      │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  EditLogTailer (编辑日志跟踪器)                       │   │
│  │  - 跟踪编辑日志                                      │   │   │
│  │  - 同步编辑日志                                      │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 主要组件

**ZKFailoverController**:
- `ZKFailoverController` - ZKFC 主类
- `HealthMonitor` - 健康监控
- `ActiveStandbyElector` - 主备选举器
- `ZKFCRpcServer` - RPC 服务

**JournalNode**:
- `JournalNode` - JournalNode 主类
- `JournalNodeRpcServer` - RPC 服务
- `JournalNodeHttpServer` - HTTP 服务
- `Journal` - 编辑日志
- `JournalNodeSyncer` - 日志同步器

**Router**:
- `Router` - Router 主类
- `RouterClientRMService` - 客户端 RPC 服务
- `RouterRMAdminService` - 管理 RPC 服务
- `RouterWebApp` - Web 服务
- `SubClusterCleaner` - 子集群清理器

---

## 三、ZKFailoverController 详解

### 3.1 ZKFailoverController - ZKFC 主类

**位置**: `ZKFailoverController.java:65`

**作用**: ZKFC 的主类，负责协调 NameNode 的主备切换和故障转移

**核心字段**:
```java
// 配置
protected Configuration conf;                    // 配置
private String zkQuorum;                           // ZooKeeper 集群地址
protected final HAServiceTarget localTarget;    // 本地服务目标

// 组件
private HealthMonitor healthMonitor;              // 健康监控
private ActiveStandbyElector elector;              // 主备选举器
protected ZKFCRpcServer rpcServer;               // RPC 服务

// 状态
private State lastHealthState;                    // 最后健康状态
private volatile HAServiceState serviceState;    // 服务状态
private String fatalError;                         // 致命错误

// 延迟加入选举
private long delayJoiningUntilNanotime;           // 延迟加入选举时间
```

**核心方法**:

#### 1. run - 运行 ZKFC
```java
public int run(final String[] args) throws Exception {
    // 1. 检查自动故障转移是否启用
    if (!localTarget.isAutoFailoverEnabled()) {
        LOG.error("Automatic failover is not enabled for " + localTarget + ".");
        return ERR_CODE_AUTO_FAILOVER_NOT_ENABLED;
    }
    
    // 2. 登录为 FC 用户
    loginAsFCUser();
    
    // 3. 运行 ZKFC
    return SecurityUtil.doAsLoginUserOrFatal(new PrivilegedAction<Integer>() {
        @Override
        public Integer run() {
            try {
                return doRun(args);
            } catch (Exception t) {
                throw new RuntimeException(t);
            } finally {
                if (elector != null) {
                    elector.terminateConnection();
                }
            }
        }
    });
}
```

#### 2. doRun - 执行 ZKFC
```java
private int doRun(final String[] args) throws Exception {
    // 1. 处理命令行参数
    if (args.length > 0) {
        if ("-formatZK".equals(args[0])) {
            return formatZK(args);
        }
    }
    
    // 2. 初始化 ZooKeeper 连接
    initZK();
    
    // 3. 启动健康监控
    startHealthMonitor();
    
    // 4. 启动主备选举器
    startElector();
    
    // 5. 启动 RPC 服务
    startRpcServer();
    
    // 6. 进入主循环
    while (true) {
        Thread.sleep(1000);
    }
}
```

#### 3. formatZK - 格式化 ZooKeeper
```java
private int formatZK(final String[] args) throws Exception {
    // 1. 检查参数
    boolean force = false;
    boolean nonInteractive = false;
    
    for (int i = 1; i < args.length; i++) {
        if ("-force".equals(args[i])) {
            force = true;
        } else if ("-nonInteractive".equals(args[i])) {
            nonInteractive = true;
        }
    }
    
    // 2. 格式化 ZooKeeper
    return formatZK(force, nonInteractive);
}
```

---

### 3.2 HealthMonitor - 健康监控

**位置**: `HealthMonitor.java`

**作用**: 监控 NameNode 的健康状态，检测故障

**核心字段**:
```java
// 监控配置
private final long checkIntervalMs;                  // 检查间隔
private final int sleepAfterDisconnectMs;             // 断开后睡眠时间

// 状态
private volatile State lastState;                      // 最后状态
private volatile boolean isHealthy;                 // 是否健康
```

**核心方法**:

#### 1. run - 运行健康监控
```java
@Override
public void run() {
    while (shouldRun) {
        try {
            // 1. 检查健康状态
            boolean healthy = checkHealth();
            
            // 2. 更新状态
            if (healthy != isHealthy) {
                isHealthy = healthy;
                notifyStateChanged();
            }
            
            // 3. 等待下一个检查周期
            Thread.sleep(checkIntervalMs);
        } catch (InterruptedException e) {
            break;
        }
    }
}
```

#### 2. checkHealth - 检查健康状态
```java
private boolean checkHealth() {
    try {
        // 1. 检查 RPC 服务是否可用
        boolean rpcHealthy = checkRpcService();
        
        // 2. 检查文件系统是否可用
        boolean fsHealthy = checkFileSystem();
        
        // 3. 检查内存是否充足
        boolean memoryHealthy = checkMemory();
        
        // 4. 返回综合健康状态
        return rpcHealthy && fsHealthy && memoryHealthy;
    } catch (Exception e) {
        return false;
    }
}
```

---

### 3.3 ActiveStandbyElector - 主备选举器

**位置**: `ActiveStandbyElector.java`

**作用**: 通过 ZooKeeper 实现主备选举和故障转移

**核心字段**:
```java
// ZooKeeper 连接
private final ZooKeeper zk;                          // ZooKeeper 客户端
private final String parentZnodeName;               // 父节点名称

// 选举状态
private volatile boolean active;                      // 是否活跃
private volatile long lastActiveAttempt;             // 最后活跃尝试时间

// 回调
private final ActiveStandbyElectorCallback callback;  // 回调接口
```

**核心方法**:
```java
@Override
public void run() {
    // 1. 创建 ZooKeeper 连接
    zk = connectToZK();
    
    // 2. 创建临时节点
    createLockNode();
    
    // 3. 进入选举循环
    while (true) {
        try {
            // 4. 检查是否是活跃节点
            if (isActive()) {
                // 5. 保持活跃状态
                maintainActiveState();
            } else {
                // 6. 等待成为活跃节点
                waitForActive();
            }
        } catch (Exception e) {
            // 7. 处理异常
            handleException(e);
        }
    }
}
```

---

## 四、JournalNode 详解

### 4.1 JournalNode - JournalNode 主类

**位置**: `JournalNode.java:75`

**作用**: JournalNode 的主类，负责提供共享编辑日志服务

**核心字段**:
```java
// 配置
private Configuration conf;                          // 配置
private String httpServerURI;                         // HTTP 服务 URI

// 服务
private JournalNodeRpcServer rpcServer;            // RPC 服务
private JournalNodeHttpServer httpServer;          // HTTP 服务

// 日志
private final Map<String, Journal> journalsById;    // 日志映射
private final Map<String, JournalNodeSyncer> journalSyncersById;  // 日志同步器映射

// 存储
private final ArrayList<File> localDir;               // 本地目录列表
```

**核心方法**:

#### 1. getOrCreateJournal - 获取或创建日志
```java
synchronized Journal getOrCreateJournal(String jid,
                                        String nameServiceId,
                                        StartupOption startOpt)
    throws IOException {
    // 1. 检查日志 ID
    QuorumJournalManager.checkJournalId(jid);
    
    // 2. 获取或创建日志
    Journal journal = journalsById.get(jid);
    if (journal == null) {
        // 3. 创建日志目录
        File logDir = getLogDir(jid, nameServiceId);
        
        // 4. 创建日志
        journal = new Journal(conf, logDir, jid, startOpt, new ErrorReporter());
        journalsById.put(jid, journal);
        
        // 5. 启动日志同步器
        if (conf.getBoolean(
            DFSConfigKeys.DFS_JOURNALNODE_ENABLE_SYNC_KEY,
            DFSConfigKeys.DFS_JOURNALNODE_ENABLE_SYNC_DEFAULT)) {
            startSyncer(journal, jid, nameServiceId);
        }
    }
    
    return journal;
}
```

#### 2. run - 运行 JournalNode
```java
@Override
public int run(String[] args) throws Exception {
    // 1. 初始化配置
    initConf(args);
    
    // 2. 启动 RPC 服务
    startRpcServer();
    
    // 3. 启动 HTTP 服务
    startHttpServer();
    
    // 4. 进入主循环
    while (true) {
        Thread.sleep(1000);
    }
}
```

---

### 4.2 Journal - 编辑日志

**位置**: `Journal.java`

**作用**: 存储和管理编辑日志

**核心字段**:
```java
// 日志配置
private final File logDir;                            // 日志目录
private final String journalId;                       // 日志 ID
private final StartupOption startOpt;                // 启动选项

// 日志文件
private final File currentLog;                        // 当前日志文件
private final File currentLogIndex;                    // 当前日志索引文件
```

**核心方法**:
```java
public void startLogSegment(long txId) throws IOException {
    // 1. 创建新的日志段
    File logFile = new File(logDir, "edits_" + txId + "_" + System.currentTimeMillis());
    
    // 2. 创建日志文件
    currentLog = logFile;
    
    // 3. 创建日志索引
    currentLogIndex = new File(logDir, "edits_" + txId + "_" + System.currentTimeMillis() + ".inprogress");
    
    // 4. 写入日志头
    writeLogHeader();
}
```

---

### 4.3 JournalNodeSyncer - 日志同步器

**位置**: `JournalNodeSyncer.java`

**作用**: 同步编辑日志，确保数据一致性

**核心字段**:
```java
// 同步配置
private final Journal journal;                      // 日志
private final String journalId;                     // 日志 ID
private final Configuration conf;                    // 配置

// 同步状态
private volatile boolean isRunning;                 // 是否运行中
private volatile long lastSyncedTxId;               // 最后同步的事务 ID
```

**核心方法**:
```java
@Override
public void run() {
    // 1. 进入运行状态
    isRunning = true;
    
    // 2. 进入主循环
    while (isRunning) {
        try {
            // 3. 获取最新事务 ID
            long latestTxId = journal.getLatestTxId();
            
            // 4. 同步事务
            syncTransactions(lastSyncedTxId, latestTxId);
            
            // 5. 更新最后同步的事务 ID
            lastSyncedTxId = latestTxId;
            
            // 6. 等待下一个同步周期
            Thread.sleep(syncIntervalMs);
        } catch (InterruptedException e) {
            break;
        }
    }
    
    // 7. 退出运行状态
    isRunning = false;
}
```

---

## 五、Router 详解

### 5.1 Router - Router 主类

**位置**: `Router.java:99`

**作用**: Router 的主类，作为联邦架构的统一入口

**核心字段**:
```java
// 服务组件
private RouterClientRMService clientRMProxyService;  // 客户端 RPC 服务
private RouterRMAdminService rmAdminProxyService;    // 管理 RPC 服务
private WebApp webApp;                               // Web 应用

// 清理器
private SubClusterCleaner subClusterCleaner;         // 子集群清理器
private ScheduledThreadPoolExecutor scheduledExecutorService;  // 定时执行器

// 配置
private Configuration conf;                          // 配置
private String webAppAddress;                         // Web 地址
```

**核心方法**:

#### 1. serviceInit - 服务初始化
```java
@Override
protected void serviceInit(Configuration config) throws Exception {
    this.conf = config;
    UserGroupInformation.setConfiguration(this.conf);

    // 1. 创建 ClientRM Proxy 服务
    clientRMProxyService = createClientRMProxyService();
    addService(clientRMProxyService);

    // 2. 创建 RMAdmin Proxy 服务
    rmAdminProxyService = createRMAdminProxyService();
    addService(rmAdminProxyService);

    // 3. 初始化 Web 服务地址
    webAppAddress = WebAppUtils.getWebAppBindURL(this.conf,
        YarnConfiguration.ROUTER_BIND_HOST,
        WebAppUtils.getRouterWebAppURLWithoutScheme(this.conf));

    // 4. 初始化 Metrics
    DefaultMetricsSystem.initialize(METRICS_NAME);
    JvmMetrics jm = JvmMetrics.initSingleton("Router", null);
    pauseMonitor = new JvmPauseMonitor();
    addService(pauseMonitor);
    jm.setPauseMonitor(pauseMonitor);

    // 5. 初始化 SubClusterCleaner
    this.subClusterCleaner = new SubClusterCleaner(this.conf);
    int scheduledExecutorThreads = conf.getInt(ROUTER_SCHEDULED_EXECUTOR_THREADS,
        DEFAULT_ROUTER_SCHEDULED_EXECUTOR_THREADS);
    this.scheduledExecutorService = new ScheduledThreadPoolExecutor(scheduledExecutorThreads);

    WebServiceClient.initialize(config);
    super.serviceInit(conf);
}
```

#### 2. serviceStart - 服务启动
```java
@Override
protected void serviceStart() throws Exception {
    // 1. 安全登录
    try {
        doSecureLogin();
    } catch (IOException e) {
        throw new YarnRuntimeException("Failed Router login", e);
    }

    // 2. 启动 SubClusterCleaner
    boolean isDeregisterSubClusterEnabled = this.conf.getBoolean(
        ROUTER_DEREGISTER_SUBCLUSTER_ENABLED, DEFAULT_ROUTER_DEREGISTER_SUBCLUSTER_ENABLED);
    if (isDeregisterSubClusterEnabled) {
        long scCleanerIntervalMs = this.conf.getTimeDuration(ROUTER_SUBCLUSTER_CLEANER_INTERVAL_TIME,
            DEFAULT_ROUTER_SUBCLUSTER_CLEANER_INTERVAL_TIME, TimeUnit.MILLISECONDS);
        this.scheduledExecutorService.scheduleAtFixedRate(this.subClusterCleaner,
            0, scCleanerIntervalMs, TimeUnit.MILLISECONDS);
        LOG.info("Scheduled SubClusterCleaner With Interval: {}.",
            DurationFormatUtils.formatDurationISO(scCleanerIntervalMs));
    }

    // 3. 启动 Web 应用
    startWepApp();
    super.serviceStart();
}
```

#### 3. startWepApp - 启动 Web 应用
```java
@VisibleForTesting
public void startWepApp() {
    // 1. 初始化 CORS 支持
    boolean enableCors = conf.getBoolean(YarnConfiguration.ROUTER_WEBAPP_ENABLE_CORS_FILTER,
        YarnConfiguration.DEFAULT_ROUTER_WEBAPP_ENABLE_CORS_FILTER);
    if (enableCors) {
        conf.setBoolean(HttpCrossOriginFilterInitializer.PREFIX
            + HttpCrossOriginFilterInitializer.ENABLED_SUFFIX, true);
    }

    LOG.info("Instantiating RouterWebApp at {}.", webAppAddress);

    // 2. 设置安全和过滤器
    RMWebAppUtil.setupSecurityAndFilters(conf, null);

    // 3. 构建 Web 应用
    Builder<Object> builder =
        WebApps.$for("cluster", null, null, "router-ws").with(conf).at(webAppAddress);

    // 4. 如果启用了 Web 代理
    if (RouterServerUtil.isRouterWebProxyEnable(conf)) {
        fetcher = new FedAppReportFetcher(conf);
        builder.withServlet(ProxyUriUtils.PROXY_SERVLET_NAME, ProxyUriUtils.PROXY_PATH_SPEC,
            WebAppProxyServlet.class);
        builder.withAttribute(WebAppProxy.FETCHER_ATTRIBUTE, fetcher);
        String proxyHostAndPort = getProxyHostAndPort(conf);
        String[] proxyParts = proxyHostAndPort.split(":");
        builder.withAttribute(WebAppProxy.PROXY_HOST_ATTRIBUTE, proxyParts[0]);
    }

    // 5. 启动 RouterWebApp
    RouterWebApp routerWebApp = new RouterWebApp(this);
    builder.withResourceConfig(routerWebApp.resourceConfig());
    webApp = builder.start(routerWebApp, getUIWebAppContext());
}
```

---

## 六、工作流程

### 6.1 ZKFC 主备切换流程

```
1. ZKFC 启动
   ↓
2. 初始化 ZooKeeper 连接
   ↓
3. 启动健康监控
   ↓
4. 启动主备选举器
   ↓
5. 进入选举循环
   ↓
6. 检查健康状态
   ↓
7. 如果健康，参与选举
   ↓
8. 如果选举成功，成为 Active
   ↓
9. 如果选举失败，成为 Standby
   ↓
10. 监控健康状态
   ↓
11. 如果 Active 故障，触发故障转移
   ↓
12. Standby 成为 Active
```

### 6.2 JournalNode 日志服务流程

```
1. JournalNode 启动
   ↓
2. 初始化配置
   ↓
3. 启动 RPC 服务
   ↓
4. 启动 HTTP 服务
   ↓
5. 等待 NameNode 连接
   ↓
6. NameNode 请求创建日志段
   ↓
7. 创建日志段
   ↓
8. NameNode 写入编辑日志
   ↓
9. JournalNode 存储编辑日志
   ↓
10. NameNode 请求读取编辑日志
   ↓
11. JournalNode 返回编辑日志
   ↓
12. Standby NameNode 同步编辑日志
```

### 6.3 Router 请求处理流程

```
1. 客户端提交应用
   ↓
2. 请求到达 Router
   ↓
3. RouterClientRMService 接收请求
   ↓
4. 获取用户管道（RequestInterceptorChain）
   ↓
5. 拦截器链处理请求
   - 第一个拦截器：获取应用归属
   - 第二个拦截器：选择子集群
   - 第三个拦截器：转发请求
   ↓
6. 获取应用归属
   - 查询 FederationApplicationHomeSubClusterStore
   - 如果应用已存在，返回归属子集群
   - 如果应用不存在，根据策略选择子集群
   ↓
7. 选择子集群
   - 根据队列策略选择子集群
   - 考虑子集群状态、负载等因素
   ↓
8. 转发请求到子集群
   - 调用子集群的 ResourceManager
   - 提交应用
   ↓
9. 记录应用归属
   - 调用 FederationApplicationHomeSubClusterStore.addApplicationHomeSubCluster()
   - 保存应用到子集群的映射
   ↓
10. 返回响应给客户端
```

---

## 七、关键特性

### 7.1 ZKFC 特性

**优势**:
- 自动故障转移
- 主备切换
- 健康监控
- 选举机制

**实现**:
- ZooKeeper 选举
- 健康监控
- 故障检测
- 自动切换

### 7.2 JournalNode 特性

**优势**:
- 共享编辑日志
- 高可用性
- 数据一致性
- 故障恢复

**实现**:
- 共享存储
- 日志同步
- 数据校验
- 故障恢复

### 7.3 Router 特性

**优势**:
- 统一访问接口
- 请求拦截器链
- 状态存储
- 策略管理

**实现**:
- 统一入口
- 拦截器链
- 状态存储
- 策略管理

---

## 八、配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `ha.zookeeper.quorum` | - | ZooKeeper 集群地址 |
| `ha.zookeeper.session-timeout.ms` | 10000ms | ZooKeeper 会话超时 |
| `ha.zookeeper.parent-znode` | /hadoop-ha | ZooKeeper 父节点 |
| `dfs.journalnode.edits.dir` | /hadoop/hdfs/journal | JournalNode 编辑日志目录 |
| `dfs.journalnode.enable.sync` | true | 是否启用日志同步 |
| `yarn.router.bind-host` | 0.0.0.0 | Router 绑定主机 |
| `yarn.router.clientrm.address` | 0.0.0.0:8050 | Router ClientRM 服务地址 |
| `yarn.router.rmadmin.address` | 0.0.0.0:8051 | Router RMAdmin 服务地址 |
| `yarn.router.webapp.address` | 0.0.0.0:8089 | Router Web 应用地址 |

---

## 九、总结

ZKFC、JournalNode 和 Router 是 HDFS 高可用和联邦架构的核心组件，具有以下特点：

1. **ZKFC**: 自动故障转移、主备切换、健康监控、选举机制
2. **JournalNode**: 共享编辑日志、高可用性、数据一致性、故障恢复
3. **Router**: 统一访问接口、请求拦截器链、状态存储、策略管理

**关键设计思想**:
- ZooKeeper 选举机制
- 共享编辑日志
- 统一访问接口
- 拦截器链设计
- 状态集中管理

通过深入理解 ZKFC、JournalNode 和 Router 的实现，可以更好地构建和管理高可用的 HDFS 集群，实现故障自动转移和联邦架构。
