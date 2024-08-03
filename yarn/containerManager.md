
# 简介
ContainerManager主要负责NM中管理所有Container生命周期，其主要包含启动Container、恢复Container、停止Container等功能。
主要功能由ContainerManagerImpl类实现，具体代码可以参考当前类。

# 初始化

初始化主要分为两部分：

ContainerManagerImpl实例的构造函数和serviceInit函数。

## 构造函数
当前函数为构造函数，主要初始化必须要的一些变量等。

- dispatcher ： 事件的中央调度器，主要用于管理单个Container的生命周期。在当前函数里面会通过`dispatcher.register()`注册支持的事件。

- containersLauncher： 主要用于启动Container，可以通过配置项yarn.nodemanager.containers-launcher.class指定。默认为containersLauncher.class


## serviceInit函数

主要是服务启动时的初始化函数，ContainerManager在NodeManager内部属于一个服务。所以初始化的时候会调用这个函数初始化一些服务相关的东西。

在这个函数里面总结下来主要做了几件事：
- 继续初始化一些事件，主要包含LogHandlerEventType、SharedCacheUploadEventType。
- 初始化AMRMProxyService。
- 从本地的LevelDB恢复Container信息。

### 恢复当前NodeManager的所有作业信息


#### 第一步：恢复Application 信息

首先是从LevelDB里面加载Application信息。循环加载。

```java
RecoveredApplicationsState appsState = stateStore.loadApplicationsState();
 try (RecoveryIterator<ContainerManagerApplicationProto> rasIterator =
          appsState.getIterator()) {
   while (rasIterator.hasNext()) {
     ContainerManagerApplicationProto proto = rasIterator.next();
     LOG.debug("Recovering application with state: {}", proto);
     recoverApplication(proto);
   }
 }
```

加载Application的时候会将Application的上下文信息从LevelDB里面读出来，通过上下文信息等初始化新的ApplicationImpl，并且触发ApplicationInitEvent事件。
会根据当前作业上下文中实际的状态等信息跳转到实际的状态。

```java
ApplicationImpl app = new ApplicationImpl(dispatcher, p.getUser(), fc,
    appId, creds, context, p.getAppLogAggregationInitedTime());
context.getApplications().put(appId, app);
metrics.runningApplication();
app.handle(new ApplicationInitEvent(appId, acls, logAggregationContext));
```

#### 第二步：恢复所有的Container信息

第二步是从LevelDB里面加载Container信息。循环加载。

```java
try (RecoveryIterator<RecoveredContainerState> rcsIterator =
          stateStore.getContainerStateIterator()) {
   while (rcsIterator.hasNext()) {
     RecoveredContainerState rcs = rcsIterator.next();
     LOG.debug("Recovering container with state: {}", rcs);
     recoverContainer(rcs);
   }
 }
```

recoverContainer函数用于恢复单个Container信息。对于已经存在的Application对应的Container会通过LevelDB里面加载到的信息初始化Container对象，
将其加到所有Container的列表里面并且触发ApplicationContainerInitEvent，后续会根据实际状态信息跳转到指定状态继续处理。

```java
Container container = new ContainerImpl(getConfig(), dispatcher,
    launchContext, credentials, metrics, token, context, rcs);
context.getContainers().put(token.getContainerID(), container);
containerScheduler.recoverActiveContainer(container, rcs);
app.handle(new ApplicationContainerInitEvent(container));
```

如果发现作业的状态为KILL状态，则会为当前Container重新触发KILL事件，保证Container已经停止。

对于Application找不见的Container，认为作业已经结束了，直接标记为已经完成。


在恢复完成之后会触发事件： ContainerSchedulerEventType.RECOVERY_COMPLETED

此状态会重新拉起所有的Container。


# 启动Containers 

## 获取NMToken

在Container启动之前需要获取NMToken,可以通过下面命令获取，一般情况下获取第一个NMTokenIdentifier类型的Token。

```java
Set<TokenIdentifier> tokenIdentifiers = remoteUgi.getTokenIdentifiers();
```

## 开始启动Container 

启动之前需要做的就是初始化ContainerImpl信息，方便后续启动Container。

```java
Container container =
    new ContainerImpl(getConfig(), this.dispatcher,
        launchContext, credentials, metrics, containerTokenIdentifier,
        context, containerStartTime);

```

如果是第一次启动(满足条件：`!context.getApplications().containsKey(applicationID))`，也就是AM，会通过下面命令触发作业的启动：

```java
context.getNMStateStore().storeApplication(applicationID,
    buildAppProto(applicationID, user, credentials, appAcls,
        logAggregationContext, flowContext));
dispatcher.getEventHandler().handle(new ApplicationInitEvent(
    applicationID, appAcls, logAggregationContext));
```

满足下面条件则是恢复Application：

`containerTokenIdentifier.getContainerType() == ContainerType.APPLICATION_MASTER && context.getApplications().containsKey(applicationID))`

启动Container主要是触发Container的Init事件：

```java
this.context.getNMStateStore().storeContainer(containerId,
    containerTokenIdentifier.getVersion(), containerStartTime, request);
dispatcher.getEventHandler().handle(
  new ApplicationContainerInitEvent(container));
```

## Container事件处理



