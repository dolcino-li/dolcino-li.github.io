---
layout: post
title: Spark Executor
---

# Spark Executor Related

这一篇讲解一些 spark 比较基础的内容。主要 force 在 spark-submit 之后，launch driver 和 master 的这个过程，以及后续的一些资源调度的内容。

这里考虑的配置情况是，使用 yarn 作为调度器，开启 dynamic allocation，并且 deploy mode 是 cluster。

## Launch Driver & Application Master

先高度概括一下。

![launchDriver](/assets/img/spark/launchDriver.png)

1. 用户用 spark submit 提交一个 application 到 ResourceManager, ResourceManager 给他分配一个 container 来启动 ApplicationMaster.
2. ApplicationMaster 启动之后，先运行了 userClassThread，其实就是 driver，主要是为了创建出来 sparkContext，此时 ApplicationMaster 这个线程在等待 sparkContextPromise。
3. userClassThread / driver 创建 sparkContext
4. UserClassThread 创建出来 sparkContext 之后，会给 sparkContextPromise 赋值，并且掉了 sparkContextPromise.wait()，阻塞在这里，等待唤醒。
5. ApplicationMaster 向 ResourceManager 注册。
6. 创建 Allocator Thread，用来为 driver allocate container。
7. sparkContextPromise.notify(), 唤醒 userClassThread / driver。
8. applicationMaster 等待 driver 死亡，driver 继续运行 user code。

### 提交到 RM

提交一个 spark job 的入口是 sparkSubmit 方法。

```
org.apache.spark.deploy.SparkSubmit#doSubmit

  def doSubmit(args: Array[String]): Unit = {
    // Initialize logging if it hasn't been done yet. Keep track of whether logging needs to
    // be reset before the application starts.
    val uninitLog = initializeLogIfNecessary(true, silent = true)

    val appArgs = parseArguments(args)
    if (appArgs.verbose) {
      logInfo(appArgs.toString)
    }
    appArgs.action match {
      case SparkSubmitAction.SUBMIT => submit(appArgs, uninitLog)
      case SparkSubmitAction.KILL => kill(appArgs)
      case SparkSubmitAction.REQUEST_STATUS => requestStatus(appArgs)
      case SparkSubmitAction.PRINT_VERSION => printVersion()
    }
  }
```

然后就会继续调用 submit 方法里，会继续调用 `org.apache.spark.deploy.SparkSubmit#runMain` 方法，在这个方法里会进行一些参数的配置，以及一些环境变量的配置。 `runMain` 方法运行到最后会跑 `app.start(childArgs.toArray, sparkConf)`, 这个 app 来自于下面这样 代码
```
    val app: SparkApplication = if (classOf[SparkApplication].isAssignableFrom(mainClass)) {
      mainClass.getConstructor().newInstance().asInstanceOf[SparkApplication]
    } else {
      new JavaMainApplication(mainClass)
    }
```
由于我们使用的 yarn 模式，SparkApplication 这个 trait 的具体实现类就是`org.apache.spark.deploy.yarn.YarnClusterApplication`。在 yarnCluasterApplication 里实现了 一个 Client，用来和对应的 ResourceManager 做通讯。 `YarnClusterApplication.start()` 里，运行了 `client.run()`。

```
org.apache.spark.deploy.yarn.YarnClusterApplication#Client.run
  def run(): Unit = {
    this.appId = submitApplication()
    if (!launcherBackend.isConnected() && fireAndForget) {
      val report = getApplicationReport(appId)
      val state = report.getYarnApplicationState
      logInfo(s"Application report for $appId (state: $state)")
      logInfo(formatReportDetails(report, getDriverLogsLink(report)))
      if (state == YarnApplicationState.FAILED || state == YarnApplicationState.KILLED) {
        throw new SparkException(s"Application $appId finished with status: $state")
      }
    } else {
        //Ignore For Now.
  }
```

在 client.run 方法里，首先 call 了 `submitApplication()`，在这个方法将 App 提交给 ResourceManager。

```
org.apache.spark.deploy.yarn.YarnClusterApplication#Client.submitApplication

    // Set up the appropriate contexts to launch our AM
    val containerContext = createContainerLaunchContext(newAppResponse)
    val appContext = createApplicationSubmissionContext(newApp, containerContext)
```

