# HDFS FSImage 工具详解

## 概述

Hadoop 提供了一套围绕 FSImage 的离线分析、转换和验证工具，全部位于 `hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/tools/offlineImageViewer/` 包下。这些工具**无需运行 HDFS 集群**即可操作 FSImage 文件，覆盖了**查看→分析→校验→逆向重建→模拟**全链路。

| 工具 | 入口类 | 用途 |
|------|--------|------|
| `oiv` | `OfflineImageViewerPB` | 新版 FSImage 查看器（Protobuf 格式） |
| `oiv_legacy` | `OfflineImageViewer` | 旧版 FSImage 查看器（旧二进制格式） |
| `OfflineImageReconstructor` | `OfflineImageReconstructor` | XML → 二进制 FSImage 逆向重构 |
| `FsImageValidation` | `server/namenode/FsImageValidation` | NameNode 启动时镜像完整性校验 |
| `LegacyOIVImage` | `FSImage.saveLegacyOIVImage()` | 生成兼容旧版查看器的镜像 |
| `Dynamometer` | `hadoop-tools/hadoop-dynamometer/` | 基于 FSImage 的大规模集群模拟 |

文件路径：
- 工具源码：`hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/tools/offlineImageViewer/`
- 校验器：`.../server/namenode/FsImageValidation.java`
- Dynamometer：`hadoop-tools/hadoop-dynamometer/`

---

## 一、`oiv` — 新版离线查看器（`OfflineImageViewerPB`）

### 1.1 架构总览

```
OfflineImageViewerPB (入口, 298行)
  ├── main()                     命令行入口
  │     └── buildOptions()       解析 -i -o -p 等参数
  ├── selectProcessor()          根据 -p 值选择 7 种处理器之一
  └── processImage()             程序化 API 入口
```

**命令格式**：
```
bin/hdfs oiv [OPTIONS] -i INPUTFILE -o OUTPUTFILE

选项：
  -i,--inputFile   FSImage 或 XML 文件路径
  -o,--outputFile  输出文件路径（默认 stdout）
  -p,--processor   处理器类型（默认 Web）
  -addr            监听地址（仅 Web 处理器）
  -maxSize         最大文件大小（仅 FileDistribution）
  -step            文件分布粒度（仅 FileDistribution）
  -format          人类可读输出（仅 FileDistribution）
  -delimiter       分隔符（仅 Delimited/DetectCorruption）
  -sp              输出存储策略（仅 Delimited）
  -ec              输出纠删码策略（仅 Delimited）
  -t,--temp        临时缓存目录
  -m,--multiThread 多线程并行处理
  -h,--help        帮助信息
```

### 1.2 处理器调度机制

```java
// OfflineImageViewerPB.java 第 157-230 行
private static void selectProcessor(String processor, ...) {
    switch (processor.toUpperCase()) {
        case "XML"               → PBImageXmlWriter
        case "FILEDISTRIBUTION"  → FileDistributionCalculator
        case "REVERSEXML"        → OfflineImageReconstructor
        case "WEB"               → WebImageViewer
        case "DELIMITED"         → PBImageDelimitedTextWriter
        case "DETECTCORRUPTION"  → PBImageCorruptionDetector
    }
}
```

### 1.3 各处理器详解

---

#### 处理器 1：XML

**类**：`PBImageXmlWriter.java`

将 FSImage 转换为完整的 XML 文档，包含所有元数据信息。

**输出结构**：
```xml
<fsimage>
  <NameSection>
    <namespaceId>1076270814</namespaceId>
    <genstampV1>1000</genstampV1>
    <genstampV2>2000</genstampV2>
    <transactionId>1234</transactionId>
    <lastInodeId>16386</lastInodeId>
  </NameSection>
  <INodeSection>
    <inode>
      <id>16386</id>
      <type>DIRECTORY</type>
      <name></name>
      <replication>0</replication>
      <mtime>1700000000000</mtime>
      <atime>1700000000000</atime>
      <permission>root:supergroup:0755</permission>
      <nsquota>9223372036854775807</nsquota>
      <dsquota>-1</dsquota>
    </inode>
    <inode>
      <id>16387</id>
      <type>FILE</type>
      <name>test.txt</name>
      <replication>3</replication>
      <blocks>
        <block><id>1073741825</id><numBytes>128</numBytes><genstamp>1001</genstamp></block>
      </blocks>
    </inode>
  </INodeSection>
  <SnapshotSection>
    <snapshotCounter>0</snapshotCounter>
    <snapshottableDirNum>0</snapshottableDirNum>
  </SnapshotSection>
</fsimage>
```

