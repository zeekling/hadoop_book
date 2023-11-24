
# 简介

ResourceManager(RM)，RM是全局的资源管理器，负责整个系统的资源管理和分配。主要由以下两部分组成：

- 调度器：根据容量、队列限制条件将系统资源分配给各个应用。
  - 资源分配的单位是container，container是一个动态资源单位，它将内存、CPU、磁盘、网络等资源封装在一起，从而限定了资源使用量。
  - 调度器是一个可插拔的组件，用户可以自己定制，也可以选择Fair或Capacity调度器.
- 应用程序管理器：负责管理所有应用程序的以下内容：
  -  应用提交
  -  与调度器协商资源以启动AM.
  -  监控AM运行状态并在失败时重启它 


# RM内部架构 

- 交互模块：RM对普通用户、管理员、Web提供了三种对外服务：
  - ClientRMService:为普通用户提供服务，它处理来自客户端的各种RPC，比如:
    - 应用提交
    - 终止应用
    - 获取应用状态等
  - AdminService:为管理员提供的独立接口，主要目的是为了防止大量普通用户请求阻塞管理员通道，提供如下功能：
    - 动态更新节点列表
    - 更新ACL列表
    - 更新队列信息等
  - WebApp:提供一个Web界面来让用户更友好的获知集群和应用的状态 
- NM管理模块：用来管理NM的模块，主要包含以下三个组件： 
  - ResourceTrackerService:处理来自NodeManager的请求，主要包括：
    - 注册：注册是NM启动时发生的行为，NM提供的信息包括：
      - 节点ID、可用资源上限信息等. 
    - 心跳：心跳是周期行为
      - NM提供的信息包括：
        - 各个Container运行状态、运行的Application列表、节点健康状态等.
      - RM返回的信息包括：
        - 等待释放的Container列表、Application列表等.
  - NMLivelinessMonitor:监控NM是否活着，如果NM在一定时间(默认10m)内未上报心跳，则认为它死掉，需要移除.
  - NodesListManager:维护正常节点和异常节点列表，管理exclude(类似黑名单)和include(类似白名单)节点列表，
    这两个列表均是在配置文件中设置的，可以动态加载。
- AM管理模块：主要是用来管理所有AM，主要包括：
  - ApplicationMasterService(AMS):处理来自AM的请求，包括：
    - 注册：是AM启动时发生的行为，信息包括：
      - AM的启动节点、对外RPC端口、tracking URL等.
    - 心跳：是周期行为
      - AM提供的信息包括：所需资源的描述、待释放Container列表、黑名单列表等.
      - AMS返回的信息包括：新分配的Container、失败的Container、待抢占的Container列表等
  - AMLivelinessMonitor:监控AM是否活着，如果AM在一定时间(默认10m)内未上报心路，
    则认为它死掉，它上面正在运行的Container将会被置为失败状态，而AM本身会被分配到另一个节点上(用户可以指定重试次数，默认5)
  - ApplicationMasterLauncher：与某个NM通信，要求它为某个应用程序启动AM.
- 应用管理模块：主要是各个应用外围的管理，并不涉及到应用内部
  - ApplicationACLsManager:管理应用程序访问权限，包含两部分：
    - 查看权限：主要用于查看应用程序基本信息
    - 修改权限：主要用于修改应用程序优先级、杀死应用程序等
  - RMAppManager:管理应用程序的启动和关闭.
  - ContainerAllocationExpirer:当AM收到RM新分配的Container后，必须在一定时间(默认10m)内在对应的NM上启动该Container，
    否则RM将强制回收该Container，而一个已经分配的Container是否该被回收则是由ContainerAllocationExpirer决定和执行的
- 状态机管理模块：RM使用有限状态机维护有状态对象的生命周期，状态机的引入使得Yarn的架构设计清晰，RM内部的状态机有：
  - RMApp:维护一个应用程序的整个运行周期，包括从启动到运行结束的整个过程
    - 由于一个APP的生命周期可能会启动多个运行实例(Attempt)，RMApp维护的是所有的这些Attempt 
  - RMAppAttempt:一次应用程序的运行实例的整个生命周期，可以理解为APP的一次尝试运行 
  - RMContainer:一个Container的运行周期，包括从创建到运行结束的整个过程。
    - RM将资源封装成Container发送给应用程序的AM，AM在Container描述的运行环境中启动任务
    - Yarn不支持Container重用，一个Container用完后会立刻释放 
  - RMNode:维护了一个NM的生命周期，包括从启动到运行结束的整个过程 
- 安全模块：RM自带了非常全面的权限管理机制，主要包括：
  - ClientToAMSecretManager
  - ContainerTokenSecretManager 
  - ApplicationTokenSecretManager 
- 调度模块：主要包含一个组件ResourceScheduler。 
  - 资源调度器，它按照一定的约束条件(比如队列容量限制等)将集群中的资源分配给各个应用程序，目前主要考虑内存和CPU。
  - ResourceScheduler是一个可插拔式的模块，自带三个调度器，用户可以自己定制。
    - FIFO：先进先出，单用户。
    - Fair Scheduler:公平调度器(FairScheduler基本上具备其它两种的所有功能)
    - Capacity Scheduler:容量调度器

# RM事件与事件处理器 

Yarn采用了事件驱动机制，而RM是的实现则是最好的例证。所有服务和组件均是通过中央异步调度器组织在一起的，
不同组件之间通过事件交互，从而实现了一个异步并行的高效系统。

## 服务

|组件名称 | 输出事件类型| 用途 |
|-----|------|-------|
| ClientRMService | RMAppAttemptEvent <br> RMAppEvent <br>  RMNodeEvent | | 
| NMLivelinessMonitor | RMNodeEvent | |
| ResourceTrackerService | RMNodeEvent <br> RMAppAttemptEvent |  |
| AMLivelinessMonitor | RMAppAttemptEvent | |
| ContainerAllocationExpirer | SchedulerEvent | |


## 事件处理器

|组件名称 | 处理的事件类型 | 输出事件类型 | 用途 |
|-----|------|-------|-------|
| ApplicationMasterLauncher | AMLauncherEvent | -  | |
| RMAppManager | RMAppManagerEvent | RMAppEvent | | 
| NodesListManager | NodesListManagerEvent | RMNodeEvent <br> RMAppEvent | |
| RMApp | RMAppEvent | RMAppAttemptEvent <br> RMNodeEvent <br> SchedulerEvent <br> RMAppManagerEvent | |
| RMAppAttempt | RMAppAttemptEvent | SchedulerEvent <br> RMAppAttemptEvent <br> RMAppEvent <br> AMLauncherEvent <br> RMNodeEvent | |
| RMNode | RMNodeEvent | RMAppEvent <br> SchedulerEvent <br> NodesListManagerEvent <br> RMNodeEvent | |
| ResourceScheduler | SchedulerEvent | RMAppEvent <br> RMAppAttemptEvent | |
| RMContainer |  RMContainerEvent | RMAppEvent <br> RMAppAttemptEvent <br> RMNodeEvent | |

### 事件处理器实现类

- RMApp 实现类：
  - ApplicationEventDispatcher 
  - RMAppImpl

- RMAppAttempt 实现类
  - ApplicationAttemptEventDispatcher 
  - RMAppAttemptImpl 

- RMNode实现类
  - NodeEventDispatcher 
  - RMNodeImpl 
  
- ResourceScheduler实现类
  - EventDispatcher 
  - FairScheduler 

- RMContainer实现类
  - RMContainerImpl 

