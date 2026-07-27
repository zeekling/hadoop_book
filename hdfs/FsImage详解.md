# HDFS FsImage 分析报告

## 一、架构概述

FsImage 是 HDFS NameNode 的元数据快照文件，记录了整个文件系统的目录树和文件块映射关系。其核心功能围绕 **加载（Load）** 和 **保存（Save）** 两个主线展开，分布在 `namenode` 包下的约 15 个主要类中。

```
FSImage 子系统核心文件：
  FSImage.java                 — 主控类，协调所有加载/保存/恢复/升级操作
  FSImageFormat.java           — LoaderDelegator + 旧版格式
  FSImageFormatProtobuf.java   — Protobuf 格式的 Loader/Saver
  FSImageFormatPBINode.java    — INode 序列化（PB 格式）
  FSImageFormatPBSnapshot.java — Snapshot 序列化（PB 格式）
  FSImageUtil.java             — 文件头/摘要/压缩包装工具
  FSImageCompression.java      — 压缩编码管理
  FSImageStorageInspector.java — 存储检查抽象
  FSImageTransactionalStorageInspector.java — 当前格式检查器
  FSImagePreTransactionalStorageInspector.java — 旧格式检查器
  FsImageValidation.java       — 镜像校验
  fsimage.proto                — Protobuf schema 定义

传输/工具相关：
  ImageServlet.java            — HTTP 镜像传输端点
  TransferFsImage.java         — 镜像下载/上传客户端
  FSImageLoader.java           — OfflineImageViewer 加载器
  FSImageHandler.java          — OIV Web 处理
  BackupImage.java             — BackupNode 镜像
```

文件物理路径：
- 核心源码：`hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/namenode/`
- Proto 定义：`hadoop-hdfs-project/hadoop-hdfs/src/main/proto/fsimage.proto`
- 离线查看器：`hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/tools/offlineImageViewer/`

---

## 二、磁盘文件格式（`fsimage.proto`）

### 2.1 物理布局

```
FILE := MAGIC SECTION* <FileSummary> FileSummaryLength
MAGIC := 'HDFSIMG1' (8 字节)
SECTION := <NameSystemSection> | <INODE> | <INODE_DIR> | ...
FileSummaryLength := 4 字节 int（小端）
```

关键点：
- **MAGIC**：固定 `HDFSIMG1`（`FSImageUtil.MAGIC_HEADER`）
- **FileSummary**：位于文件末尾，包含所有 section 的偏移量和长度、layoutVersion、ondiskVersion、压缩编码
- **所有 Protobuf 消息使用 delimited 格式**（varint 长度前缀 + 消息体）
- **每个 section 可独立压缩**，但 MAGIC 和 FileSummary 不压缩

### 2.2 文件校验

- 每个 FsImage 文件附带一个 `.md5` 校验文件（如 `fsimage_0000000000000000001.md5`）
- 加载时：启动 `DigestThread` 异步计算 MD5，与 `.md5` 文件对比
- 保存时：`MD5FileUtils.saveMD5File()` 写入 MD5 文件
- `FileSummary` 在 Protobuf 序列化后，末尾附加 4 字节的 trunk size

### 2.3 Section 类型

支持 11 种 Section 类型（`FSImageFormatProtobuf.SectionName` 枚举）：

| Section 枚举值 | 名称 | 内容 |
|----------------|------|------|
| NS_INFO | 命名空间信息 | namespaceId、generation stamps、transactionId、lastAllocatedBlockId |
| STRING_TABLE | 字符串表 | 序列号到字符串的映射（用户/组名去重） |
| EXTENDED_ACL | 扩展 ACL | 预留 |
| ERASURE_CODING | EC 策略 | 纠删码策略列表 |
| INODE | INode 数据 | 所有文件/目录/符号链接的序列化 |
| INODE_SUB | INode 子分区 | INode section 的子分区（并行加载用） |
| INODE_REFERENCE | INode 引用 | Snapshot 相关的引用节点 |
| INODE_REFERENCE_SUB | 引用子分区 | 预留 |
| SNAPSHOT | 快照 | 快照计数和快照根目录 |
| SNAPSHOT_DIFF | 快照差异 | 目录 diff 和文件 diff |
| SNAPSHOT_DIFF_SUB | 快照差异子分区 | 预留 |
| INODE_DIR | 目录结构 | 目录父子关系 |
| INODE_DIR_SUB | 目录子分区 | INodeDir section 的子分区（并行加载用） |
| FILES_UNDERCONSTRUCTION | 正在写入文件 | 客户端名和机器名 |
| SECRET_MANAGER | 密钥管理 | DelegationToken 密钥和 token |
| CACHE_MANAGER | 缓存管理 | 缓存指令和池 |

