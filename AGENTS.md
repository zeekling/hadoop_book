# Apache Hadoop 文档仓库 - 代理指南

## 仓库概述

本仓库包含关于 Apache Hadoop 的中文技术文档，重点分析 Hadoop 组件、源码实现细节和架构模式。

## 仓库结构

```
hadoop_book/
├── common/              # 通用工具（身份验证、公平调用队列、NativeIO）
├── hdfs/                # HDFS 组件（Namenode、DataNode、FsImage等）
├── yarn/                # YARN 组件（ResourceManager、ContainerManager等）
├── zookeeper/           # ZooKeeper 协调服务
├── ozone/               # 对象存储服务
└── watch/               # 特性追踪
```

## 文档规范

### 文件命名
- 模块 README：`{模块名}/README.md`
- 技术详解：`{模块}/{组件}详解.md`
- 文件使用 kebab-case 命名（如 `fair_call_queue.md`）

### 内容结构
1. 标题（# 标题）
2. 简介
3. 技术细节和代码示例
4. 架构图：`![pic](https://pan.zeekling.cn/zeekling/hadoop/...)`
5. 比较表格

### 代码示例
- 使用 Java 编写 Hadoop 实现细节
- 包含方法签名、类定义、关键逻辑
- 添加中文注释
- 展示请求/响应和错误处理
- 保持正确的缩进格式

### 语气和风格
- 专业技术中文
- 简洁但全面
- 聚焦实现细节和架构

## 模块特定规范

**HDFS：** Namenode/NameNode、DataNode、块管理、JournalNode、Router/WebHDFS
**YARN：** ResourceManager、容器管理、调度器、ApplicationMaster 生命周期
**Common：** 身份验证（Kerberos/SPNEGO）、Native I/O、公平调用队列、故障转移
**ZooKeeper：** 启动、会话管理、监视器、领导选举

## 代码风格规范

### 语言
- 所有文档使用中文
- Hadoop 示例使用 Java

### 格式化
- 使用带有反引号的 Java 语法高亮
- 保持一致的缩进
- 遵循 Markdown 表格结构

### 内容覆盖
- 架构概述
- 核心类和接口
- 配置参数
- 工作流/过程流程
- 代表性代码片段
- 架构图
- 边界情况和错误处理

## 命令

```bash
# 统计文档文件数量
find . -name "*.md" -type f | wc -l

# 搜索特定组件
grep -r "NameNode" --include="*.md"

# 按模块统计文件数
ls -1 */ | while read dir; do echo "$dir: $(find "$dir" -name "*.md" -type f | wc -l)"; done
```

## 重要说明

1. **语言：** 所有内容使用中文
2. **代码：** Hadoop 示例使用 Java
3. **图表：** 始终引用 https://pan.zeekling.cn/zeekling/hadoop/
4. **链接：** 内部文档使用相对路径
5. **风格：** 与现有文档保持一致
6. **完整性：** 包含概览和详细实现信息
7. **一致性：** 跨文档使用相同的术语
8. **版本：** 引用 GitHub 问题（如 YARN-8234）

## 参考资料来源

- 源代码：Apache Hadoop 源代码和官方文档
- 图表：https://pan.zeekling.cn/zeekling/hadoop/

如有问题，请咨询相应的模块文档或主 README.md。
