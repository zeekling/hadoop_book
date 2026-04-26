
# 简介

Shuffle是MapReduce框架的核心功能之一，负责将Map阶段产生的海量中间数据，经过高效、有序的处理后，输送给Reduce阶段。这个过程定义了数据如何从Map任务流向Reduce任务，是影响作业性能的关键环节。
Shuffle过程对于MapReduce程序的正确执行至关重要，因为它确保了数据按照正确的顺序传递给Reduce函数。

总体框架如下：

<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width=1366 height=1366 src="https://edrawcloudpubliccn.oss-cn-shenzhen.aliyuncs.com/viewer/self/18010812/share/2026-3-8/1772981212/main.svg"></iframe>

---

# ShuffleHandler 源码解读

## 一、概述

ShuffleHandler 是 Hadoop MapReduce 中负责处理 Shuffle 阶段数据传输的核心服务，运行在 NodeManager 上作为辅助服务（AuxiliaryService）。它使用 Netty 框架提供 HTTP 服务，响应 Reducer 的数据请求，将 Map 输出数据传输给 Reducer。

**核心职责**：
- 接收 Reducer 的 Shuffle 请求
- 验证请求的合法性（Token 认证）
- 查找并读取 Map 输出文件
- 通过 HTTP 流式传输数据给 Reducer

---

## 二、核心架构

### 2.1 类继承关系

```
ShuffleHandler extends AuxiliaryService
```

作为 YARN 的辅助服务，实现了以下关键方法：
- `initializeApplication()` - 应用初始化时调用
- `stopApplication()` - 应用结束时调用
- `serviceInit()` - 服务初始化
- `serviceStart()` - 服务启动
- `serviceStop()` - 服务停止
- `getMetaData()` - 获取服务元数据（端口号）

### 2.2 主要组件

```
ShuffleHandler
├── ShuffleChannelHandlerContext (上下文对象)
├── ShuffleChannelInitializer (Netty 通道初始化器)
├── ShuffleChannelHandler (HTTP 请求处理器)
├── IndexCache (索引缓存)
├── JobTokenSecretManager (安全认证)
├── ShuffleMetrics (指标收集)
└── LevelDB (状态恢复)
```

---

## 三、主要组件详解

### 3.1 ShuffleChannelHandlerContext

**位置**: `ShuffleChannelHandlerContext.java`

**作用**: 封装 ShuffleHandler 运行所需的所有配置和状态

**核心字段**:
```java
- conf: Configuration                    // Hadoop 配置
- secretManager: JobTokenSecretManager   // 安全认证管理器
- userRsrc: Map<String, String>          // 用户资源映射
- pathCache: LoadingCache               // 路径缓存
- indexCache: IndexCache                 // 索引缓存
- metrics: ShuffleMetrics               // 指标收集器
- allChannels: ChannelGroup             // Netty 连接组
- activeConnections: AtomicInteger       // 活跃连接计数
```

**关键配置参数**:
- `connectionKeepAliveEnabled`: 是否启用连接保活
- `connectionKeepAliveTimeOut`: 连接保活超时时间（秒）
- `mapOutputMetaInfoCacheSize`: Map 输出元信息缓存大小（默认 1000）
- `manageOsCache`: 是否使用 posix_fadvise 管理 OS 缓存
- `readaheadLength`: 预读长度（默认 4MB）
- `maxShuffleConnections`: 最大 Shuffle 连接数（0 表示无限制）
- `shuffleBufferSize`: Shuffle 缓冲区大小（默认 128KB）
- `shuffleTransferToAllowed`: 是否允许零拷贝传输
- `maxSessionOpenFiles`: 每个会话最大打开文件数（默认 3）

---

### 3.2 ShuffleChannelInitializer

**位置**: `ShuffleChannelInitializer.java`

**作用**: 为每个新连接设置 Netty 处理管道

**处理管道顺序**:
```
1. SSL Handler (可选)          - SSL 加密处理
2. HttpServerCodec              - HTTP 编解码器
3. HttpObjectAggregator         - HTTP 消息聚合（最大 64KB）
4. ChunkedWriteHandler          - 支持分块写入
5. ShuffleChannelHandler        - 核心业务逻辑处理器
6. TimeoutHandler               - 连接超时处理
```

**关键代码**:
```java
protected void initChannel(SocketChannel ch) {
    ChannelPipeline pipeline = ch.pipeline();
    
    // SSL Handler（如果启用）
    if (sslFactory != null) {
        pipeline.addLast("ssl", sslFactory.createSSLEngine());
    }
    
    // HTTP 编解码
    pipeline.addLast("codec", new HttpServerCodec());
    
    // 消息聚合
    pipeline.addLast("aggregator", 
        new HttpObjectAggregator(MAX_CONTENT_LENGTH));
    
    // 分块写入
    pipeline.addLast("chunkedWriter", new ChunkedWriteHandler());
    
    // 核心处理器
    pipeline.addLast("handler", new ShuffleChannelHandler(handlerContext));
    
    // 超时处理
    pipeline.addLast("timeout", new TimeoutHandler(connectionKeepAliveTimeOut));
}
```

---

### 3.3 ShuffleChannelHandler

**位置**: `ShuffleChannelHandler.java`

**作用**: 处理 HTTP 请求的核心业务逻辑

**继承关系**:
```java
extends SimpleChannelInboundHandler<FullHttpRequest>
```

**核心方法**:

#### 1. channelActive - 连接激活
```java
public void channelActive(ChannelHandlerContext ctx) {
    int numConnections = handlerCtx.activeConnections.incrementAndGet();
    
    // 检查连接数限制
    if (maxShuffleConnections > 0 && numConnections > maxShuffleConnections) {
        // 返回 429 Too Many Requests
        sendError(ctx, "", TOO_MANY_REQ_STATUS, 
                  Map.of("Retry-After", "1000"));
    } else {
        super.channelActive(ctx);
        handlerCtx.allChannels.add(ctx.channel());
    }
}
```

#### 2. channelRead0 - 处理 HTTP 请求
```java
public void channelRead0(ChannelHandlerContext ctx, FullHttpRequest request) {
    // 1. 验证 HTTP 方法（只允许 GET）
    if (request.method() != GET) {
        sendError(ctx, METHOD_NOT_ALLOWED);
        return;
    }
    
    // 2. 验证 Shuffle 版本
    if (!validateShuffleVersion(request)) {
        sendError(ctx, "Incompatible shuffle request version", BAD_REQUEST);
        return;
    }
    
    // 3. 解析请求参数
    QueryStringDecoder decoder = new QueryStringDecoder(request.uri());
    String jobId = decoder.parameters().get("job").get(0);
    int reduceId = Integer.parseInt(decoder.parameters().get("reduce").get(0));
    List<String> mapIds = splitMaps(decoder.parameters().get("map"));
    
    // 4. 验证请求（Token 认证）
    verifyRequest(jobId, ctx, request, response, requestUri);
    
    // 5. 填充响应头
    populateHeaders(mapIds, jobId, user, reduceId, response, 
                   keepAliveParam, mapOutputInfoMap);
    
    // 6. 发送响应头
    channel.write(response);
    
    // 7. 创建 ReduceContext 并开始发送数据
    ReduceContext reduceContext = new ReduceContext(
        mapIds, reduceId, ctx, user, mapOutputInfoMap, jobId, keepAlive);
    sendMap(reduceContext);
}
```

