# AGENTS.md

## 仓库性质
**纯文档仓库**——分析 Apache Hadoop 3.3.1 源码并撰写架构文档。非 Hadoop 源码库，**不要执行任何构建/编译命令**。
- 内容：Markdown 技术文档/论文翻译/问题分析 + PDF（仅存 `research/`）
- 无 CI/CD、package.json、lint 配置、可执行的构建命令
- 根 `README.md` 含有 Hadoop 源码编译命令（`mvn`），仅作为参考，**不要在此仓库执行**
- 共 87 个 Markdown 文件，148 次 Git 提交

## 目录结构
| 目录 | 内容 |
|------|------|
| `common/` | 通用模块：认证、Fair Call Queue、NativeIO、ZK 主备倒换、CLI |
| `common/applicationhistoryservice/` | RollingLevelDB 时间线存储（`README.md` 为空占位；实际内容在 `RollingLevelDBTimelineStore.md`） |
| `hdfs/` | NameNode、DataNode、FsImage、JournalNode、RBF、WebHDFS/HttpFS |
| `yarn/` | ResourceManager、调度器、Container、状态机、联邦 |
| `yarn/federation/` | YARN Federation 源码解读 |
| `zookeeper/` | ZooKeeper 启动流程 + 版本差异 |
| `ozone/` | Ozone 分布式对象存储（简介/模块/功能/CLI+安全） |
| `research/` | 学术论文 PDF + 中文翻译/解析 + 分析报告 |
| `research/hdfs/` | HDFS 专项论文综述（含 README 索引） |
| `watch/` | 版本特性追踪、兼容性分析（3.3.1→3.4.1→3.5.0）、Issue 追踪 |

各模块根目录有 `README.md` 作为索引（**`watch/` 无 README.md**——新增文档直接在根 `README.md` 知识目录树添加条目）。

### 配图源文件
`common/attach/`、`hdfs/attach/`、`yarn/attach/` 存放 `.eddx` 文件（EdrawMax 格式）。图片托管于 `pan.zeekling.cn/zeekling/hadoop/`，仓库内仅存引用 URL `![pic](https://pan.zeekling.cn/zeekling/hadoop/{模块}_xxx_00001.png)`。

### 根目录特殊文件
- `hadoop_300_question.md` — Hadoop 面试题集（当前为占位桩，仅 1 个条目）

## 文件类型与编码
- **Markdown**：全部中文。**UTF-8 无 BOM**（曾因 GBK/UTF-8 混合编码导致显示异常）
- **PDF**：仅存 `research/`，二进制文件，不要用文本工具编辑
- **图片**：不直接存储，通过外部 URL 引用

## Git 工作流
- **禁止推 master**——仅通过 Gitea PR 合并
- 分支命名：`feature/描述`
- 提交信息：中文简洁描述动作+对象（例："添加DataNode优化详解文档"）
- 远程：`origin` → Gitea（`ssh://git@git.zeekling.cn:222/big-data/hadoop_book.git`），`github` → GitHub 镜像
- **PR 编码**：标题和正文必须 UTF-8（避免中文/emoji 乱码）。API 请求建议用 `curl.exe` 或指定 `Content-Type: application/json; charset=utf-8`

## 文件命名
命名风格不一致：仓库混用 `snake_case`（`fair_call_queue.md`）和 `kebab-case`（`container-executor.md`），新增文件参考所在模块已有风格。
- 技术详解：`{模块}/{组件}详解.md`
- 论文翻译：`research/{原文前缀}_翻译解析.md` 或 `research/{英文标题}论文全文中文翻译.md`
- 版本追踪：`watch/{主题}.md`

## 内容规范
- 编码：UTF-8 无 BOM | 代码示例：Java
- 类引用：全限定名 `org.apache.hadoop.hdfs.server.namenode.FSImage`
- 方法引用：`ClassName#methodName`
- 配置引用：反引号包裹 `` `dfs.namenode.name.dir` ``
- 版本号：`3.3.1`（不加 v 前缀）
- 内部链接：相对路径（`watch/` → `hdfs/` 需 `../hdfs/`）
- JIRA issue 表格：加粗关键 issue（`**HDFS-15382**`）

## 新增文档必做
1. 在对应模块 `README.md` 添加链接（`watch/` 无 README.md，跳过此步）
2. 在根 `README.md` 知识目录树添加条目（按模块分类表格）
3. 文档内推荐阅读使用正确的相对路径

## 新增论文翻译必做
1. PDF 存于 `research/`，命名保持原文标题
2. 翻译文件名遵循命名规则
3. 在 `research/README.md` 论文列表添加条目（PDF → 翻译双向链接）
4. 在根 `README.md`「研究论文」表格添加条目
5. 注意：`research/README.md` 论文列表可能不全，根 `README.md` 表格为权威入口，需同时更新

## OpenCode 配置
- 仓库无 `opencode.json`（通过 `AGENTS.md` 内部指令驱动）
- 仓库无 `.opencode/` 目录（配置文件在用户级全局目录，不跟踪到仓库）

## 模块 README 概况
| 文件 | 行数 | 类型 |
|------|------|------|
| `hdfs/README.md` | 854 | 类和方法级详细列表 |
| `yarn/README.md` | 2051 | 含调度器详解（最大模块 README） |
| `research/README.md` | 18 | 论文列表（部分缺失，根 README 为权威入口） |
| `research/hdfs/README.md` | 95 | HDFS 小文件问题综述索引 |
| `common/README.md` | 13 | 轻索引（仅列出 3 个文档，实际根 README 列出了 6 个） |
| `ozone/README.md` | 5 | 轻索引 |
| `zookeeper/README.md` | 5 | 轻索引 |
| `common/applicationhistoryservice/README.md` | 空占位 |
注意：新增文档需同时在根 `README.md` 和模块 `README.md` 添加链接。

## JIRA 数据获取
```
https://issues.apache.org/jira/rest/api/2/search?jql=project=HDFS AND component=datanode AND fixVersion=3.4.0&fields=key,summary,priority&maxResults=100
```
- 大项目（HADOOP/HDFS）可能超 100 条，需分页 `&startAt=100`
- 请求偶尔超时，需重试或缩小参数范围