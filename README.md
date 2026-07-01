# 简介

本仓库用于记录学习Hadoop相关笔记，菜就多做笔记。

[![Star History Chart](https://api.star-history.com/svg?repos=zeekling/hadoop_book&type=Date)](https://star-history.com/#zeekling/hadoop_book&Date)

[![Last Commit](https://img.shields.io/github/last-commit/zeekling/hadoop_book)](https://github.com/zeekling/hadoop_book/graphs/commit-activity) [![Commits per Year](https://img.shields.io/github/commit-activity/y/zeekling/hadoop_book)](https://github.com/zeekling/hadoop_book/graphs/commit-activity) [![Monthly Activity](https://img.shields.io/github/commit-activity/m/zeekling/hadoop_book)](https://github.com/zeekling/hadoop_book/graphs/commit-activity) [![Watchers](https://img.shields.io/github/watchers/zeekling/hadoop_book?style=social&label=Watch)](https://github.com/zeekling/hadoop_book/watchers) ![Visitors](https://visitor-badge.laobi.icu/badge?page_id=zeekling.hadoop_book)

# 编译

执行下面编译命令：

- Linux 下的编译命令如下：

```bash
# -T 是编译的线程数，可以按照具体操作系统增加或者减少
mvn -T 8 package -Pdist,native -DskipTests -Dmaven.javadoc.skip=true
# or 忽略 本地方法编译
mvn -T 1C clean package -DskipTests -P\!sign -Pnative -P\!resource-bundle -PskipShade  -Dmaven.javadoc.skip=true -Dcheckstyle.skip=true -Dpmd.skip=true
```

- Windows 下面的编译命令如下（Windows下的二进制编译未调试通过）：

```bash
# 本地编译，忽略二进制编译
mvn -T 1C clean install -DskipTests -PskipShade -P\!native-win -PskipShade  -Dmaven.javadoc.skip=true -Dcheckstyle.skip=true -Dpmd.skip=true
```

# 知识目录树

## HDFS 相关

### NameNode

| 知识库名称 | 二级分类 | 所属模块 |
|---|---|---|
| [namenode全景](./hdfs/namenode全景.md) | 全景 | [HDFS 模块](./hdfs/README.md) |
| [NameNode启动过程](./hdfs/nameNode启动过程.md) | 启动 | [HDFS 模块](./hdfs/README.md) |
| [NameNode 组件详解](./hdfs/NameNode%20组件详解.md) | 组件 | [HDFS 模块](./hdfs/README.md) |
| [FsImage详解](./hdfs/fsImages.md) | 镜像 | [HDFS 模块](./hdfs/README.md) |
| [NamenodeProtocols详解](./hdfs/NamenodeProtocols详解.md) | 协议 | [HDFS 模块](./hdfs/README.md) |
| [leaseManager详解](./hdfs/leaseManager详解.md) | 租赁 | [HDFS 模块](./hdfs/README.md) |
| [FSDirectory详解](./hdfs/FSDirectory详解.md) | 目录树 | [HDFS 模块](./hdfs/README.md) |
| [BPServiceActor详解](./hdfs/BPServiceActor详解.md) | 块报告 | [HDFS 模块](./hdfs/README.md) |

### 高可用

| 知识库名称 | 二级分类 | 所属模块 |
|---|---|---|
| [HDFS 高可用组件详解](./hdfs/HDFS%20高可用组件详解.md) | 高可用 | [HDFS 模块](./hdfs/README.md) |
| [journalnode](./hdfs/journalnode.md) | JournalNode | [HDFS 模块](./hdfs/README.md) |
| [主备倒换模块详解](./common/zk_failover.md) | 主备倒换 | [Common 模块](./common/README.md) |

### DataNode

| 知识库名称 | 二级分类 | 所属模块 |
|---|---|---|
| [dataNode启动过程](./hdfs/dataNode启动过程.md) | 启动 | [HDFS 模块](./hdfs/README.md) |
| [DataNode 组件详解](./hdfs/DataNode%20组件详解.md) | 组件 | [HDFS 模块](./hdfs/README.md) |
| [BlockRecoveryWorker详解](./hdfs/BlockRecoveryWorker详解.md) | 块恢复 | [HDFS 模块](./hdfs/README.md) |
| [DiskBalancer详解](./hdfs/DiskBalancer详解.md) | 磁盘平衡 | [HDFS 模块](./hdfs/README.md) |
| [DataNode优化详解](./watch/datanode优化详解.md) | 优化 | [Watch 模块](./watch/) |

### 客户端

| 知识库名称 | 二级分类 | 所属模块 |
|---|---|---|
| [HDFS 客户端组件详解](./hdfs/HDFS%20客户端组件详解.md) | 客户端 | [HDFS 模块](./hdfs/README.md) |
| [文件上传](./hdfs/file_upload.md) | 写流程 | [HDFS 模块](./hdfs/README.md) |
| [webhdfs详解](./hdfs/webhdfs详解.md) | WebHDFS | [HDFS 模块](./hdfs/README.md) |
| [httpfs详解](./hdfs/httpfs详解.md) | HttpFS | [HDFS 模块](./hdfs/README.md) |

### 联邦

| 知识库名称 | 二级分类 | 所属模块 |
|---|---|---|
| [router启动详解](./hdfs/router启动详解.md) | Router | [HDFS 模块](./hdfs/README.md) |

### HDFS 功能与优化

| 知识库名称 | 二级分类 | 所属模块 |
|---|---|---|
| [HDFS 功能详解](./hdfs/HDFS%20功能详解.md) | 功能 | [HDFS 模块](./hdfs/README.md) |
| [HDFS 功能总结](./hdfs/HDFS%20功能总结.md) | 总结 | [HDFS 模块](./hdfs/README.md) |

## YARN 相关

### ResourceManager

| 知识库名称 | 二级分类 | 所属模块 |
|---|---|---|
| [ResourceManager详解](./yarn/resourcemanager.md) | 核心 | [YARN 模块](./yarn/README.md) |
| [容量调度器](./yarn/capacity.md) | 调度器 | [YARN 模块](./yarn/README.md) |
| [ResourceEstimatorServer](./yarn/ResourceEstimatorServer.md) | 资源估算 | [YARN 模块](./yarn/README.md) |
| [RMStateStore对比分析](./yarn/RMStateStore对比分析.md) | 状态存储 | [YARN 模块](./yarn/README.md) |
| [RMDelegationTokenSecretManager](./yarn/RMDelegationTokenSecretManager.md) | 安全 | [YARN 模块](./yarn/README.md) |

### NodeManager

| 知识库名称 | 二级分类 | 所属模块 |
|---|---|---|
| [containerManager详解](./yarn/containerManager.md) | 容器管理 | [YARN 模块](./yarn/README.md) |
| [container-executor详解](./yarn/container-executor.md) | 容器执行 | [YARN 模块](./yarn/README.md) |
| [shuffle详解](./yarn/shuffle详解.md) | Shuffle | [YARN 模块](./yarn/README.md) |

### 事件系统

| 知识库名称 | 二级分类 | 所属模块 |
|---|---|---|
| [Yarn状态机](./yarn/yarn_event.md) | 状态机 | [YARN 模块](./yarn/README.md) |
| [Yarn事件处理机制](./yarn/yarn_event_detail.md) | 事件处理 | [YARN 模块](./yarn/README.md) |

### 联邦

| 知识库名称 | 二级分类 | 所属模块 |
|---|---|---|
| [YARN Federation 源码解读](./yarn/federation/YARN%20Federation%20源码解读.md) | 联邦 | [YARN 模块](./yarn/README.md) |

### 工具与作业

| 知识库名称 | 二级分类 | 所属模块 |
|---|---|---|
| [DistributedShell](./yarn/DistributedShell.md) | 示例 | [YARN 模块](./yarn/README.md) |
| [Distcp详解](./yarn/distcp.md) | 工具 | [YARN 模块](./yarn/README.md) |
| [作业启动](./yarn/job_start.md) | 作业 | [YARN 模块](./yarn/README.md) |
| [作业历史缓存](./yarn/jobhistory_cache.md) | JobHistory | [YARN 模块](./yarn/README.md) |
| [SharedCacheManager](./yarn/SharedCacheManager.md) | 共享缓存 | [YARN 模块](./yarn/README.md) |

## ZooKeeper

| 知识库名称 | 二级分类 | 所属模块 |
|---|---|---|
| [Zookeeper启动源码详解](./zookeeper/Zookeeper启动源码详解.md) | 启动 | [ZooKeeper 模块](./zookeeper/README.md) |
| [Zookeeper版本差异详解](./zookeeper/Zookeeper版本差异详解.md) | 版本差异 | [ZooKeeper 模块](./zookeeper/README.md) |

## Ozone

| 知识库名称 | 二级分类 | 所属模块 |
|---|---|---|
| [Ozone 简介](./ozone/Ozone简介.md) | 简介 | [Ozone 模块](./ozone/README.md) |
| [Ozone 模块分析](./ozone/模块分析.md) | 模块分析 | [Ozone 模块](./ozone/README.md) |
| [Ozone 功能分析](./ozone/功能分析.md) | 功能分析 | [Ozone 模块](./ozone/README.md) |
| [Ozone 模块详细分析报告](./ozone/模块详细分析报告.md) | 详细分析 | [Ozone 模块](./ozone/README.md) |
| [Ozone 模块详细分析报告（补充）](./ozone/模块详细分析报告-补充.md) | 详细分析 | [Ozone 模块](./ozone/README.md) |
| [Ozone 模块详细分析报告（深度补充）](./ozone/模块详细分析报告-深度补充.md) | 详细分析 | [Ozone 模块](./ozone/README.md) |
| [CLI和安全模块详细分析报告](./ozone/CLI和安全模块详细分析报告.md) | 安全/CLI | [Ozone 模块](./ozone/README.md) |
| [Ozone与HDFS对比分析](./watch/Ozone与HDFS对比分析.md) | 对比分析 | [Watch 模块](./watch/) |

## Common 公共模块

| 知识库名称 | 二级分类 | 所属模块 |
|---|---|---|
| [Hadoop 模块概览](./common/hadoop_modules_overview.md) | 模块概览 | [Common 模块](./common/README.md) |
| [认证模块详解](./common/authenticated.md) | 认证 | [Common 模块](./common/README.md) |
| [Fair Call Queue](./common/fair_call_queue.md) | 队列 | [Common 模块](./common/README.md) |
| [NativeIO](./common/NativeIO.md) | IO 相关 | [Common 模块](./common/README.md) |
| [org.apache.hadoop.cli 模块详解](./common/org.apache.hadoop.cli%20模块详解.md) | CLI | [Common 模块](./common/README.md) |
| [RollingLevelDB时间线存储详解](./common/applicationhistoryservice/RollingLevelDBTimelineStore.md) | 作业历史服务 | [Common 模块](./common/README.md) |

## Watch 版本追踪与特性分析

| 知识库名称 | 二级分类 | 所属模块 |
|---|---|---|
| [DataNode优化详解（3.3.1后）](./watch/datanode优化详解.md) | DataNode 优化 | [Watch 模块](./watch/) |
| [HDFS 多管道并行写入优化](./watch/hdfs-multi-pipeline-optimization.md) | HDFS 优化 | [Watch 模块](./watch/) |
| [3.3.1-3.4.1 兼容性分析](./watch/3.3.1-3.4.1兼容性分析.md) | 版本兼容性 | [Watch 模块](./watch/) |
| [3.3.1-3.5.0 兼容性预测分析](./watch/3.3.1-3.5.0兼容性预测分析.md) | 版本兼容性 | [Watch 模块](./watch/) |
| [2026.03.28 开源最新问题分析](./watch/2026.03.28开源最新问题分析.md) | Issue 追踪 | [Watch 模块](./watch/) |
| [2026.04.19 开源最新问题分析](./watch/2026.04.19开源最新问题分析.md) | Issue 追踪 | [Watch 模块](./watch/) |
| [值得关注的特性](./watch/值得关注的特性.md) | 特性 | [Watch 模块](./watch/) |
| [Ozone与HDFS对比分析](./watch/Ozone与HDFS对比分析.md) | 对比分析 | [Watch 模块](./watch/) |

## 研究论文

| 知识库名称 | 研究方向 | 所属模块 |
|---|---|---|
| [A Trust-Driven Optimization Model 翻译解析](./research/A_Trust-Driven_Optimization_Model_翻译解析.md) | 安全与授权 | [Research 模块](./research/README.md) |
| [Fletch 论文全文中文翻译](./research/Fletch论文全文中文翻译.md) (FAST'26) | 元数据缓存 | [Research 模块](./research/README.md) |
| [AsyncFS 论文全文中文翻译](./research/AsyncFS论文全文中文翻译.md) | 元数据异步更新 | [Research 模块](./research/README.md) |
| [Origami 翻译解析](./research/Origami_ICPP25_翻译解析.md) (ICPP'25) | ML 驱动元数据负载均衡 | [Research 模块](./research/README.md) |
| [AITURBO 翻译解析](./research/AITURBO_FAST26_翻译解析.md) (FAST'26) | AI 作业存储优化 | [Research 模块](./research/README.md) |
| [BYOM 翻译解析](./research/BYOM-ML-Driven-Storage-Placement_翻译解析.md) | ML 驱动存储放置 | [Research 模块](./research/README.md) |
| [UpFuzz 翻译解析](./research/UpFuzz_NSDI26_翻译解析.md) (NSDI'26) | 存储系统升级兼容性 | [Research 模块](./research/README.md) |
| [智能配置调优 翻译解析](./research/Intelligent_Configuration_Tuning_ML_DL_RL_翻译解析.md) | ML/DL/RL 配置调优 | [Research 模块](./research/README.md) |
| [Origami 论文笔记](./Origami_ICPP25.md) (ICPP'25) | ML 驱动元数据负载均衡 | [仓库根目录](./) |
| [HDFS & YARN 高质量论文分析报告 (2021-2026)](./research/HDFS_YARN_高质量论文分析报告_2021-2026.md) | 综述分析 | [Research 模块](./research/README.md) |
| [HDFS × YARN × AI 交叉论文分析 (2021-2026)](./research/HDFS_YARN_AI_交叉论文分析_2021-2026.md) | AI 交叉分析 | [Research 模块](./research/README.md) |

## 其他资源

| 知识库名称 | 说明 | 所属模块 |
|---|---|---|
| [学习 Hadoop 300问](./hadoop_300_question.md) | Hadoop 面试题集 | [仓库根目录](./) |
