
# 简介
BPServiceActor 主要在DataNode中用于和NameNode沟通的类。主要功能如下：
- 与 namenode 进行预注册握手。
- 向 namenode 注册。
- 定期向 namenode 发送心跳。
- 处理从 namenode 收到的命令。


# 核心功能

## 预注册


## 注册DataNode

### DataNode注册核心逻辑

注册DataNode的核心函数是register

```java
void register(NamespaceInfo nsInfo) throws IOException {
    // The handshake() phase loaded the block pool storage
    // off disk - so update the bpRegistration object from that info
    DatanodeRegistration newBpRegistration = bpos.createRegistration();

    LOG.info(this + " beginning handshake with NN");

    while (shouldRun()) {
        try {
            // 向NN注册DataNodde
            newBpRegistration = bpNamenode.registerDatanode(newBpRegistration);
            newBpRegistration.setNamespaceInfo(nsInfo);
            bpRegistration = newBpRegistration;
            break;
            // ... 省略部分异常处理
        }
    }

    if (bpRegistration == null) {
        throw new IOException("DN shut down before block pool registered");
    }

    // .. 省略 ..
    // reset lease id whenever registered to NN.
    // ask for a new lease id at the next heartbeat.
    fullBlockReportLeaseId = 0;

    // random short delay - helps scatter the BR from all DatanodeRegistration 
    // 定时汇报block块信息给NN
    scheduler.scheduleBlockReport(dnConf.initialBlockReportDelayMs, true);
}
```

### NameNode 处理注册信息

NameNode处理DataNode的注册信息，主要接口是在NameNodeRpcServer的ipc接口registerDatanode。
其主要的实现实在DatanodeManager中的函数registerDatanode。


主要实现如下：

```java
// update cluster map
getNetworkTopology().remove(nodeS);
if(shouldCountVersion(nodeS)) {
    decrementVersionCount(nodeS.getSoftwareVersion());
}
nodeS.updateRegInfo(nodeReg);

nodeS.setSoftwareVersion(nodeReg.getSoftwareVersion());
nodeS.setDisallowed(false); // Node is in the include list

// resolve network location
if(this.rejectUnresolvedTopologyDN) {
    nodeS.setNetworkLocation(resolveNetworkLocation(nodeS));
    nodeS.setDependentHostNames(getNetworkDependencies(nodeS));
} else {
    nodeS.setNetworkLocation(
            resolveNetworkLocationWithFallBackToDefaultLocation(nodeS));
    nodeS.setDependentHostNames(
            getNetworkDependenciesWithDefault(nodeS));
}
getNetworkTopology().add(nodeS);
resolveUpgradeDomain(nodeS);

// also treat the registration message as a heartbeat
heartbeatManager.register(nodeS);
incrementVersionCount(nodeS.getSoftwareVersion());
startAdminOperationIfNecessary(nodeS);
success = true;

```

主要做了下面两件事：
- 更新节点信息，比如网络拓扑信息。
- 在心跳管理里面注册当前节点。只要将节点信息添加到节点列表里面即可。


## 发送心跳


## 处理NN的命令