**特点**：
- 完整无损，可被 XML 工具链处理（XSLT、XPath 等）
- 输出体积最大（XML 语法冗余）
- **可用于 ReverseXML 逆向还原**

---

#### 处理器 2：FileDistribution

**类**：`FileDistributionCalculator.java`

分析 FSImage 中所有文件的**大小分布统计**。

**参数**：
| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-maxSize` | 128GB | 文件大小分析范围 [0, maxSize] |
| `-step` | 2MB | 桶的粒度 |
| `-format` | false | 是否使用人类可读单位（KB/MB/GB） |

**算法**：
```java
bucket = min(fileSize / step, maxBuckets - 1);
distribution[bucket]++;
```

**输出示例**（`-format` 模式）：
```
Size Range            NumFiles
(0 B, 2 MB]          45231
(2 MB, 4 MB]         12890
(4 MB, 6 MB]          5672
...
(126 MB, 128 MB]        23
(128 MB, ∞)              4
totalFiles = 67890
totalDirectories = 1234
totalBlocks = 98765
totalSpace = 1099511627776
maxFileSize = 10737418240
```

---

#### 处理器 3：ReverseXML

**类**：`OfflineImageReconstructor.java`（1857 行）

XML → 二进制 FSImage 的**逆向转换器**。与 XML 处理器配对使用。

**完整流程**：

```
process(XML文件) {
  Step 1: parseXML()
          → DocumentBuilderFactory.newInstance().newDocumentBuilder()
          → document = builder.parse(xmlFile)

  Step 2: 验证兼容性
          layoutVersion 必须 ≤ oiv 工具的 LAYOUT_VERSION
          ondiskVersion 必须匹配

  Step 3: NameSection 处理
          读取 namespaceID、genstampV1/V2、stampAtBlockIdSwitch、
          lastAllocatedBlockId、transactionId、lastInodeId
          写入二进制文件头

  Step 4: INodeSection 处理（递归）
          遍历 XML <inode> 元素
          type="DIRECTORY"  → 递归创建子 INodeDirectory
          type="FILE"       → 创建 INodeFile + BlockInfo 列表
          type="SYMLINK"    → 创建 INodeSymlink
          序列化到二进制输出流

  Step 5: SnapshotSection 处理
          重建 Snapshot 元数据
          重建 DirectoryDiff 列表
          重建 FileDiff 列表

  Step 6: 写入 FileSummary（section 索引）
          计算 MD5 校验和
          写入 .md5 文件
}
```

**容错**：XML 格式错误或 I/O 异常均抛出中断，不做静默容错。

---

#### 处理器 4：Web

**类**：`WebImageViewer.java`（143 行） + `FSImageLoader.java`（687 行） + `FSImageHandler.java`

启动 HTTP 服务器，暴露**只读 WebHDFS REST API**，支持在浏览器中交互浏览命名空间。

**启动方式**：
```bash
bin/hdfs oiv -i fsimage
# 默认监听 localhost:5978
# 可通过 -addr host:port 指定地址
```

**架构**：
```
WebImageViewer
  └── HttpServer2 (Jetty 嵌入式)
        └── FSImageHandler (HttpServlet)
              └── FSImageLoader (只读命名空间)
