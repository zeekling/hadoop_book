# 简介

# 启动源码分析

Zookeeper启动的主类为QuorumPeerMain.java 。入口函数文为initializeAndRun，如下所示，在往下的核心函数为runFromConfig。

```java
 QuorumPeerMain main = new QuorumPeerMain();
 try {
     main.initializeAndRun(args);
} catch (IllegalArgumentException e) {
// 启动异常处理。
 }
LOG.info("Exiting normally");
ServiceUtils.requestSystemExit(ExitCode.EXECUTION_FINISHED.getValue());
```

runFromConfig函数里面主要做了下面几件事：

- 初始化log4j相关的jmx。

- 初始化监控相关组件。

- 初始化认证相关组件。

- 设置基础配置信息。

- 启动Zookeeper。由`quorumPeer.start();` 开始。相关的类为：QuorumPeer.java

```java
public void runFromConfig(QuorumPeerConfig config) throws IOException, AdminServerException {
        try {
            // 注册和log4j相关的jmx监控
            ManagedUtil.registerLog4jMBeans();
        } catch (JMException e) {
            LOG.warn("Unable to register log4j JMX control", e);
        }

        LOG.info("Starting quorum peer, myid=" + config.getServerId());
        final MetricsProvider metricsProvider;
        try {
            metricsProvider = MetricsProviderBootstrap.startMetricsProvider(
                config.getMetricsProviderClassName(),
                config.getMetricsProviderConfiguration());
        } catch (MetricsProviderLifeCycleException error) {
            throw new IOException("Cannot boot MetricsProvider " + config.getMetricsProviderClassName(), error);
        }
        try {
            // 初始化监控相关
            ServerMetrics.metricsProviderInitialized(metricsProvider);
            // 初始化认证相关信息
            ProviderRegistry.initialize();
            // 省略部分

            quorumPeer = getQuorumPeer();
            // 设置基础配置文件
            quorumPeer.setTxnFactory(new FileTxnSnapLog(config.getDataLogDir(), config.getDataDir()));
            quorumPeer.enableLocalSessions(config.areLocalSessionsEnabled());
            quorumPeer.enableLocalSessionsUpgrading(config.isLocalSessionsUpgradingEnabled());
            // 省略部分

            // sets quorum sasl authentication configurations
            quorumPeer.setQuorumSaslEnabled(config.quorumEnableSasl);
            if (quorumPeer.isQuorumSaslAuthEnabled()) {
                // 开启sasl之后，设置相关参数
                quorumPeer.setQuorumServerSaslRequired(config.quorumServerRequireSasl);
                quorumPeer.setQuorumLearnerSaslRequired(config.quorumLearnerRequireSasl);
                quorumPeer.setQuorumServicePrincipal(config.quorumServicePrincipal);
                quorumPeer.setQuorumServerLoginContext(config.quorumServerLoginContext);
                quorumPeer.setQuorumLearnerLoginContext(config.quorumLearnerLoginContext);
            }
            quorumPeer.setQuorumCnxnThreadsSize(config.quorumCnxnThreadsSize);
            quorumPeer.initialize();

            if (config.jvmPauseMonitorToRun) {
                quorumPeer.setJvmPauseMonitor(new JvmPauseMonitor(config));
            }

            // 开始启动
            quorumPeer.start();
            ZKAuditProvider.addZKStartStopAuditLog();
            quorumPeer.join();
        } catch (InterruptedException e) {
            // warn, but generally this is ok
            LOG.warn("Quorum Peer interrupted", e);
        } finally {
            try {
                metricsProvider.stop();
            } catch (Throwable error) {
                LOG.warn("Error while stopping metrics", error);
            }
        }
    }
```

真正启动的函数为QuorumPeer的start函数。主要做了下面事：

- 加载数据，包括log文件里面和snapshot里面的数据，在数据量较大的情况下，当前步骤可能比较慢。

- 启动管理服务，主要用于管理Zookeeper服务端。主要实现方式包含：
  
  - JettyAdminServer：提供http方式的servier，通过CommandServlet实现管理接口，主要是四字命令。
  - DummyAdminServer：实际上就是啥也没有，不只是管理的意思。

- 开始参与选举Leader。

- 启动JVM 延时检测线程。

```java
 public synchronized void start() {
        if (!getView().containsKey(myid)) {
            throw new RuntimeException("My id " + myid + " not in the peer list");
        }
        // 加载数据，当前步骤可能比较慢。
        loadDataBase();
        startServerCnxnFactory();
        try {
            adminServer.start();
        } catch (AdminServerException e) {
            LOG.warn("Problem starting AdminServer", e);
        }
        // 开始Leader选举
        startLeaderElection();
        startJvmPauseMonitor();
        super.start();
    }
```

zookeeper数据加载主要通过ZKDatabase.java实现。加载数据的入口函数为loadDataBase。核心还是FileTxnSnapLog.restore函数。

```java
    public long loadDataBase() throws IOException {
        long startTime = Time.currentElapsedTime();
        // 加载snapshot文件和log文件
        long zxid = snapLog.restore(dataTree, sessionsWithTimeouts, commitProposalPlaybackListener);
        initialized = true;
        long loadTime = Time.currentElapsedTime() - startTime;
        ServerMetrics.getMetrics().DB_INIT_TIME.add(loadTime);
        LOG.info("Snapshot loaded in {} ms, highest zxid is 0x{}, digest is {}",
                loadTime, Long.toHexString(zxid), dataTree.getTreeDigest());
        return zxid;
    }
```


