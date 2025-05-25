# 说明

DistCp（分布式拷贝）是用于大规模集群内部和集群之间拷贝的工具。 
它使用Map/Reduce实现文件分发，错误处理和恢复，以及报告生成。 
它把文件和目录的列表作为map任务的输入，每个任务会完成源列表中部分文件的拷贝。 
由于使用了Map/Reduce方法，这个工具在语义和执行上都会有特殊的地方。 这篇文档会为常用DistCp操作提供指南并阐述它的工作模型。

# 源码详解

## 作业启动

作业的启动主要包含初始化和作业提交，在初始化阶段主要是list左右需要拷贝的文件信息，根据文件信息构造split信息。
作业提交阶段就是根据初始化阶段构造的split信息，将作业提交到Yarn上面。

### 作业初始化

初始化阶段主要是list左右需要拷贝的文件信息，根据文件信息构造split信息。

DistCp的入口函数是main函数，在main函数里面主要做了两件事：

- 注册Cleanup。
- 初始化和启动作业，核心处理函数为execute函数里面的createAndSubmitJob

创建Job对象,主要是指定Map的处理类，InputFormat 和outputFormat 信息:

```java
Job job = Job.getInstance(getConf());
job.setJobName(jobName);
job.setInputFormatClass(DistCpUtils.getStrategy(getConf(), context));
job.setJarByClass(CopyMapper.class);
configureOutputFormat(job);

job.setMapperClass(CopyMapper.class);
job.setOutputFormatClass(CopyOutputFormat.class);
job.getConfiguration().set(JobContext.MAP_SPECULATIVE, "false");
```

根据需要拷贝的目录获取所有的文件信息。支持snapshot模式和普通模式。

#### snapshot 模式

核心函数为SimpleCopyListing.doBuildListingWithSnapshotDiff。主要是通过DistCpSync.getAllDiffs获取Snapshot的差异文件。
差异文件主要包含创建、修改、删除类型，将差别的的文件输出到fileList.seq文件里面。fileList.seq文件在staging目录下面的的`_distcp_随机的int值`。

#### 普通模式

核心函数为SimpleCopyListing.doBuildListing。对于非snapshot模式，核心处理逻辑就是通过list将所有的文件获取出来。添加到fileList.seq里面。
对于XAttrs等权限信息也会按照-p参数指定的来获取。

### 作业提交

由于DistCp也是MapReduce作业，所以作业提交沿用了MapReduce作业提交的框架，对于Map和Reduce的处理类，
以及InputFormat和outputFormat都是DistCp自己实现的。

其中比较常用的是DynamicInputFormat，DynamicInputFormat主要是通过主要是按照文件数量分配的。



## 作业运行

### AM运行

在创建作业的时候定义了outputFormat，在CopyOutputFormat中定义了getOutputCommitter。

```java
job.setOutputFormatClass(CopyOutputFormat.class);
```

Distcp的AM结束时的核心处理类是CopyCommitter。结束的时候会调用commitJob函数，在commitJob函数里面。

#### deleteMissing函数

从目标端删除多余的文件，需要配置-delete参数。

#### preserveFileAttributesForDirectories函数



### Map运行

### Reduce运行