```

**支持的 REST API**：

| API | 方法 | 说明 |
|-----|------|------|
| `GET /webhdfs/v1/<path>?op=LISTSTATUS` | `FSImageLoader.getListing()` | 列出目录内容 |
| `GET /webhdfs/v1/<path>?op=GETFILESTATUS` | `FSImageLoader.getFileStatus()` | 获取文件状态 |
| `GET /webhdfs/v1/<path>?op=GETCONTENTSUMMARY` | `FSImageLoader.getContentSummary()` | 计算目录空间使用 |
| `GET /webhdfs/v1/<path>?op=GETSNAPSHOTDIFF` | `FSImageLoader.listDiffStatus()` | 快照差异 |
| `GET /webhdfs/v1/<path>?op=GETXATTRS` | `FSImageLoader.getXAttrs()` | 获取扩展属性 |
| `GET /webhdfs/v1/<path>?op=LISTXATTRS` | `FSImageLoader.listXAttrs()` | 列出扩展属性 |

**FSImageLoader 内部结构**：
```java
public class FSImageLoader {
    // 核心字段
    private final INodeMap inodeMap;           // 所有 INode（id → INode 对象）
    private final Map<Long, IMU> inodeMapMU;   // 优化用的同步锁映射
    private final Map<Long, byte[]> xattrMap;  // xattr 缓存

    // 初始化
    load(File file, Configuration conf) {
        // 1. 解析 FileSummary
        // 2. 解析 StringTableSection（用户/组名去重映射）
        // 3. 解析 INodeSection → 构建 inodeMap
        // 4. 解析 INodeDirSection → 构建目录树
        // 5. 解析 SnapshotSection
    }
}
```

**限制**：
- 不启用安全模式
- 不支持 HTTPS
- 仅只读

**使用示例**：
```bash
# 命令行
hdfs dfs -ls webhdfs://127.0.0.1:5978/
hdfs dfs -ls -R webhdfs://127.0.0.1:5978/

# HTTP REST
curl -i "http://127.0.0.1:5978/webhdfs/v1/?op=liststatus"
curl -i "http://127.0.0.1:5978/webhdfs/v1/user?op=getfilestatus"
```

---

#### 处理器 5：Delimited

**类**：`PBImageDelimitedTextWriter.java`（151 行）

**继承链**：
```
PBImageTextWriter (475行, 抽象基类)
  └── PBImageDelimitedTextWriter
  └── PBImageCorruptionDetector (同一基类，不同输出逻辑)
```

**PBImageTextWriter 基类架构**：
```java
public abstract class PBImageTextWriter {
    // 多线程并行加载（-m 参数控制线程数）
    loadSubSections() {
        ExecutorService exec = Executors.newFixedThreadPool(numThreads);
        for (SubSection section : subSections) {
            exec.submit(() -> loadSubSection(section));
        }
        exec.shutdown();
        exec.awaitTermination(Long.MAX_VALUE, TimeUnit.SECONDS);
    }

    // 子类实现的两个核心抽象方法
    abstract processDirectory(DB, dirNode);  // 目录处理
    abstract processFiles(DB, fileNode, ...); // 文件处理
}
```

**输出格式**（Tab 分隔，每行一个文件/目录）：
```
Path  Replication  ModificationTime  AccessTime  PreferredBlockSize
BlocksCount  FileSize  NSQUOTA  DSQUOTA  Permission  User  Group
```

**可选扩展列**（`-sp` / `-ec`）：
```
StoragePolicy  ErasureCodingPolicy
```

---

#### 处理器 6：DetectCorruption

**类**：`PBImageCorruptionDetector.java`（344 行）

**继承 PBImageTextWriter**，在遍历目录树时检测**命名空间不一致性**。

**检测逻辑**：
```java
// processDirectory() 阶段
for each child in directory.children：
    if child.id not in inodeMap：
        → 记录 CorruptionEntry(missingNode, parentPath, childId, isDir)

// 输出格式
CorruptionType \t Path \t MissingNodeId \t IsDir
```

**类型**：
- `MISSING_INODE` — 子节点在 inodeMap 中不存在

**限制**：非穷举检查，仅捕获命名空间重建过程中丢失的节点。

---

## 二、`oiv_legacy` — 旧版离线查看器（`OfflineImageViewer`）

### 2.1 架构：Visitor 模式

```
ImageVisitor (抽象基类, 279行)
  ├── TextWriterImageVisitor (文本输出抽象)
  │     ├── LsImageVisitor            (默认, ls -R 风格)
  │     ├── DelimitedImageVisitor     (定界符分隔, -delimiter)
  │     └── IndentedImageVisitor      (缩进树形)
  ├── XmlImageVisitor                 (XML 输出)
  ├── FileDistributionVisitor         (文件分布)
  └── NameDistributionVisitor         (名称分布)