在 createContainerLaunchContext 里，在设置 application master 相关的内容，类似运行方式，以及需要设置的环境变量。
在 createApplicationSubmissionContext 里，在设置 和 ResourceManager 相关的配置，类似于 app 的名字，提交到哪个Queue，application master 需要使用的资源等。

这两步设置结束之后，就会调用 `yarnClient.submitApplication(appContext)` 将这个 spark job 提交给 ResourceManager。等待 ResourceManager 来进行调度。


### Application Master 启动

在 ResourceManager 给这个 Spark Job 一个 container 之后，会按照上一步中 createContainerLaunchContext 约定的步骤来启动这个 container。对于一个 spark job 而言，可以简单理解为，启动这个 spark 的 main class 是 `org.apache.spark.deploy.yarn.ApplicationMaster` (cluster mode)。

那现在这个 container 的入口就是 `org.apache.spark.deploy.yarn.ApplicationMaster#main` 。

```
org.apache.spark.deploy.yarn.ApplicationMaster#main
 
 def main(args: Array[String]): Unit = {

    //Setup Conf, ignore.
    master = new ApplicationMaster(amArgs, sparkConf, yarnConf)

    //Get User Information, ignore.
    ugi.doAs(new PrivilegedExceptionAction[Unit]() {
      override def run(): Unit = System.exit(master.run())  //Pay Attention !!!! master.run() !!!
    })
  }
```

在 `ApplicationMaster#main` 方法里 call 了 `ApplicationMaster.run()`方法，`run` 方法里中，运行了 `runDriver()` 方法（ `run` 里其他的内容不是很重要，这里先忽略。）

`runDriver` 里，首先运行了一个方法 `userClassThread = startUserApplication()`。这个方法是在运行 user 写的代码，并且另外起了一个线程，点开 `startUserApplication()` 可以看到下面这些代码。
```
org.apache.spark.deploy.yarn.ApplicationMaster#startUserApplication

    val mainMethod = userClassLoader.loadClass(args.userClass)
      .getMethod("main", classOf[Array[String]])

    val userThread = new Thread {
      override def run(): Unit = {
        try {
          if (!Modifier.isStatic(mainMethod.getModifiers)) {
            //Error check. Ignore
          } else {
            mainMethod.invoke(null, userArgs.toArray)
            finish(FinalApplicationStatus.SUCCEEDED, ApplicationMaster.EXIT_SUCCESS)
            logDebug("Done running user class")
          }
        } catch {
            //Ignore
        } finally {
            //Ignore
        }
      }
    }
    userThread.setContextClassLoader(userClassLoader)
    userThread.setName("Driver")
    userThread.start()
    userThread
```
这个时候，回到 `runDriver` 这个方法里，可以看到 `val sc = ThreadUtils.awaitResult(sparkContextPromise.future, Duration(totalWaitTime, TimeUnit.MILLISECONDS))` 可以看到，原来的这个线程，在等待 sparkContextPromise 这个 Promise 里的值能够有一个结果。

那谁会给 sparkContextPromise 这个变量 赋 Success 呢？那其实就是 刚才启动的 user Thread, 现在 user Thread 的名称就是 Driver。对于所有的 spark app，第一句都是 `var spark = SparkSession.builder().getOrCreate()`。
那在 `org.apache.spark.sql.SparkSession#getOrCreate()` 里，会对 sparkContext 做设置，`SparkContext.getOrCreate(sparkConf)`

```
org.apache.spark.SparkContext#getOrCreate()

  def getOrCreate(config: SparkConf): SparkContext = {
    // Synchronize to ensure that multiple create requests don't trigger an exception
    // from assertNoOtherContextIsRunning within setActiveContext
    SPARK_CONTEXT_CONSTRUCTOR_LOCK.synchronized {
      if (activeContext.get() == null) {
        setActiveContext(new SparkContext(config))
      } else {
        if (config.getAll.nonEmpty) {
          logWarning("Using an existing SparkContext; some configuration may not take effect.")
        }
      }
      activeContext.get()
    }
  }
```

