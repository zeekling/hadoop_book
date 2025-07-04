# 简介

hdfs提供了一种除了通过rpc的方式进行文件操作的方式之外，还提供了http的方式对文件进行操作的方式：webhdfs。支持HDFS 的完整[FileSystem](https://hadoop.org.cn/docs/api/org/apache/hadoop/fs/FileSystem.html) / [FileContext](https://hadoop.org.cn/docs/api/org/apache/hadoop/fs/FileContext.html)接口。

其中Router和NameNode都支持了webhdfs的功能，具体实现有差别。



# 使用

## 文件系统URI与HTTP URL

WebHDFS的文件系统方案为“ webhdfs：// ”。WebHDFS文件系统URI具有以下格式。

```textile
webhdfs://<主机>:<HTTP_PORT>/<PATH>
```

上面的WebHDFS URI对应于下面的HDFS URI。

```textile
 hdfs://<主机>:<RPC_PORT>/<PATH>
```



在REST API中，在路径中插入前缀“ /webhdfs/v1 ”，并在末尾附加查询。因此，相应的HTTP URL具有以下格式。

```url
http://<主机>:<HTTP_PORT>/webhdfs/v1/<PATH>?op=create
```

详细可以参考：[https://hadoop.apache.org/docs/r3.4.1/hadoop-project-dist/hadoop-hdfs/WebHDFS.html](https://hadoop.apache.org/docs/r3.4.1/hadoop-project-dist/hadoop-hdfs/WebHDFS.html)



# 源码实现分析

## NameNode webhdfs 源码实现分析

### 启动和初始化

在[NameNode启动过程](./nameNode启动过程.md)中，启动NameNode的http模块的时候，启动了NameNode的webhdfs模块。核心入口函数(NameNodeHttpServer.java)：

```java
void start() throws IOException {
//...
initWebHdfs(conf, bindAddress.getHostName(), httpKeytab, httpServer,
        NamenodeWebHdfsMethods.class.getPackage().getName());
//...

}
```

从上面代码可以看出webhdfs的核心处理类为NamenodeWebHdfsMethods.java。当前类是每个请求都是由一个NamenodeWebHdfsMethods对象处理的，在处理每个请求的时候，需要做下面的初始化：

```java
 public NamenodeWebHdfsMethods(@Context HttpServletRequest request) {
    // the request object is a proxy to thread-locals so we have to extract
    // what we want from it since the external call will be processed in a
    // different thread.
    scheme = request.getScheme();
    userPrincipal = request.getUserPrincipal();
    // get the remote address, if coming in via a trusted proxy server then
    // the address with be that of the proxied client
    remoteAddr = JspHelper.getRemoteAddr(request);
    remotePort = JspHelper.getRemotePort(request);
    supportEZ =
        Boolean.valueOf(request.getHeader(WebHdfsFileSystem.EZ_HEADER));
  }
```



主要获取了当前登录的用户的相关信息，hdfs的nameService以及是否开启EC。



### 请求处理

NamenodeWebHdfsMethods里面定义的请求类型主要是：

- PUT请求：主要处理写入类型的求情。
  
  - CREATE操作。
  
  - MKDIRS操作。
  
  - CREATESYMLINK操作。
  
  - RENAME操作。
  
  - SETREPLICATION操作。
  
  - SETOWNER操作。
  
  - .....

- DELETE请求：主要处理删除类的请求。主要包含：
  
  - DELETE操作：
  
  - DELETESNAPSHOT操作：

- GET请求：主要处理查询类的请求。

- POST请求：主要处理写入类的请求。主要包含：
  
  - APPEND操作。
  
  - CONCAT操作。
  
  - TRUNCATE操作。
  
  - UNSETSTORAGEPOLICY操作。
  
  - UNSETECPOLICY操作。

定义参考如下：

```java
 @GET
  @Path("{" + UriFsPathParam.NAME + ":.*}")
  @Produces({MediaType.APPLICATION_OCTET_STREAM + "; " + JettyUtils.UTF_8,
      MediaType.APPLICATION_JSON + "; " + JettyUtils.UTF_8})
  public Response get(
      @Context final UserGroupInformation ugi,
      @QueryParam(DelegationParam.NAME) @DefaultValue(DelegationParam.DEFAULT)
          final DelegationParam delegation,
     //...
      ) throws IOException, InterruptedException {

    init(ugi, delegation, username, doAsUser, path, op, offset, length,
        renewer, bufferSize, xattrEncoding, excludeDatanodes, fsAction,
        snapshotName, oldSnapshotName, tokenKind, tokenService, startAfter);

    return doAs(ugi, new PrivilegedExceptionAction<Response>() {
      @Override
      public Response run() throws IOException, URISyntaxException {
        return get(ugi, delegation, username, doAsUser, path.getAbsolutePath(),
            op, offset, length, renewer, bufferSize, xattrNames, xattrEncoding,
            excludeDatanodes, fsAction, snapshotName, oldSnapshotName,
            tokenKind, tokenService, noredirect, startAfter);
      }
    });
  }
```



### webhdfs 操作实现

#### 1、 CREATE操作

当前操作主要是使用webhdfs上传文件的操作，核心操作在DN和NN上面都有。在NN里面的操作主要是选择合适的DN节点，然后跳转到DN上面进行文件上传。

核心处理流程如下：

![pic](https://pan.zeekling.cn/zeekling/hadoop/namenode/nn_webhdfs.jpg)

##### NN部分源码实现

入口位置代码如下，在  redirectURI里面主要是选择合适的DN，选择合适DN的代码在函数redirectURI里面的chooseDatanode函数，但最终还是有BlockManager提供结果。

```java
case CREATE:
    {
      final NameNode namenode = (NameNode)context.getAttribute("name.node");
      final URI uri = redirectURI(null, namenode, ugi, delegation, username,
          doAsUser, fullpath, op.getValue(), -1L, blockSize.getValue(conf),
          exclDatanodes.getValue(), permission, unmaskedPermission,
          overwrite, bufferSize, replication, blockSize, createParent,
          createFlagParam);
      if(!noredirectParam.getValue()) {
        return Response.temporaryRedirect(uri)
          .type(MediaType.APPLICATION_OCTET_STREAM).build();
      } else {
        final String js = JsonUtil.toJsonString("Location", uri);
        return Response.ok(js).type(MediaType.APPLICATION_JSON).build();
      }
    }
```

下面代码是为上传的文件选择块的逻辑的入口，选块的逻辑最后还是有BlockManager完成，核心函数为chooseTarget4WebHDFS。

```java
if (op == PutOpParam.Op.CREATE) {
      //choose a datanode near to client 
      final DatanodeDescriptor clientNode = bm.getDatanodeManager(
          ).getDatanodeByHost(remoteAddr);
      if (clientNode != null) {
        final DatanodeStorageInfo[] storages = bm.chooseTarget4WebHDFS(
            path, clientNode, excludes, blocksize);
        if (storages.length > 0) {
          return storages[0].getDatanodeDescriptor();
        }
      }
    }
```

 chooseTarget4WebHDF函数里面实现如下，由此可以看出webhdfs写入的是单副本。

```java
 /** Choose target for WebHDFS redirection. */
  public DatanodeStorageInfo[] chooseTarget4WebHDFS(String src,
      DatanodeDescriptor clientnode, Set<Node> excludes, long blocksize) {
    return placementPolicies.getPolicy(CONTIGUOUS).chooseTarget(src, 1,
        clientnode, Collections.<DatanodeStorageInfo>emptyList(), false,
        excludes, blocksize, storagePolicySuite.getDefaultPolicy(), null);
  }
```


