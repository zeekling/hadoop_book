# AGENTS.md

## 仓库性质
**纯文档仓库**——分析 Apache Hadoop 3.3.1 源码并撰写架构文档。非 Hadoop 源码库，无需构建/测试/编译。
- 内容：Markdown（技术文档/论文翻译/问题分析）+ PDF（仅存 `research/`）
- 无 CI/CD、package.json、lint 配置、任何可执行的构建命令
- README.md 配有 shields.io 徽章（GitHub 统计），但无 CI 流水线

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
| `watch/` | 版本特性追踪、兼容性分析（3.3.1→3.4.1→3.5.0）、Issue 追踪 |

各模块根目录有 `README.md` 作为索引（`watch/` 除外——它无 README.md，新增文档直接在根 `README.md` 知识目录树添加条目）。

### 配图源文件
`common/attach/`、`hdfs/attach/`、`yarn/attach/` 存放 `.eddx` 文件（EdrawMax 格式），文件名中文。生成的图片托管于 `pan.zeekling.cn/zeekling/hadoop/`，仓库内仅存引用 URL。

### 根目录特殊文件
- `hadoop_300_question.md` — Hadoop 面试题集（当前为占位桩，仅 1 个条目）
- `Origami_ICPP25.md` — Origami 论文笔记（ICPP'25），存根目录而非 `research/`

## File types / 文件类型
- **Markdown**：技术文档、论文翻译。全部中文。**UTF-8 无 BOM**（曾因 GBK/UTF-8 混合编码导致显示异常）。
- **PDF**：仅存 `research/`，二进制文件，不要用文本工具编辑。
- **图片**：不直接存储，通过 URL `![pic](https://pan.zeekling.cn/zeekling/hadoop/...)` 引用。

## Git & PR 工作流
- **禁止推 master**：工作流规范——仅通过 Gitea PR 合并（不接受直接 push master）
- 分支命名：`feature/描述`（例外：`shuffle` 为历史遗留）
- 提交信息：中文简洁描述动作+对象（例："添加DataNode优化详解文档"）
- 远程：`origin` → Gitea（`ssh://git@git.zeekling.cn:222/big-data/hadoop_book.git`），`github` → GitHub 镜像
- **PR 编码**：创建/更新 PR 时，标题和正文必须 UTF-8（避免中文/emoji 乱码）。API 请求建议用 `curl.exe` 或指定 `Content-Type: application/json; charset=utf-8`

## 文件命名约定
- 模块索引：`{模块}/README.md`
- 技术详解：`{模块}/{组件}详解.md`（中英文混排，如 `NameNode 组件详解.md`）
- 论文翻译：`research/{原文前缀}_翻译解析.md`；例外格式 `{英文标题}论文全文中文翻译.md`（如 `Fletch论文全文中文翻译.md`）
- 版本追踪：`watch/{主题}.md`
- 命名风格不一致：仓库混用 `snake_case`（`fair_call_queue.md`）和 `kebab-case`（`container-executor.md`），新增文件参考所在模块已有风格

## 内容规范
- 语言：全部中文 | 编码：UTF-8 无 BOM | 代码示例：Java
- 类引用：全限定名 `org.apache.hadoop.hdfs.server.namenode.FSImage`
- 方法引用：`ClassName#methodName`
- 配置引用：反引号包裹 `` `dfs.namenode.name.dir` ``
- 版本号：`3.3.1`（不加 v 前缀）
- 图片命名：`{模块}_xxx_00001.png`，URL 前缀 `https://pan.zeekling.cn/zeekling/hadoop/`
- 内部链接：相对路径（`watch/` → `hdfs/` 需 `../hdfs/`）
- JIRA issue：表格中加粗关键 issue（`**HDFS-15382**`）

## 新增文档必做
1. 在对应模块 `README.md` 添加链接（`watch/` 无 README.md，跳过此步）
2. 在根 `README.md` 知识目录树添加条目（按模块分类表格）
3. 文档内推荐阅读使用正确的相对路径

## 新增论文翻译必做
1. PDF 存于 `research/`，命名保持原文标题
2. 翻译文件名：`{原文前缀}_翻译解析.md` 或 `{英文标题}论文全文中文翻译.md`
3. 在 `research/README.md` 论文列表表格添加条目（PDF → 翻译双向链接）
4. 在根 `README.md`「研究论文」表格添加条目
5. 注意：`research/README.md` 论文列表可能不全，根 `README.md` 表格为权威入口，需同时更新

## OpenCode 配置
- 仓库无 `opencode.json` 文件（OpenCode 通过当前 `AGENTS.md` 内部指令驱动）
- 仓库无 `.opencode/` 目录（OpenCode 配置文件/插件在用户级全局目录，不跟踪到仓库）

## 模块 README 概况
- `hdfs/README.md` — ~590 行，类和方法级详细列表
- `yarn/README.md` — ~1600 行，含调度器详解（最大模块 README）
- `common/README.md` — 模块索引，较短（仅列出3个文档，实际根 `README.md` 列出了6个；新增文档仍需在根 `README.md` 和 `common/README.md` 同时添加）
- `common/applicationhistoryservice/README.md` — 空占位
- `ozone/README.md` — 模块索引
- `research/README.md` — 论文列表 + 分析报告（部分论文暂无翻译以 `—` 标记；论文列表/分析报告均可能不全）
- `zookeeper/README.md` — 模块索引

## JIRA 数据获取
```bash
# Apache JIRA REST API
https://issues.apache.org/jira/rest/api/2/search?jql=project=HDFS AND component=datanode AND fixVersion=3.4.0&fields=key,summary,priority&maxResults=100
```
- 大项目（HADOOP/HDFS）可能超 100 条，需分页 `&startAt=100`
- 请求偶尔超时，需重试或缩小参数范围