#### 3. sendMap - 发送 Map 输出数据
```java
public void sendMap(ReduceContext reduceContext) {
    if (reduceContext.getMapsToSend().get() < 
        reduceContext.getMapIds().size()) {
        
        int nextIndex = reduceContext.getMapsToSend().getAndIncrement();
        String mapId = reduceContext.getMapIds().get(nextIndex);
        
        // 获取 Map 输出信息
        MapOutputInfo info = getMapOutputInfo(
            mapId, reduceContext.getReduceId(), 
            reduceContext.getJobId(), reduceContext.getUser());
        
        // 发送 Map 输出
        ChannelFuture nextMap = sendMapOutput(
            channel, user, mapId, reduceId, info);
        
        // 添加监听器，完成后继续发送下一个
        nextMap.addListener(new ReduceMapFileCount(this, reduceContext));
    }
}
```

#### 4. sendMapOutput - 发送单个 Map 输出
```java
protected ChannelFuture sendMapOutput(Channel ch, String user, 
                                       String mapId, int reduce,
                                       MapOutputInfo mapOutputInfo) {
    // 1. 写入 ShuffleHeader
    ch.write(shuffleHeaderToBytes(
        new ShuffleHeader(mapId, info.partLength, info.rawLength, reduce)));
    
    // 2. 打开文件
    RandomAccessFile spill = SecureIOUtils.openForRandomRead(
        spillFile, "r", user, null);
    
    // 3. 发送文件数据
    if (ch.pipeline().get(SslHandler.class) == null) {
        // HTTP - 使用零拷贝
        FadvisedFileRegion partition = new FadvisedFileRegion(
            spill, info.startOffset, info.partLength, 
            manageOsCache, readaheadLength, readaheadPool, 
            spillFile.getAbsolutePath(), shuffleBufferSize, 
            shuffleTransferToAllowed);
        writeFuture = ch.writeAndFlush(partition);
    } else {
        // HTTPS - 使用分块传输
        FadvisedChunkedFile chunk = new FadvisedChunkedFile(
            spill, info.startOffset, info.partLength, 
            sslFileBufferSize, manageOsCache, readaheadLength, 
            readaheadPool, spillFile.getAbsolutePath());
        writeFuture = ch.writeAndFlush(chunk);
    }
    
    return writeFuture;
}
```

#### 5. verifyRequest - 验证请求合法性
```java
protected void verifyRequest(String appid, ChannelHandlerContext ctx,
                            HttpRequest request, HttpResponse response, 
                            URL requestUri) {
    // 1. 获取 Job Token 密钥
    SecretKey tokenSecret = secretManager.retrieveTokenSecret(appid);
    if (tokenSecret == null) {
        throw new IOException("Could not find jobid");
    }
    
    // 2. 构建加密 URL
    String encryptedURL = SecureShuffleUtils.buildMsgFrom(requestUri);
    
    // 3. 获取请求中的 URL Hash
    String urlHashStr = request.headers().get(
        SecureShuffleUtils.HTTP_HEADER_URL_HASH);
    
    // 4. 验证 Hash
    SecureShuffleUtils.verifyReply(urlHashStr, encryptedURL, tokenSecret);
    
    // 5. 生成响应 Hash
    String reply = SecureShuffleUtils.generateHash(
        urlHashStr.getBytes(StandardCharsets.UTF_8), tokenSecret);
    response.headers().set(
        SecureShuffleUtils.HTTP_HEADER_REPLY_URL_HASH, reply);
}
```

**内部类**:

#### ReduceContext - 维护每次请求的上下文
```java
public static class ReduceContext {
    private final List<String> mapIds;           // 要发送的 Map ID 列表
    private final AtomicInteger mapsToWait;       // 等待完成的 Map 数量
    private final AtomicInteger mapsToSend;      // 已发送的 Map 数量
    private final int reduceId;                  // Reduce ID
    private final ChannelHandlerContext ctx;     // Netty 上下文
    private final String user;                   // 用户名
    private final Map<String, MapOutputInfo> infoMap;  // Map 输出信息缓存
    private final String jobId;                  // Job ID
    private final boolean keepAlive;             // 是否保持连接
}
```

#### ReduceMapFileCount - 监听 Map 发送完成
```java
static class ReduceMapFileCount implements ChannelFutureListener {
    @Override
    public void operationComplete(ChannelFuture future) {
        if (!future.isSuccess()) {
            future.channel().close();
            return;
        }
        
        int waitCount = reduceContext.getMapsToWait().decrementAndGet();
        if (waitCount == 0) {
            // 所有 Map 发送完成
            future.channel().writeAndFlush(LastHttpContent.EMPTY_LAST_CONTENT);
            
            if (reduceContext.getKeepAlive()) {
                // 保持连接，启用超时处理
                timeoutHandler.setEnabledTimeout(true);
            } else {
                // 关闭连接
                lastContentFuture.addListener(ChannelFutureListener.CLOSE);
            }
        } else {
            // 继续发送下一个 Map
            handler.sendMap(reduceContext);
        }
    }
}
```

---

### 3.4 IndexCache

**位置**: `IndexCache.java`

**作用**: 缓存 Map 输出的索引文件，避免重复读取磁盘

**核心数据结构**:
```java
private final ConcurrentHashMap<String, IndexInformation> cache;
private final LinkedBlockingQueue<String> queue;  // LRU 队列
private final long totalMemoryAllowed;            // 最大允许内存（默认 10MB）
private final long totalMemoryUsed;              // 当前已使用内存
```

**核心方法**:

#### getIndexInformation - 获取索引信息
```java
public IndexRecord getIndexInformation(String mapId, int reduce, 
                                       Path indexFile, String user) 
    throws IOException {
    String cacheKey = mapId + "-" + reduce;
    
    // 1. 先从缓存查找
    IndexInformation info = cache.get(cacheKey);
    if (info != null) {
        return info.indexRecord;
    }
    
    // 2. 缓存未命中，从磁盘读取
    synchronized (this) {
        // 双重检查
        info = cache.get(cacheKey);
        if (info != null) {
            return info.indexRecord;
        }
        
        // 读取索引文件
        SpillRecord spillRecord = new SpillRecord(indexFile, jobConf);
        IndexRecord indexRecord = spillRecord.getIndex(reduce);
        
        // 检查内存限制
        while (totalMemoryUsed + indexRecord.getRawLength() > totalMemoryAllowed) {
            freeIndexInformation();
        }
        
        // 添加到缓存
        info = new IndexInformation(spillRecord);
        cache.put(cacheKey, info);
        queue.add(cacheKey);
        totalMemoryUsed += info.getSize();
        
        return indexRecord;
    }
}
```

#### freeIndexInformation - 清理索引信息
```java
private void freeIndexInformation() {
    String toRemove = queue.poll();
    if (toRemove != null) {
        IndexInformation info = cache.remove(toRemove);
        if (info != null) {
            totalMemoryUsed -= info.getSize();
        }
    }
}
```

---

### 3.5 JobTokenSecretManager

**位置**: `JobTokenSecretManager.java`

**作用**: 管理 Job Token 的密钥，用于 Shuffle 阶段的安全认证

**继承关系**:
```java
extends SecretManager<JobTokenIdentifier>
```

**核心字段**:
```java
private SecretKey masterKey;                          // 主密钥
private Map<String, SecretKey> currentJobTokens;       // Job ID 到 Token 密钥的映射
```

**核心方法**:

#### createPassword - 创建密码
```java
public byte[] createPassword(JobTokenIdentifier identifier) {
    byte[] identifierBytes = identifier.getBytes();
    return computeHash(identifierBytes, masterKey);
}
```

#### addTokenForJob - 添加 Job Token
```java
public void addTokenForJob(String jobId, Token<JobTokenIdentifier> jobToken) {
    SecretKey tokenSecret = createSecretKey(jobToken.getPassword());
    currentJobTokens.put(jobId, tokenSecret);
}
```

