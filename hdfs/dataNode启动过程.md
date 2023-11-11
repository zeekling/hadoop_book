
# 启动

## createDataNode

DataNode核心启动类是从createDataNode开始的。

```java
public static DataNode createDataNode(String args[], Configuration conf,
    SecureResources resources) throws IOException {
  // 初始化DataNode,核心函数在makeInstance里面的new DataNode()进行初始化。
  DataNode dn = instantiateDataNode(args, conf, resources);
  if (dn != null) {
    // 启动DataNode
    dn.runDatanodeDaemon();
  }
  return dn;
}
```

### makeInstance 

初始化DataNode实例。

```java 
static DataNode makeInstance(Collection<StorageLocation> dataDirs,
    Configuration conf, SecureResources resources) throws IOException {
  List<StorageLocation> locations;
  StorageLocationChecker storageLocationChecker =
      new StorageLocationChecker(conf, new Timer());
  try {
    // 对存储路径进行检查。返回可用的位置。
    locations = storageLocationChecker.check(conf, dataDirs);
  } catch (InterruptedException ie) {
    throw new IOException("Failed to instantiate DataNode", ie);
  }
  DefaultMetricsSystem.initialize("DataNode");

  assert locations.size() > 0 : "number of data directories should be > 0";
  // 初始化DataNode实例。在构造函数里面startDataNode比较重要，是启动DataNode的核心函数，需要重点关注。
  return new DataNode(conf, locations, storageLocationChecker, resources);
}
```

### startDataNode 


```java 
void startDataNode(List<StorageLocation> dataDirectories,
                   SecureResources resources
                   ) throws IOException {

  // settings global for all BPs in the Data Node
  this.secureResources = resources;
  synchronized (this) {
    this.dataDirs = dataDirectories;
  }
  this.dnConf = new DNConf(this);
  checkSecureConfig(dnConf, getConf(), resources);
  // ... 省略 ....

  storage = new DataStorage();
  
  // global DN settings
  registerMXBean();
  // 初始化dataXceiver服务：用于DN接受客户端和其他DN发送过来的数据的服务。
  initDataXceiver();
  // 初始化HTTP服务。
  startInfoServer();
  //TODO 待定。
  pauseMonitor = new JvmPauseMonitor();
  pauseMonitor.init(getConf());
  pauseMonitor.start();

  // BlockPoolTokenSecretManager is required to create ipc server.
  this.blockPoolTokenSecretManager = new BlockPoolTokenSecretManager();

  // Login is done by now. Set the DN user name.
  dnUserName = UserGroupInformation.getCurrentUser().getUserName();
  LOG.info("dnUserName = {}", dnUserName);
  LOG.info("supergroup = {}", supergroup);
  // 初始化DN的RPC调用服务。
  initIpcServer();

  metrics = DataNodeMetrics.create(getConf(), getDisplayName());
  peerMetrics = dnConf.peerStatsEnabled ?
      DataNodePeerMetrics.create(getDisplayName(), getConf()) : null;
  metrics.getJvmMetrics().setPauseMonitor(pauseMonitor);

  ecWorker = new ErasureCodingWorker(getConf(), this);
  blockRecoveryWorker = new BlockRecoveryWorker(this);

  blockPoolManager = new BlockPoolManager(this);
  // DN向NN注册，其中核心的函数为doRefreshNamenodes
  blockPoolManager.refreshNamenodes(getConf());

  // Create the ReadaheadPool from the DataNode context so we can
  // exit without having to explicitly shutdown its thread pool.
  readaheadPool = ReadaheadPool.getInstance();
  saslClient = new SaslDataTransferClient(dnConf.getConf(),
      dnConf.saslPropsResolver, dnConf.trustedChannelResolver);
  saslServer = new SaslDataTransferServer(dnConf, blockPoolTokenSecretManager);
  startMetricsLogger();

  if (dnConf.diskStatsEnabled) {
    diskMetrics = new DataNodeDiskMetrics(this,
        dnConf.outliersReportIntervalMs, getConf());
  }
}
```

### doRefreshNamenodes

主要是向NN注册当前DN。