```

### 2.2 Visitor 生命周期

```
visitor.start()         ← 文件头初始化

for each INode in FSImage (拓扑顺序):
    visitor.visit(inode) ← 处理每个 INode

visitor.finish()        ← 关闭输出
```

### 2.3 各 Visitor 输出示例

**LsImageVisitor**（默认）：
```
drwxr-xr-x  - root supergroup          0 2024-01-15 10:00 /user
-rw-r--r--  3 root supergroup       1024 2024-01-15 10:01 /user/file1.txt
-rw-r--r--  2 root supergroup      65536 2024-01-15 10:02 /user/file2.dat
```

**IndentedImageVisitor**：
```
/
  user
    foo
      bar.txt  1024 bytes
    baz.dat   65536 bytes
  tmp
    temp.log   512 bytes
```

**DelimitedImageVisitor**（Tab 分隔）：
```
Path\tReplication\tModificationTime\t...
/user\t0\t1700000000000\t...
/user/file1.txt\t3\t1700000001000\t1024\t...
```

**NameDistributionVisitor**（文件名长度分布）：
```
Length\tCount
1\t2
3\t5
5\t23
8\t156
16\t789
```

### 2.4 与新版 oiv 的对比

| 对比项 | `oiv` (OfflineImageViewerPB) | `oiv_legacy` (OfflineImageViewer) |
|--------|------------------------------|-----------------------------------|
| 支持格式 | Protobuf（Hadoop 2.4+） | 旧二进制格式（Hadoop 2.4 之前） |
| 架构 | 函数式调度 `selectProcessor()` | Visitor 模式 |
| 处理器数量 | 7 种 | 6 种（Ls/XML/Delimited/Indented/FileDist/NameDist） |
| Web 服务 | 支持 | 不支持 |
| 逆向重构 | 支持（ReverseXML） | 不支持 |
| 多线程 | 支持（Delimited/DetectCorruption） | 不支持 |
| 推荐状态 | **当前推荐** | **已废弃**，仅兼容旧版本 |

---

## 三、`OfflineImageReconstructor` — FSImage 逆向重构器

**类**：`OfflineImageReconstructor.java`（1857 行，目录 `tools/offlineImageViewer/`）

### 3.1 用途

将 XML 处理器输出的 XML 文件还原为**二进制 FSImage 文件**，使得用户可以：

1. **手动编辑 FSImage** — 导出 XML → 编辑 → 重构为二进制 → 注入 NameNode
2. **自动化测试** — 通过 XML 模板生成特定命名空间的 FSImage
3. **损坏修复** — 在 XML 级别修复损坏数据后重建

### 3.2 重构流程详解

```
Step 1: XML 解析
        DocumentBuilderFactory.newInstance()
        .newDocumentBuilder().parse(xmlFile)
        ↓ 得到 DOM Document

Step 2: 版本校验
        int expectedLayoutVersion = LAYOUT_VERSION;
        if (xmlLayoutVersion > expectedLayoutVersion) {
            throw IOException("版本不兼容");
        }

Step 3: 写入文件头
        ┌─────────────────────────────────┐
        │ layoutVersion: int              │
        │ namespaceID: int                │
        │ numINodes: long                 │
        │ genstampV1: long                │
        │ genstampV2: long                │
        │ stampAtBlockIdSwitch: long      │
        │ lastAllocatedBlockId: long      │
        │ transactionId: long             │
        │ lastInodeId: long               │
        │ snapshotManager state           │
        └─────────────────────────────────┘

Step 4: INode 重构
        for each <inode> in XML:
            type = "DIRECTORY"
                → new INodeDirectory(...)
                → 递归处理子节点
                → 填充配额 (nsQuota, dsQuota)
                → 填充权限 (permission string → PermissionStatus)
            type = "FILE"
                → new INodeFile(...)
                → 创建 BlockInfo 列表
                → 设置 replication, blockSize
            type = "SYMLINK"
                → new INodeSymlink(...)
                → 设置 symlinkString

Step 5: Snapshot 重构
        重建 SnapshotCounter
        重建每个 Snapshot 的根目录
        重建 DirectoryDiff / FileDiff