#### retrieveTokenSecret - 查找 Token 密钥
```java
public SecretKey retrieveTokenSecret(String jobId) 
    throws InvalidToken {
    SecretKey tokenSecret = currentJobTokens.get(jobId);
    if (tokenSecret == null) {
        throw new InvalidToken("Invalid job token for " + jobId);
    }
    return tokenSecret;
}
```

---

### 3.6 FadvisedFileRegion

**位置**: `FadvisedFileRegion.java`

**作用**: 扩展 Netty 的 DefaultFileRegion，支持 OS 缓存管理和预读

**继承关系**:
```java
extends DefaultFileRegion
```

**核心字段**:
```java
private final boolean manageOsCache;        // 是否管理 OS 缓存
private final int readaheadLength;          // 预读长度
private final ReadaheadPool readaheadPool;  // 预读线程池
private final FileDescriptor fd;            // 文件描述符
private final String identifier;            // 文件标识符
private final int shuffleBufferSize;        // Shuffle 缓冲区大小
private final boolean shuffleTransferToAllowed;  // 是否允许零拷贝
```

**核心方法**:

#### transferTo - 传输数据
```java
public long transferTo(WritableByteChannel target, long position) {
    // 1. 预读
    if (readaheadPool != null && readaheadLength > 0) {
        readaheadRequest = readaheadPool.readaheadStream(
            identifier, fd, position() + position, 
            readaheadLength, position() + count(), readaheadRequest);
    }
    
    // 2. 传输数据
    if (shuffleTransferToAllowed) {
        // 使用零拷贝（sendfile）
        return super.transferTo(target, position);
    } else {
        // 使用自定义缓冲区传输
        return customShuffleTransfer(target, position);
    }
}
```

#### customShuffleTransfer - 自定义传输
```java
long customShuffleTransfer(WritableByteChannel target, long position) {
    ByteBuffer byteBuffer = ByteBuffer.allocate(
        Math.min(shuffleBufferSize, trans));
    
    while (trans > 0 && 
           (readSize = fileChannel.read(byteBuffer, position)) > 0) {
        byteBuffer.flip();
        while (byteBuffer.hasRemaining()) {
            target.write(byteBuffer);
        }
        byteBuffer.clear();
    }
    
    return actualCount - trans;
}
```

#### transferSuccessful - 传输成功后的处理
```java
public void transferSuccessful() {
    if (fd.valid() && manageOsCache && count() > 0) {
        // 通知 OS 不再需要缓存这些数据
        NativeIO.POSIX.getCacheManipulator().posixFadviseIfPossible(
            identifier, fd, position(), count(), POSIX_FADV_DONTNEED);
    }
}
```

---

### 3.7 FadvisedChunkedFile

**位置**: `FadvisedChunkedFile.java`

**作用**: 用于 HTTPS 传输的分块文件处理器

**继承关系**:
```java
extends ChunkedFile
```

**核心方法**:

#### readChunk - 读取数据块
```java
public ByteBuf readChunk(ByteBufAllocator allocator) {
    // 预读
    if (manageOsCache && readaheadPool != null) {
        readaheadRequest = readaheadPool.readaheadStream(
            identifier, fd, currentOffset(), readaheadLength,
            endOffset(), readaheadRequest);
    }
    
    return super.readChunk(allocator);
}
```

#### close - 关闭文件
```java
public void close() {
    // 取消预读请求
    if (readaheadRequest != null) {
        readaheadRequest.cancel();
    }
    
    // 通知 OS 不再需要缓存
    if (fd.valid() && manageOsCache && 
        endOffset() - startOffset() > 0) {
        NativeIO.POSIX.getCacheManipulator().posixFadviseIfPossible(
            identifier, fd, startOffset(), 
            endOffset() - startOffset(), POSIX_FADV_DONTNEED);
    }
    
    super.close();
}
```

---

## 四、工作流程

### 4.1 完整的 Shuffle 请求处理流程

```
1. Reducer 发起 HTTP GET 请求
   ↓
2. ShuffleChannelInitializer 初始化连接管道
   ↓
3. ShuffleChannelHandler.channelRead0() 处理请求
   ↓
4. 验证 HTTP 方法（只允许 GET）
   ↓
5. 验证 Shuffle 版本
   ↓
6. 解析请求参数（job, reduce, map）
   ↓
7. verifyRequest() - Token 认证
   - 获取 Job Token 密钥
   - 验证 URL Hash
   - 生成响应 Hash
   ↓
8. populateHeaders() - 填充响应头
   - 遍历所有 mapId
   - 调用 getMapOutputInfo() 获取每个 Map 的输出信息
   - 计算 contentLength
   - 设置响应头
   ↓
9. 发送 HTTP 响应头
   ↓
10. 创建 ReduceContext
   ↓
11. sendMap() - 开始发送 Map 输出数据
   ↓
12. 循环发送每个 Map 输出
   - sendMapOutput() 发送单个 Map 输出
   - 写入 ShuffleHeader
   - 使用 FadvisedFileRegion 或 FadvisedChunkedFile 发送文件数据
   ↓
13. ReduceMapFileCount 监听发送完成
   - 如果还有未发送的 Map，继续调用 sendMap()
   - 如果所有 Map 发送完成，发送 LastHttpContent
   - 根据 keepAlive 参数决定是否关闭连接
```

### 4.2 HTTP 请求示例

**请求格式**:
```
GET /mapOutput?job=job_1111111111111_0001&reduce=0&map=attempt_1111111111111_0001_m_000001_0,attempt_1111111111111_0002_m_000002_0 HTTP/1.1
name: mapreduce
version: 1.0.0
UrlHash: 9zS++qE0/7/D2l1Rg0TqRoSguAk=
```

**响应格式**:
```
HTTP/1.1 200 OK
ReplyHash: GcuojWkAxXUyhZHPnwoV/MW2tGA=
name: mapreduce
version: 1.0.0
connection: close
content-length: 138

[ShuffleHeader for attempt_1]
[Map output data for attempt_1]
[ShuffleHeader for attempt_2]
[Map output data for attempt_2]
...
```

### 4.3 ShuffleHeader 格式

```java
class ShuffleHeader {
    String mapId;        // Map 尝试 ID
    long partLength;     // 分区长度（该 Reduce 需要的数据长度）
    long rawLength;      // 原始长度（整个 Map 输出长度）
    int reduceId;       // Reduce ID
}
```

---

## 五、关键特性

### 5.1 安全认证

**Token 认证机制**:
1. ApplicationMaster 启动时生成 Job Token
2. Job Token 通过 `initializeApplication()` 传递给 ShuffleHandler
3. ShuffleHandler 将 Token 存储在 `JobTokenSecretManager` 中
4. Reducer 发起请求时携带 URL Hash
5. ShuffleHandler 验证 Hash 并生成响应 Hash
6. 双向验证确保请求的合法性

**关键代码**:
```java
// Reducer 端生成 URL Hash
String urlHash = SecureShuffleUtils.generateHash(
    url.getBytes(StandardCharsets.UTF_8), jobToken.getPassword());

// ShuffleHandler 端验证
SecureShuffleUtils.verifyReply(urlHash, encryptedURL, tokenSecret);
```

### 5.2 连接管理

**连接数限制**:
```java
if (maxShuffleConnections > 0 && 
    numConnections > maxShuffleConnections) {
    // 返回 429 Too Many Requests
    sendError(ctx, "", TOO_MANY_REQ_STATUS, 
              Map.of("Retry-After", "1000"));
}
```

**连接保活**:
```java
if (keepAliveParam || connectionKeepAliveEnabled) {
    // 保持连接，启用超时处理
    timeoutHandler.setEnabledTimeout(true);
} else {
    // 关闭连接
    lastContentFuture.addListener(ChannelFutureListener.CLOSE);
}
```

