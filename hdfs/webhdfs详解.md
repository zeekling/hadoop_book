## 简介

hdfs提供了一种除了通过rpc的方式进行文件操作的方式之外，还提供了http的方式对文件进行操作的方式：webhdfs。支持HDFS 的完整[FileSystem](https://hadoop.org.cn/docs/api/org/apache/hadoop/fs/FileSystem.html) / [FileContext](https://hadoop.org.cn/docs/api/org/apache/hadoop/fs/FileContext.html)接口。



## 使用

### 文件系统URI与HTTP URL

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



## 源码实现


