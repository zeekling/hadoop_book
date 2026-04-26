# YARN Federation 源码解读

## 一、概述

YARN Federation 是 Hadoop YARN 的联邦架构，允许将多个独立的 YARN 集群（称为 SubCluster）联合起来，形成一个逻辑上的统一集群。通过 Router 组件作为统一入口，客户端可以透明地访问联邦中的所有资源，实现跨集群的资源管理和作业调度。

**核心目标**：
- 扩展 YARN 集群规模，突破单集群的资源限制
- 提供统一的访问接口，屏蔽底层多个集群的复杂性
- 支持跨集群的作业提交和资源调度
- 实现集群间的负载均衡和故障转移

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
│              FederationStateStore (联邦状态存储)              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  FederationMembershipStateStore (成员状态存储)       │   │
│  │  - SubClusterInfo (子集群信息)                        │   │
│  │  - SubClusterState (子集群状态)                       │   │
│  │  - 注册/注销/心跳                                     │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  FederationApplicationHomeSubClusterStore (应用存储) │   │
│  │  - ApplicationHomeSubCluster (应用归属)              │   │
│  │  - 应用到子集群的映射                                 │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  FederationPolicyStore (策略存储)                     │   │
│  │  - SubClusterPolicyConfiguration (策略配置)          │   │
│  │  - 队列到策略的映射                                   │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  FederationReservationHomeSubClusterStore (预留存储)  │   │
│  │  - ReservationHomeSubCluster (预留归属)              │   │
│  │  - 预留到子集群的映射                                 │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  FederationDelegationTokenStateStore (令牌存储)      │   │
│  │  - RouterRMToken (路由器令牌)                         │   │
│  │  - RouterMasterKey (主密钥)                           │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    SubCluster (子集群)                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ SubCluster 1 │  │ SubCluster 2 │  │ SubCluster 3 │ ... │
│  │  - RM        │  │  - RM        │  │  - RM        │     │
│  │  - NM        │  │  - NM        │  │  - NM        │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 核心组件

**Router (路由器)**:
- `Router` - 主服务类，继承 CompositeService
- `RouterClientRMService` - 客户端 RPC 服务
- `RouterRMAdminService` - 管理 RPC 服务
- `RouterWebApp` - Web 服务
- `SubClusterCleaner` - 子集群清理器

**StateStore (状态存储)**:
- `FederationStateStore` - 联邦状态存储接口
- `FederationMembershipStateStore` - 成员状态存储
- `FederationApplicationHomeSubClusterStore` - 应用存储
- `FederationPolicyStore` - 策略存储
- `FederationReservationHomeSubClusterStore` - 预留存储
- `FederationDelegationTokenStateStore` - 令牌存储

**Resolver (解析器)**:
- `SubClusterResolver` - 子集群解析器接口

**Policy (策略)**:
- `SubClusterPolicyConfiguration` - 子集群策略配置

---

## 三、主要组件详解

### 3.1 Router - 路由器

**位置**: `Router.java:99`

**作用**: YARN Federation 的核心组件，作为统一入口，代理客户端请求到各个子集群