在 `new SparkConext` 中会做很多的事情，类似于设置 DAGScheduler / TaskScheduler 等，这里我们先不做讨论。在 `new SparkContext` 的过程中，会 call 一个方法。`_taskScheduler.postStartHook()`。

```
org.apache.spark.scheduler.cluster#YarnClusterScheduler

/**
 * This is a simple extension to YarnScheduler - to ensure that appropriate initialization of
 * ApplicationMaster, etc is done
 */
private[spark] class YarnClusterScheduler(sc: SparkContext) extends YarnScheduler(sc) {

  logInfo("Created YarnClusterScheduler")

  override def postStartHook(): Unit = {
    ApplicationMaster.sparkContextInitialized(sc)
    super.postStartHook()
    logInfo("YarnClusterScheduler.postStartHook done")
  }
}
```

ApplicationMaster.sparkContextInitialized(sc) 方法如下:
```
  private[spark] def sparkContextInitialized(sc: SparkContext): Unit = {
    master.sparkContextInitialized(sc)
  }

  
  private def sparkContextInitialized(sc: SparkContext) = {
    sparkContextPromise.synchronized {
      // Notify runDriver function that SparkContext is available
      sparkContextPromise.success(sc)
      // Pause the user class thread in order to make proper initialization in runDriver function.
      sparkContextPromise.wait()
    }
  }
```
可以看到，`ApplicationMaster.sparkContextInitialized` 是一个静态方法，所以可以直接使用 ApplicationMaster 这个类来调用，又由于是先创建的 `master` 然后才创建的 driver thread，所以 master 也不会为 null。这样调用都是安全的，(`master` 变量是 static 的 / 在 object 里)。

`master.sparkContextInitialized` 方法中，在调用了 `sparkContextPromise.success(sc)` 之后，原先的 application master thread 就可以继续运行了，而此时 driver thread 调用了 `sparkContextPromise.wait` 方法在这里继续等待。那又会是谁唤醒 driver 线程呢？

那我们先回到 ApplicationMaster 这个线程，`val sc = ThreadUtils.awaitResult(sparkContextPromise.future, Duration(totalWaitTime, TimeUnit.MILLISECONDS))`，那么此时 ApplicationMaster thread 拿到 spark context 之后，会先向 ResourceManager `registerAM`，并且开始申请 container。然后运行了下面两行

```
org.apache.spark.deploy.yarn.ApplicationMaster#runDriver

    resumeDriver()
    userClassThread.join()
```

在 resumeDriver 里会 `sparkContextPromise.notify()` 来唤醒 driver thread，然后再 `userClassThread.join()` 来等待 driver thread 运行结束。

以上就完成了 driver 和 ApplicationMaster 的启动。

## Executor Launch

这一部分讲一下 Executor Launch 这一部分。向 RM 请求 Container 的入口，我这里觉得是`org.apache.spark.deploy.yarn.YarnAllocator#allocateResources()` 因为这个 `allocateResource()` 有两个地方在调用，一个是刚刚新建完 allocator 之后，一个是在 `allocationThreadImpl` 里周期性调用。

```
org.apache.spark.deploy.yarn.YarnAllocator#allocateResources()

  /**
   * Request resources such that, if YARN gives us all we ask for, we'll have a number of containers
   * equal to maxExecutors.
   *
   * Deal with any containers YARN has granted to us by possibly launching executors in them.
   *
   * This must be synchronized because variables read in this method are mutated by other methods.
   */
  def allocateResources(): Unit = synchronized {
    updateResourceRequests()

    val progressIndicator = 0.1f
    // Poll the ResourceManager. This doubles as a heartbeat if there are no pending container
    // requests.
    val allocateResponse = amClient.allocate(progressIndicator)

    val allocatedContainers = allocateResponse.getAllocatedContainers()
    allocatorNodeHealthTracker.setNumClusterNodes(allocateResponse.getNumClusterNodes)

    if (allocatedContainers.size > 0) {
      logDebug(("Allocated containers: %d. Current executor count: %d. " +
        "Launching executor count: %d. Cluster resources: %s.")
        .format(
          allocatedContainers.size,
          getNumExecutorsRunning,
          getNumExecutorsStarting,
          allocateResponse.getAvailableResources))

      handleAllocatedContainers(allocatedContainers.asScala.toSeq)
    }

    val completedContainers = allocateResponse.getCompletedContainersStatuses()
    if (completedContainers.size > 0) {
      logDebug("Completed %d containers".format(completedContainers.size))
      processCompletedContainers(completedContainers.asScala.toSeq)
      logDebug("Finished processing %d completed containers. Current running executor count: %d."
        .format(completedContainers.size, getNumExecutorsRunning))
    }
  }
```