**超时处理**:
```java
class TimeoutHandler extends IdleStateHandler {
    @Override
    public void channelIdle(ChannelHandlerContext ctx, IdleStateEvent e) {
        if (e.state() == IdleState.WRITER_IDLE && enabledTimeout) {
            ctx.channel().close();
        }
    }
}
```

### 5.3 文件描述符限制

**问题**: 如果同时打开所有 Map 输出文件，可能会耗尽文件描述符

**解决方案**: 限制每个会话同时打开的文件数

```java
// 配置参数
public static final String SHUFFLE_MAX_SESSION_OPEN_FILES =
    "mapreduce.shuffle.max.session-open-files";
public static final int DEFAULT_SHUFFLE_MAX_SESSION_OPEN_FILES = 3;

// 实现
public void sendMap(ReduceContext reduceContext) {
    // 最多同时打开 maxSessionOpenFiles 个文件
    // 每完成一个文件传输，再打开下一个
}
```

### 5.4 状态恢复

**LevelDB 存储**:
```java
private void recordJobShuffleInfo(JobID jobId, String user,
                                  Token<JobTokenIdentifier> jobToken) {
    if (stateDb != null) {
        // 序列化 JobShuffleInfo
        JobShuffleInfoProto proto = JobShuffleInfoProto.newBuilder()
            .setUser(user)
            .setJobToken(tokenProto)
            .build();
        
        // 存储到 LevelDB
        stateDb.put(bytes(jobId.toString()), proto.toByteArray());
    }
    
    addJobToken(jobId, user, jobToken);
}
```

**恢复流程**:
```java
private void recoverState(Configuration conf) {
    Path recoveryRoot = getRecoveryPath();
    if (recoveryRoot != null) {
        startStore(recoveryRoot);
        
        // 遍历 LevelDB 中的所有 Job
        LeveldbIterator iter = new LeveldbIterator(stateDb);
        iter.seek(bytes(JobID.JOB));
        while (iter.hasNext()) {
            Map.Entry<byte[], byte[]> entry = iter.next();
            recoverJobShuffleInfo(key, entry.getValue());
        }
    }
}
```

### 5.5 性能优化

#### 1. 路径缓存
```java
LoadingCache<AttemptPathIdentifier, AttemptPathInfo> pathCache =
    CacheBuilder.newBuilder()
        .expireAfterAccess(5, TimeUnit.MINUTES)
        .softValues()
        .concurrencyLevel(16)
        .maximumWeight(10 * 1024 * 1024)
        .build(new CacheLoader<...>() {
            @Override
            public AttemptPathInfo load(AttemptPathIdentifier key) {
                // 加载路径信息
                return new AttemptPathInfo(indexFileName, mapOutputFileName);
            }
        });
```

#### 2. 索引缓存
```java
IndexCache indexCache = new IndexCache(new JobConf(conf));
// 缓存 Map 输出索引，避免重复读取磁盘
IndexRecord info = indexCache.getIndexInformation(
    mapId, reduce, indexPath, user);
```

#### 3. 预读
```java
if (readaheadPool != null && readaheadLength > 0) {
    readaheadRequest = readaheadPool.readaheadStream(
        identifier, fd, position, readaheadLength, endOffset, 
        readaheadRequest);
}
```

#### 4. 零拷贝传输
```java
if (shuffleTransferToAllowed) {
    // 使用 sendfile 系统调用，零拷贝
    return super.transferTo(target, position);
} else {
    // 使用缓冲区传输
    return customShuffleTransfer(target, position);
}
```

#### 5. OS 缓存管理
```java
public void transferSuccessful() {
    if (manageOsCache && count() > 0) {
        // 通知 OS 不再需要缓存这些数据
        NativeIO.POSIX.getCacheManipulator().posixFadviseIfPossible(
            identifier, fd, position(), count(), 
            POSIX_FADV_DONTNEED);
    }
}
```

---

## 六、配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `mapreduce.shuffle.port` | 13562 | Shuffle 服务端口 |
| `mapreduce.shuffle.listen.queue.size` | 128 | 监听队列大小 |
| `mapreduce.shuffle.connection-keep-alive.enable` | false | 是否启用连接保活 |
| `mapreduce.shuffle.connection-keep-alive.timeout` | 5 | 连接保活超时时间（秒） |
| `mapreduce.shuffle.mapoutput-info.meta.cache.size` | 1000 | Map 输出元信息缓存大小 |
| `mapreduce.shuffle.manage.os.cache` | true | 是否管理 OS 缓存 |
| `mapreduce.shuffle.readahead.bytes` | 4MB | 预读长度 |
| `mapreduce.shuffle.pathcache.max-weight` | 10MB | 路径缓存最大权重 |
| `mapreduce.shuffle.pathcache.expire-after-access-minutes` | 5 | 路径缓存过期时间（分钟） |
| `mapreduce.shuffle.pathcache.concurrency-level` | 16 | 路径缓存并发级别 |
| `mapreduce.shuffle.max.connections` | 0 | 最大 Shuffle 连接数（0 表示无限制） |
| `mapreduce.shuffle.max.threads` | 0 | 最大 Shuffle 线程数（0 表示 2 * CPU 核心数） |
| `mapreduce.shuffle.transfer.buffer.size` | 128KB | Shuffle 传输缓冲区大小 |
| `mapreduce.shuffle.transferTo.allowed` | true | 是否允许零拷贝传输 |
| `mapreduce.shuffle.max.session-open-files` | 3 | 每个会话最大打开文件数 |

---

## 七、指标收集

**ShuffleMetrics 类**:
```java
@Metrics(about="Shuffle output metrics", context="mapred")
static class ShuffleMetrics implements ChannelFutureListener {
    @Metric("Shuffle output in bytes")
    MutableCounterLong shuffleOutputBytes;
    
    @Metric("# of failed shuffle outputs")
    MutableCounterInt shuffleOutputsFailed;
    
    @Metric("# of succeeded shuffle outputs")
    MutableCounterInt shuffleOutputsOK;
    
    @Metric("# of current shuffle connections")
    MutableGaugeInt shuffleConnections;
    
    @Override
    public void operationComplete(ChannelFuture future) {
        if (future.isSuccess()) {
            shuffleOutputsOK.incr();
        } else {
            shuffleOutputsFailed.incr();
        }
        shuffleConnections.decr();
    }
}
```

---

## 八、总结

ShuffleHandler 是 Hadoop MapReduce Shuffle 阶段的核心组件，具有以下特点：

1. **高性能**: 使用 Netty 框架，支持零拷贝传输、预读、缓存等优化
2. **安全性**: 基于 Token 的双向认证机制
3. **可靠性**: 支持状态恢复，可以从 LevelDB 恢复作业信息
4. **可配置性**: 丰富的配置参数，可根据场景调优
5. **可观测性**: 完善的指标收集，便于监控和诊断

**关键设计思想**:
- 使用 Netty 的异步非阻塞模型，提高并发性能
- 多级缓存（路径缓存、索引缓存、元信息缓存）减少磁盘 I/O
- 限制文件描述符使用，避免资源耗尽
- 支持 HTTP 和 HTTPS 两种传输方式
- OS 缓存管理，避免缓存污染

通过深入理解 ShuffleHandler 的实现，可以更好地优化 Hadoop MapReduce 的 Shuffle 性能，排查相关问题。

---

# Map 端 Shuffle 源码解读

## 一、概述