**核心字段**:
```java
// 服务组件
private RouterClientRMService clientRMProxyService;  // 客户端 RPC 服务
private RouterRMAdminService rmAdminProxyService;    // 管理 RPC 服务
private WebApp webApp;                               // Web 应用
private FedAppReportFetcher fetcher;                 // 应用报告获取器

// 清理器
private SubClusterCleaner subClusterCleaner;         // 子集群清理器
private ScheduledThreadPoolExecutor scheduledExecutorService;  // 定时执行器

// 配置
private Configuration conf;                          // 配置
private String webAppAddress;                         // Web 地址
private static long clusterTimeStamp;                // 集群时间戳
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

### 3.2 RouterClientRMService - 客户端 RPC 服务

**位置**: `RouterClientRMService.java:132`

**作用**: 实现 ApplicationClientProtocol，拦截客户端请求并通过拦截器链转发到子集群

**核心字段**:
```java
private Server server;                                          // RPC 服务器
private InetSocketAddress listenerEndpoint;                     // 监听端点
private Map<String, RequestInterceptorChainWrapper> userPipelineMap;  // 用户管道缓存
private URL redirectURL;                                        // 重定向 URL
private RouterDelegationTokenSecretManager routerDTSecretManager;  // 令牌管理器
```

**核心方法**:

#### 1. serviceStart - 服务启动
```java
@Override
protected void serviceStart() throws Exception {
    LOG.info("Starting Router ClientRMService.");
    Configuration conf = getConfig();
    YarnRPC rpc = YarnRPC.create(conf);
    UserGroupInformation.setConfiguration(conf);

    // 1. 获取监听地址
    this.listenerEndpoint =
        conf.getSocketAddr(YarnConfiguration.ROUTER_BIND_HOST,
            YarnConfiguration.ROUTER_CLIENTRM_ADDRESS,
            YarnConfiguration.DEFAULT_ROUTER_CLIENTRM_ADDRESS,
            YarnConfiguration.DEFAULT_ROUTER_CLIENTRM_PORT);

    // 2. 如果启用了 Web 代理，获取重定向 URL
    if (RouterServerUtil.isRouterWebProxyEnable(conf)) {
        redirectURL = getRedirectURL();
    }

    // 3. 初始化用户管道缓存
    int maxCacheSize =
        conf.getInt(YarnConfiguration.ROUTER_PIPELINE_CACHE_MAX_SIZE,
            YarnConfiguration.DEFAULT_ROUTER_PIPELINE_CACHE_MAX_SIZE);
    this.userPipelineMap = Collections.synchronizedMap(new LRUCacheHashMap<>(maxCacheSize, true));

    // 4. 初始化 RPC 服务器配置
    Configuration serverConf = new Configuration(conf);
    int numWorkerThreads =
        serverConf.getInt(YarnConfiguration.RM_CLIENT_THREAD_COUNT,
            YarnConfiguration.DEFAULT_RM_CLIENT_THREAD_COUNT);

    // 5. 初始化 RouterRMDelegationTokenSecretManager
    routerDTSecretManager = createRouterRMDelegationTokenSecretManager(conf);
    routerDTSecretManager.startThreads();

    // 6. 启动 RPC 服务器
    this.server = rpc.getServer(ApplicationClientProtocol.class, this,
        listenerEndpoint, serverConf, routerDTSecretManager, numWorkerThreads);

    // 7. 启用服务授权
    if (conf.getBoolean(
        CommonConfigurationKeysPublic.HADOOP_SECURITY_AUTHORIZATION, false)) {
        refreshServiceAcls(conf, RouterPolicyProvider.getInstance());
    }

    this.server.start();
    LOG.info("Router ClientRMService listening on address: {}.", this.server.getListenerAddress());
    super.serviceStart();
}
```

#### 2. submitApplication - 提交应用
```java
@Override
public SubmitApplicationResponse submitApplication(
    SubmitApplicationRequest request) throws YarnException, IOException {

    // 1. 获取用户管道
    RequestInterceptorChainWrapper pipeline = getUserPipeline();

    // 2. 通过拦截器链提交应用
    return pipeline.submitApplication(request);
}
```

#### 3. getUserPipeline - 获取用户管道
```java
private RequestInterceptorChainWrapper getUserPipeline() throws YarnException {
    // 1. 获取当前用户
    String user = UserGroupInformation.getCurrentUser().getShortUserName();

    // 2. 从缓存中获取管道
    RequestInterceptorChainWrapper pipeline = userPipelineMap.get(user);

    // 3. 如果缓存中没有，创建新管道
    if (pipeline == null) {
        pipeline = createPipeline(user);
        userPipelineMap.put(user, pipeline);
    }

    return pipeline;
}
```

---

### 3.3 FederationStateStore - 联邦状态存储

**位置**: `FederationStateStore.java:35`

**作用**: 联邦状态存储的统一接口，管理所有联邦相关的状态信息

**核心接口**:
```java
public interface FederationStateStore extends
    FederationApplicationHomeSubClusterStore,    // 应用存储
    FederationMembershipStateStore,              // 成员存储
    FederationPolicyStore,                      // 策略存储
    FederationReservationHomeSubClusterStore,   // 预留存储
    FederationDelegationTokenStateStore {       // 令牌存储

    // 初始化
    void init(Configuration conf) throws YarnException;

    // 关闭
    void close() throws Exception;

    // 版本管理
    Version getCurrentVersion();
    Version loadVersion() throws Exception;
    void storeVersion() throws Exception;
    void checkVersion() throws Exception;

    // 删除状态存储
    void deleteStateStore() throws Exception;
}
```

**核心方法**:

#### 1. checkVersion - 检查版本
```java
default void checkVersion() throws Exception {
    Version loadedVersion = loadVersion();
    LOG.info("Loaded Router State Version Info = {}.", loadedVersion);
    Version currentVersion = getCurrentVersion();

    if (loadedVersion != null && loadedVersion.equals(currentVersion)) {
        return;
    }

    // 如果没有版本信息，使用当前版本
    if (loadedVersion == null) {
        loadedVersion = currentVersion;
    }

    // 检查兼容性
    if (loadedVersion.isCompatibleTo(currentVersion)) {
        LOG.info("Storing Router State Version Info {}.", currentVersion);
        storeVersion();
    } else {
        throw new FederationStateVersionIncompatibleException(
           "Expecting Router state version " + currentVersion +
           ", but loading version " + loadedVersion);
    }
}
```

---

### 3.4 FederationMembershipStateStore - 成员状态存储

**位置**: `FederationMembershipStateStore.java:41`

**作用**: 管理所有参与联邦的子集群的成员信息

**核心方法**:

#### 1. registerSubCluster - 注册子集群
```java
/**
 * 注册子集群到联邦
 * @param registerSubClusterRequest 子集群注册请求
 * @return 注册响应
 * @throws YarnException 如果注册失败
 */