支持并行加载的 **子 section**（`INODE_SUB`、`INODE_DIR_SUB`），将大 section 拆分为多个小块由线程池并行处理。

### 2.4 Proto 消息结构（核心）

**FileSummary**：文件末尾的索引
```protobuf
message FileSummary {
  required uint32 ondiskVersion = 1;    // 文件格式版本（当前=1）
  required uint32 layoutVersion = 2;    // NameNode layout 版本
  optional string codec         = 3;    // 压缩编码器全类名
  message Section {
    optional string name = 1;           // section 名称
    optional uint64 length = 2;         // section 长度
    optional uint64 offset = 3;         // section 起始偏移
  }
  repeated Section sections = 4;        // section 索引列表
}
```

**NameSystemSection**：命名空间全局元数据
```protobuf
message NameSystemSection {
  optional uint32 namespaceId = 1;
  optional uint64 genstampV1 = 2;       // legacy generation stamp
  optional uint64 genstampV2 = 3;       // generation stamp of latest version
  optional uint64 genstampV1Limit = 4;
  optional uint64 lastAllocatedBlockId = 5;
  optional uint64 transactionId = 6;
  optional uint64 rollingUpgradeStartTime = 7;
  optional uint64 lastAllocatedStripedBlockId = 8;
}
```

**INode**：文件/目录/符号链接
```protobuf
message INode {
  enum Type { FILE = 1; DIRECTORY = 2; SYMLINK = 3; };
  required Type type = 1;
  required uint64 id = 2;
  optional bytes name = 3;
  optional INodeFile file = 4;
  optional INodeDirectory directory = 5;
  optional INodeSymlink symlink = 6;
}
```

**INodeFile** 包含：文件 block 列表（Contiguous 或 Striped）、权限、ACL、XAttr、存储策略、EC 策略 ID、UnderConstruction 信息

**INodeDirectory** 包含：修改时间、命名空间配额、磁盘配额、类型配额、权限、ACL、XAttr

**StringTableSection**：字符串去重表，用户/组名通过 ID 引用，减少存储空间

---

## 三、核心类 `FSImage.java` 详细分析

### 3.1 关键字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `editLog` | `FSEditLog` | 编辑日志，与 FsImage 配合使用 |
| `lastAppliedTxId` | `long` | 最后加载的应用 txid |
| `storage` | `NNStorage` | 存储目录管理（IMAGE + EDITS 目录） |
| `archivalManager` | `NNStorageRetentionManager` | 旧文件清理策略 |
| `currentlyCheckpointing` | `Set<Long>` | 并发 checkpoint 保护集合 |
| `exitAfterSave` | `AtomicBoolean` | 检测到镜像损坏时的退出标记 |
| `isUpgradeFinalized` | `boolean` | 升级完成标记 |

### 3.2 启动加载流程

