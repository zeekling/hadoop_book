
## NameNode客户端协议详解

协议的定义主要在类NamenodeProtocols中。如下：

```java
public interface NamenodeProtocols
  extends ClientProtocol,
          DatanodeProtocol,
          DatanodeLifelineProtocol,
          NamenodeProtocol,
          RefreshAuthorizationPolicyProtocol,
          ReconfigurationProtocol,
          RefreshUserMappingsProtocol,
          RefreshCallQueueProtocol,
          GenericRefreshProtocol,
          GetUserMappingsProtocol,
          HAServiceProtocol {
}
```

根据交互对象的不同，将协议进行了不同的归类。要想了解协议内容，需要将其单独分开分析。

### NamenodeProtocol 详解

```java
BlocksWithLocations getBlocks(DatanodeInfo datanode, long size, long
      minBlockSize) throws IOException;
```
当前协议主要是备NameNode和主NameNode之间的通信协议。

获取指定DataNode中的块信息。
- size： 请求的块数量。
- minBlockSize： 查询的block块需要小于当前值。

```java
public ExportedBlockKeys getBlockKeys() throws IOException;
```
获取NameNode产生的所有的blockkey信息。blockKey是由BlockTokenSecretManager产生的，BlockTokenSecretManager有两种模式：master模式和worker模式。
master主要产生token，并且将token导入给workers。master和worker都可以校验token。一般情况下，NN是master模式，DN是worker模式。主要用于加密。

```java
public long getTransactionID() throws IOException;
```

获取最新的事务ID。

### DatanodeProtocol

DataNode和NameNode之间的协议。

### DatanodeLifelineProtocol

DN和NN之间心跳协议。


