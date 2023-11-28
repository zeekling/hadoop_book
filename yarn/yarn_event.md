
# 简介

Yarn采用了基于事件驱动的并发模型：

- 所有状态机都实现了EventHandler接口，很多服务（类名通常带有Service后缀）也实现了该接口，它们都是事件处理器。
- 需要异步处理的事件由中央异步调度器（类名通常带有Dispatcher后缀）统一接收/派发，需要同步处理的事件直接交给相应的事件处理器。

![pic](https://pan.zeekling.cn/zeekling/hadoop/event/state_event_001.png)

某些事件处理器不仅处理事件，也会向中央异步调度器发送事件。


# 事件处理器定义

事件处理器定义如下：

```java
@SuppressWarnings("rawtypes")
@Public
@Evolving
public interface EventHandler<T extends Event> {

  void handle(T event);

}
```

只有一个handler函数，如参是事件：


# 中央处理器AsyncDispatcher 

AsyncDispatcher 实现了接口Dispatcher，Dispatcher中定义了事件Dispatcher的接口。主要提供两个功能：
- 注册不同类型的事件，主要包含事件类型和事件处理器。
- 获取事件处理器，用来派发事件，等待异步执行真正的EventHandler。

```java
@Public
@Evolving
public interface Dispatcher {

  EventHandler<Event> getEventHandler();

  void register(Class<? extends Enum> eventType, EventHandler handler);

}
```

AsyncDispatcher实现了Dispatcher接口，也扩展了AbstractService，表明AsyncDispatcher也是一个服务，
是一个典型的生产者消费这模型。

```java
public class AsyncDispatcher extends AbstractService implements Dispatcher {
 ...
}
```

# 事件处理器的注册

事件注册就是将事件写入到eventDispatchers里面，eventDispatchers的定义：`Map<Class<? extends Enum>, EventHandler> eventDispatchers`，键是事件类型，value是事件的处理器。

对于同一事件类型注册多次handler处理函数时，将使用MultiListenerHandler代替，MultiListenerHandler里面保存了多个handler，调用handler函数时，会依次调用每个handler。

```java 
public void register(Class<? extends Enum> eventType,
      EventHandler handler) {
    /* check to see if we have a listener registered */
    EventHandler<Event> registeredHandler = (EventHandler<Event>) eventDispatchers.get(eventType);
    LOG.info("Registering " + eventType + " for " + handler.getClass());
    if (registeredHandler == null) {
      eventDispatchers.put(eventType, handler);
    } else if (!(registeredHandler instanceof MultiListenerHandler)){
      /* for multiple listeners of an event add the multiple listener handler */
      MultiListenerHandler multiHandler = new MultiListenerHandler();
      multiHandler.addHandler(registeredHandler);
      multiHandler.addHandler(handler);
      eventDispatchers.put(eventType, multiHandler);
    } else {
      /* already a multilistener, just add to it */
      MultiListenerHandler multiHandler
      = (MultiListenerHandler) registeredHandler;
      multiHandler.addHandler(handler);
    }
  }
```


# 事件处理

AsyncDispatcher#getEventHandler()是异步派发的关键：

```java 
private final EventHandler<Event> handlerInstance = new GenericEventHandler();

// 省略.....

@Override
public EventHandler<Event> getEventHandler() {
   return handlerInstance;
}
```

## GenericEventHandler：一个特殊的事件处理器

GenericEventHandler是一个特殊的事件处理器，用于接受各种事件。由指定线程处理接收到的事件。

```java
public void handle(Event event) {
  if (blockNewEvents) {
    return;
  }
  drained = false;
  /* all this method does is enqueue all the events onto the queue */
  int qSize = eventQueue.size();
  if (qSize != 0 && qSize % 1000 == 0
      && lastEventQueueSizeLogged != qSize) {
    lastEventQueueSizeLogged = qSize;
    LOG.info("Size of event-queue is " + qSize);
  }
  if (qSize != 0 && qSize % detailsInterval == 0
          && lastEventDetailsQueueSizeLogged != qSize) {
    lastEventDetailsQueueSizeLogged = qSize;
    printEventQueueDetails();
    printTrigger = true;
  }
  int remCapacity = eventQueue.remainingCapacity();
  if (remCapacity < 1000) {
    LOG.warn("Very low remaining capacity in the event-queue: "
        + remCapacity);
  }
  try {
    eventQueue.put(event);
  } catch (InterruptedException e) {
    if (!stopped) {
      LOG.warn("AsyncDispatcher thread interrupted", e);
    }
    // Need to reset drained flag to true if event queue is empty,
    // otherwise dispatcher will hang on stop.
    drained = eventQueue.isEmpty();
    throw new YarnRuntimeException(e);
  }
};
```



- blockNewEvents: 是否阻塞事件处理，只有当中央处理器停止之后才会停止接受事件。

- eventQueue：将接收到的请求放置到当前阻塞队列里面。方便指定线程及时处理。

  

## 事件处理线程

在服务启动时（serviceStart函数）创建一个线程，会循环处理接受到的事件。核心处理逻辑在函数dispatch里面。

```java
Runnable createThread() {
  return new Runnable() {
    @Override
    public void run() {
      while (!stopped && !Thread.currentThread().isInterrupted()) {
        drained = eventQueue.isEmpty();
        // 省略。。。
        Event event;
        try {
          event = eventQueue.take();
        } catch(InterruptedException ie) {
          if (!stopped) {
            LOG.warn("AsyncDispatcher thread interrupted", ie);
          }
          return;
        }
        if (event != null) {
          // 省略。。。
          dispatch(event);
          // 省略。。。
        }
      }
    }
  };
}
```



### dispatch详解

- 从已经注册的eventDispatchers列表里面查找当前事件对应的处理器，调用当前处理器的handler函数。
- 如果当前handler处理出现异常时，默认会退出RM。

```java
protected void dispatch(Event event) {
  //all events go thru this loop
  LOG.debug("Dispatching the event {}.{}", event.getClass().getName(),
      event);

  Class<? extends Enum> type = event.getType().getDeclaringClass();

  try{
    EventHandler handler = eventDispatchers.get(type);
    if(handler != null) {
      handler.handle(event);
    } else {
      throw new Exception("No handler for registered for " + type);
    }
  } catch (Throwable t) {
    //TODO Maybe log the state of the queue
    LOG.error(FATAL, "Error in dispatcher thread", t);
    // If serviceStop is called, we should exit this thread gracefully.
    if (exitOnDispatchException
        && (ShutdownHookManager.get().isShutdownInProgress()) == false
        && stopped == false) {
      stopped = true;
      Thread shutDownThread = new Thread(createShutDownThread());
      shutDownThread.setName("AsyncDispatcher ShutDown handler");
      shutDownThread.start();
    }
  }
}
```





# 状态机

状态转换由成员变量StateMachine管理，所有的StateMachine都由StateMachineFactory进行管理。由addTransition函数实现状态机。

```java
private static final StateMachineFactory<RMAppImpl,
                                           RMAppState,
                                           RMAppEventType,
                                           RMAppEvent> stateMachineFactory
                               = new StateMachineFactory<RMAppImpl,
                                           RMAppState,
                                           RMAppEventType,
                                           RMAppEvent>(RMAppState.NEW)


     // Transitions from NEW state
    .addTransition(RMAppState.NEW, RMAppState.NEW,
        RMAppEventType.NODE_UPDATE, new RMAppNodeUpdateTransition())
    .addTransition(RMAppState.NEW, RMAppState.NEW_SAVING,
        RMAppEventType.START, new RMAppNewlySavingTransition())
    .addTransition(RMAppState.NEW, EnumSet.of(RMAppState.SUBMITTED,
            RMAppState.ACCEPTED, RMAppState.FINISHED, RMAppState.FAILED,
            RMAppState.KILLED, RMAppState.FINAL_SAVING),
        RMAppEventType.RECOVER, new RMAppRecoveredTransition())
    .addTransition(RMAppState.NEW, RMAppState.KILLED, RMAppEventType.KILL,
        new AppKilledTransition())
    .addTransition(RMAppState.NEW, RMAppState.FINAL_SAVING,
        RMAppEventType.APP_REJECTED,
        new FinalSavingTransition(new AppRejectedTransition(),
          RMAppState.FAILED))

    .addTransition(
        RMAppState.KILLED,
        RMAppState.KILLED,
        EnumSet.of(RMAppEventType.APP_ACCEPTED,
            RMAppEventType.APP_REJECTED, RMAppEventType.KILL,
            RMAppEventType.ATTEMPT_FINISHED, RMAppEventType.ATTEMPT_FAILED,
            RMAppEventType.NODE_UPDATE, RMAppEventType.START))

     .installTopology();
```



Transition定义了“从一个状态转换到另一个状态”的行为，由转换操作、开始状态、事件类型、事件组成：

```java
public interface StateMachine
                 <STATE extends Enum<STATE>,
                  EVENTTYPE extends Enum<EVENTTYPE>, EVENT> {
  public STATE getCurrentState();
  public STATE getPreviousState();
  public STATE doTransition(EVENTTYPE eventType, EVENT event)
        throws InvalidStateTransitionException;
}
```





## ResourceManager中状态机

- RMApp：用于维护一个Application的生命周期，实现类 - RMAppImpl
- RMAppAttempt：用于维护一次试探运行的生命周期，实现类 - RMAppAttemptImpl
- RMContainer：用于维护一个已分配的资源最小单位Container的生命周期，实现类 - RMContainerImpl
- RMNode：用于维护一个NodeManager的生命周期，实现类 - RMNodeImpl

NodeManager中状态机：

- Application：用于维护节点上一个Application的生命周期，实现类 - ApplicationImpl
- Container：用于维护节点上一个容器的生命周期，实现类 - ContainerImpl
- LocalizedResource：用于维护节点上资源本地化的生命周期，没有使用接口即实现类 - LocalizedResource