```
recoverTransitionRead(startOpt, target, recovery)
  │
  ├── recoverStorageDirs()
  │     遍历所有存储目录，分析状态（NON_EXISTENT / NOT_FORMATTED / NORMAL），
  │     执行恢复（doRecover），读取 VERSION 文件验证一致性
  │
  ├── processStartupOptionsForUpgrade()
  │     处理 UPGRADE / ROLLBACK / REGULAR 等启动选项
  │
  └── loadFSImage(target, startOpt, recovery)
       │
       ├── storage.readAndInspectDirs(nnfs, startOpt)
       │     使用 FSImageTransactionalStorageInspector 扫描所有存储目录，
       │     找到所有镜像文件并记录 txid
       │
       ├── initEditLog(startOpt)
       │     根据 HA 状态初始化编辑日志（write mode / read mode）
       │
       ├── editLog.selectInputStreams()
       │     选择需要回放的编辑日志流
       │
       ├── 循环 loadFSImageFile()（失败自动尝试下一个镜像副本）
       │    │
       │    ├── storage.readProperties() 读取存储属性
       │    │
       │    └── loadFSImage(curFile, expectedMd5, target, ...)
       │         │
       │         ├── FSImageFormat.newLoader(conf, fsn)
       │         │    └── LoaderDelegator.load(file)
       │         │         ├── 读取前 8 字节检测 MAGIC
       │         │         ├── MATCH → FSImageFormatProtobuf.Loader
       │         │         └── 不匹配 → FSImageFormat.Loader（旧版）
       │         │
       │         ├── loader.load(curFile, requireSameLayoutVersion)
       │         │    加载镜像数据到 FSNamesystem
       │         │
       │         ├── 验证 MD5 校验和
       │         └── storage.setMostRecentCheckpointInfo()
       │
       ├── loadEdits()（加载编辑日志回放）
       │     FSEditLogLoader 按顺序回放 EditLogInputStream 中的操作
       │
       ├── rollingRollback()（如适用）
       │
       └── needsResaveBasedOnStaleCheckpoint()
             判断 checkpoint 是否过时，返回 needToSave
```

### 3.3 保存流程

```
saveNamespace(source, NameNodeFile, canceler)
  │
  ├── editLog.endCurrentLogSegment(true)
  │     结束当前日志段，确保一致性
  │
  ├── getCorrectLastAppliedOrWrittenTxId()
  │     获取当前 txid 作为镜像 txid
  │
  ├── addToCheckpointing(txid) 防并发
  │
  ├── saveFSImageInAllDirs(source, nnf, txid, canceler)
  │    │
  │    ├── 为每个 IMAGE 类型存储目录创建 SaveNamespaceContext
  │    │
  │    ├── 启动 N 个 FSImageSaver 线程（每个目录一个）
  │    │    │
  │    │    └── saveFSImage(context, sd, dstType)
  │    │         │
  │    │         ├── FSImageFormatProtobuf.Saver 实例化
  │    │         ├── FSImageCompression.createCompression(conf)
  │    │         ├── saver.save(newFile, compression)
  │    │         │    ├── saveInternal() → 写各 section
  │    │         │    └── 计算全文件 MD5
  │    │         ├── MD5FileUtils.saveMD5File() 写入 .md5 文件
  │    │         └── storage.setMostRecentCheckpointInfo()
  │    │
  │    ├── waitForThreads() 等待所有线程完成
  │    │
  │    ├── renameCheckpoint(IMAGE_NEW → IMAGE) 原子重命名
  │    │    fsimage_NEW_00000... → fsimage_00000...
  │    │
  │    ├── purgeOldStorage() 清理旧文件
  │    └── archivalManager.purgeCheckpoints() 清理旧 checkpoint
  │
  ├── editLog.startLogSegmentAndWriteHeaderTxn(txid+1)
  │     重新打开编辑日志
  │
  └── removeFromCheckpointing(txid)
```

当检测到保存过程中出现非致命错误（`numErrors > 0`），设置 `exitAfterSave` 标记：

```java
if (exitAfterSave.get()) {
  LOG.error("NameNode process will exit now... The saved FsImage " +
      nnf + " is potentially corrupted.");
  ExitUtil.terminate(-1);
}
```

---

## 四、Protobuf 格式加载器（`FSImageFormatProtobuf.Loader`）

### 4.1 加载入口

```java
class Loader implements FSImageFormat.AbstractLoader {
  // 关键字段
  private final Configuration conf;
  private final FSNamesystem fsn;
  private final LoaderContext ctx;     // 包含 stringTable + refList
  private MD5Hash imgDigest;           // 加载文件的 MD5
  private long imgTxId;                // 最后 txid
  private boolean requireSameLayoutVersion; // 滚动升级回滚时要求版本一致
  private File filename;
}
```

### 4.2 加载步骤