SubClusterRegisterResponse registerSubCluster(
    SubClusterRegisterRequest registerSubClusterRequest) throws YarnException;
```

#### 2. deregisterSubCluster - 注销子集群
```java
/**
 * 从联邦中注销子集群
 * @param subClusterDeregisterRequest 子集群注销请求
 * @return 注销响应
 * @throws YarnException 如果注销失败
 */
SubClusterDeregisterResponse deregisterSubCluster(
    SubClusterDeregisterRequest subClusterDeregisterRequest)
    throws YarnException;
```

#### 3. subClusterHeartbeat - 子集群心跳
```java
/**
 * 子集群心跳，保持活跃状态
 * @param subClusterHeartbeatRequest 心跳请求
 * @return 心跳响应
 * @throws YarnException 如果心跳失败
 */
SubClusterHeartbeatResponse subClusterHeartbeat(
    SubClusterHeartbeatRequest subClusterHeartbeatRequest)
    throws YarnException;
```

#### 4. getSubCluster - 获取子集群信息
```java
/**
 * 获取指定子集群的信息
 * @param subClusterRequest 子集群请求
 * @return 子集群信息
 * @throws YarnException 如果获取失败
 */
GetSubClusterInfoResponse getSubCluster(
    GetSubClusterInfoRequest subClusterRequest) throws YarnException;
```

#### 5. getSubClusters - 获取所有子集群信息
```java
/**
 * 获取所有子集群的信息
 * @param subClustersRequest 子集群请求
 * @return 所有子集群信息
 * @throws YarnException 如果获取失败
 */
GetSubClustersInfoResponse getSubClusters(
    GetSubClustersInfoRequest subClustersRequest) throws YarnException;
