# AGENTS.md

## 仓库说明
本文档仓库，分析 Apache Hadoop 3.3.1 源码并撰写架构文档。**非 Hadoop 源码库，无需构建/测试**。无 CI/CD、无 package.json、无 lint 配置——纯 Markdown + PDF。**所有操作均为文件编辑，无任何编译/测试/构建命令可执行。**

## 目录结构
| 目录 | 内容 |
|------|------|
| `common/` | 身份验证、Fair Call Queue、NativeIO、ZK 主备倒换 |
| `hdfs/` | NameNode、DataNode、FsImage、JournalNode、RBF、客户端 |
| `yarn/` | ResourceManager、调度器、Container、状态机、联邦 |
| `zookeeper/` | ZooKeeper 源码分析 |
| `ozone/` | Apache Ozone 分布式对象存储 |
| `research/` | 学术论文 PDF 原文 + 中文翻译/解析（含 FAST'26、NSDI'26、ICPP'25 等） |
| `watch/` | 版本特性追踪（3.3.1→3.4.1→3.5.0）、兼容性分析、开源问题追踪 |
| `common/attach/` `hdfs/attach/` `yarn/attach/` | 各模块配图（图片托管于 `pan.zeekling.cn/zeekling/hadoop/`，本地仅存引用） |

各模块根目录有 `README.md` 作为索引。根 `README.md` 是知识目录树，新增文档必须同步更新。

## 文件类型
- **Markdown**：技术文档、论文翻译、问题分析。所有内容中文。
- **PDF**：仅存于 `research/`，论文原文。**二进制文件，不要用文本工具编辑。**
- **图片**：不直接存储在仓库；通过 URL `![pic](https://pan.zeekling.cn/zeekling/hadoop/...)` 引用。

## Git 工作流
- **禁止推 master**：本地 pre-push hook 阻止，仅通过 Gitea PR 合并
- 分支命名：`feature/描述`（如 `feature/add-datanode-optimization-analysis`）
  - 例外：`shuffle` 分支（历史遗留，非 feature/ 前缀）
- 提交信息：中文，简洁描述动作+对象（如"添加DataNode优化详解文档"）
- 远程仓库：`origin` → Gitea（`ssh://git@git.zeekling.cn:222/big-data/hadoop_book.git`），`github` → GitHub 镜像
- **PR 编码**：创建/更新 PR 时，标题和正文必须使用 **UTF-8 编码**（避免中文及 emoji 字符乱码）；API 请求建议使用 `curl.exe` 或指定 `Content-Type: application/json; charset=utf-8`

## 文件命名
- 模块索引：`{模块名}/README.md`
- 技术详解：`{模块}/{组件}详解.md`（中英文混排，如 `NameNode 组件详解.md`）
- 论文翻译：`research/{论文名}_翻译解析.md`（保持原文前缀+中文后缀，如 `AITURBO_FAST26_翻译解析.md`）
- 版本追踪：`watch/{主题}.md`（如 `3.3.1-3.4.1兼容性分析.md`）
- **注意**：仓库同时使用 `snake_case`（`fair_call_queue.md`）和 `kebab-case`（`container-executor.md`）两种命名风格，新增文件时参考所在模块已有文件的风格。

## 文档结构
1. 标题（`# 标题`）
2. 简介
3. 技术细节和代码示例
4. 架构图：`![pic](https://pan.zeekling.cn/zeekling/hadoop/...)`
5. 比较表格
6. 推荐阅读（相对路径链接）

## 内容规范
- **语言**：所有内容中文
- **编码**：所有 `.md` 文件必须为 **UTF-8 编码**（无 BOM），禁止 GBK/GB2312（曾有文件因混合编码导致显示异常）
- **代码示例**：Java
- **类引用**：全限定名 `org.apache.hadoop.hdfs.server.namenode.FSImage`
- **方法引用**：`ClassName#methodName`
- **配置引用**：反引号包裹 `` `dfs.namenode.name.dir` ``
- **版本号**：3.3.1（不用 v3.3.1）
- **图片**：托管于 `pan.zeekling.cn/zeekling/hadoop/`，图片格式 `hadoop_xxx_00001.png`，通过 `![pic](URL)` 引用
- **内部链接**：相对路径（`watch/` 内引用 `hdfs/` 需 `../hdfs/`）
- **JIRA issue**：表格中加粗关键 issue（如 `**HDFS-15382**`）

## 新增文档必做
1. 在对应模块 `README.md` 添加链接（`watch/` 无 README.md，新增 `watch/` 文档需在根 `README.md` 知识目录树中添加条目，但不在 `watch/` 下创建 README）
2. 在根 `README.md` 知识目录树添加条目（按模块分类表格）
3. 文档内推荐阅读使用正确的相对路径

## 新增论文翻译必做
1. 学术 PDF 存于 `research/`，命名保持原文标题（如 `AITURBO_FAST26.pdf`）
2. 中文翻译/解析文件命名：`{原文前缀}_翻译解析.md`（如 `AITURBO_FAST26_翻译解析.md`）；部分文档可能使用 `{英文标题}论文全文中文翻译.md`（如 `Fletch论文全文中文翻译.md`）
3. 论文 PDF 和翻译解析文件之间通过 README 表格链接关联
4. 在 `research/README.md` 论文列表表格中添加条目（含 PDF → 翻译解析双向链接）
5. 在根 `README.md` 研究论文表格中添加条目

## JIRA 数据获取
Apache JIRA REST API 获取 issue 数据：
```
https://issues.apache.org/jira/rest/api/2/search?jql=project=HDFS AND component=datanode AND fixVersion=3.4.0&fields=key,summary,priority&maxResults=100
```
- 大项目（HADOOP/HDFS）结果可能超 100 条，需分页 `&startAt=100`
- 请求偶尔超时，需重试或换参数缩小范围