```
load(File file)
  │
  ├── 启动 DigestThread 异步计算文件 MD5（与加载并行进行）
  │
  ├── 打开 RandomAccessFile + FileInputStream
  │
  └── loadInternal(raFile, fin)
       │
       ├── FSImageUtil.checkFileFormat(raFile)
       │     验证前 8 字节是否为 "HDFSIMG1"
       │
       ├── FSImageUtil.loadSummary(raFile)
       │     从文件末尾读取 FileSummary（4 字节 trunkSize 确定位置）
       │     验证 ondiskVersion=1、layoutVersion 支持 PROTIBUF_FORMAT
       │
       ├── 解析 FileSummary.sections
       │     按 SectionName.ordinal() 排序，分离 _SUB 子 section
       │
       ├── 判断是否启用并行加载
       │     conf.getBoolean("dfs.image.parallel.load", false)
       │
       └── 按顺序处理每个 section：
            │
            ├── NS_INFO → loadNameSystemSection()
            │   GenerationStamp、BlockId、TxId、RollingUpgrade 信息
            │
            ├── STRING_TABLE → loadStringTableSection()
            │   重建 SerialNumberManager.StringTable
            │
            ├── INODE → inodeLoader.loadINodeSection()
            │   或 loadINodeSectionInParallel() — 并行版
            │   读取 header（lastInodeId, numInodes），然后循环读取每个 INode
            │   Root INode 特殊处理，其他添加到 inodeMap
            │   异步更新 NameCache 和 BlockMap（各用单线程 ExecutorService）
            │
            ├── INODE_DIR → inodeLoader.loadINodeDirectorySection()
            │   或并行版
            │   重建父-子关系（children + refChildren）
            │   完成后等待 BlockMap 和 NameCache 更新完成
            │
            ├── INODE_REFERENCE → snapshotLoader.loadINodeReferenceSection()
            │   加载引用节点到 refList，顺序必须严格一致
            │
            ├── FILES_UNDERCONSTRUCTION → 兼容性读取
            │   Lease 信息已从 INode 文件属性中读取
            │
            ├── SNAPSHOT → 快照元数据
            ├── SNAPSHOT_DIFF → 快照差异
            ├── SECRET_MANAGER → DelegationToken
            ├── CACHE_MANAGER → 缓存指令/池
            └── ERASURE_CODING → EC 策略
       │
       └── 等待 DigestThread 结束，获取 imgDigest
```

### 4.3 并行加载机制

- 配置：`dfs.image.parallel.load=true`，`dfs.image.parallel.threads=N`
- INode 和 INodeDir 的 `_SUB` 子 section 被分配给线程池
- 每个线程独立处理一个子 section
- 使用 `CountDownLatch` 同步等待所有线程完成
- 完成后验证加载的 INode 数是否匹配 header 中的 `numInodes`
- 异步更新：`blocksMapUpdateExecutor` 和 `nameCacheUpdateExecutor` 各为单线程

---

## 五、Protobuf 格式保存器（`FSImageFormatProtobuf.Saver`）

### 5.1 保存入口

```java
class Saver {
  private final SaveNamespaceContext context;
  private final SaverContext saverContext;
  private long currentOffset = MAGIC_HEADER.length;
  private long subSectionOffset = currentOffset;
  private MD5Hash savedDigest;
  private FileChannel fileChannel;
  private OutputStream sectionOutputStream;
  private CompressionCodec codec;
}
```

### 5.2 保存步骤