Step 6: 收尾
        写入 FileSummary (section 索引, 记录偏移量和长度)
        计算 MD5Digest
        写入 .md5 文件
```

### 3.3 应用场景示例

```bash
# 导出 XML
hdfs oiv -i fsimage_0001 -o fsimage_0001.xml -p XML

# 编辑 XML（如修改权限、添加文件）
vim fsimage_0001.xml

# 逆向重构为二进制
hdfs oiv -i fsimage_0001.xml -o fsimage_0001_rebuilt -p ReverseXML

# 将重构的 FSImage 用于 NameNode 启动
# （需要替换原 fsimage 文件）
```

---

## 四、`FsImageValidation` — 镜像完整性校验器

**类**：`FsImageValidation.java`（448 行，包 `server/namenode/`）

### 4.1 位置与时机

FsImageValidation 运行在 **NameNode 进程内**，在 `FSImage.recoverTransitionRead()` 阶段调用，作为启动过程的一部分。

### 4.2 校验范围

```
validate()
  ├── validateINodes()              ← INode 树完整性
  │     ├── validateDirectory()     ← 目录校验
  │     │     ├── checkChildCount()           ← 子节点数量一致性
  │     │     ├── checkParentChildLinks()     ← 父子引用双向校验
  │     │     ├── checkSnapshotLinks()        ← 快照引用一致性
  │     │     └── checkQuota()                ← 配额计数
  │     ├── validateFile()          ← 文件校验
  │     │     ├── checkBlockReplication()     ← 副本数
  │     │     └── checkBlockReferences()      ← BlockInfo 引用
  │     └── validateSymlink()       ← 符号链接校验
  │           └── checkTargetPath()           ← 目标路径有效性
  ├── validateFilesUnderConstruction() ← 构建中文件校验
  │     ├── checkLeases()                    ← Lease 一致性
  │     └── checkUCPaths()                   ← 路径正确性
  ├── validateSecretManager()    ← 委托令牌校验
  │     ├── checkDelegationKeys()            ← 密钥一致性
  │     └── checkDelegationTokens()          ← Token 有效性
  └── validateCacheManager()     ← 缓存指令校验
        ├── checkCacheDirectives()           ← 指令一致性
        └── checkCachePools()                ← 池存在性
```

### 4.3 关键校验逻辑

```java
// 第 195-230 行：递归目录校验
private void validateDirectory(INodeDirectory dir) {
    // 1. 校验子节点数量
    int actualChildren = dir.getChildrenList(Snapshot.CURRENT_STATE_ID).size();
    if (actualChildren != dir.getChildrenNum()) {
        corruptionReporter.report("children num mismatch", dir);
    }

    // 2. 校验父引用
    for (INode child : dir.getChildrenList(...)) {
        if (child.getParent() != dir) {
            corruptionReporter.report("parent link broken", child);
        }
    }

    // 3. 校验快照引用
    DirectoryWithSnapshotFeature sf = dir.getDirectoryWithSnapshotFeature();
    if (sf != null) {
        for (DirectoryDiff diff : sf.getDiffs()) {
            // 校验 diff 中的引用节点可达性
        }
    }

    // 4. 校验配额
    QuotaCounts q = dir.getQuotaCounts();
    if (q.getNameSpace() < -1 || q.getStorageSpace() < -1) {
        corruptionReporter.report("invalid quota", dir);
    }
}
```

### 4.4 校验结果处理

| 结果 | 行为 |
|------|------|
| 无错误 | NameNode 正常启动 |
| 可修复错误 | 记录 WARN 日志，尝试修复后继续 |
| 不可修复错误 | 记录 FATAL 日志，NameNode 启动失败 |

---

## 五、`LegacyOIVImage` — 旧版兼容镜像生成

### 5.1 背景

新版 `oiv` 只支持 Protobuf 格式的 FSImage（Hadoop 2.4+），而一些旧工具仍需要旧版二进制格式。为此，NameNode 在 checkpoint 时可选生成**旧版格式的副本**供旧版 `oiv_legacy` 使用。

### 5.2 触发路径

```
SecondaryNameNode.doCheckpoint()    ← SNN 定期 checkpoint
  └── FSImage.saveLegacyOIVImage()
        ├── 1. 读取当前 Protobuf 格式 fsimage
        ├── 2. 用旧版 FSImageFormat.Saver 重新序列化
        │     (layoutVersion = -51)
        ├── 3. 写入文件 fsimage_legacy_oiv_TXID
        ├── 4. 计算 MD5 校验和
        └── 5. 清理过期旧版镜像

