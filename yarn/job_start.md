# 作业启动

作业提交的客户端比较核心的类是Job.java，看作业启动的源码需要从这个类开始看。

## Job.java

作业启动的入口函数为waitForCompletion函数。当前函数的核心函数为submit()，主要如下：

```java
public void submit() 
      throws IOException, InterruptedException, ClassNotFoundException {
 ensureState(JobState.DEFINE);
 setUseNewAPI();
 connect();
 final JobSubmitter submitter = 
     getJobSubmitter(cluster.getFileSystem(), cluster.getClient());
 status = ugi.doAs(new PrivilegedExceptionAction<JobStatus>() {
   public JobStatus run() throws IOException, InterruptedException, 
   ClassNotFoundException {
     return submitter.submitJobInternal(Job.this, cluster);
   }
 });
 state = JobState.RUNNING;
 LOG.info("The url to track the job: " + getTrackingURL());
}
```

其中，connect主要为连接ResourceManager。核心提交类为submitJobInternal，在submitJobInternal中主要包含：

- 检查是否开启分布式缓存，核心函数为：`addMRFrameworkToDistributedCache(conf);`
- 从yarn上面获取Yarn ApplicationId。
- 将需要上传的文件拷贝到submitJobDir下面，将上传的结果添加到指定的配置中。主要实现在函数`copyAndConfigureFiles(job, submitJobDir);`里面，主要上传当前作业需要的jar包等信息到staging目录。当上传Jar包比较频繁的时候可以考虑开启分布式缓存。
- 初始化核心配置，主要实现在函数：`writeConf(conf, submitJobFile);`里面。
- 最后才是真正提交作业的部分：`status = submitClient.submitJob(jobId, submitJobDir.toString(), job.getCredentials());`通过submitClient.submitJob之后是远程调用到ResourceManager的类：YARNRunner.java，开始作业提交。



## YARNRunner.java

在当前类中，处理逻辑主要包含下面几步：

- 创建上下问信息：ApplicationSubmissionContext，当前这一步当中主要是构造AM相关参数，比如AM的启动命令等。在AM的启动命令中会设置AM的启动主函数MRAppMaster，在资源调度到当前作业时，会先启动AM的主函数MRAppMaster
- 提交作业。最后会调用到`rmClient.submitApplication(request);`发送启动作业的请求，在发送请求之后会一直等到作业启动完成。启动成功之后会返回appilicationId

## 资源调度

Yarn资源调度过程待完善，后面会单独章节学习。

## MRAppMaster.java

当前类是启动AM的入口函数，所以要从main函数开始读代码。main函数里面主要做了下面几件事：

- 初始化MRAppMaster实例。
- 加载job.xml信息。
- 初始化web信息。主要包含： MR history server、MR Server。
- 启动APPMaster。



### initAndStartAppMaster：启动AppMaster

MRAppMaster在yarn内部是一个服务，最终启动的时候会调用到serviceStart函数里面，所以我们主要看这个函数里面做了什么。



#### 1、创建并且初始化Job

创建Job对象并且将其初始化掉。但是不会启动当前作业。

- 初始化JobImpl对象。在JobImpl初始化的时候做了下面几件事：

  - 初始化线程池。

  - 初始化作业状态机的核心代码如下：

    ```java
    protected static final
      StateMachineFactory<JobImpl, JobStateInternal, JobEventType, JobEvent> 
         stateMachineFactory
       = new StateMachineFactory<JobImpl, JobStateInternal, JobEventType, JobEvent>
                (JobStateInternal.NEW)
            // Transitions from NEW state
            .addTransition(JobStateInternal.NEW, JobStateInternal.NEW,
                JobEventType.JOB_DIAGNOSTIC_UPDATE,
                DIAGNOSTIC_UPDATE_TRANSITION)
            .addTransition(JobStateInternal.NEW, JobStateInternal.NEW,
                JobEventType.JOB_COUNTER_UPDATE, COUNTER_UPDATE_TRANSITION)
            // ....省略...
            .addTransition(JobStateInternal.REBOOT, JobStateInternal.REBOOT,
                JobEventType.JOB_COUNTER_UPDATE, COUNTER_UPDATE_TRANSITION)
            // create the topology tables
            .installTopology();

  - 初始化其他配置。

- 在中央处理器里面注册JobFinishEvent类型事件以及事件处理的handler。

```java
protected Job createJob(Configuration conf, JobStateInternal forcedState, 
    String diagnostic) {
  // create single job
  Job newJob =
      new JobImpl(jobId, appAttemptID, conf, dispatcher.getEventHandler(),
          taskAttemptListener, jobTokenSecretManager, jobCredentials, clock,
          completedTasksFromPreviousRun, metrics,
          committer, newApiCommitter,
          currentUser.getUserName(), appSubmitTime, amInfos, context, 
          forcedState, diagnostic);
  ((RunningAppContext) context).jobs.put(newJob.getID(), newJob);
  dispatcher.register(JobFinishEvent.Type.class,
      createJobFinishEventHandler());     
  return newJob;
}
```



#### 2、发送inited事件

发送inited事件的对象主要是下面两个：

- 通过dispatcher给历史AM发送。
- 当前AM。代码如下：

```java
// Send out an MR AM inited event for this AM.
dispatcher.getEventHandler().handle(
    new JobHistoryEvent(job.getID(), new AMStartedEvent(amInfo
        .getAppAttemptId(), amInfo.getStartTime(), amInfo.getContainerId(),
        amInfo.getNodeManagerHost(), amInfo.getNodeManagerPort(), amInfo
            .getNodeManagerHttpPort(), this.forcedState == null ? null
                : this.forcedState.toString(), appSubmitTime)));