Map 端的 Shuffle 是 MapReduce 框架中 Map 任务将输出数据写入磁盘的过程。这个过程包括数据收集、排序、溢写（Spill）和合并（Merge）等关键步骤，最终生成可供 Reducer 读取的输出文件和索引文件。

**核心职责**：
- 收集 Map 输出的 key-value 对
- 将数据写入环形缓冲区
- 当缓冲区达到阈值时触发溢写
- 对溢写数据进行排序和分区
- 合并所有溢写文件生成最终输出

---

## 二、核心架构

### 2.1 类继承关系

```
MapTask
└── MapOutputBuffer<K,V> implements MapOutputCollector<K,V>, IndexedSortable
    ├── SpillThread (溢写线程)
    ├── BlockingBuffer (阻塞缓冲区)
    └── Buffer (输出流)
```

**关键接口**:
- `MapOutputCollector<K,V>` - Map 输出收集器接口
- `IndexedSortable` - 可排序接口

### 2.2 主要组件

```
MapOutputBuffer
├── 环形缓冲区 (kvbuffer)          - 存储序列化的 key-value 数据
├── 元数据缓冲区 (kvmeta)          - 存储记录的元数据
├── SpillThread                    - 后台溢写线程
├── SpillRecord                    - 溢写索引记录
├── IndexRecord                    - 索引记录
├── MapOutputFile                  - 文件路径管理
└── CombinerRunner                 - Combiner 运行器
```

---

## 三、主要组件详解

### 3.1 MapOutputBuffer

**位置**: `MapTask.java:890`

**作用**: Map 端的核心缓冲区管理类，负责收集、排序和溢写 Map 输出数据

**核心字段**:
```java
// 缓冲区相关
private byte[] kvbuffer;              // 主输出缓冲区（环形缓冲区）
private IntBuffer kvmeta;             // 元数据缓冲区（覆盖在 kvbuffer 上）
int kvstart;                          // 溢写元数据起始位置
int kvend;                            // 溢写元数据结束位置
int kvindex;                          // 当前元数据写入位置
int bufstart;                         // 序列化数据起始位置
int bufend;                           // 序列化数据结束位置
int bufindex;                         // 当前序列化数据写入位置
int bufmark;                          // 记录结束标记
int bufvoid;                          // 缓冲区有效边界

// 元数据常量
private static final int VALSTART = 0;         // value 偏移
private static final int KEYSTART = 1;         // key 偏移
private static final int PARTITION = 2;       // 分区偏移
private static final int VALLEN = 3;           // value 长度
private static final int NMETA = 4;            // 元数据字段数
private static final int METASIZE = NMETA * 4; // 元数据大小（16 字节）

// 溢写相关
private int maxRec;                   // 最大记录数
private int softLimit;                // 软限制（触发溢写的阈值）
boolean spillInProgress;             // 是否正在溢写
int numSpills = 0;                   // 溢写次数
final ReentrantLock spillLock = new ReentrantLock();  // 溢写锁
final Condition spillDone = spillLock.newCondition(); // 溢写完成条件
final Condition spillReady = spillLock.newCondition(); // 溢写就绪条件
final SpillThread spillThread = new SpillThread();    // 溢写线程

// 索引缓存
final ArrayList<SpillRecord> indexCacheList = new ArrayList<SpillRecord>();
private int totalIndexCacheMemory;    // 索引缓存总内存
private int indexCacheMemoryLimit;    // 索引缓存内存限制（默认 1MB）
```

**关键配置参数**:
- `mapreduce.task.io.sort.mb`: 排序缓冲区大小（默认 100MB）
- `mapreduce.map.sort.spill.percent`: 溢写百分比（默认 0.8，即 80%）
- `mapreduce.task.index.cache.memory.limit.bytes`: 索引缓存内存限制（默认 1MB）
- `mapreduce.task.spill.files.count.limit`: 溢写文件数量限制（默认 -1，无限制）

---

### 3.2 环形缓冲区设计

**设计思想**: 使用环形缓冲区实现高效的内存管理，避免频繁的内存分配和拷贝

**缓冲区布局**:
```
kvbuffer (环形缓冲区)
┌─────────────────────────────────────────────────────────┐
│  元数据区 (kvmeta)  │  序列化数据区 (kvbuffer)          │
│  [PARTITION|KEYSTART|VALSTART|VALLEN]  │  [key|value]   │
└─────────────────────────────────────────────────────────┘
         ↑ kvstart/kvend                    ↑ bufstart/bufend
         ↑ kvindex                          ↑ bufindex
```

**关键方法**:

#### 1. collect - 收集 Map 输出
```java
public synchronized void collect(K key, V value, final int partition) 
    throws IOException {
    // 1. 检查缓冲区剩余空间
    bufferRemaining -= METASIZE;
    if (bufferRemaining <= 0) {
        // 2. 检查是否需要触发溢写
        spillLock.lock();
        try {
            if (!spillInProgress && kvindex != kvend) {
                // 触发溢写
                startSpill();
                // 重新计算 equator 和 bufferRemaining
                setEquator(newPos);
                bufferRemaining = Math.min(
                    distanceTo(bufend, newPos),
                    Math.min(
                        distanceTo(newPos, serBound),
                        softLimit)) - 2 * METASIZE;
            }
        } finally {
            spillLock.unlock();
        }
    }
    
    // 3. 序列化 key
    int keystart = bufindex;
    keySerializer.serialize(key);
    if (bufindex < keystart) {
        // key 跨越了缓冲区边界，需要移动数据
        bb.shiftBufferedKey();
        keystart = 0;
    }
    
    // 4. 序列化 value
    final int valstart = bufindex;
    valSerializer.serialize(value);
    bb.write(b0, 0, 0);  // 零长度写入，确保边界检查
    
    // 5. 记录元数据
    int valend = bb.markRecord();
    kvmeta.put(kvindex + PARTITION, partition);
    kvmeta.put(kvindex + KEYSTART, keystart);
    kvmeta.put(kvindex + VALSTART, valstart);
    kvmeta.put(kvindex + VALLEN, distanceTo(valstart, valend));
    
    // 6. 更新索引
    kvindex = (kvindex - NMETA + kvmeta.capacity()) % kvmeta.capacity();
}
```

#### 2. setEquator - 设置分界线
```java
private void setEquator(int pos) {
    equator = pos;
    // 对齐到元数据边界
    final int aligned = pos - (pos % METASIZE);
    // 设置 kvindex 到第一个元数据记录之前
    kvindex = (int)
        (((long)aligned - METASIZE + kvbuffer.length) % kvbuffer.length) / 4;
}
```

#### 3. resetSpill - 重置溢写状态
```java
private void resetSpill() {
    final int e = equator;
    bufstart = bufend = e;
    final int aligned = e - (e % METASIZE);
    // 设置 start/end 到第一个元数据记录
    kvstart = kvend = (int)
        (((long)aligned - METASIZE + kvbuffer.length) % kvbuffer.length) / 4;
}
```

---

### 3.3 SpillThread - 溢写线程

**位置**: `MapTask.java:1553`

**作用**: 后台线程，负责将缓冲区数据排序并写入磁盘

**核心方法**:
```java
protected class SpillThread extends SubjectInheritingThread {
    @Override
    public void work() {
        spillLock.lock();
        spillThreadRunning = true;
        try {
            while (true) {
                // 1. 通知主线程 spill 线程已就绪
                spillDone.signal();
                
                // 2. 等待溢写触发
                while (!spillInProgress) {
                    spillReady.await();
                }
                
                try {
                    // 3. 释放锁，执行溢写
                    spillLock.unlock();
                    sortAndSpill();
                } catch (Throwable t) {
                    sortSpillException = t;
                } finally {
                    // 4. 重新获取锁，重置状态
                    spillLock.lock();
                    if (bufend < bufstart) {
                        bufvoid = kvbuffer.length;
                    }
                    kvstart = kvend;
                    bufstart = bufend;
                    spillInProgress = false;
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            spillLock.unlock();
            spillThreadRunning = false;
        }
    }
}
```

