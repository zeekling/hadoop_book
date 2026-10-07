# Apache Hadoop 版本特性分析报告

**分析时间**: 2026-09-30
**数据来源**:
- Apache JIRA: https://issues.apache.org/jira/browse/HADOOP（REST API fixVersion 查询）
- GitHub: https://github.com/apache/hadoop（Releases / PR / branches）
- 本地仓库: D:\project\github\hadoop（git tags / trunk commits 交叉验证）

---

## 📊 版本状态全景

| 版本 | 状态 | 发布日期 | 说明 |
|------|------|---------|------|
| **3.4.0** | ✅ 已发布 | 2024-03-17 | 3.4.x 线起点 |
| **3.4.1** | ✅ 已发布 | 2024-10-18 | 维护版本 |
| **3.4.2** | ✅ 已发布 | 2025-08-29 | 维护版本 |
| **3.4.3** | ✅ 已发布 | 2026-02-24 | JIRA 标记已发布（本地仅 RC0 标签，2026-01-08） |
| **3.5.0** | ✅ 已发布 | 2026-04-01 | **当前最新正式版** |
| **3.5.1** | ⏳ 未发布 | — | 开发中（HADOOP-19973 Windows 修复目标版本） |
| **3.4.4** | ⏳ 未发布 | — | 开发中 |
| **3.6.0** | 🔧 开发中 | — | **trunk 当前版本（3.6.0-SNAPSHOT）**，JIRA 128 条关联 Issue |

> 注：Hadoop 仓库不使用 GitHub Releases 发布（Releases API 返回空），正式版信息以 JIRA fixVersions + Apache 官网为准。
> trunk 最新提交：e8783aa5925（2026-09-26，HADOOP-19973 Windows 冒号文件夹修复回移目标确认中）。

---

## 🚀 3.5.0 核心特性（当前最新正式版，2026-04-01）

### 1. Java 17 完整支持（重大主线）

| Issue | 描述 | 类型 | 组件 |
|-------|------|------|------|
| HADOOP-17177 | Java 17 support | Improvement | build |
| HADOOP-18887 | Java 17 Runtime Support | **Feature** | build |
| HADOOP-19310 | [JDK17] Add JPMS options required by Java 17+ | Other | build, common |
| HADOOP-19530 | Add --enable-native-access=ALL-UNNAMED JVM option | Improvement | — |
| HADOOP-19540 | [JDK17] Add ubuntu:noble as a build platform with JDK-17 as default | Other | hadoop-common, hdfs-client |
| HADOOP-18716 | [JDK-17] Failed unit tests（修复） | Bug（Critical） | — |

- Java 17 从"编译支持"升级为"**运行时支持**"（HADOOP-18887）
- KMS 服务默认 Java 17 运行（HADOOP-18902）
- JDK25 适配进行中：UserGroupInformation Subject API 迁移（HADOOP-19212）

### 2. S3A 云存储重大改进

| Issue | 描述 | 类型 |
|-------|------|------|
| HADOOP-19559 | S3A: **Analytics accelerator for S3 to be enabled by default** | **Feature** |
| HADOOP-19303 | VectorIO API to support releasing buffers on failure | Other |
| HADOOP-19278 | S3A: remove option to delete directory markers | Improvement |

S3A Analytics Accelerator（AAL）配套子任务：
- IoStatistics 支持（HADOOP-19364）
- 审计头支持（HADOOP-19365）
- Vector IO 支持（HADOOP-19394）
- SSE-C 支持（HADOOP-19587）
- 迁移到 Java sync client（HADOOP-19664）
- AAL 版本升级至 1.3.1（HADOOP-19742）

### 3. ABFS 增强

| Issue | 描述 | 类型 |
|-------|------|------|
| HADOOP-19518 | ABFS: [FnsOverBlob] WASB to ABFS Migration Config Support Script | Other |

ReadAheadV2 系列（3.4.3/3.5.0）：
- Read Buffer Manager V2 实现（HADOOP-19622）
- Prefetch Aggressiveness 提升（HADOOP-19596）
- ReadBufferManager 重构隔离（HADOOP-19613）
- FNS Over Blob 幂等性（HADOOP-19658）

### 4. 基础设施升级