StandbyCheckpointer.doCheckpoint()   ← HA Standby NN checkpoint
  └── 同上
```

### 5.3 文件命名

```java
// NNStorage.java 第 774 行
public static String getLegacyOIVImageFileName(long txid) {
    return getNameNodeFileName(NameNodeFile.IMAGE_LEGACY_OIV, txid);
}
// 输出示例: fsimage_legacy_oiv_0000000000000000123
```

### 5.4 配置

```xml
<property>
  <name>dfs.namenode.legacy-oiv-image.dir</name>
  <value>/path/to/legacy/oiv/dir</value>
  <description>旧版 OIV 镜像输出目录</description>
</property>
```

---

## 六、Dynamometer — 大规模集群模拟工具

### 6.1 概述

Dynamometer 是 Hadoop 生态中的一个独立工具集（`hadoop-tools/hadoop-dynamometer/`），使用**真实的 FSImage 数据**在 YARN 容器中启动模拟 HDFS 集群，用于：

- NameNode 性能基准测试
- 集群规模容量规划
- 新特性/配置变更的影响评估

### 6.2 与 FSImage 的关系

```
┌─────────────────────────────────────────────────────────┐
│                   数据准备阶段                             │
├─────────────────────────────────────────────────────────┤
│ Step 1: 从 NameNode 收集                                │
│   ├── fsimage_TXID          (二进制 FSImage)             │
│   ├── fsimage_TXID.md5      (MD5 校验和)                 │
│   └── VERSION               (集群元信息)                  │
│                                                          │
│ Step 2: 转换为 XML                                       │
│   hdfs oiv -i fsimage_TXID -o fsimage_TXID.xml -p XML   │
│                                                          │
│ Step 3: 部署到 HDFS                                       │
│   /dynamometer/fsimage/fsimage_TXID.xml                  │
│   /dynamometer/fsimage/VERSION                           │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                   模拟运行阶段                             │
├─────────────────────────────────────────────────────────┤
│ DynamometerInfra (YARN ApplicationMaster)                │
│   └── 启动 MiniHDFSCluster 在 YARN Container 中          │
│         ├── NameNode 加载 fsimage                        │
│         ├── DataNode 模拟块存储                          │
│         └── 运行 Benchmark 工作负载                       │
└─────────────────────────────────────────────────────────┘
```

### 6.3 组件

| 组件 | 目录 | 职责 |
|------|------|------|
| `dynamometer-infra` | `.../dynamometer-infra/` | YARN ApplicationMaster，管理模拟集群生命周期 |
| `dynamometer-workload` | `.../dynamometer-workload/` | 客户端工作负载模拟器 |
| `dynamometer-blockgen` | `.../dynamometer-blockgen/` | DataNode 块数据生成器 |

---

## 七、工具关系全景图

```
                ┌──────────────────────────────────────┐
                │           FSImage 文件                │
                │  (fsimage_TXID, Protobuf 格式)        │
                └──────────┬───────────────────────────┘
                           │
           ┌───────────────┼───────────────────┐
           │               │                    │
           ▼               ▼                    ▼
    ┌────────────┐  ┌──────────────┐    ┌──────────────┐
    │ oiv        │  │ oiv_legacy   │    │ NameNode     │
    │ (新式查看器)│  │ (旧版查看器)   │    │ 启动时加载   │
    └─────┬──────┘  └──────┬───────┘    └──────┬───────┘
          │                │                    │
          ▼                ▼                    ▼
   ┌───────────┐    ┌────────────┐     ┌──────────────┐
   │ XML       │    │ Ls         │     │ FsImageValid │
   │ Web       │    │ XML        │     │ ation 校验器  │
   │ Delimited │    │ Indented   │     └──────────────┘
   │ FileDist  │    │ Delimited  │
   │ DetectCorr│    │ FileDist   │
   │ ReverseXML│───→│ NameDist   │
   └─────┬─────┘    └────────────┘
         │
         ▼
  ┌──────────────┐
  │ OfflineImage │
  │ Reconstructor│  (XML→二进制 FSImage)
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ 二进制 FSImage│  (可注入 NameNode 启动)
  └──────────────┘
         │
         ▼
  ┌──────────────┐
  │ Dynamometer  │  (YARN 容器中启动模拟集群)
  └──────────────┘