---

### 3.4 sortAndSpill - 排序和溢写

**位置**: `MapTask.java:1617`

**作用**: 对缓冲区数据进行排序，按分区写入磁盘

**核心流程**:
```java
private void sortAndSpill() throws IOException, ClassNotFoundException,
                                   InterruptedException {
    // 1. 创建 SpillRecord
    final SpillRecord spillRec = new SpillRecord(partitions);
    
    // 2. 创建溢写文件
    final Path filename = mapOutputFile.getSpillFileForWrite(numSpills, size);
    FSDataOutputStream out = rfs.create(filename);
    
    // 3. 排序
    final int mstart = kvend / NMETA;
    final int mend = 1 + (kvstart >= kvend ? kvstart : kvmeta.capacity() + kvstart) / NMETA;
    sorter.sort(MapOutputBuffer.this, mstart, mend, reporter);
    
    // 4. 按分区写入数据
    int spindex = mstart;
    final IndexRecord rec = new IndexRecord();
    for (int i = 0; i < partitions; ++i) {
        IFile.Writer<K, V> writer = null;
        try {
            long segmentStart = out.getPos();
            writer = new Writer<K, V>(job, partitionOut, keyClass, valClass, codec,
                                      spilledRecordsCounter);
            
            if (combinerRunner == null) {
                // 直接写入
                while (spindex < mend &&
                    kvmeta.get(offsetFor(spindex % maxRec) + PARTITION) == i) {
                    final int kvoff = offsetFor(spindex % maxRec);
                    int keystart = kvmeta.get(kvoff + KEYSTART);
                    int valstart = kvmeta.get(kvoff + VALSTART);
                    key.reset(kvbuffer, keystart, valstart - keystart);
                    getVBytesForOffset(kvoff, value);
                    writer.append(key, value);
                    ++spindex;
                }
            } else {
                // 使用 Combiner
                combineCollector.setWriter(writer);
                RawKeyValueIterator kvIter = new MRResultIterator(spstart, spindex);
                combinerRunner.combine(kvIter, combineCollector);
            }
            
            // 5. 记录索引
            rec.startOffset = segmentStart;
            rec.rawLength = writer.getRawLength() + CryptoUtils.cryptoPadding(job);
            rec.partLength = writer.getCompressedLength() + CryptoUtils.cryptoPadding(job);
            spillRec.putIndex(rec, i);
            
            writer.close();
        } finally {
            if (null != writer) writer.close();
        }
    }
    
    // 6. 写入索引文件
    if (totalIndexCacheMemory >= indexCacheMemoryLimit) {
        Path indexFilename = mapOutputFile.getSpillIndexFileForWrite(numSpills, 
            partitions * MAP_OUTPUT_INDEX_RECORD_LENGTH);
        spillRec.writeToFile(indexFilename, job);
    } else {
        indexCacheList.add(spillRec);
        totalIndexCacheMemory += spillRec.size() * MAP_OUTPUT_INDEX_RECORD_LENGTH;
    }
    
    incrementNumSpills();
}
```

---

### 3.5 SpillRecord - 溢写索引记录

**位置**: `SpillRecord.java`

**作用**: 管理溢写文件的索引信息

**核心字段**:
```java
private final ByteBuffer buf;          // 背景存储
private final LongBuffer entries;      // long 数组视图
```

**索引格式**:
```
每个分区的索引占用 3 个 long（24 字节）：
- startOffset: 分区在文件中的起始偏移
- rawLength: 原始数据长度
- partLength: 压缩后的数据长度
```

**核心方法**:

#### 1. 构造函数
```java
public SpillRecord(int numPartitions) {
    buf = ByteBuffer.allocate(
        numPartitions * MapTask.MAP_OUTPUT_INDEX_RECORD_LENGTH);
    entries = buf.asLongBuffer();
}
```

#### 2. getIndex - 获取索引
```java
public IndexRecord getIndex(int partition) {
    final int pos = partition * MapTask.MAP_OUTPUT_INDEX_RECORD_LENGTH / 8;
    return new IndexRecord(entries.get(pos), entries.get(pos + 1),
                           entries.get(pos + 2));
}
```

#### 3. putIndex - 设置索引
```java
public void putIndex(IndexRecord rec, int partition) {
    final int pos = partition * MapTask.MAP_OUTPUT_INDEX_RECORD_LENGTH / 8;
    entries.put(pos, rec.startOffset);
    entries.put(pos + 1, rec.rawLength);
    entries.put(pos + 2, rec.partLength);
}
```

#### 4. writeToFile - 写入文件
```java
public void writeToFile(Path loc, JobConf job) throws IOException {
    final FileSystem rfs = FileSystem.getLocal(job).getRaw();
    CheckedOutputStream chk = null;
    final FSDataOutputStream out = rfs.create(loc);
    try {
        if (crc != null) {
            crc.reset();
            chk = new CheckedOutputStream(out, crc);
            chk.write(buf.array());
            out.writeLong(chk.getChecksum().getValue());
        } else {
            out.write(buf.array());
        }
    } finally {
        if (chk != null) {
            chk.close();
        } else {
            out.close();
        }
    }
}
```

---

### 3.6 IndexRecord - 索引记录

**位置**: `IndexRecord.java`

**作用**: 表示单个分区的索引信息

**核心字段**:
```java
public long startOffset;    // 分区在文件中的起始偏移
public long rawLength;      // 原始数据长度
public long partLength;     // 压缩后的数据长度
```

---

### 3.7 MapOutputFile - 文件路径管理

**位置**: `MapOutputFile.java`

**作用**: 管理 Map 输出文件的路径

**文件命名规则**（YarnOutputFiles 实现）:
```java
// 溢写文件
private static final String SPILL_FILE_PATTERN = "%s_spill_%d.out";
// 溢写索引文件
private static final String SPILL_INDEX_FILE_PATTERN = SPILL_FILE_PATTERN + ".index";
// 最终输出文件
static final String MAP_OUTPUT_FILENAME_STRING = "file.out";
// 最终索引文件
static final String MAP_OUTPUT_INDEX_SUFFIX_STRING = ".index";
```

**文件路径示例**:
```
output/
└── attempt_1234567890123_0001_m_000000_0/
    ├── attempt_1234567890123_0001_m_000000_0_spill_0.out
    ├── attempt_1234567890123_0001_m_000000_0_spill_0.out.index
    ├── attempt_1234567890123_0001_m_000000_0_spill_1.out
    ├── attempt_1234567890123_0001_m_000000_0_spill_1.out.index
    ├── ...
    ├── file.out
    └── file.out.index
```

**核心方法**:
```java
// 获取溢写文件
public Path getSpillFileForWrite(int spillNumber, long size) {
    return lDirAlloc.getLocalPathForWrite(
        String.format(SPILL_FILE_PATTERN,
            conf.get(JobContext.TASK_ATTEMPT_ID), spillNumber), size, conf);
}

// 获取溢写索引文件
public Path getSpillIndexFileForWrite(int spillNumber, long size) {
    return lDirAlloc.getLocalPathForWrite(
        String.format(SPILL_INDEX_FILE_PATTERN,
            conf.get(JobContext.TASK_ATTEMPT_ID), spillNumber), size, conf);
}

// 获取最终输出文件
public Path getOutputFileForWrite(long size) {
    Path attemptOutput = new Path(getAttemptOutputDir(), MAP_OUTPUT_FILENAME_STRING);
    return lDirAlloc.getLocalPathForWrite(attemptOutput.toString(), size, conf);
}

// 获取最终索引文件
public Path getOutputIndexFileForWrite(long size) {
    Path attemptIndexOutput = new Path(getAttemptOutputDir(), 
        MAP_OUTPUT_FILENAME_STRING + MAP_OUTPUT_INDEX_SUFFIX_STRING);
    return lDirAlloc.getLocalPathForWrite(attemptIndexOutput.toString(), size, conf);
}
```