| Issue | 描述 | 类型 |
|-------|------|------|
| HADOOP-15984 | **Update Jersey from 1.19 to 2.x**（跨 common/hdfs/mapreduce/yarn） | Improvement |
| HADOOP-19012 | Use CRC tables to speed up galoisFieldMultiply in CrcUtil | Improvement |
| HADOOP-19269 | Upgrade maven-shade-plugin to 3.6.0 | Other |
| HADOOP-19738 | Upgrade to hadoop-thirdparty 1.5.0 | Improvement |
| HADOOP-19696 | hadoop binary distribution to move cloud connectors to hadoop common/lib | Other |
| HADOOP-19165 | Explore dropping protobuf 2.5.0 from the distro | Improvement |

---

## 🔒 3.4.3 安全修复清单（维护版本，2026-02-24）

### CVE 修复（5 条显式 CVE）

| Issue | 组件 | 版本升级 | CVE |
|-------|------|---------|-----|
| HADOOP-19690 | commons-lang3 | 3.18.0 | CVE-2025-48924 |
| HADOOP-19689 | netty | 4.1.127 | CVE-2025-58057 |
| HADOOP-19730 | bouncycastle | 1.82 | CVE-2025-8916 |
| HADOOP-19806 | Jetty | 9.4.60+ | CVE-2024-6763 |
| HADOOP-19764 | amazon-s3-encryption-client-java | 4.0.0+ | CVE-2025-14763（Invisible Salamanders） |
| HADOOP-19747 | lz4-java | at.yawk.lz4 版本 | CVE-2025-12183 |
| HADOOP-19632 | nimbus-jose-jwt | 10.4 | CVE-2025-53864 |

### 代码级安全修复

| Issue | 描述 | 项目 |
|-------|------|------|
| HDFS-17874 | Unsafe Jackson Polymorphic Deserialization in HDFS DiskBalancer NodePlan | HDFS |
| YARN-10972 | Remove stack traces from Jetty's response for Security Reasons | YARN |
| YARN-11916 | FileSystemTimelineReaderImpl vulnerable to race conditions | YARN |

### 3.4.3 其他关键修复

| Issue | 描述 | 项目 |
|-------|------|------|
| HADOOP-19712 | S3A: Deadlock in IOStatistics EvaluatingStatisticsMap.entryset() | HADOOP |
| HADOOP-19282 | S3A: STSClientFactory hard-coded dependency on shaded HttpClient | HADOOP |
| YARN-11839 | [RM HA] RM stay in ACTIVE with RMStateStore in FENCED state | YARN |
| MAPREDUCE-7448 | Skipping cleanup with FileOutputCommitter V1 can corrupt output | MAPREDUCE |
| HDFS-17380/17528 | FsImageValidation: remove inaccessible nodes / set txid | HDFS |
| HDFS-17834 | Fix invalid HTTP links for DataNodes in IPv6 environment | HDFS |

---

## 🔧 3.6.0 开发中（trunk 方向，JIRA 128 条关联 Issue）

| Issue | 描述 | 类型 | 状态 |
|-------|------|------|------|
| HADOOP-10579 | **Find command - add match expressions to find command** | Other | Resolved |

- 2014 年创建的老需求在 trunk 落地——find 命令支持 match 表达式，向更现代的文件系统查询能力演进
- 3.6.0 完整清单需分页抓取（JIRA total=128，已获取 100 条）

---

## ⏳ 3.5.1 / 3.4.4 在途版本

| Issue | 描述 | 类型 | 状态 | 目标版本 |
|-------|------|------|------|---------|
| **HADOOP-19973** | Hadoop file system can not browse the folder that has children folder with **colon in Windows** | Bug（Critical） | **Open** | 3.4.4 / 3.5.1 |
| HADOOP-19154 | upgrade bouncy castle to 1.78.1 due to CVEs | Improvement | Resolved | 3.5.1 |

---

## 📈 版本演进时间线

```
2024-03-17: 3.4.0 发布
2024-10-18: 3.4.1 发布
2025-08-29: 3.4.2 发布
2026-01-08: release-3.4.3-RC0 打标
2026-02-24: 3.4.3 发布
2026-04-01: 3.5.0 发布 ← 当前最新正式版
2025-11-04: thirdparty-1.6.0 开发开始
进行中:     3.5.1（Windows 修复）、3.4.4、3.6.0（trunk）
```

---

## ⚠️ 值得关注的重点

### 1. Jersey 1.19 → 2.x 大迁移（HADOOP-15984）⭐
- **影响面极广**：跨 common/hdfs/mapreduce/yarn 四大模块，多年遗留的架构级升级，3.5.0 完成
- **升级建议**：3.4.x → 3.5.0 时重点测试 Web UI / REST 接口兼容性