```java 
private void doRefreshNamenodes(
    Map<String, Map<String, InetSocketAddress>> addrMap,
    Map<String, Map<String, InetSocketAddress>> lifelineAddrMap)
    throws IOException {

  Set<String> toRefresh = Sets.newLinkedHashSet();
  Set<String> toAdd = Sets.newLinkedHashSet();
  Set<String> toRemove;
  
  synchronized (this) {
    // Step 1. For each of the new nameservices, figure out whether
    // it's an update of the set of NNs for an existing NS,
    // or an entirely new nameservice.
    // 判断当前DN中的nameservice是否已经存在：如果已经存在则需要刷新；否则需要添加。
    for (String nameserviceId : addrMap.keySet()) {
      if (bpByNameserviceId.containsKey(nameserviceId)) {
        toRefresh.add(nameserviceId);
      } else {
        toAdd.add(nameserviceId);
      }
    }
    
    // Step 2. Any nameservices we currently have but are no longer present
    // need to be removed.
    // 计算当前DN需要删除的nameservice
    toRemove = Sets.newHashSet(Sets.difference(
        bpByNameserviceId.keySet(), addrMap.keySet()));
    
    // Step 3. 启动新的nameservice
    if (!toAdd.isEmpty()) {
    
      for (String nsToAdd : toAdd) {
        Map<String, InetSocketAddress> nnIdToAddr = addrMap.get(nsToAdd);
        Map<String, InetSocketAddress> nnIdToLifelineAddr =
            lifelineAddrMap.get(nsToAdd);
        ArrayList<InetSocketAddress> addrs =
            Lists.newArrayListWithCapacity(nnIdToAddr.size());
        ArrayList<String> nnIds =
            Lists.newArrayListWithCapacity(nnIdToAddr.size());
        ArrayList<InetSocketAddress> lifelineAddrs =
            Lists.newArrayListWithCapacity(nnIdToAddr.size());
        for (String nnId : nnIdToAddr.keySet()) {
          addrs.add(nnIdToAddr.get(nnId));
          nnIds.add(nnId);
          lifelineAddrs.add(nnIdToLifelineAddr != null ?
              nnIdToLifelineAddr.get(nnId) : null);
        }
        BPOfferService bpos = createBPOS(nsToAdd, nnIds, addrs,
            lifelineAddrs);
        bpByNameserviceId.put(nsToAdd, bpos);
        offerServices.add(bpos);
      }
    }
    startAll();
  }

  // Step 4. 停止掉老的nameservice. This happens outside
  // of the synchronized(this) lock since they need to call
  // back to .remove() from another thread
  if (!toRemove.isEmpty()) {
    
    for (String nsToRemove : toRemove) {
      BPOfferService bpos = bpByNameserviceId.get(nsToRemove);
      bpos.stop();
      bpos.join();
      // they will call remove on their own
    }
  }
  
  // Step 5. Update nameservices whose NN list has changed 
  // 更新nameservices
  if (!toRefresh.isEmpty()) {
    
    for (String nsToRefresh : toRefresh) {
      BPOfferService bpos = bpByNameserviceId.get(nsToRefresh);
      Map<String, InetSocketAddress> nnIdToAddr = addrMap.get(nsToRefresh);
      Map<String, InetSocketAddress> nnIdToLifelineAddr =
          lifelineAddrMap.get(nsToRefresh);
      ArrayList<InetSocketAddress> addrs =
          Lists.newArrayListWithCapacity(nnIdToAddr.size());
      ArrayList<InetSocketAddress> lifelineAddrs =
          Lists.newArrayListWithCapacity(nnIdToAddr.size());
      ArrayList<String> nnIds = Lists.newArrayListWithCapacity(
          nnIdToAddr.size());
      for (String nnId : nnIdToAddr.keySet()) {
        addrs.add(nnIdToAddr.get(nnId));
        lifelineAddrs.add(nnIdToLifelineAddr != null ?
            nnIdToLifelineAddr.get(nnId) : null);
        nnIds.add(nnId);
      }
      try {
        UserGroupInformation.getLoginUser()
            .doAs(new PrivilegedExceptionAction<Object>() {
              @Override
              public Object run() throws Exception {
                // 刷新NNList 核心实现.
                bpos.refreshNNList(nsToRefresh, nnIds, addrs, lifelineAddrs);
                return null;
              }
            });
      } catch (InterruptedException ex) {
        IOException ioe = new IOException();
        ioe.initCause(ex.getCause());
        throw ioe;
      }
    }
  }
}
```


