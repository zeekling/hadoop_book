# 简介

distributedShell是Yarn自带的应用程序，和MR类似，当前工具可以用来对Yarn进行压测。

# 使用示例

参考命令如下：

```java
./bin/hadoop jar ./share/hadoop/yarn/hadoop-yarn-applications-distributedshell-3.4.1.jar  -jar ./share/hadoop/yarn/hadoop-yarn-applications-distributedshell-3.4.1.jar  -shell_command '/bin/date' -num_containers 5
```

可以提交一个样例作业到Yarn上面。

# 源码阅读

当前样例的入口类是`org.apache.hadoop.yarn.applications.distributedshell.Client` ，在pom文件里面默认定义了当前类为主类。所以在提交的时候可以不用指定主类。

```xml
 <plugin>
        <artifactId>maven-jar-plugin</artifactId>
        <executions>
         <!-- 省略部分参数 -->
        </executions>
        <configuration>
           <archive>
             <manifest>
               <mainClass>org.apache.hadoop.yarn.applications.distributedshell.Client</mainClass>
             </manifest>
           </archive>
        </configuration>
      </plugin>
```

核心流程主要包含下面3个：

- 初始化CLient对象

- 初始化Client

- 提交作业到yarn

其中前面两个主要在客户端，第3个主要是在yarn上面。

## 客户端提交核心代码



### 初始化

初始化阶段包括下面两部分：

- 初始化Client对象，主要是创建Yarn的连接以及初始化支持的参数列表

- 初始化Client

下面是初始化Client对象的核心代码。

```java
Client(String appMasterMainClass, Configuration conf) {
    this.conf = conf;
    this.conf.setBoolean(
        YarnConfiguration.YARN_CLIENT_LOAD_RESOURCETYPES_FROM_SERVER, true);
    this.appMasterMainClass = appMasterMainClass;
    // 创建和RM的连接
    yarnClient = YarnClient.createYarnClient();
    yarnClient.init(conf);
    opts = new Options();
    // 初始化支持的参数列表
    stopSignalReceived = new AtomicBoolean(false);
    isRunning = new AtomicBoolean(false);
  }
```

初始化Client，在初始化Client阶段主要是读取命令行参数。

```java
// 初始化Client函数入口
boolean doRun = client.init(args);
```



### 运行作业

首先还是建立和Yarn服务端的连接，为作业提交做准备。

```java
 isRunning.set(true);
 yarnClient.start();
```

在连接建立之后会查询并且在控制台打印Yarn服务端的一些信息。主要包含下面内容：

- 当前集群NM的个数，通过`yarnClient.getYarnClusterMetrics()` 查询到并且显示。

- 当前集群中运行中NM的详细信息，通过`yarnClient.getNodeReports(NodeState.RUNNING)`查询到。

- 当前任务提交的队列的详细信息，通过`yarnClient.getQueueInfo(this.amQueue)`查询到。

- 当前集群的ACL信息，通过`yarnClient.getQueueAclsInfo()`查询。

- 当前集群的ResourceProfile信息，通过`yarnClient.getResourceProfiles()`查询。



在打印完集群信息之后才是作业提交的开始。



提交作业之前，是需要先向RM申请AppId的。AppId可以通过`YarnClientApplication app = yarnClient.createApplication();`获取。作业提交信息一般都在ApplicationSubmissionContext里面，包含下面信息：

- AM申请资源的请求。通过`appContext.setAMContainerResourceRequests(amResourceRequests);`设置。

- AM的上下文信息：
  
  - 访问hdfs等所需要的token。当前token会伴随着整个作业，直到作业结束才会异步销毁。
  
  - 需要本地话的文件。
  
  - AM或者Container所需要的环境变量。
  
  - AM的启动命令,AM启动的类也是在这里指定的。类似于 java运行jar或者某个主类。

- App名称。通过`appContext.setApplicationName(appName);`设置。

- app tag信息。

- 资源标签信息。

- 作业的优先级。

- 作业提交的队列信息。

- 日志聚合相关配置。主要是和日志归集的Rolling模式有关系。可以设置需要通过rolling的方式归集哪些日志。通过`appContext.setLogAggregationContext(logAggregationContext);`设置。



作业真正提交的代码只有一行：

```java
yarnClient.submitApplication(appContext);
```

当前样例做到了作业所需要的信息可配置。是一个比较适合开发作业的样例。

## AM核心代码

AM的核心代码是在ApplicationMaster.java里面的。在启动AM的时候会调用到当前函数的main函数。

在构造函数里面和init函数里面，主要是加载配置项以及命令行参数。真正运行的函数是run，核心在run函数里面，

首先需要创建和RM以及NM的连接。

```java
amRMClient = AMRMClientAsync.createAMRMClientAsync(1000, allocListener);
amRMClient.init(conf);
amRMClient.start();

containerListener = createNMCallbackHandler();
nmClientAsync = new NMClientAsyncImpl(containerListener);
nmClientAsync.init(conf);
nmClientAsync.start();  
startTimelineClient(conf);
```

在AM启动OK了第一件事就是需要去RM上面注册，证明当前AM已经启动完成了。

```java
RegisterApplicationMasterResponse response = amRMClient
        .registerApplicationMaster(appMasterHostname, appMasterRpcPort,
            appMasterTrackingUrl, placementConstraintMap);
```

普通Container的申请是在AM里面处理的，类似下面代码，下面代码是异步申请的。

```java
ContainerRequest containerAsk = setupContainerAskForRM();
amRMClient.addContainerRequest(containerAsk);
```

当Container申请好之后，可以通过下面代码获取，在样例中触发onContainerAllocated事件。

```java
List<Container> allocated = response.getAllocatedContainers();
if (!allocated.isEmpty()) {
    handler.onContainersAllocated(allocated);
}
```

通过下面代码启动Container.

```java
ContainerLaunchContext ctx = ContainerLaunchContext.newInstance(
        localResources, myShellEnv, commands, null, allTokens.duplicate(),
          null, containerRetryContext); 
nmClientAsync.startContainerAsync(container, ctx);
```



在作业结束的时候，AM需要做下面事：

- 停止nmClient。

- 从RM上取消AppMaster

- 停止amClient。

```java
nmClientAsync.stop();
try {
    amRMClient.unregisterApplicationMaster(appStatus, message, null);
} catch (YarnException | IOException ex) {
    LOG.error("Failed to unregister application", ex);
}
amRMClient.stop();
```