```

---

### 3.5 SubClusterInfo - 子集群信息

**位置**: `SubClusterInfo.java:44`

**作用**: 表示子集群的运行时信息

**核心字段**:
```java
private SubClusterId subClusterId;              // 子集群 ID
private String amRMServiceAddress;              // AM-RM 服务地址
private String clientRMServiceAddress;          // Client-RM 服务地址
private String rmAdminServiceAddress;           // RM Admin 服务地址
private String rmWebServiceAddress;             // RM Web 服务地址
private long lastHeartBeat;                     // 最后心跳时间
private SubClusterState state;                   // 子集群状态
private long lastStartTime;                     // 最后启动时间
private String capability;                      // 能力信息（ClusterMetrics）
```

**核心方法**:

#### 1. newInstance - 创建实例
```java
public static SubClusterInfo newInstance(SubClusterId subClusterId,
    String amRMServiceAddress, String clientRMServiceAddress,
    String rmAdminServiceAddress, String rmWebServiceAddress,
    long lastHeartBeat, SubClusterState state, long lastStartTime,
    String capability) {
    SubClusterInfo subClusterInfo = Records.newRecord(SubClusterInfo.class);
    subClusterInfo.setSubClusterId(subClusterId);
    subClusterInfo.setAMRMServiceAddress(amRMServiceAddress);
    subClusterInfo.setClientRMServiceAddress(clientRMServiceAddress);
    subClusterInfo.setRMAdminServiceAddress(rmAdminServiceAddress);
    subClusterInfo.setRMWebServiceAddress(rmWebServiceAddress);
    subClusterInfo.setLastHeartBeat(lastHeartBeat);
    subClusterInfo.setState(state);
    subClusterInfo.setLastStartTime(lastStartTime);
    subClusterInfo.setCapability(capability);
    return subClusterInfo;
}
```

---

### 3.6 SubClusterState - 子集群状态

**位置**: `SubClusterState.java:32`

**作用**: 表示子集群的状态

**状态枚举**:
```java
public enum SubClusterState {
    /** 新注册的子集群，第一次心跳之前 */
    SC_NEW,

    /** 子集群已注册且 RM 最近发送了心跳 */
    SC_RUNNING,

    /** 子集群不健康 */
    SC_UNHEALTHY,

    /** 子集群正在下线过程中 */
    SC_DECOMMISSIONING,

    /** 子集群已下线 */
    SC_DECOMMISSIONED,

    /** RM 在配置的时间阈值内未发送心跳 */
    SC_LOST,

    /** 子集群已注销 */
    SC_UNREGISTERED;
}
```

**核心方法**:

#### 1. isUsable - 是否可用
```java
public boolean isUsable() {
    return (this == SC_RUNNING || this == SC_NEW);
}
```

#### 2. isActive - 是否活跃
```java
public boolean isActive() {
    return this == SC_RUNNING;
}
```

#### 3. isFinal - 是否为最终状态
```java
public boolean isFinal() {
    return (this == SC_UNREGISTERED || this == SC_DECOMMISSIONED
        || this == SC_LOST);
}
```

---

### 3.7 FederationApplicationHomeSubClusterStore - 应用存储

**位置**: `FederationApplicationHomeSubClusterStore.java:49`

**作用**: 管理所有提交到联邦的应用的归属信息

**核心方法**:

#### 1. addApplicationHomeSubCluster - 添加应用归属
```java
/**
 * 注册新提交的应用的归属子集群
 * @param request 应用归属请求
 * @return 应用归属响应
 * @throws YarnException 如果添加失败
 */
AddApplicationHomeSubClusterResponse addApplicationHomeSubCluster(
    AddApplicationHomeSubClusterRequest request) throws YarnException;
```

#### 2. updateApplicationHomeSubCluster - 更新应用归属
```java
/**
 * 更新应用的归属子集群
 * @param request 应用归属更新请求
 * @return 更新响应
 * @throws YarnException 如果更新失败
 */
UpdateApplicationHomeSubClusterResponse updateApplicationHomeSubCluster(
    UpdateApplicationHomeSubClusterRequest request) throws YarnException;
```

#### 3. getApplicationHomeSubCluster - 获取应用归属
```java
/**
 * 获取指定应用的归属信息
 * @param request 应用请求
 * @return 应用归属信息
 * @throws YarnException 如果获取失败
 */