```
save(File file, FSImageCompression compression)
  │
  ├── enableSubSectionsIfRequired()
  │     根据 INode 数、阈值、目标 section 数判断是否启用子 section
  │     阈值：dfs.image.parallel.inode.threshold（默认 1000000）
  │     目标 section：dfs.image.parallel.target.sections（默认 4）
  │
  └── saveInternal(fout, compression, filePath)
       │
       ├── 写入 MAGIC_HEADER ("HDFSIMG1")
       │
       ├── 构建 FileSummary.Builder
       │
       ├── 确定压缩编码器（codec）
       │
       ├── 按固定顺序写各 section：
       │
       │   1. NS_INFO
       │      写入 GenerationStamp、BlockId、TxId、NamespaceId
       │
       │   2. ERASURE_CODING（如果 layout 版本支持）
       │      写入 EC 策略列表
       │
       │   3. INODE
       │      serializeINodeSection():
       │        - 写 header（lastInodeId, numInodes）
       │        - 遍历 inodeMap，对每个 INode 调用 save()
       │        - 每 inodesPerSubSection 个 INode 提交一个 _SUB 子 section
       │      serializeINodeDirectorySection():
       │        - 遍历所有目录，序列化子节点列表
       │        - 普通 children + refChildren 引用索引
       │      serializeFilesUCSection():
       │        - 从 LeaseManager 获取正在写入的文件列表
       │
       │   4. SNAPSHOT
       │      序列化快照计数器、可快照目录、快照根节点
       │
       │   5. SNAPSHOT_DIFF
       │      序列化 DiffEntry（DirectoryDiff / FileDiff）
       │
       │   6. INODE_REFERENCE
       │      序列化 WithCount/WithName/DstReference 引用节点
       │
       │   7. SECRET_MANAGER
       │      序列化 DelegationToken 密钥和 token
       │
       │   8. CACHE_MANAGER
       │      序列化缓存指令和池
       │
       │   9. STRING_TABLE
       │      序列化字符串去重表（用户/组名映射）
       │
       ├── 刷新压缩流，写 FileSummary + 4 字节 trunk size 到文件末尾
       │
       └── 计算全文件 MD5 摘要
```

### 5.3 Section 管理

- `commitSection()`：记录 section 在文件中的偏移量和长度到 `FileSummary.Builder`
- `commitSubSection()`：记录子 section 的偏移量和长度
- 压缩流需要刷新（flush）后才能获取正确长度
- offset 追踪：`currentOffset` 从 MAGIC 末尾开始累加

---

## 六、INode 序列化（`FSImageFormatPBINode`）

### 6.1 加载器（Loader）

**INode 加载**：
- 先读 header（`lastInodeId`、`numInodes`）
- 循环读每个 Protobuf INode 消息
- `ROOT_INODE_ID` → `loadRootINode()`：只更新属性，不替换根目录对象
- 其他 INode → `loadINode()`：按类型分派
  - `FILE` → `loadINodeFile()`：还原 block 列表、权限、ACL、XAttr、UnderConstruction、Lease
  - `DIRECTORY` → `loadINodeDirectory()`：还原配额、类型配额、ACL、XAttr
  - `SYMLINK` → `loadINodeSymlink()`

**目录结构重建**：
- `loadINodeDirectorySection()`：读取每个 `DirEntry`
- 将 children 和 refChildren 添加到父目录
- refChildren 通过 `refList` 索引引用

**异步更新**：
- `addToCacheAndBlockMap()`：每 1000 个文件节点触发一次
- `blocksMapUpdateExecutor`：更新 BlockManager 的块映射
- `nameCacheUpdateExecutor`：更新名称缓存
- 加载完 INodeDir section 后，等待两者完成

### 6.2 保存器（Saver）

- `serializeINodeSection()`：遍历 inodeMap
  - 按 INode 类型调用不同 save 方法
  - 每 `inodesPerSubSection` 提交一个子 section
- `serializeINodeDirectorySection()`：遍历所有目录
  - 序列化子节点（children + refChildren）
  - 检测 dangling child 错误
  - 按输出 INode 数提交子 section
- `serializeFilesUCSection()`：从 LeaseManager 获取文件列表
- 构建 INode 消息时，ACL 条目编码为 32-bit int，XAttr 编码为 compact 格式

### 6.3 ACL 编码

ACL 条目编码为 32-bit int（Big Endian）：
- `[0:2)`: 保留
- `[2:26)`: 名称 ID（指向 STRING_TABLE）
- `[26:27)`: 作用域（AclEntryScopeProto）
- `[27:29)`: 类型（AclEntryTypeProto）
- `[29:32)`: 权限（FsActionProto）

### 6.4 XAttr 编码

XAttr 名称编码为 32-bit int：
- `[0:2)`: 命名空间（XAttrNamespaceProto）
- `[2:26)`: 名称 ID（指向 STRING_TABLE）
- `[26:27)`: 命名空间扩展（支持第 5 个命名空间 "raw"）
- `[27:32)`: 保留