上面这一段函数里，`updateResourceRequest()` 这一步会更新 container 的request，并将这些信息更新到 `amClient`, 所以到了 `amClient.allocate(progressIndicator)` 这一步就不需要传递和 resource request 相关的object了 (amClient是YarnAllocator 构造函数的一部分，可以认为是全局变量)，

在获得 allocatedContainers 之后，调用 `handleAllocatedContainers()` 来准备部署container。`runAllocatedContainers(containersToUse: ArrayBuffer[Container])` 这个 function 里会在 launcherPool 里运行 `new ExecutorRunnable()` 方法。在 `ExecutorRunnable().run()` 方法里，会新建一个 `nmClient`, 然后会调用 `startContainer()` 方法。在这个方法里，调用 `prepareCommand()` 方法里，做了 Executor 的 class `org.apache.spark.executor.YarnCoarseGrainedExecutorBackend` 的定义，以及运行方式的定义。最后在 `nmClient.startContainer()` 方法，来运行一个 container。

在一个新的 Executor 里，会先运行 `org.apache.spark.executor.YarnCoarseGrainedExecutorBackend` 的 main 方法。

在 main 方法里会启动 CoarseGrainedExecutorBackend，然后再 `onStart()` 方法中，向 driver 发送注册 RegisterExecutor 事件。


## Driver & Application 里的其他 daemon thread

### Allocation 相关的 thread
#### reporterThread
```
org.apache.spark.deploy.yarn.ApplicationMaster#allocationThreadImpl

  private def allocationThreadImpl(): Unit = {
    // The number of failures in a row until the allocation thread gives up.
    val reporterMaxFailures = sparkConf.get(MAX_REPORTER_THREAD_FAILURES)
    var failureCount = 0
    while (!finished) {
      try {
        if (allocator.getNumExecutorsFailed >= maxNumExecutorFailures) {
          //Ignore
        } else if (allocator.isAllNodeExcluded) {
          //Ignore
        } else {
          //allocate container.
          logDebug("Sending progress")
          allocator.allocateResources()
        }
        failureCount = 0
      } catch {
        //Ignore
      }
      try {
        val numPendingAllocate = allocator.getNumContainersPendingAllocate
        //Ignore
        allocatorLock.synchronized {
          sleepInterval = //Get SleepInterval
          sleepStartNs = System.nanoTime()
          allocatorLock.wait(sleepInterval)
        }
        //Ignore
      } catch {
        //Ignore
      }
    }
  }

```
上面这个方法是 reportThread 里的主要的方法，可以看到这个方法是一个类似 while(true) 的循环，其中 `allocator.allocateResources()` 这个方法用于申请 containers。之后这个线程会 wait(sleepInterval)，之后再进行这么一个循环。

`allocator.allocateResources()` 这个方法，即是上面 `Executor Launch` 里讲解的方法，具体的 Executor launch 的步骤就不重复说明了。在这个方法里，有一步 `updateResourceRequests()`，在这个方法里会从一个全局变量`targetNumExecutorsPerResourceProfileId` 里来生成，这个线程最终想要获得的 container 的数量。

#### AMEndpoint
这个 RPCEndpoint 会处理和 ResourceManager 打交道的所有事件。
```
org.apache.spark.deploy.yarn.ApplicationMaster#AMEndpoint

    //向 driver 注册自己，即让Driver知道 AMEndpoint 的地址。
    override def onStart(): Unit = {
      driver.send(RegisterClusterManager(self))
    }

    override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {
      //  处理申请 Executor request 的请求。
      case r: RequestExecutors =>
        Option(allocator) match {
          case Some(a) =>
          //Ignoe
        
      //  处理 Executor 被kill 的请求。
      case KillExecutors(executorIds) =>
          //Ignoe

      //  处理 Executor Loss Reason 的请求。
      case GetExecutorLossReason(eid) =>
          //Ignore
        }
    }
```

