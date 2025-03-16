
# 简介

在HDFS当中可能存在比较重的操作，比如：大目录的List、大目录的count操作，在默认情况下，所有的操作都是按照先进先出的顺序执行的，
在这种情况下可能导致其他轻量大操作也会耗时比较高。为了缓解这个问题，Hadoop社区推出了FairCallQueue。


# 原理

下图是FairCallQueue的原理图：

![pic](https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/images/faircallqueue-overview.png)

## listen queue和Reader threads
客户端发送的请求会进入listen queue。Reader threads将客户端发送的请求移动到RpcScheduler，
RpcScheduler中会根据历史RPC请求信息计算并且为当前的请求分配优先级。

## RpcScheduler
配置了FairCallQueue之后，默认的RpcScheduler将是DecayRpcScheduler。在DecayRpcScheduler中记录这每个用户的请求平均耗时，
平均耗时时间每隔一个衰减周期（默认5s）会衰减一次，衰减算法：`上个周期的平均耗时 * 衰减系数 + 当前周期平均耗时。`

每隔周期内，会对所有的用户按照平均耗时


## WeightedMultiplexer