GetApplicationHomeSubClusterResponse getApplicationHomeSubCluster(
    GetApplicationHomeSubClusterRequest request) throws YarnException;
```

#### 4. getApplicationsHomeSubCluster - 获取所有应用归属
```java
/**
 * 获取所有应用的归属信息
 * @param request 应用请求
 * @return 所有应用归属信息
 * @throws YarnException 如果获取失败
 */
GetApplicationsHomeSubClusterResponse getApplicationsHomeSubCluster(
    GetApplicationsHomeSubClusterRequest request) throws YarnException;
```

#### 5. deleteApplicationHomeSubCluster - 删除应用归属
```java
/**
 * 删除应用的归属信息
 * @param request 应用删除请求
 * @return 删除响应
 * @throws YarnException 如果删除失败
 */
DeleteApplicationHomeSubClusterResponse deleteApplicationHomeSubCluster(
    DeleteApplicationHomeSubClusterRequest request) throws YarnException;
```

---

### 3.8 FederationPolicyStore - 策略存储

**位置**: `FederationPolicyStore.java:45`

**作用**: 管理联邦的策略配置，支持为不同队列配置不同的策略

**核心方法**:

#### 1. getPolicyConfiguration - 获取策略配置
```java
/**
 * 获取指定队列的策略配置
 * @param request 队列请求
 * @return 策略配置
 * @throws YarnException 如果获取失败
 */
GetSubClusterPolicyConfigurationResponse getPolicyConfiguration(
    GetSubClusterPolicyConfigurationRequest request) throws YarnException;
```

#### 2. setPolicyConfiguration - 设置策略配置
```java
/**
 * 设置指定队列的策略配置
 * @param request 策略配置请求
 * @return 设置响应
 * @throws YarnException 如果设置失败
 */
SetSubClusterPolicyConfigurationResponse setPolicyConfiguration(
    SetSubClusterPolicyConfigurationRequest request) throws YarnException;
```

#### 3. getPoliciesConfigurations - 获取所有策略配置
```java
/**
 * 获取所有队列的策略配置
 * @param request 请求
 * @return 所有策略配置
 * @throws YarnException 如果获取失败
 */
GetSubClusterPoliciesConfigurationsResponse getPoliciesConfigurations(
    GetSubClusterPoliciesConfigurationsRequest request) throws YarnException;
```

#### 4. deletePoliciesConfigurations - 删除策略配置
```java
/**
 * 删除指定队列的策略配置
 * @param request 删除请求
 * @return 删除响应
 * @throws YarnException 如果删除失败
 */
DeleteSubClusterPoliciesConfigurationsResponse deletePoliciesConfigurations(
    DeleteSubClusterPoliciesConfigurationsRequest request) throws YarnException;
```

#### 5. deleteAllPoliciesConfigurations - 删除所有策略配置
```java
/**
 * 删除所有队列的策略配置
 * @param request 删除请求
 * @return 删除响应
 * @throws Exception 如果删除失败
 */
DeletePoliciesConfigurationsResponse deleteAllPoliciesConfigurations(
    DeletePoliciesConfigurationsRequest request) throws Exception;
```

---

### 3.9 SubClusterResolver - 子集群解析器

**位置**: `SubClusterResolver.java:31`

**作用**: 确定指定节点或机架所属的子集群

**核心方法**:

#### 1. getSubClusterForNode - 获取节点所属子集群
```java
/**
 * 获取指定节点所属的子集群
 * @param nodename 节点名称
 * @return 子集群 ID
 * @throws YarnException 如果解析失败
 */
SubClusterId getSubClusterForNode(String nodename) throws YarnException;
```

#### 2. getSubClustersForRack - 获取机架上的子集群
```java
/**
 * 获取指定机架上的所有子集群
 * @param rackname 机架名称
 * @return 子集群 ID 集合
 * @throws YarnException 如果解析失败
 */
Set<SubClusterId> getSubClustersForRack(String rackname) throws YarnException;
```

#### 3. load - 加载映射
```java
/**
 * 从文件加载节点到子集群的映射
 */