---

## 七、Snapshot 序列化（`FSImageFormatPBSnapshot`）

位于 `namenode/snapshot` 子包。

### 7.1 加载

```java
class Loader {
  private final FSNamesystem fsn;
  private final Map<Integer, Snapshot> snapshotMap;  // snapshotId → Snapshot
}
```

加载 3 种 section：
- **INODE_REFERENCE**：`loadINodeReferenceSection()`
  - 读取 `INodeReferenceSection.INodeReference`
  - 构建 WithCount + WithName / DstReference 引用节点
  - 添加到 `refList`
- **SNAPSHOT**：`loadSnapshotSection()`
  - 读取 SnapshotCounter、snapshottableDir 列表
  - 创建 Snapshot 及其根目录
- **SNAPSHOT_DIFF**：`loadSnapshotDiffSection()`
  - 读取 FileDiff 和 DirectoryDiff
  - 附着到对应 INode 的 diff 列表

**关键约束**：引用节点在内存中的顺序必须与磁盘完全一致，`refList` 通过索引关联。

### 7.2 保存

- `serializeINodeReferenceSection()`：序列化引用节点
- `serializeSnapshotSection()`：序列化快照元数据
- `serializeSnapshotDiffSection()`：只有 `numSnapshots > 0` 时才序列化差异

---

## 八、Legacy 格式（`FSImageFormat`）

### 8.1 LoaderDelegator（自动检测）

```java
class LoaderDelegator implements AbstractLoader {
  public void load(File file, boolean requireSameLayoutVersion) {
    is = Files.newInputStream(file.toPath());
    // 读取前 8 字节检测 MAGIC
    IOUtils.readFully(is, magic, 0, magic.length);
    if (Arrays.equals(magic, FSImageUtil.MAGIC_HEADER)) {
      // 新格式 → Protobuf Loader
      impl = new FSImageFormatProtobuf.Loader(conf, fsn, ...);
    } else {
      // 旧格式 → FSImageFormat.Loader
      impl = new Loader(conf, fsn);
    }
  }
}
```

### 8.2 旧版格式布局

定义在 `FSImageFormat.java` 的 Javadoc 中，使用 `DataInput/DataOutput` 手工序列化：

```
FSImage {
  layoutVersion: int, namespaceID: int, numItems: long,
  genstampV1: long, genstampV2: long,
  genstampAtBlockIdSwitch: long, lastAllocatedBlockId: long,
  transactionID: long, snapshotCounter: int, numSnapshots: int,
  numSnapshottableDirs: int,
  FSDirectoryTree | FilesUnderConstruction | SecretManagerState
}
```

INode 序列化使用 `FSImageSerialization` 工具类，当前主要用于 `saveLegacyOIVImage()` 向后兼容输出给 OIV（OfflineImageViewer）。

---

## 九、存储检查器体系

### 9.1 类层次

```
FSImageStorageInspector（抽象）
  ├── FSImageTransactionalStorageInspector（当前格式，txid-based）
  └── FSImagePreTransactionalStorageInspector（旧格式，文件名 fsimage）
```

### 9.2 核心接口

| 方法 | 说明 |
|------|------|
| `inspectDirectory(StorageDirectory)` | 检查存储目录中的镜像文件 |
| `isUpgradeFinalized()` | 是否有未完成的升级 |
| `getLatestImages()` | 返回最大 txid 的镜像文件列表 |
| `getMaxSeenTxId()` | 从 `seen_txid` 文件获取最大 txid |
| `needToSave()` | 是否需要重新保存 |

### 9.3 FSImageTransactionalStorageInspector 实现

- 文件名匹配：`image_(\d+)` 或 `image_rollback_(\d+)`
- `inspectDirectory()` 流程：
  1. 检查 VERSION 文件是否存在
  2. 读取 `seen_txid` 获取 `maxSeenTxId`
  3. 列出 `current/` 目录下所有文件
  4. 使用正则匹配 `NameNodeFile` 模式
  5. 记录解析到的 `FSImageFile(sd, file, txid)`
  6. 检查 `previous/` 目录判断升级是否完成
- `getLatestImages()`：按 txid 排序，返回最大 txid 的镜像（可能有多个存储目录的同 txid 副本）

