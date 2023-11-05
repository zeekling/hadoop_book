
# 简介

本章详细介绍NameNode启动过程。主要是代码级别的解释。

nameNode的启动主要是有NameNode.java主导的，由main函数开始了解。

下面是main函数里面的主要内容，可以看到主要由createNameNode实现NameNode的启动。
```java
NameNode namenode = createNameNode(argv, null);
if (namenode != null) {
  namenode.join();
}
```

在createNameNode函数里面主要是分为两部分：
- 参数解析：主要关心解析startOpt，startOpt可以控制具体操作，比如format、rockback等。主要操作如下,后续会详细介绍。
  ```java
    FORMAT  ("-format"),
    CLUSTERID ("-clusterid"),
    GENCLUSTERID ("-genclusterid"),
    REGULAR ("-regular"),
    BACKUP  ("-backup"),
    CHECKPOINT("-checkpoint"),
    UPGRADE ("-upgrade"),
    ROLLBACK("-rollback"),
    ROLLINGUPGRADE("-rollingUpgrade"),
    IMPORT  ("-importCheckpoint"),
    BOOTSTRAPSTANDBY("-bootstrapStandby"),
    INITIALIZESHAREDEDITS("-initializeSharedEdits"),
    RECOVER  ("-recover"),
    FORCE("-force"),
    NONINTERACTIVE("-nonInteractive"),
    SKIPSHAREDEDITSCHECK("-skipSharedEditsCheck"),
    RENAMERESERVED("-renameReserved"),
    METADATAVERSION("-metadataVersion"),
    UPGRADEONLY("-upgradeOnly"),
    HOTSWAP("-hotswap"),
    OBSERVER("-observer");
  ```
  模型情况下会走到启动的启动的流程里面。
- 启动NameNode或者其他操作，比如format等。

## 启动

NameNode的核心主要在NameNode的构造函数里面。

```java 
this.haEnabled = HAUtil.isHAEnabled(conf, nsId);
// 检查HA的状态，主要是判断当前启动的是主实例还是备实例
state = createHAState(getStartupOption(conf));
this.allowStaleStandbyReads = HAUtil.shouldAllowStandbyReads(conf);
this.haContext = createHAContext();
try {
  initializeGenericKeys(conf, nsId, namenodeId);
  // 启动NameNode
  initialize(getConf());
  state.prepareToEnterState(haContext);
  try {
    haContext.writeLock();
    state.enterState(haContext);
  } finally {
    haContext.writeUnlock();
  }
} catch (IOException e) {
  this.stopAtException(e);
  throw e;
} catch (HadoopIllegalArgumentException e) {
  this.stopAtException(e);
  throw e;
}
```

initialize函数详解如下：

```java 
protected void initialize(Configuration conf) throws IOException {
  // .... 省略
  //登录kerberos
  UserGroupInformation.setConfiguration(conf);
  loginAsNameNodeUser(conf);

  // 初始化监控信息
  NameNode.initMetrics(conf, this.getRole());
  StartupProgressMetrics.register(startupProgress);

  pauseMonitor = new JvmPauseMonitor();
  pauseMonitor.init(conf);
  pauseMonitor.start();
  metrics.getJvmMetrics().setPauseMonitor(pauseMonitor);

  // .... 省略

  if (NamenodeRole.NAMENODE == role) {
    // 启动HTTPServer,会调用NameNodeHttpServer中的start函数，是基于org.eclipse.jetty.server.Server实现的
    startHttpServer(conf);
  }
  // 从本地加载FSImage，并且与Editlog合并产生新的FSImage
  loadNamesystem(conf);
  //TODO 待确认用途
  startAliasMapServerIfNecessary(conf);

  //创建rpcserver，封装了NameNodeRpcServer、ClientRPCServer
  //支持ClientNameNodeProtocol、DataNodeProtocolPB等协议
  rpcServer = createRpcServer(conf);

  initReconfigurableBackoffKey();

  // .... 省略

  if (NamenodeRole.NAMENODE == role) {
    httpServer.setNameNodeAddress(getNameNodeAddress());
    httpServer.setFSImage(getFSImage());
    if (levelDBAliasMapServer != null) {
      httpServer.setAliasMap(levelDBAliasMapServer.getAliasMap());
    }
  }

  //启动执行多个重要的工作线程
  startCommonServices(conf);
  startMetricsLogger(conf);
}
```