void load();
```

---

### 3.10 SubClusterCleaner - 子集群清理器

**位置**: `SubClusterCleaner.java`

**作用**: 定期清理失效的子集群

**核心功能**:
- 定期检查子集群状态
- 清理失效的子集群
- 更新联邦状态存储

---

## 四、工作流程

### 4.1 子集群注册流程

```
1. ResourceManager 启动
   ↓
2. 调用 FederationStateStore.registerSubCluster()
   - 提供子集群信息（SubClusterInfo）
   - 包含服务地址、能力信息等
   ↓
3. FederationStateStore 验证请求
   - 检查子集群 ID 是否已存在
   - 如果不存在，创建新记录
   - 如果存在，更新现有记录
   ↓
4. 设置子集群状态为 SC_NEW
   ↓
5. 返回注册响应
   - 包含子集群 ID
   ↓
6. ResourceManager 开始发送心跳
   ↓
7. 子集群状态更新为 SC_RUNNING
```

### 4.2 应用提交流程

```
1. 客户端提交应用
   ↓
2. 请求到达 Router
   ↓
3. RouterClientRMService 接收请求
   ↓
4. 获取用户管道（RequestInterceptorChain）
   - 从缓存中获取
   - 如果缓存中没有，创建新管道
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

### 4.3 子集群心跳流程

```
1. ResourceManager 定期发送心跳
   ↓
2. 心跳到达 Router
   ↓
3. Router 调用 FederationStateStore.subClusterHeartbeat()
   - 提供子集群信息（SubClusterInfo）
   - 包含当前能力、状态等
   ↓
4. FederationStateStore 更新子集群信息
   - 更新最后心跳时间
   - 更新能力信息
   - 更新状态
   ↓
5. 检查子集群状态
   - 如果心跳超时，状态设置为 SC_LOST
   - 如果子集群不健康，状态设置为 SC_UNHEALTHY
   ↓
6. 返回心跳响应
   ↓
7. ResourceManager 继续发送心跳
```

### 4.4 应用查询流程

```
1. 客户端查询应用状态
   ↓
2. 请求到达 Router
   ↓
3. RouterClientRMService 接收请求
   ↓
4. 获取用户管道
   ↓
5. 拦截器链处理请求
   - 第一个拦截器：获取应用归属
   - 第二个拦截器：转发请求
   ↓
6. 获取应用归属
   - 查询 FederationApplicationHomeSubClusterStore
   - 获取应用所属的子集群
   ↓
7. 转发请求到子集群
   - 调用子集群的 ResourceManager
   - 查询应用状态
   ↓
8. 返回响应给客户端
```

---

## 五、关键特性

### 5.1 统一访问接口

**优势**:
- 客户端无需知道底层有多个集群
- 使用标准的 YARN API
- 支持多种协议（RPC、REST）

**实现**:
- Router 实现 ApplicationClientProtocol
- Router 实现 ResourceManagerAdminProtocol
- Router 提供 Web UI 和 REST API

### 5.2 请求拦截器链

**优势**:
- 灵活的请求处理
- 可扩展的拦截器
- 支持请求/响应修改

**实现**:
```java
// 拦截器链示例
RequestInterceptorChain pipeline = new RequestInterceptorChain();
pipeline.addInterceptor(new GetApplicationHomeSubClusterInterceptor());
pipeline.addInterceptor(new SubClusterPolicyInterceptor());
pipeline.addInterceptor(new ForwardingInterceptor());
```

### 5.3 状态存储

**优势**:
- 集中式状态管理
- 支持多种存储后端（ZooKeeper、SQL、文件系统）
- 高可用性

**实现**:
- FederationStateStore 接口
- 支持多种实现（ZooKeeper、SQL、文件系统）
- 版本管理和兼容性检查

### 5.4 策略管理

**优势**:
- 灵活的子集群选择策略
- 支持队列级别的策略配置
- 可扩展的策略实现

**实现**:
- FederationPolicyStore 接口
- SubClusterPolicyConfiguration 配置
- 支持多种策略（RoundRobin、LoadBased 等）

