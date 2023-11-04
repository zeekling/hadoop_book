
## 简介

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

### 启动

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