#### CoarseGrainedSchedulerBackend 里的 thread
**reviveThread**

reviveThread，这个 thread 的名字是 driver-revive-thread，这个 thread 会定期向 driverEndpoint 发送一个 ReviveOffers 事件。
```
org.apache.spark.scheduler.cluster.CoarseGrainedSchedulerBackend#DriverEndpoint

      reviveThread.scheduleAtFixedRate(() => Utils.tryLogNonFatalError {
        Option(self).foreach(_.send(ReviveOffers))
      }, 0, reviveIntervalMs, TimeUnit.MILLISECONDS)
```

**SpeculationScheduler Thread**
speculationScheduler Thread，这个 thread 的名字是 task-scheduler-speculation。

```
org.apache.spark.scheduler.TaskSchedulerImpl

  speculationScheduler.scheduleWithFixedDelay(
        () => Utils.tryOrStopSparkContext(sc) { checkSpeculatableTasks() },
        SPECULATION_INTERVAL_MS, SPECULATION_INTERVAL_MS, TimeUnit.MILLISECONDS)
```
这个 thread 会定期调用 `checkSpeculatableTasks()`这个方法，最终会调用到 `org.apache.spark.scheduler.TaskSetManager#checkSpeculatableTasks` 这个方法。

这个 thread 的目的主要是，当 75% 的 task 都已经完成了之后，会去检查一下剩下的 task，如果有的 task 远超运行结束的平均时间，会重新跑这个 task。另外，如果启动了 spark.decommission.enable 并且设置了 spark.executor.decommission.killInterval，会判断一下，在 spark executor decommission 结束之前，这个 task 是否能够结束，不能结束的话，就在另外一个 executor 上重跑这个 task。（判断 task 什么时间结束，使用的是已经结束的 task 运行时间的中位数）

**DriverEndpoint**
这个是 driverEndpoint rpc 事件的接收 thread，这里列一下现在要接受的主要事件。

StatusUpdate(更新TaskID的status) / killTask / KillExecutorsOnHost / RemoveExecutor / LaunchedExecutor / RegisterExecutor / StopExecutors / ExecutorDecommissioning

**YarnSchedularEndpoint**

YarnSchedulerEndpoint 用于转发 CoarseGrainedSchedulerBackend 的一些请求到 amEndpoint.

RequestExecutors / KillExecutors / RegisterClusterManager(向YarnSchedulerEndpoint注册 amEndpoint) / RemoveExecutor 


### executorAllocationManager

```
org.apache.spark.ExecutorAllocationManager#schedule

  /**
   * This is called at a fixed interval to regulate the number of pending executor requests
   * and number of executors running.
   *
   * First, adjust our requested executors based on the add time and our current needs.
   * Then, if the remove time for an existing executor has expired, kill the executor.
   *
   * This is factored out into its own method for testing.
   */
  private def schedule(): Unit = synchronized {
    val executorIdsToBeRemoved = executorMonitor.timedOutExecutors()
    if (executorIdsToBeRemoved.nonEmpty) {
      initializing = false
    }

    // Update executor target number only after initializing flag is unset
    updateAndSyncNumExecutorsTarget(clock.nanoTime())
    if (executorIdsToBeRemoved.nonEmpty) {
      removeExecutors(executorIdsToBeRemoved)
    }
  }
```
executorAllocatorManager 这个 thread 会定期运行上面这个 `scheduler()` 方法，其中主要是 `updateAndSyncNumExecutorsTarget()` 这个方法。 在这个方法里，会运行 `val maxNeeded = maxNumExecutorsNeededPerResourceProfile(rpId)` 这个方法，这个方法主要是在计算，基于当前的 running task 和 pending task 来计算需要多少个 executor，然后根据这个 maxNeeded 和 当前已经拥有的executor 数量做对比，如果还需要更多的 executor，则会去申请。