```

---

## 八、类/文件清单

| 文件路径 | 类名 | 行数 | 职责 |
|----------|------|------|------|
| `tools/offlineImageViewer/OfflineImageViewerPB.java` | `OfflineImageViewerPB` | 298 | 新版 oiv 入口 |
| `tools/offlineImageViewer/PBImageXmlWriter.java` | `PBImageXmlWriter` | - | XML 输出 |
| `tools/offlineImageViewer/PBImageTextWriter.java` | `PBImageTextWriter` | 475 | 文本输出抽象基类（多线程） |
| `tools/offlineImageViewer/PBImageDelimitedTextWriter.java` | `PBImageDelimitedTextWriter` | 151 | 定界文本输出 |
| `tools/offlineImageViewer/PBImageCorruptionDetector.java` | `PBImageCorruptionDetector` | 344 | 损坏检测 |
| `tools/offlineImageViewer/PBImageCorruption.java` | `PBImageCorruption` | - | 损坏条目 POJO |
| `tools/offlineImageViewer/FileDistributionCalculator.java` | `FileDistributionCalculator` | - | 文件大小分布分析 |
| `tools/offlineImageViewer/OfflineImageReconstructor.java` | `OfflineImageReconstructor` | 1857 | XML→二进制逆向重构 |
| `tools/offlineImageViewer/WebImageViewer.java` | `WebImageViewer` | 143 | Web 服务器 |
| `tools/offlineImageViewer/FSImageLoader.java` | `FSImageLoader` | 687 | 只读命名空间加载器 |
| `tools/offlineImageViewer/FSImageHandler.java` | `FSImageHandler` | - | WebHDFS Servlet |
| `tools/offlineImageViewer/OfflineImageViewer.java` | `OfflineImageViewer` | 279 | 旧版 oiv 入口 |
| `tools/offlineImageViewer/ImageVisitor.java` | `ImageVisitor` | - | Visitor 抽象基类 |
| `tools/offlineImageViewer/TextWriterImageVisitor.java` | `TextWriterImageVisitor` | - | 文本输出抽象 |
| `tools/offlineImageViewer/LsImageVisitor.java` | `LsImageVisitor` | - | ls 风格输出 |
| `tools/offlineImageViewer/DelimitedImageVisitor.java` | `DelimitedImageVisitor` | - | 定界文本（旧版） |
| `tools/offlineImageViewer/IndentedImageVisitor.java` | `IndentedImageVisitor` | - | 缩进树形输出 |
| `tools/offlineImageViewer/XmlImageVisitor.java` | `XmlImageVisitor` | - | XML（旧版） |
| `tools/offlineImageViewer/FileDistributionVisitor.java` | `FileDistributionVisitor` | - | 文件分布（旧版） |
| `tools/offlineImageViewer/NameDistributionVisitor.java` | `NameDistributionVisitor` | - | 名称分布 |
| `tools/offlineImageViewer/ImageLoader.java` | `ImageLoader` | - | 旧版 fsimage 加载接口 |
| `tools/offlineImageViewer/ImageLoaderCurrent.java` | `ImageLoaderCurrent` | - | 旧版 fsimage 加载实现 |
| `server/namenode/FsImageValidation.java` | `FsImageValidation` | 448 | 启动时完整性校验 |
| `server/namenode/FSImage.java` | `FSImage` | 1606 | `saveLegacyOIVImage()` 方法 |
| `server/namenode/NNStorage.java` | `NNStorage` | - | `IMAGE_LEGACY_OIV` 文件类型定义 |
| `server/namenode/NNStorageRetentionManager.java` | `NNStorageRetentionManager` | - | 旧版 OIV 镜像清理 |
