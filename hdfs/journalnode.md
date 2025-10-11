# 简介
Active Namenode与StandBy Namenode之间的绿色区域就是JournalNode,当然数量不一定只有1个,作用相当于NFS共享文件系统.Active Namenode往里写editlog数据,StandBy再从里面读取数据进行同步.

JournalNodede 在hdfs架构中的角色：

![pic](https://pan.zeekling.cn/zeekling/hadoop/jn/jn_01,png)


# 源码解析

解读JournalNodede的原理。JN的核心功能主要包含下面几个：
- JN启动
- 读写editLog。
- JN之间editLog数据同步


## JN 启动
JN的启动入口类是JournalNode.java， 启动函数是main。在启动阶段主要启动了两个核心部件：
- JournalNodeHttpServer，主要是jn的http服务端。读取editlog使用。
- JournalNodeRpcServer， 主要是jn的rpc服务端。主要是写入editlog使用当前协议。

### JournalNodeHttpServer
当前类主要是jn的http服务端，在启动阶段最关键的功能是初始化http的Servlet：GetJournalEditServlet。
```java
httpServer.addInternalServlet("getJournal", "/getJournal", GetJournalEditServlet.class, true);
httpServer.start();
```
核心是读取editLog，没有其他特别的功能。

### JournalNodeRpcServer
当前类主要是jn的rpc服务端，核心在于启动关键的rpc服务，通过下面函数初始化rpc服务端。
```java
 this.server = new RPC.Builder(confCopy)
    .setProtocol(QJournalProtocolPB.class)
    .setInstance(service)
    .setBindAddress(bindHost)
    .setPort(addr.getPort())
    .setNumHandlers(this.handlerCount)
    .setVerbose(false)
    .build();
```
启动服务端定义的rpc协议主要包含QJournalProtocol.java，核心函数例如：
```java
public interface QJournalProtocol {
  public static final long versionID = 1L;

  boolean isFormatted(String journalId,
                      String nameServiceId) throws IOException;

  GetJournalStateResponseProto getJournalState(String journalId,
                                               String nameServiceId)
      throws IOException;
  
  void format(String journalId, String nameServiceId,
      NamespaceInfo nsInfo, boolean force) throws IOException;

  NewEpochResponseProto newEpoch(String journalId,
                                        String nameServiceId,
                                        NamespaceInfo nsInfo,
                                        long epoch) throws IOException;
  
  public void journal(RequestInfo reqInfo,
                      long segmentTxId,
                      long firstTxnId,
                      int numTxns,
                      byte[] records) throws IOException;

  
  public void heartbeat(RequestInfo reqInfo) throws IOException;
  
  public void startLogSegment(RequestInfo reqInfo,
      long txid, int layoutVersion) throws IOException;

  public void finalizeLogSegment(RequestInfo reqInfo,
      long startTxId, long endTxId) throws IOException;

  public void purgeLogsOlderThan(RequestInfo requestInfo, long minTxIdToKeep)
      throws IOException;
  
  GetEditLogManifestResponseProto getEditLogManifest(String jid,
                                                     String nameServiceId,
                                                     long sinceTxId,
                                                     boolean inProgressOk)
      throws IOException;


  GetJournaledEditsResponseProto getJournaledEdits(String jid,
      String nameServiceId, long sinceTxId, int maxTxns) throws IOException;


  public PrepareRecoveryResponseProto prepareRecovery(RequestInfo reqInfo,
      long segmentTxId) throws IOException;

  public void acceptRecovery(RequestInfo reqInfo,
      SegmentStateProto stateToAccept, URL fromUrl) throws IOException;

  void doPreUpgrade(String journalId) throws IOException;

  public void doUpgrade(String journalId, StorageInfo sInfo) throws IOException;

  void doFinalize(String journalId,
                         String nameServiceid) throws IOException;

  Boolean canRollBack(String journalId, String nameServiceid,
                      StorageInfo storage, StorageInfo prevStorage,
                      int targetLayoutVersion) throws IOException;

  void doRollback(String journalId,
                         String nameServiceid) throws IOException;


  @Idempotent
  void discardSegments(String journalId,
                       String nameServiceId,
                       long startTxId)
      throws IOException;

  Long getJournalCTime(String journalId,
                       String nameServiceId) throws IOException;
}
```



## 读写editLog



## JN之间editLog数据同步