```
org.apache.spark.ExecutorAllocationManager#maxNumExecutorsNeededPerResourceProfile

  private def maxNumExecutorsNeededPerResourceProfile(rpId: Int): Int = {
    val pending = listener.totalPendingTasksPerResourceProfile(rpId)
    val pendingSpeculative = listener.pendingSpeculativeTasksPerResourceProfile(rpId)
    val unschedulableTaskSets = listener.pendingUnschedulableTaskSetsPerResourceProfile(rpId)
    val running = listener.totalRunningTasksPerResourceProfile(rpId)
    val numRunningOrPendingTasks = pending + running
    val rp = resourceProfileManager.resourceProfileFromId(rpId)
    val tasksPerExecutor = rp.maxTasksPerExecutor(conf)

    val maxNeeded = math.ceil(numRunningOrPendingTasks * executorAllocationRatio /
      tasksPerExecutor).toInt
      //Ignore
  }
```


之后，会调用 `doRequestTotalExecutors` 这个方法，yarnSchedulerEndpoint 会接收一个 `RequestExecutors` 事件，然后转发给 `amEndpoint`， `amEndpoint` 会继续向 RM 申请 container。

```
org.apache.spark.scheduler.cluster.YarnSchedulerBackend#doRequestTotalExecutors

  /**
   * Request executors from the ApplicationMaster by specifying the total number desired.
   * This includes executors already pending or running.
   */
  override def doRequestTotalExecutors(
      resourceProfileToTotalExecs: Map[ResourceProfile, Int]): Future[Boolean] = {
    yarnSchedulerEndpointRef.ask[Boolean](prepareRequestExecutors(resourceProfileToTotalExecs))
  }
```


**ExecutorAllocationListener**

org.apache.spark。ExecutorAllocationManager # ExecutorAllocationListener

executorAllocationManager 在启动的时候，在 listenerBus 里加入了 ExecutorAllocationListener ，这样的话，当有一些 spark job 相关的 event 被放到了 listenerBus 上之后，ExecutorAllocationListener 就会收到对应的Event，对于不同的 event 可以做不同的动作。

- onStageSubmitted： 收到stage submit event，记录stage 相关的信息，以及总的 task 数量。
- onTaskStart: running task 的数量 + 1
- onTaskEnd: running task 的数量 -1

在上述的 `maxNumExecutorsNeededPerResourceProfile()` 的方法中，获取 pending task 的数量 和 running task 的数量的来源就是通过上述 executorAllocationListener 在 ExecutorAllocationManager 里维护一些 share 的变量来维护的。


最后简单总结一下，spark在运行过程中，executor 的申请的过程

![launchExecutor](/assets/img/spark/launchExecutor.png)

1. executorAllocationListener 收到 sparkListener 发来的一些事件，例如 stageSubmitted / taskStart / taskEnd 等。
2. executorAllocationListener 和 executorAllocationManager 共享一些变量，listener 会维护 task 状态和数量。
3. executorAllocationManager 会根据 task 的数量和状态，来计算所需要的总的 executor 的数量，并发送给 YarnSchedulerEndpoint。YarnSchedulerEndpoint 会转发这个事件给 AMEndpoint
4. AMEndpoint 会和 allocation thread 共享一些变量，AMEndpoint 会维护 spark job 所需的 executor 的个数。
5. allocation thread 会根据 所需的 executor 一直向 Resource Manager 申请 container。allocaiton thread 申请到这些 container 之后，会 launch Executor。
6. Executor 启动之后，会向 driverEndpoint 发送 registerExecutor 事件
7. DriverEndpoint 会将 executor 加入到 ExecutorSet，并标注这个 executor 的 memory 和 core 信息。
8. Speculation Scheduler Thread 会在 75% 的 task 运行结束之后，去判断这些 task 是否过慢，需要复制一份让其他的 executor 也同时运行。如果有这样的 task，将会被加入到 task set 里（有可能会 trigger listenBus，产生新的 taskStart 事件。
9. reviveThread 会定期 匹配 taskSet 和 executorSet 里的 task 和 executor，并发送给对应的 executor 去运行。

（图中的 cycle 表示这个 thread 会定期运行， Endpoint 都是 RpcEndpoint，接收 Rpc 调用）
