# 简介

Yarn状态机的基础部分参见：[Yarn 状态机以及事件机制](./yarn_event.md)

本章主要将Yarn当中详细的事件以及处理过程。

AsyncDispatcher是中央处理器的核心线程。通过使用AsyncDispatcher的对象可以分析Yarn里面有多少个中央处理器，每个处理器都由什么用途。



## Component  dispatcher

当前的处理器注册了下面几个事件：

- ServiceEventHandler:
- ComponentEventHandler:
- ComponentInstanceEventHandler:

```java
dispatcher.register(ServiceEventType.class, new ServiceEventHandler());
dispatcher.register(ComponentEventType.class, new ComponentEventHandler());
dispatcher.register(ComponentInstanceEventType.class,
        new ComponentInstanceEventHandler());
```