---

### 3.8 mergeParts - 合并溢写文件

**位置**: `MapTask.java:1862`

**作用**: 合并所有溢写文件，生成最终的输出文件

**核心流程**:
```java
private void mergeParts() throws IOException, InterruptedException, 
                                 ClassNotFoundException {
    // 1. 计算最终文件大小
    long finalOutFileSize = 0;
    long finalIndexFileSize = 0;
    final Path[] filename = new Path[numSpills];
    
    for(int i = 0; i < numSpills; i++) {
        filename[i] = mapOutputFile.getSpillFile(i);
        finalOutFileSize += rfs.getFileStatus(filename[i]).getLen();
    }
    
    // 2. 如果只有一个溢写文件，直接重命名
    if (numSpills == 1) {
        Path indexFileOutput = mapOutputFile.getOutputIndexFileForWriteInVolume(filename[0]);
        sameVolRename(filename[0], mapOutputFile.getOutputFileForWriteInVolume(filename[0]));
        if (indexCacheList.size() == 0) {
            Path indexFilePath = mapOutputFile.getSpillIndexFile(0);
            sameVolRename(indexFilePath, indexFileOutput);
        } else {
            indexCacheList.get(0).writeToFile(indexFileOutput, job);
        }
        sortPhase.complete();
        return;
    }
    
    // 3. 读取所有索引文件
    for (int i = indexCacheList.size(); i < numSpills; ++i) {
        Path indexFileName = mapOutputFile.getSpillIndexFile(i);
        indexCacheList.add(new SpillRecord(indexFileName, job));
    }
    
    // 4. 创建最终输出文件
    Path finalOutputFile = mapOutputFile.getOutputFileForWrite(finalOutFileSize);
    Path finalIndexFile = mapOutputFile.getOutputIndexFileForWrite(finalIndexFileSize);
    FSDataOutputStream finalOut = rfs.create(finalOutputFile, true, 4096);
    
    // 5. 按分区合并
    final SpillRecord spillRec = new SpillRecord(partitions);
    for (int parts = 0; parts < partitions; parts++) {
        // 5.1 创建 Segment 列表
        List<Segment<K,V>> segmentList = new ArrayList<Segment<K, V>>(numSpills);
        for(int i = 0; i < numSpills; i++) {
            IndexRecord indexRecord = indexCacheList.get(i).getIndex(parts);
            Segment<K,V> s = new Segment<K,V>(job, rfs, filename[i], 
                indexRecord.startOffset, indexRecord.partLength, codec, true);
            segmentList.add(i, s);
        }
        
        // 5.2 合并 Segment
        int mergeFactor = job.getInt(MRJobConfig.IO_SORT_FACTOR,
            MRJobConfig.DEFAULT_IO_SORT_FACTOR);
        boolean sortSegments = segmentList.size() > mergeFactor;
        RawKeyValueIterator kvIter = Merger.merge(job, rfs,
            keyClass, valClass, codec,
            segmentList, mergeFactor,
            new Path(mapId.toString()),
            job.getOutputKeyComparator(), reporter, sortSegments,
            null, spilledRecordsCounter, sortPhase.phase(),
            TaskType.MAP);
        
        // 5.3 写入合并后的数据
        long segmentStart = finalOut.getPos();
        Writer<K, V> writer = new Writer<K, V>(job, finalPartitionOut, 
            keyClass, valClass, codec, spilledRecordsCounter);
        if (combinerRunner == null || numSpills < minSpillsForCombine) {
            Merger.writeFile(kvIter, writer, reporter, job);
        } else {
            combineCollector.setWriter(writer);
            combinerRunner.combine(kvIter, combineCollector);
        }
        
        // 5.4 记录索引
        rec.startOffset = segmentStart;
        rec.rawLength = writer.getRawLength() + CryptoUtils.cryptoPadding(job);
        rec.partLength = writer.getCompressedLength() + CryptoUtils.cryptoPadding(job);
        spillRec.putIndex(rec, parts);
        
        writer.close();
    }
    
    // 6. 写入最终索引文件
    spillRec.writeToFile(finalIndexFile, job);
    finalOut.close();
    
    // 7. 删除溢写文件
    for(int i = 0; i < numSpills; i++) {
        rfs.delete(filename[i], true);
    }
}
```

---

## 四、工作流程

### 4.1 完整的 Map 端 Shuffle 流程

```
1. MapTask 启动
   ↓
2. 初始化 MapOutputBuffer
   - 创建环形缓冲区（kvbuffer）
   - 启动 SpillThread
   ↓
3. Map 函数输出 key-value 对
   ↓
4. MapOutputBuffer.collect() 收集数据
   - 序列化 key 和 value
   - 写入环形缓冲区
   - 记录元数据（分区、key 位置、value 位置、value 长度）
   ↓
5. 检查缓冲区是否达到软限制
   ↓
6. 如果达到软限制，触发溢写
   - startSpill() 设置溢写标志
   - spillReady.signal() 唤醒 SpillThread
   ↓
7. SpillThread 执行 sortAndSpill()
   - 对缓冲区数据排序（按分区，然后按 key）
   - 按分区写入溢写文件
   - 为每个分区创建索引记录
   - 写入索引文件或缓存到内存
   ↓
8. 主线程继续收集数据
   ↓
9. Map 函数执行完成
   ↓
10. MapOutputBuffer.flush()
    - 等待所有溢写完成
    - 执行最后一次溢写
    ↓
11. mergeParts() 合并所有溢写文件
    - 如果只有一个溢写文件，直接重命名
    - 如果有多个溢写文件，按分区合并
    - 生成最终输出文件和索引文件
    ↓
12. 设置文件权限
    ↓
13. MapTask 完成
```

### 4.2 环形缓冲区工作原理

**初始状态**:
```
kvbuffer (100MB)
┌─────────────────────────────────────────────────────────┐
│  元数据区 (kvmeta)  │  序列化数据区 (kvbuffer)          │
│  [空闲]              │  [空闲]                          │
└─────────────────────────────────────────────────────────┘
         ↑ kvstart/kvend                    ↑ bufstart/bufend
         ↑ kvindex                          ↑ bufindex
```

**写入数据后**:
```
kvbuffer (100MB)
┌─────────────────────────────────────────────────────────┐
│  元数据区 (kvmeta)  │  序列化数据区 (kvbuffer)          │
│  [rec1|rec2|rec3]   │  [key1|val1|key2|val2|key3|val3]  │
└─────────────────────────────────────────────────────────┘
         ↑ kvstart                    ↑ bufstart
         ↑ kvindex                   ↑ bufindex
                                     ↑ bufend
```

**触发溢写后**:
```
kvbuffer (100MB)
┌─────────────────────────────────────────────────────────┐
│  元数据区 (kvmeta)  │  序列化数据区 (kvbuffer)          │
│  [空闲]              │  [空闲]                          │
└─────────────────────────────────────────────────────────┘
         ↑ kvstart/kvend                    ↑ bufstart/bufend
         ↑ kvindex                          ↑ bufindex
```

### 4.3 溢写文件格式