---

## 十、压缩管理（`FSImageCompression`）

### 10.1 配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `dfs.image.compress` | `false` | 是否压缩 FsImage |
| `dfs.image.compress.codec` | `org.apache.hadoop.io.compress.DefaultCodec` | 压缩编码器全类名 |

### 10.2 工作机制

**PB 格式保存**：
- `FileSummary.codec` 字段记录压缩编码器名称
- 每个 section 的数据流通过 `CompressionOutputStream` 包装
- MAGIC 和 FileSummary 本身不压缩

**PB 格式加载**：
- `FSImageUtil.wrapInputStreamForCompression()` 按 codec 名称包装解压流

**Legacy 格式**：
- 压缩头写在流前部（`writeHeaderAndWrapStream()` / `readCompressionHeader()`）
- 格式：`boolean isCompressed + String codecClassName`

---

## 十一、镜像传输（`ImageServlet` + `TransferFsImage`）

### 11.1 ImageServlet

- 端点：`/imagetransfer`
- 用途：Secondary NameNode 拉取镜像、Standby NN 上传 checkpoint
- 核心方法：
  - `doGet()` 处理下载请求（参数：txid / "latest"）
  - `doPut()` 处理上传请求
- 特性：
  - `DataTransferThrottler` 支持限速
  - 并发 checkpoint 下载保护（`currentlyDownloadingCheckpoints` 排序集合）
  - 支持 bootstrapstandby 模式
  - `CONTENT_DISPOSITION` 和 `HADOOP_IMAGE_EDITS_HEADER` 头

### 11.2 TransferFsImage

- 客户端工具，通过 HTTP 从 NameNode 获取/上传镜像和编辑日志
- 方法：
  - `getLatestImage()`：获取最新镜像
  - `uploadImageFromStorage()`：上传本地镜像
  - `downloadEditsToStorage()`：下载编辑日志
- 支持校验和验证、自动重试
- `TransferResult` 枚举：SUCCESS / AUTHENTICATION_FAILURE / NOT_ACTIVE_NAMENODE_FAILURE / OLD_TRANSACTION_ID_FAILURE / UNEXPECTED_FAILURE

---

## 十二、OfflineImageViewer（`FSImageLoader`）

### 12.1 离线加载

`FSImageLoader` 独立于 NameNode 环境运行：

- 解析 FsImage 的 Protobuf section
- 将 INode 以 `byte[][]` 形式按 id 排序存储在内存中
- `dirmap` (`Map<Long, long[]>`) 记录每个目录的子节点 id 列表
- 反序列化：`INode.parseFrom()` 从 byte 数组解析

### 12.2 JSON 输出

- `getFileStatus()`：返回 JSON 格式的文件状态（路径、类型、权限、大小等）
- `listStatus()`：返回目录下的文件列表
- 支持 ACL、XAttr 查询
- 用于 `oiv` 命令行工具

---

## 十三、关键配置参数汇总

| 参数 | 默认值 | 作用域 | 说明 |
|------|--------|--------|------|
| `dfs.image.compress` | `false` | NameNode | 是否压缩 FsImage |
| `dfs.image.compress.codec` | `DefaultCodec` | NameNode | 压缩编码器 |
| `dfs.image.parallel.load` | `false` | NameNode | 是否并行加载 FsImage |
| `dfs.image.parallel.threads` | `2` | NameNode | 并行加载线程数 |
| `dfs.image.parallel.inode.threshold` | `1000000` | NameNode | 启用并行保存的最小 INode 数 |
| `dfs.image.parallel.target.sections` | `4` | NameNode | 子 section 目标数量 |
| `dfs.namenode.max.op.size` | `1048576` (1MB) | NameNode | 编辑日志最大操作大小 |
| `dfs.namenode.checkpoint.period` | `3600` (1h) | NameNode | checkpoint 间隔秒数 |
| `dfs.namenode.checkpoint.txns` | `1000000` | NameNode | checkpoint 触发的事务数 |
| `dfs.namenode.name.dir.restore` | `false` | NameNode | 失败存储目录自动恢复 |
| `dfs.image.parallel.inode.threshold` | `1000000` | NameNode | INode 阈值 |

