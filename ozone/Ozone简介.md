# Apache Ozone 简介

Apache Ozone 是 Hadoop 的可扩展、可靠、高性能的分布式对象存储系统，专为数据分析和对象存储工作负载优化。

## 核心特性

| 特性 | 说明 |
|------|------|
| **Scalable** | 支持 PB 级数据、数十亿对象，支持大小文件 |
| **Resilient** | 内置复制、纠删码、强一致性模型 |
| **Secure** | Kerberos 认证、TDE 加密、ACL/Ranger 授权 |
| **Performant** | 高吞吐、低延迟，支持分层存储 |
| **Multi-Protocol** | 原生 S3 协议支持，兼容 Hadoop 文件系统接口 |
| **Efficient** | 密集存储节点，可合并多集群降低成本 |

## 典型用例

- **Data Lakes**: 支持分层存储的分析型数据湖
- **Object Storage**: S3 兼容的对象存储（应用备份、非结构化数据）
- **Big Data Storage**: Hadoop/Spark 等数据处理框架的存储底座

## 集成生态

Hive、Spark、Iceberg、Trino、Impala、Ranger、Knox、HBase、DistCP、Prometheus、Grafana

## 推荐阅读

- [Ozone 官方文档](https://ozone.apache.org/docs/)