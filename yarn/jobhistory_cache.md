## jobhistory 作业缓存

jobhistory 一般会保存一部分作业信息到内存中，查询作业信息的时候一般会从内存查询，如果内存查询不到就会从磁盘上扫描。

jobhistory 缓存一般分为两层，第一层是guava缓存，默认情况下guava的缓存个数是5，可以通过配置项`mapreduce.jobhistory.loadedjobs.cache.size`控制。

当guava的一级缓存中不存在的时候，默认是需要重新加载的，jobhistory中定义了加载规则,定义代码如下：

```java
CacheLoader<JobId, Job> loader;
loader = new CacheLoader<JobId, Job>() {
  @Override
  public Job load(JobId key) throws Exception {
    return loadJob(key);
  }
};
```

其中loadJob实现如下，其中hsManager为加载具体实现，

```java
private Job loadJob(JobId jobId) throws RuntimeException, IOException {
  if (LOG.isDebugEnabled()) {
    LOG.debug("Looking for Job " + jobId);
  }
  HistoryFileInfo fileInfo;

  fileInfo = hsManager.getFileInfo(jobId);

  if (fileInfo == null) {
    throw new HSFileRuntimeException("Unable to find job " + jobId);
  }

  fileInfo.waitUntilMoved();

  if (fileInfo.isDeleted()) {
    throw new HSFileRuntimeException("Cannot load deleted job " + jobId);
  } else {
    return fileInfo.loadJob();
  }
}
```

hsManager中定义了jobhistory的二级缓存：jobListCache，jobListCache的大小可以通过配置项`mapreduce.jobhistory.joblist.cache.size`控制。
默认可以保存20000个。当然缓存超时指定时间可会被清理，具体可以有配置项`mapreduce.jobhistory.max-age-ms`控制，默认为1周。

查找的顺序为：

- 优先从内存查找（二级缓存），为jobListCache。
- 如果缓存找不见，优先扫描刚完成的作业所在的目录，会刷新jobListCache缓存，由配置项mapreduce.jobhistory.intermediate-done-dir控制。
- 如果还是找不见，从已经完成的作业的目录扫描，具体目录由配置项mapreduce.jobhistory.done-dir控制。

```java
public HistoryFileInfo getFileInfo(JobId jobId) throws IOException {
  // 优先从内存查找（二级缓存）
  HistoryFileInfo fileInfo = jobListCache.get(jobId);
  if (fileInfo != null) {
    return fileInfo;
  }
  // 如果缓存找不见，优先扫描刚完成的作业所在的目录，由配置项mapreduce.jobhistory.intermediate-done-dir控制
  scanIntermediateDirectory();
  fileInfo = jobListCache.get(jobId);
  if (fileInfo != null) {
    return fileInfo;
  }

  // 如果还是找不见，从已经完成的作业的目录扫描，具体目录由配置项mapreduce.jobhistory.done-dir控制
  fileInfo = scanOldDirsForJob(jobId);
  if (fileInfo != null) {
    return fileInfo;
  }
  return null;
}
```