---

## 十四、整体架构图

```
                      ┌──────────────────────────────┐
                      │       FSImage.java            │  ← 主控协调
                      │  (recoverTransitionRead /     │
                      │   saveNamespace / format /    │
                      │   doUpgrade / doRollback)     │
                      └───┬──────┬──────────┬─────────┘
                          │      │          │
               ┌──────────┘      │          └──────────┐
               ▼                 ▼                     ▼
     ┌─────────────────┐  ┌──────────────┐  ┌───────────────────┐
     │  LoaderDelegator │  │SaveNamespace │  │   FSImageSaver    │
     │ (FSImageFormat)  │  │  Context     │  │ (内部线程类)       │
     └────────┬─────────┘  └──────────────┘  └────────┬──────────┘
              │                                        │
              ▼                                        ▼
     ┌──────────────────┐                   ┌──────────────────────┐
     │ Protobuf Loader  │                   │ Protobuf Saver       │
     │ (并行加载/顺序)   │                   │ (子 section 划分)     │
     ├──────────────────┤                   ├──────────────────────┤
     │ INode Loader     │                   │ INode Saver          │
     │ Snapshot Loader  │                   │ Snapshot Saver       │
     └──────────────────┘                   └──────────────────────┘
              │                                        │
              ▼                                        ▼
     ┌──────────────────┐                   ┌──────────────────────┐
     │FSImageStorage    │                   │ FSImageCompression   │
     │Inspector         │                   │ FSImageUtil          │
     ├──────────────────┤                   ├──────────────────────┤
     │ Transactional    │                   │ fsimage.proto        │
     │ PreTransactional │                   │ (磁盘格式定义)         │
     └──────────────────┘                   └──────────────────────┘

     ┌──────────────────┐
     │   传输工具         │
     ├──────────────────┤
     │ ImageServlet     │  ← HTTP 端点 /imagetransfer
     │ TransferFsImage  │  ← 客户端 HTTP 下载/上传
     │ FSImageLoader    │  ← OIV 离线加载器
     └──────────────────┘
```

---

## 十五、主要类职责速查

| 类 | 文件 | 行数 | 职责 |
|----|------|------|------|
| `FSImage` | `FSImage.java` | 1606 | 主控：加载、保存、格式化、升级、回滚、checkpoint |
| `FSImageFormat` | `FSImageFormat.java` | 1498 | LoaderDelegator（自动选择格式）+ 旧版 Loader/Saver |
| `FSImageFormatProtobuf` | `FSImageFormatProtobuf.java` | 1056 | PB 格式 Loader/Saver、SectionName、并行控制 |
| `FSImageFormatPBINode` | `FSImageFormatPBINode.java` | 927 | INode/Directory/File/ACL/XAttr 的 PB 序列化 |
| `FSImageFormatPBSnapshot` | `snapshot/FSImageFormatPBSnapshot.java` | 656 | Snapshot/Reference/Diff 的 PB 序列化 |
| `FSImageUtil` | `FSImageUtil.java` | 94 | MAGIC 验证、FileSummary 读取、压缩包装 |
| `FSImageCompression` | `FSImageCompression.java` | 181 | 压缩编码管理 |
| `FSImageStorageInspector` | `FSImageStorageInspector.java` | 95 | 存储检查抽象 + FSImageFile |
| `FSImageTransactionalStorageInspector` | `FSImageTransactionalStorageInspector.java` | 177 | 当前 txid-based 格式检查 |
| `FSImagePreTransactionalStorageInspector` | `FSImagePreTransactionalStorageInspector.java` | - | 旧格式检查 |
| `FsImageValidation` | `FsImageValidation.java` | - | 镜像完整性校验 |
| `ImageServlet` | `ImageServlet.java` | 795 | HTTP 镜像传输端点 |
| `TransferFsImage` | `TransferFsImage.java` | 463 | 镜像 HTTP 下载/上传客户端 |
| `FSImageLoader` | `tools/offlineImageViewer/FSImageLoader.java` | 687 | OIV 离线加载器 |
| `BackupImage` | `BackupImage.java` | - | BackupNode 使用的镜像 |