```



#### 3、创建job init事件，并且处理

创建init事件，核心代码如下：

```java
JobEvent initJobEvent = new JobEvent(job.getID(), JobEventType.JOB_INIT);
jobEventDispatcher.handle(initJobEvent);
```

事件处理的核心类为InitTransition，核心代码如下：

```java
public JobStateInternal transition(JobImpl job, JobEvent event) {
  job.metrics.submittedJob(job);
  job.metrics.preparingJob(job);
  // 初始化上下文。
  if (job.newApiCommitter) {
    job.jobContext = new JobContextImpl(job.conf,
        job.oldJobId);
  } else {
    job.jobContext = new org.apache.hadoop.mapred.JobContextImpl(
        job.conf, job.oldJobId);
  }
  
  try {
    // 初始化token等信息。
    setup(job);
    job.fs = job.getFileSystem(job.conf);

    //log to job history
    JobSubmittedEvent jse = new JobSubmittedEvent(job.oldJobId,
          job.conf.get(MRJobConfig.JOB_NAME, "test"), 
        job.conf.get(MRJobConfig.USER_NAME, "mapred"),
        job.appSubmitTime,
        job.remoteJobConfFile.toString(),
        job.jobACLs, job.queueName,
        job.conf.get(MRJobConfig.WORKFLOW_ID, ""),
        job.conf.get(MRJobConfig.WORKFLOW_NAME, ""),
        job.conf.get(MRJobConfig.WORKFLOW_NODE_NAME, ""),
        getWorkflowAdjacencies(job.conf),
        job.conf.get(MRJobConfig.WORKFLOW_TAGS, ""), job.conf);
    job.eventHandler.handle(new JobHistoryEvent(job.jobId, jse));
    //TODO JH Verify jobACLs, UserName via UGI?
    // 初始化并行度等信息。
    TaskSplitMetaInfo[] taskSplitMetaInfo = createSplits(job, job.jobId);
    job.numMapTasks = taskSplitMetaInfo.length;
    job.numReduceTasks = job.conf.getInt(MRJobConfig.NUM_REDUCES, 0);

    if (job.numMapTasks == 0 && job.numReduceTasks == 0) {
      job.addDiagnostic("No of maps and reduces are 0 " + job.jobId);
    } else if (job.numMapTasks == 0) {
      job.reduceWeight = 0.9f;
    } else if (job.numReduceTasks == 0) {
      job.mapWeight = 0.9f;
    } else {
      job.mapWeight = job.reduceWeight = 0.45f;
    }

    checkTaskLimits();
    
   // 加载其他参数，具体代码省略。。

    cleanupSharedCacheUploadPolicies(job.conf);

    // create the Tasks but don't start them yet，， 创建map task
    createMapTasks(job, inputLength, taskSplitMetaInfo);
    // 创建reduce tasks
    createReduceTasks(job);

    job.metrics.endPreparingJob(job);
    return JobStateInternal.INITED;
  } catch (Exception e) {
    LOG.warn("Job init failed", e);
    job.metrics.endPreparingJob(job);
    job.addDiagnostic("Job init failed : "
        + StringUtils.stringifyException(e));
    // Leave job in the NEW state. The MR AM will detect that the state is
    // not INITED and send a JOB_INIT_FAILED event.
    return JobStateInternal.NEW;
  }
}
```



#### 4、检查初始化结果并且启动作业

当init成功时，handler返回的结果是JobStateInternal.INITED；如果是失败了则返回的结果是JobStateInternal.NEW。

对于初始化失败的作业会触发JobEventType.JOB_INIT_FAILED事件。

对于初始化成功的作业会调用函数startJobs，继续启动作业。触发

```java
protected void startJobs() {
  /** create a job-start event to get this ball rolling */
  JobEvent startJobEvent = new JobStartEvent(job.getID(),
      recoveredJobStartTime);
  /** send the job-start event. this triggers the job execution. */
  dispatcher.getEventHandler().handle(startJobEvent);
}
```

核心处理逻辑如下，主要是触发了几个事件：

- JobHistoryEvent：事件处理的handler为JobHistoryEventHandler。
- JobInfoChangeEvent：
- CommitterJobSetupEvent：作业启动的事件，核心处理逻辑在EventProcessor中的函数handleJobSetup中。

```java
public void transition(JobImpl job, JobEvent event) {
  JobStartEvent jse = (JobStartEvent) event;
  if (jse.getRecoveredJobStartTime() != -1L) {
    job.startTime = jse.getRecoveredJobStartTime();
  } else {
    job.startTime = job.clock.getTime();
  }
  JobInitedEvent jie =
    new JobInitedEvent(job.oldJobId,
         job.startTime,
         job.numMapTasks, job.numReduceTasks,
         job.getState().toString(),
         job.isUber());
  job.eventHandler.handle(new JobHistoryEvent(job.jobId, jie));
  JobInfoChangeEvent jice = new JobInfoChangeEvent(job.oldJobId,
      job.appSubmitTime, job.startTime);
  job.eventHandler.handle(new JobHistoryEvent(job.jobId, jice));
  job.metrics.runningJob(job);

  job.eventHandler.handle(new CommitterJobSetupEvent(
          job.jobId, job.jobContext));
}
```

handleJobSetup的核心处理逻辑：

- 创建attempt路径。
- 触发JobSetupCompletedEvent事件。从事件实现来看会触发JobImpl里面的JOB_SETUP_COMPLETED事件类型，由SetupCompletedTransition来处理当前事件。在当前函数里面会触发JOB_COMPLETED事件。最终会走到JobImpl的checkReadyForCommit函数里面。

```java
protected void handleJobSetup(CommitterJobSetupEvent event) {
  try {
    // 主要是创建attempt路径
    committer.setupJob(event.getJobContext());
    context.getEventHandler().handle(
        new JobSetupCompletedEvent(event.getJobID()));
  } catch (Exception e) {
    LOG.warn("Job setup failed", e);
    context.getEventHandler().handle(new JobSetupFailedEvent(
        event.getJobID(), StringUtils.stringifyException(e)));
  }
}
```

SetupCompletedTransition的处理逻辑如下，可以看到会定时启动MapTask和ReduceTask。

```java
public void transition(JobImpl job, JobEvent event) {
  job.setupProgress = 1.0f;
  job.scheduleTasks(job.mapTasks, job.numReduceTasks == 0);
  job.scheduleTasks(job.reduceTasks, true);

  // If we have no tasks, just transition to job completed
  if (job.numReduceTasks == 0 && job.numMapTasks == 0) {
    job.eventHandler.handle(new JobEvent(job.jobId,
        JobEventType.JOB_COMPLETED));
  }
}
```

checkReadyForCommit函数的实现如下，可以看到在触发了CommitterJobCommitEvent事件,在CommitterJobCommitEvent里面会触发JOB_COMMIT事件。主要处理逻辑在handleJobCommit里面。

```java
protected JobStateInternal checkReadyForCommit() {
  JobStateInternal currentState = getInternalState();
  if (completedTaskCount == tasks.size()
      && currentState == JobStateInternal.RUNNING) {
    eventHandler.handle(new CommitterJobCommitEvent(jobId, getJobContext()));
    return JobStateInternal.COMMITTING;
  }
  // return the current state as job not ready to commit yet
  return getInternalState();
}
```

handleJobCommit处理逻辑如下，

```java
protected void handleJobCommit(CommitterJobCommitEvent event) {
  boolean commitJobIsRepeatable = false;
  try {
    // 检查作业是否重复。
    commitJobIsRepeatable = committer.isCommitJobRepeatable(
        event.getJobContext());
  } catch (IOException e) {
    LOG.warn("Exception in committer.isCommitJobRepeatable():", e);
  }

  try {
    // 创建文件：/tmp/hadoop-yarn/staging//user/.staging/{jobid}/COMMIT_STARTED
    touchz(startCommitFile, commitJobIsRepeatable);
    jobCommitStarted();
    // 检查和RM的心跳。
    waitForValidCommitWindow();
    // 提交作业，核心处理函数在commitJobInternal里面
    committer.commitJob(event.getJobContext());
    // 创建文件：/tmp/hadoop-yarn/staging//user/.staging/{jobid}/COMMIT_SUCCESS
    touchz(endCommitSuccessFile, commitJobIsRepeatable);
    context.getEventHandler().handle(
        new JobCommitCompletedEvent(event.getJobID()));
  } catch (Exception e) {
    LOG.error("Could not commit job", e);
    try {
      // 失败之后创建：/tmp/hadoop-yarn/staging//user/.staging/{jobid}/COMMIT_FAIL
      touchz(endCommitFailureFile, commitJobIsRepeatable);
    } catch (Exception e2) {
      LOG.error("could not create failure file.", e2);
    }
    context.getEventHandler().handle(
        new JobCommitFailedEvent(event.getJobID(),
            StringUtils.stringifyException(e)));
  } finally {
    jobCommitEnded();
  }
}
```



##### CommitSucceededTransition

提交成功的事件处理handler为CommitSucceededTransition，核心处理逻辑如下：

```java
job.logJobHistoryFinishedEvent();
job.finished(JobStateInternal.SUCCEEDED);
```
