# AGENTS.md

## 仓库说明
本文档仓库，分析 Apache Hadoop 3.3.1 源码并撰写架构文档。**非 Hadoop 源码库，无需构建/测试**。

## 目录结构
| 目录 | 内容 |
|------|------|
| `common/` | 身份验证、Fair Call Queue、NativeIO |
| `hdfs/` | NameNode、DataNode、FsImage、JournalNode、RBF |
| `yarn/` | ResourceManager、调度器、Container、状态机 |
| `zookeeper/` | ZooKeeper 源码分析 |
| `ozone/` | Apache Ozone 分布式对象存储 |
| `watch/` | 版本特性追踪、兼容性分析、问题汇总 |
| `attach/` | 各模块文档配图（图片托管于 `pan.zeekling.cn/zeekling/hadoop/`）|

各模块根目录有 `README.md` 作为索引。

## Git 工作流
- **禁止推 master**：本地 pre-push hook 阻止，仅通过 Gitea PR 合并
- 分支命名：`feature/描述`（如 `feature/add-datanode-optimization-analysis`）
- 提交信息：中文，简洁描述动作+对象（如"添加DataNode优化详解文档"）
- 远程仓库：`origin` → Gitea（`ssh://git@git.zeekling.cn:222/big-data/hadoop_book.git`），`github` → GitHub 镜像
- **PR 编码**：创建/更新 PR 时，标题和正文必须使用 **UTF-8 编码**（避免中文及 emoji 字符乱码）；API 请求建议使用 `curl.exe` 或指定 `Content-Type: application/json; charset=utf-8`

## 文件命名
- 模块索引：`{模块名}/README.md`
- 技术详解：`{模块}/{组件}详解.md`
- watch 分析：`watch/{主题}.md`（如 `3.3.1-3.4.1兼容性分析.md`、`datanode优化详解.md`）
- kebab-case：`fair_call_queue.md`

## 文档结构
1. 标题（`# 标题`）
2. 简介
3. 技术细节和代码示例
4. 架构图：`![pic](https://pan.zeekling.cn/zeekling/hadoop/...)`
5. 比较表格
6. 推荐阅读（相对路径链接）

## 内容规范
- **语言**：所有内容中文
- **代码示例**：Java
- **类引用**：全限定名 `org.apache.hadoop.hdfs.server.namenode.FSImage`
- **方法引用**：`ClassName#methodName`
- **配置引用**：反引号包裹 `` `dfs.namenode.name.dir` ``
- **版本号**：3.3.1（不用 v3.3.1）
- **图片**：托管于 `pan.zeekling.cn/zeekling/hadoop/`
- **内部链接**：相对路径
- **JIRA issue**：表格中加粗关键 issue（如 `**HDFS-15382**`）

## 新增文档必做
1. 在对应模块 `README.md` 添加链接
2. 在根 `README.md` 知识目录树添加条目
3. 文档内推荐阅读使用正确的相对路径（`watch/` 内引用 `hdfs/` 需 `../hdfs/`）

## JIRA 数据获取
Apache JIRA REST API 获取 issue 数据：
```
https://issues.apache.org/jira/rest/api/2/search?jql=project=HDFS AND component=datanode AND fixVersion=3.4.0&fields=key,summary,priority&maxResults=100
```
- 大项目（HADOOP/HDFS）结果可能超 100 条，需分页 `&startAt=100`
- 请求偶尔超时，需重试或换参数缩小范围