**溢写文件结构**:
```
attempt_1234567890123_0001_m_000000_0_spill_0.out
┌─────────────────────────────────────────────────────────┐
│  IFile Header                                             │
│  - Magic Number                                           │
│  - Key Class Name                                         │
│  - Value Class Name                                       │
│  - Compression Codec                                      │
│  - Metadata                                               │
├─────────────────────────────────────────────────────────┤
│  Partition 0 Data                                         │
│  - [key1, value1]                                         │
│  - [key2, value2]                                         │
│  - ...                                                    │
├─────────────────────────────────────────────────────────┤
│  Partition 1 Data                                         │
│  - [key3, value3]                                         │
│  - [key4, value4]                                         │
│  - ...                                                    │
├─────────────────────────────────────────────────────────┤
│  ...                                                      │
└─────────────────────────────────────────────────────────┘
```

**索引文件结构**:
```
attempt_1234567890123_0001_m_000000_0_spill_0.out.index
┌─────────────────────────────────────────────────────────┐
│  Partition 0 Index                                        │
│  - startOffset: 0                                         │
│  - rawLength: 1024                                        │
│  - partLength: 512                                        │
├─────────────────────────────────────────────────────────┤
│  Partition 1 Index                                        │
│  - startOffset: 1024                                      │
│  - rawLength: 2048                                        │
│  - partLength: 1024                                       │
├─────────────────────────────────────────────────────────┤
│  ...                                                      │
└─────────────────────────────────────────────────────────┘
```

---

## 五、关键特性

### 5.1 环形缓冲区

**优势**:
- 避免频繁的内存分配和拷贝
- 支持并发读写（主线程写入，溢写线程读取）
- 内存利用率高

**关键设计**:
- 元数据和序列化数据共享同一个缓冲区
- 使用 equator 分隔元数据区和序列化数据区
- 支持数据跨越缓冲区边界

### 5.2 排序

**排序规则**:
1. 先按分区排序
2. 同一分区内按 key 排序

**排序算法**:
- 默认使用 QuickSort
- 可配置其他排序算法（`mapreduce.sort.class`）

**排序时机**:
- 溢写前排序
- 合并时排序（如果 segment 数量超过 mergeFactor）

### 5.3 Combiner

**作用**: 在 Map 端进行本地聚合，减少网络传输

**触发条件**:
- 配置了 Combiner
- 溢写次数 >= `mapreduce.map.combine.min.spills`（默认 3）

**实现**:
```java
if (combinerRunner == null || numSpills < minSpillsForCombine) {
    // 直接写入
    Merger.writeFile(kvIter, writer, reporter, job);
} else {
    // 使用 Combiner
    combineCollector.setWriter(writer);
    combinerRunner.combine(kvIter, combineCollector);
}
```

### 5.4 压缩

**支持**:
- Map 输出压缩
- 可配置压缩算法（`mapreduce.map.output.compress.codec`）

**实现**:
```java
if (job.getCompressMapOutput()) {
    Class<? extends CompressionCodec> codecClass =
        job.getMapOutputCompressorClass(DefaultCodec.class);
    codec = ReflectionUtils.newInstance(codecClass, job);
}
```

### 5.5 索引缓存

**作用**: 缓存溢写文件的索引，减少磁盘 I/O

**策略**:
- 如果索引缓存内存 < 限制，缓存到内存
- 如果索引缓存内存 >= 限制，写入磁盘

**实现**:
```java
if (totalIndexCacheMemory >= indexCacheMemoryLimit) {
    // 写入磁盘
    Path indexFilename = mapOutputFile.getSpillIndexFileForWrite(numSpills, 
        partitions * MAP_OUTPUT_INDEX_RECORD_LENGTH);
    spillRec.writeToFile(indexFilename, job);
} else {
    // 缓存到内存
    indexCacheList.add(spillRec);
    totalIndexCacheMemory += spillRec.size() * MAP_OUTPUT_INDEX_RECORD_LENGTH;
}
```

### 5.6 溢写文件数量限制

**作用**: 防止产生过多的溢写文件

**配置**:
- `mapreduce.task.spill.files.count.limit`（默认 -1，无限制）

**实现**:
```java
private void incrementNumSpills() throws IOException {
    ++numSpills;
    if(spillFilesCountLimit != SPILL_FILES_COUNT_UNBOUNDED_VALUE
        && numSpills > spillFilesCountLimit) {
        throw new IOException("Too many spill files got created, control it with " +
            "mapreduce.task.spill.files.count.limit, current value: " + 
            spillFilesCountLimit + ", current spill count: " + numSpills);
    }
}
```

---

## 六、配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `mapreduce.task.io.sort.mb` | 100 | 排序缓冲区大小（MB） |
| `mapreduce.map.sort.spill.percent` | 0.8 | 溢写百分比（0-1） |
| `mapreduce.task.index.cache.memory.limit.bytes` | 1MB | 索引缓存内存限制 |
| `mapreduce.task.spill.files.count.limit` | -1 | 溢写文件数量限制（-1 表示无限制） |
| `mapreduce.map.combine.min.spills` | 3 | 触发 Combiner 的最小溢写次数 |
| `mapreduce.map.output.compress` | false | 是否压缩 Map 输出 |
| `mapreduce.map.output.compress.codec` | DefaultCodec | Map 输出压缩算法 |
| `mapreduce.sort.class` | QuickSort | 排序算法类 |
| `mapreduce.task.io.sort.factor` | 10 | 合并因子 |

---

## 七、性能优化

### 7.1 调整缓冲区大小

**原则**:
- 缓冲区越大，溢写次数越少，但内存占用越高
- 建议设置为 200-500MB

**配置**:
```xml
<property>
    <name>mapreduce.task.io.sort.mb</name>
    <value>200</value>
</property>
```

### 7.2 调整溢写百分比

**原则**:
- 百分比越高，溢写次数越少，但内存占用越高
- 建议设置为 0.8-0.9

**配置**:
```xml
<property>
    <name>mapreduce.map.sort.spill.percent</name>
    <value>0.9</value>
</property>
```

### 7.3 启用压缩

**优势**:
- 减少磁盘 I/O
- 减少网络传输

**配置**:
```xml
<property>
    <name>mapreduce.map.output.compress</name>
    <value>true</value>
</property>
<property>
    <name>mapreduce.map.output.compress.codec</name>
    <value>org.apache.hadoop.io.compress.SnappyCodec</value>
</property>
```

### 7.4 使用 Combiner

**优势**:
- 减少 Map 输出数据量
- 减少 Shuffle 网络传输

**配置**:
```java
job.setCombinerClass(MyCombiner.class);
```

### 7.5 调整合并因子

**原则**:
- 合并因子越大，合并次数越少，但内存占用越高
- 建议设置为 10-20

**配置**:
```xml
<property>
    <name>mapreduce.task.io.sort.factor</name>
    <value>20</value>
</property>
```

---

## 八、总结

Map 端的 Shuffle 是 MapReduce 框架的核心组件，具有以下特点：

1. **高效性**: 使用环形缓冲区，避免频繁的内存分配和拷贝
2. **并发性**: 主线程和溢写线程并发执行，提高吞吐量
3. **可扩展性**: 支持自定义排序算法、压缩算法等
4. **可靠性**: 支持索引缓存、溢写文件数量限制等机制
5. **可配置性**: 丰富的配置参数，可根据场景调优

**关键设计思想**:
- 环形缓冲区设计，支持并发读写
- 元数据和序列化数据共享内存，提高内存利用率
- 溢写机制，避免内存溢出
- 合并机制，减少文件数量
- 索引机制，支持快速定位分区数据

通过深入理解 Map 端 Shuffle 的实现，可以更好地优化 Hadoop MapReduce 的性能，排查相关问题。