### startCommonServices函数详解

启动NameNode关键服务

```java
private void startCommonServices(Configuration conf) throws IOException {
  // 创建NameNodeResourceChecker、激活BlockManager等
  namesystem.startCommonServices(conf, haContext);
  registerNNSMXBean();
  if (NamenodeRole.NAMENODE != role) {
    startHttpServer(conf);
    httpServer.setNameNodeAddress(getNameNodeAddress());
    httpServer.setFSImage(getFSImage());
    if (levelDBAliasMapServer != null) {
      httpServer.setAliasMap(levelDBAliasMapServer.getAliasMap());
    }
  }
  // 启动rpc服务
  rpcServer.start();
  try {
    // 获取启动插件列表
    plugins = conf.getInstances(DFS_NAMENODE_PLUGINS_KEY,
        ServicePlugin.class);
  } catch (RuntimeException e) {
    String pluginsValue = conf.get(DFS_NAMENODE_PLUGINS_KEY);
    LOG.error("Unable to load NameNode plugins. Specified list of plugins: " +
        pluginsValue, e);
    throw e;
  }
  // 启动所有插件
  for (ServicePlugin p: plugins) {
    try {
      // 调用插件的start接口，需要插件自己实现，需要实现接口ServicePlugin
      p.start(this);
    } catch (Throwable t) {
      LOG.warn("ServicePlugin " + p + " could not be started", t);
    }
  }
  LOG.info(getRole() + " RPC up at: " + getNameNodeAddress());
  if (rpcServer.getServiceRpcAddress() != null) {
    LOG.info(getRole() + " service RPC up at: "
        + rpcServer.getServiceRpcAddress());
  }
}
```

### namesystem.startCommonServices 

在当前函数中启动blockManager和NameNodeResourceChecker，blockManager比较关键。

```java 
void startCommonServices(Configuration conf, HAContext haContext) throws IOException {
  this.registerMBean(); // register the MBean for the FSNamesystemState
  writeLock();
  this.haContext = haContext;
  try {
    //创建NameNodeResourceChecker，并立即检查一次
    nnResourceChecker = new NameNodeResourceChecker(conf);
    checkAvailableResources();
    assert !blockManager.isPopulatingReplQueues();
    StartupProgress prog = NameNode.getStartupProgress();
    prog.beginPhase(Phase.SAFEMODE);
    //获取已完成的数据块总量
    long completeBlocksTotal = getCompleteBlocksTotal();
    prog.setTotal(Phase.SAFEMODE, STEP_AWAITING_REPORTED_BLOCKS,
        completeBlocksTotal);
    // 激活blockManager，blockManager负责管理文件系统中文件的物理块与实际存储位置的映射关系，
    // 是NameNode的核心功能之一。
    blockManager.activate(conf, completeBlocksTotal);
  } finally {
    writeUnlock("startCommonServices");
  }
  
  registerMXBean();
  DefaultMetricsSystem.instance().register(this);
  if (inodeAttributeProvider != null) {
    inodeAttributeProvider.start();
    dir.setINodeAttributeProvider(inodeAttributeProvider);
  }
  // 注册快照管理器
  snapshotManager.registerMXBean();
  InetSocketAddress serviceAddress = NameNode.getServiceAddress(conf, true);
  this.nameNodeHostName = (serviceAddress != null) ? serviceAddress.getHostName() : "";
}
```

### blockManager.activate

启动blockManager.activate 主要是初始化blockManager。

主要包含下面几个方面:
- pendingReconstruction
- datanodeManager 
- bmSafeMode 

```java 
public void activate(Configuration conf, long blockTotal) {
  pendingReconstruction.start();
  // 初始化datanodeManager
  datanodeManager.activate(conf);
  this.redundancyThread.setName("RedundancyMonitor");
  this.redundancyThread.start();
  this.markedDeleteBlockScrubberThread.setName("MarkedDeleteBlockScrubberThread");
  this.markedDeleteBlockScrubberThread.start();
  this.blockReportThread.start();
  mxBeanName = MBeans.register("NameNode", "BlockStats", this);
  bmSafeMode.activate(blockTotal);
}
```


详细参见：

![pic](https://pan.zeekling.cn/zeekling/hadoop/nn_0010.png)

