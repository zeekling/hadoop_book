
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