### 2. S3A Analytics Accelerator 默认开启（HADOOP-19559）⭐
- 3.5.0 中唯一标记为 **Feature** 的核心特性；IoStatistics、审计头、SSE-C、vector IO 配套完整
- **升级建议**：S3A 用户升级后关注性能变化，`fs.s3a.aal.*` 配置项可能需调整

### 3. Java 17 运行时完整支持（HADOOP-18887）⭐
- 从"编译支持"升级为"**运行时支持**"；JPMS 模块选项、native-access、Ubuntu Noble 平台
- **升级建议**：JDK17 用户可正式迁移生产环境；KMS 默认 Java 17

### 4. 云连接器迁移到 common/lib（HADOOP-19696/19800）
- 二进制发行版结构变更：hadoop-aws/hadoop-azure 等移入 common/lib
- **升级建议**：依赖 classpath 布局的部署脚本需适配

### 5. Windows 冒号文件夹修复未完成（HADOOP-19973）
- **Critical**，仍 Open，目标 3.4.4/3.5.1
- **升级建议**：Windows 用户升级 3.5.0 后仍会遇到此问题，需等 3.5.1

### 6. 3.6.0 新方向：find 命令 match 表达式（HADOOP-10579）
- 2014 年老需求 trunk 落地，向现代文件系统查询能力演进
- **跟踪建议**：关注 trunk 进展与 3.6.0 发布计划

---

## 📋 JIRA 数据统计（70 条 3.4.3 关联 Issue）

### 状态/类型/项目分布

| 维度 | 分布 |
|------|------|
| 状态 | Resolved 69（98.6%）/ Open 1（1.4%，HADOOP-19833） |
| 类型 | Bug 17 / Feature 1 / Improvement 11 / Other(Sub-task/Task) 41 |
| 项目 | HADOOP 58（82.9%）/ YARN 6 / HDFS 4 / MAPREDUCE 2 |
| 优先级 | Major 53 / Minor 9 / Critical 4 / Blocker 2 / Trivial 2 |

### 唯一 Open 的 3.4.3 关联 Issue

| Issue | 描述 | 优先级 |
|-------|------|--------|
| HADOOP-19833 | fs.s3a.classloader.isolation=false configuration does not work（fixVersions 横跨 3.4.0–3.5.1，长期遗留回归） | Minor |

---

## 📝 升级路径建议

```
3.4.x ──→ 3.5.0（最新正式版，推荐目标）
              ├─ 重点验证：Jersey 2.x 兼容性、AAL 默认启用、common/lib 布局
              ├─ Windows 用户：等 3.5.1（HADOOP-19973 修复）
              └─ JDK17 用户：可正式迁移生产环境
```

**优先级排序**：
1. 🔴 受 CVE 影响 → 立即升级（commons-lang3、netty、bouncycastle、Jetty、lz4、nimbus-jose-jwt）
2. 🔴 S3A/ABFS 用户 → 3.5.0（AAL 默认启用 + 幂等性 + 死锁修复）
3. 🟠 HDFS/YARN 用户 → 3.4.3（反序列化修复 + RM HA corner case）
4. 🟠 JDK17 用户 → 3.5.0（运行时支持完整）
5. 🟢 Windows 用户 → 等 3.5.1
6. 🟢 新特性跟踪 → 3.6.0 trunk

---

## 🔍 数据完整性说明

| 检查项 | 状态 | 说明 |
|--------|------|------|
| JIRA fixVersion 查询 | ✅ | 3.4.3（70 条）、3.5.0+3.5.1（total=950 取 100 条）、3.6.0（total=128 取 100 条） |
| GitHub Releases | ⚠️ | 返回空——Hadoop 不使用 GitHub Releases，正式版以 JIRA 为准 |
| GitHub Issues | ❌ | Hadoop 未使用 GitHub Issues 管理 Issue（搜索返回 0） |
| JIRA Issue Navigator | ⚠️ | Bug 类型 50 条 Open 已获取（updated DESC）；复杂 JQL 返回 jqlTooComplex |
| 3.6.0 完整清单 | ⚠️ | total=128，已获取 100 条，缺 28 条（需分页抓取） |

---

*由 OpenCode（PuaSE 编排）+ apache-issue-analyzer skill 自动生成 | 数据采集时间：2026-09-30*