### 5.5 子集群管理

**优势**:
- 自动注册和注销
- 心跳机制保持活跃
- 状态监控和故障检测

**实现**:
- FederationMembershipStateStore 接口
- SubClusterInfo 和 SubClusterState
- SubClusterCleaner 定期清理

### 5.6 应用归属管理

**优势**:
- 记录应用到子集群的映射
- 支持应用迁移
- 快速定位应用

**实现**:
- FederationApplicationHomeSubClusterStore 接口
- ApplicationHomeSubCluster 记录
- 支持添加、更新、删除操作

### 5.7 高可用性

**优势**:
- Router 可以部署多个实例
- 支持负载均衡
- 故障转移

**实现**:
- Router 无状态设计
- 支持多实例部署
- 使用 VIP 和负载均衡器

---

## 六、配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `yarn.router.bind-host` | 0.0.0.0 | Router 绑定主机 |
| `yarn.router.clientrm.address` | 0.0.0.0:8050 | Router ClientRM 服务地址 |
| `yarn.router.rmadmin.address` | 0.0.0.0:8051 | Router RMAdmin 服务地址 |
| `yarn.router.webapp.address` | 0.0.0.0:8089 | Router Web 应用地址 |
| `yarn.router.pipeline.cache.max-size` | 100 | 管道缓存最大大小 |
| `yarn.router.deregister.subcluster.enabled` | true | 是否启用子集群注销 |
| `yarn.router.subcluster.cleaner.interval-time` | 5min | 子集群清理器间隔时间 |
| `yarn.router.scheduled.executor.threads` | 5 | 定时执行器线程数 |
| `yarn.router.webapp.enable.cors.filter` | false | 是否启用 CORS 过滤器 |
| `yarn.federation.statestore.class` | - | 联邦状态存储实现类 |
| `yarn.federation.policy-manager.class` | - | 联邦策略管理器实现类 |

---

## 七、性能优化

### 7.1 调整管道缓存大小

**原则**:
- 缓存越大，命中率越高，但内存占用越高
- 建议设置为 100-500

**配置**:
```xml
<property>
    <name>yarn.router.pipeline.cache.max-size</name>
    <value>200</value>
</property>
```

### 7.2 调整 RPC 线程数

**原则**:
- 线程数越多，并发处理能力越强，但资源占用越高
- 建议设置为 20-50

**配置**:
```xml
<property>
    <name>yarn.resourcemanager.client.thread.count</name>
    <value>30</value>
</property>
```

### 7.3 调整心跳间隔

**原则**:
- 心跳间隔越短，故障检测越快，但网络开销越大
- 建议设置为 10-30 秒

**配置**:
```xml
<property>
    <name>yarn.resourcemanager.heartbeat.interval-ms</name>
    <value>15000</value>
</property>
```

### 7.4 使用高效的 StateStore

**原则**:
- ZooKeeper 适合小规模集群
- SQL 适合大规模集群
- 文件系统适合测试环境

**配置**:
```xml
<property>
    <name>yarn.federation.statestore.class</name>
    <value>org.apache.hadoop.yarn.server.federation.store.impl.SQLFederationStateStore</value>
</property>
```

---

## 八、总结

YARN Federation 是 Hadoop YARN 的联邦架构，具有以下特点：

1. **统一访问**: Router 提供统一的访问接口，屏蔽底层多个集群的复杂性
2. **高可用性**: 支持多实例部署，故障转移
3. **可扩展性**: 支持动态添加/删除子集群
4. **灵活性**: 支持多种策略和存储后端
5. **透明性**: 客户端无需修改代码即可使用联邦

**关键设计思想**:
- Router 作为统一入口，代理客户端请求
- StateStore 集中管理联邦状态
- 拦截器链提供灵活的请求处理
- 策略管理支持灵活的子集群选择
- 心跳机制保持子集群活跃

通过深入理解 YARN Federation 的实现，可以更好地构建和管理大规模的 Hadoop 集群，实现跨集群的资源管理和作业调度。
