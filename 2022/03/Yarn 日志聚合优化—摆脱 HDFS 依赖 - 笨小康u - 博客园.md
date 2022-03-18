# Yarn 日志聚合优化—摆脱 HDFS 依赖 - 笨小康u - 博客园
**（1）问题背景**

线上集群 Container 日志上报的事务集群 namenode rpc 持续飙高，影响到了 Yarn 分配 Container 的性能，任务提交数下降，导致整个集群的吞吐量下降。

**（2）原因简介**

作业提交到 Yarn 集群时，每个 NM 节点都会对每个 app 作业进行日志聚合操作，该操作包括初始化日志聚合服务、检测和创建日志聚合的 HDFS 目录、创建日志聚合线程执行本地日志的上传，其中

-   初始化日志聚合服务就是简单的对象创建，不和 HDFS 交互，基本无压力；
-   检测和创建日志聚合的 HDFS 目录会执行 HDFS 读和写请求，并且是同步阻塞执行，依赖写 `/tmp/logs/` 目录所在集群的 HDFS 服务。
-   创建日志聚合线程上传本地日志，代码中该线程是通过线程池异步创建，不存在阻塞，但固定大小的线程池可能会出现线程创建阻塞。

**（3）解决方案**

根据以上分析，针对日志聚合依赖读写 HDFS 数据反向影响作业的提交问题，主要有两种解决方案：

-   作业提交不依赖日志聚合对 HDFS 服务的读 / 写。（**本文主要解决这一问题**）
-   日志聚合写 HDFS 进行分流，写到多个 HDFS 集群。

日志聚集是 Yarn 提供的日志中央化管理功能，它能将运行完成的 Container 任务日志上传到 HDFS 上，从而减轻 NodeManager 负载，且提供一个中央化存储和分析机制。默认情况下，Container 任务日志存在在各个 NodeManager 的本地磁盘上，保存在 `yarn.nodemanager.log-dirs` 参数配置的目录下，保存的时间由 `yarn.nodemanager.log.retain-seconds` 参数决定（默认时 3 小时）。若启用日志聚集功能，会将作业完成的日志上传到 HDFS 的 `${yarn.nodemanager.remote-app-log-dir}/${user}/${yarn.nodemanager.remote-app-log-dir-suffix}` 下，要实现日志聚合功能，需要额外的配置。

这里的日志存储的就是具体 Mapreduce 和 Spark 任务的日志，包括框架的和应用程序里自己打印的。这日志聚合是用来看日志的，而 job history server 则是用来看某个 application 的大致统计信息的，包括作业启停时间，map 任务数，reduce 任务数以及各种计数器的值等等。job history server 是抽象概要性的统计信息，而聚合日志是该 application 所有任务节点的详细日志集合。

![](https://img2020.cnblogs.com/blog/1372882/202103/1372882-20210310174414290-1407590078.jpg)

Yarn 作业在运行过程中，聚合日志的生命周期如下：

1.  作业运行过程中，日志将暂存于 `yarn.nodemanager.log-dirs` 配置项指定的本地路径下，默认为 `/var/log/hadoop-yarn/container`。
2.  作业运行结束后（无论正常结束与否），将持久化日志到 `yarn.nodemanager.remote-app-log-dir` 和 `yarn.nodemanager.remote-app-log-dir-suffix` 配置项指定的 HDFS 路径下，前者默认为 `/tmp/logs`，后者默认为 `logs`。对应 HDFS 的实际路径为 `${yarn.nodemanager.remote-app-log-dir}/${user}/${yarn.nodemanager.remote-app-log-dir-suffix}/${application_id}`，即 `/tmp/logs/<user>/logs/`。控制日志聚合操作的服务为 LogAggregationService，具体上传日志到 HDFS 的行为由 LogAggregationService 服务创建的 AppLogAggregator 线程执行。
3.  日志持久化聚合到 HDFS 后，会删除本地的暂存日志。
4.  聚合上传到 HDFS 的日志也是有保留周期的，保存周期由 `yarn.log-aggregation.retain-seconds` 参数控制，集群可配置。

```null
参数：yarn.nodemanager.log-dirs
参数解释：日志存放地址（可配置多个目录）。
默认值：${yarn.log.dir}/userlogs

参数：yarn.log-aggregation-enable
参数解释：是否启用日志聚集功能。
默认值：false

参数：yarn.log-aggregation.retain-seconds
参数解释：在HDFS上聚集的日志最多保存多长时间。
默认值：-1

参数：yarn.log-aggregation.retain-check-interval-seconds
参数解释：多长时间检查一次日志，并将满足条件的删除，如果是0或者负数，则为上一个值的1/10。
默认值：-1

参数：yarn.nodemanager.remote-app-log-dir
参数解释：当应用程序运行结束后，日志被转移到的HDFS目录（启用日志聚集功能时有效）。
默认值：/tmp/logs

参数：yarn.nodemanager.remote-app-log-dir-suffix
参数解释：远程日志目录子目录名称（启用日志聚集功能时有效）。
默认值：logs 日志将被转移到目录${yarn.nodemanager.remote-app-log-dir}/${user}/${thisParam}下

参数：yarn.nodemanager.log.retain-seconds
参数解释：NodeManager上日志最多存放时间（不启用日志聚集功能时有效）。
默认值：10800（3小时）
```

[YARN 在字节跳动的优化与实践](https://mp.weixin.qq.com/s/0ffysIzzJIFLcyg2bXfwSQ)

> 将 HDFS 做成弱依赖  
> 对于一般的离线批处理来说，如果 HDFS 服务不可用了，那么 YARN 也没必要继续运行了。但是在字节跳动内部由于 YARN 还同时承载流式作业和模型训练，因此不能容忍 HDFS 故障影响到 YARN。为此，我们通过将 NodeLabel 存储到 ZK 中，将 Container Log 在 HDFS 的目录初始化和上传都改为异步的方式，摆脱了对 HDFS 的强依赖。

[YARN 在快手的应用实践与技术演进之路](https://mp.weixin.qq.com/s/z5HzYSqc2zHmd-DDBVcG4w)

> HDFS 是 yarn 非常底层的基础设施，ResourceManager 事件处理逻辑中有一些 HDFS 操作，HDFS 卡一下，会造成整个事件处理逻辑卡住，最终整个集群卡住。分析发现 RM 对 HDFS 的操作主要集中在失败 APP 的处理，不是非常核心的逻辑，解决方案也比较简单粗暴，把 HDFS 的操作从同步改成异步。我们还对整个 yarn 事件处理逻辑进行排查，发现有一些像 DNS 的操作，在某些情况下也会比较卡，我们就把这种比较重 IO 的操作进行相应的优化，确保事件处理逻辑中都是快速的 CPU 操作，保证事件处理的高效和稳定。

[基于 Hadoop 的 58 同城离线计算平台设计与实践](https://www.infoq.cn/article/pmlsDrWfcjZBxj2P97S1)

> 虽然有 Fedoration 机制来均衡各个 NN 的压力，但是对于单个 NN 压力仍然非常大，各种问题时刻在挑战 HDFS 稳定性，比如：NN RPC 爆炸，我们线上最大的 NS 有 15 亿的 RPC 调用，4000+ 并发连接请求，如此高的连接请求对业务稳定影响很大。针对这个问题，我们使用 "拆解 + 优化" 的两种手段相结合的方式来改进。拆解就是说我们把一些大的访问，能不能拆解到不同的集群上，或者我们能不能做些控制，具体案例如下：  
> 1.Hive Scratch：我们经过分析 Hive Scratch 的临时目录在 RPC 调用占比中达到 20%，对于 Hive Scratch 实际上每个业务不需要集中到一个 NS 上，我们把它均衡到多个 NS 上。  
> 2.Yarn 日志聚合：Yarn 的日志聚合主要是给业务查看一些日志，实际上他没有必要那个聚合到 HDFS 上，只需要访问本地就可以了。ResourceLocalize：同样把它均衡到各个 NS 上。

[HDFS Federation 在美团点评的应用与改进](https://tech.meituan.com/2017/04/14/hdfs-federation.html)

> 计算引擎（包括 MapReduce 和 Spark）在提交作业时，会向 NameNode 发送 RPC，获取 HDFS Token。在 ViewFileSystem 中，会向所有 namespace 串行的申请 Token，如果某个 namespace 的 NameNode 负载很高，或者发生故障，则任务无法提交，YARN 的 ResourceManager 在 renew Token 时，也会受此影响。随着美团点评的发展 YARN 作业并发量也在逐渐提高，保存在 HDFS 上的 YARN log 由于 QPS 过高，被拆分为独立的 namespace。但由于其并发和 YARN container 并发相同，NameNode 读写压力还是非常大，经常导致其 RPC 队列打满，请求超时，进而影响了作业的提交。针对此问题，我们做出了一下改进：  
> 1.container 日志由 NodeManager 通过 impersonate 写入 HDFS，这样客户端在提交 Job 时，就不需要 YARN log 所在 namespace 的 Token。  
> 2.ViewFileSystem 在获取 Token 时，增加了参数，用于指定不获取哪些 namespace 的 Token。  
> 3. 由于作业并不总是需要所有 namespace 中的数据，因此当单个 namespace 故障时，不应当影响其他 namespace 数据的读写，否则会降低整个集群的分区容忍性和可用性，ViewFileSystem 在获取 Token 时，即使失败，也不影响作业提交，而是在真正访问数据时作业失败，这样在不需要的 Token 获取失败时，不影响作业的运行。  
> 另外，客户端获取到的 Token 会以 namespace 为 key，保存在一个自定义数据结构中（Credentials）。ResourceManager renew 时，遍历这个数据结构。而 NodeManager 在拉取 JAR 包时，根据本地配置中的 namespace 名去该数据结构中获取对应 Token。因此需要注意的是，虽然 namespace 配置和服务端不同不影响普通 HDFS 读写，但提交作业所使用的 namespace 配置需要与 NodeManager 相同，至少会用到的 namespace 配置需要是一致的。

本文主要针对字节跳动的思路对日志聚合逻辑进行优化，将日志聚合读写 HDFS 集群改为弱依赖。

要弄清楚聚合日志如何工作的，就需要了解 Yarn 中处理聚合日志的服务在哪里创建的，根据 [ApplicationMaster 启动及资源申请源码分析](https://www.cnblogs.com/lemonu/p/13566381.html) 文章分析，我们知道 Yarn 的第一个 Container 启动是用于 AppAttmpt 角色，也就是我们通常在 Yarn UI 界面看到的 ApplicationMaster 服务。所以我们来看看一个作业的第一个 Container 是如何启动以及如何创建日志记录组件 LogHandler 的。

ApplicationMaster 通过调用 RPC 函数 ContainerManagementProtocol#startContainers() 开始启动 Container，即 startContainerInternal() 方法，这部分逻辑做了两件事：

-   发送 ApplicationEventType.INIT_APPLICATION 事件，对应用程序资源的初始化，主要是初始化各类必需的服务组件（如日志记录组件 LogHandler、资源状态追踪组件 LocalResourcesTrackerImpl 等），供后续 Container 启动，通常来自 ApplicationMaster 的第一个 Container 完成，这里的 if 逻辑针对一个 NM 节点上运行作业的所有 Containers 只调用一次，后续的 Container 跳过这段 Application 初始化过程。
-   发送 ApplicationEventType.INIT_CONTAINER 事件，对 Container 进行初始化操作。（这部分事件留在 Container 启动环节介绍）

```null

  private void startContainerInternal(NMTokenIdentifier nmTokenIdentifier,
      ContainerTokenIdentifier containerTokenIdentifier,
      StartContainerRequest request) throws YarnException, IOException {
 
    
 
    this.readLock.lock();
    try {
      if (!serviceStopped) {
        
        Application application =
            new ApplicationImpl(dispatcher, user, applicationID, credentials, context);
         
        
        if (null == context.getApplications().putIfAbsent(applicationID,
          application)) {
          LOG.info("Creating a new application reference for app " + applicationID);
          LogAggregationContext logAggregationContext =
              containerTokenIdentifier.getLogAggregationContext();
          Map<ApplicationAccessType, String> appAcls =
              container.getLaunchContext().getApplicationACLs();
          context.getNMStateStore().storeApplication(applicationID,
              buildAppProto(applicationID, user, credentials, appAcls,
                logAggregationContext));
 
 
          
          dispatcher.getEventHandler().handle(
            new ApplicationInitEvent(applicationID, appAcls,
              logAggregationContext));
        }
 
        
        this.context.getNMStateStore().storeContainer(containerId, request);
        dispatcher.getEventHandler().handle(
          new ApplicationContainerInitEvent(container));
 
        this.context.getContainerTokenSecretManager().startContainerSuccessful(
          containerTokenIdentifier);
        NMAuditLogger.logSuccess(user, AuditConstants.START_CONTAINER,
          "ContainerManageImpl", applicationID, containerId);
        
        
        metrics.launchedContainer();
        metrics.allocateContainer(containerTokenIdentifier.getResource());
      } else {
        throw new YarnException(
            "Container start failed as the NodeManager is " +
            "in the process of shutting down");
      }
    } finally {
      this.readLock.unlock();
    }
  }
```

这里主要看看第 1 件事情，即向 ApplicationImpl 发送 ApplicationEventType.INIT_APPLICATION 事件，事件对应的状态机为 AppInitTransition 状态机。

```null


           .addTransition(ApplicationState.NEW, ApplicationState.INITING,
               ApplicationEventType.INIT_APPLICATION, new AppInitTransition())
```

AppInitTransition 状态机会对日志聚合组件服务进行初始化，关键行动是向调度器发送 LogHandlerEventType.APPLICATION_STARTED 事件。

```null

  
  @SuppressWarnings("unchecked")
  static class AppInitTransition implements
      SingleArcTransition<ApplicationImpl, ApplicationEvent> {
    @Override
    public void transition(ApplicationImpl app, ApplicationEvent event) {
      ApplicationInitEvent initEvent = (ApplicationInitEvent)event;
      app.applicationACLs = initEvent.getApplicationACLs();
      app.aclsManager.addApplication(app.getAppId(), app.applicationACLs);

      
      
      app.logAggregationContext = initEvent.getLogAggregationContext();
      
      app.dispatcher.getEventHandler().handle(
          new LogHandlerAppStartedEvent(app.appId, app.user,
              app.credentials, ContainerLogsRetentionPolicy.ALL_CONTAINERS,
              app.applicationACLs, app.logAggregationContext)); 
    }
  }
```

想要弄清楚 LogHandlerEventType.APPLICATION_STARTED 事件做了什么，就要知道 LogHandlerEventType 类注册的事件处理器是什么以及事件处理器做了什么事情。这里的 register 方法对 LogHandlerEventType 类进行了注册，对应的 logHandler 事件处理器为 LogAggregationService 服务。

```null

  @Override
  public void serviceInit(Configuration conf) throws Exception {
    
    LogHandler logHandler =
      createLogHandler(conf, this.context, this.deletionService);
    addIfService(logHandler);
    
    dispatcher.register(LogHandlerEventType.class, logHandler);
    
    waitForContainersOnShutdownMillis =
        conf.getLong(YarnConfiguration.NM_SLEEP_DELAY_BEFORE_SIGKILL_MS,
            YarnConfiguration.DEFAULT_NM_SLEEP_DELAY_BEFORE_SIGKILL_MS) +
        conf.getLong(YarnConfiguration.NM_PROCESS_KILL_WAIT_MS,
            YarnConfiguration.DEFAULT_NM_PROCESS_KILL_WAIT_MS) +
        SHUTDOWN_CLEANUP_SLOP_MS;

    super.serviceInit(conf);
    recover();
  }
```

具体创建 logHandler 对象的调用，由于集群开启了日志聚合功能（由参数 `yarn.log-aggregation-enable` 控制），这里返回 LogAggregationService 服务。

```null

    protected LogHandler createLogHandler(Configuration conf, Context context,
      DeletionService deletionService) {
    if (conf.getBoolean(YarnConfiguration.LOG_AGGREGATION_ENABLED,
        YarnConfiguration.DEFAULT_LOG_AGGREGATION_ENABLED)) {
      
      return new LogAggregationService(this.dispatcher, context,
          deletionService, dirsHandler);
    } else {
      return new NonAggregatingLogHandler(this.dispatcher, deletionService,
                                          dirsHandler,
                                          context.getNMStateStore());
    }
  }
```

弄清楚了 LogHandlerEventType 类注册的服务是 LogAggregationService，我们就进入 LogAggregationService 类的 handle() 方法，看看上面的 LogHandlerEventType.APPLICATION_STARTED 事件做了什么事。

```null

  @Override
  public void handle(LogHandlerEvent event) {
    switch (event.getType()) {
      
      case APPLICATION_STARTED:
        LogHandlerAppStartedEvent appStartEvent =
            (LogHandlerAppStartedEvent) event;
        initApp(appStartEvent.getApplicationId(), appStartEvent.getUser(),
            appStartEvent.getCredentials(),
            appStartEvent.getLogRetentionPolicy(),
            appStartEvent.getApplicationAcls(),
            appStartEvent.getLogAggregationContext());
        break;
      case CONTAINER_FINISHED:
        
      case APPLICATION_FINISHED:
        
      default:
        ; 
    }
  }
```

LogHandlerEventType.APPLICATION_STARTED 事件的关键逻辑在 initApp() 方法的调用。这段逻辑主要做了三件事：

1.  判断 HDFS 上日志聚合的根目录是否存在，即 `/tmp/logs/` 目录（具体为 `hdfs://nameservice/tmp/logs`)，由参数 `yarn.nodemanager.remote-app-log-dir` 控制。（注意：这里的请求会阻塞读 HDFS）
2.  创建作业日志聚合的 HDFS 目录，并初始化 app 日志聚合实例，采用线程池的方式启动日志聚合进程。（重点，这里会有请求阻塞写 HDFS，并且通过有限大小的线程池异步创建日志聚合线程去做日志的聚合）
3.  根据构建的 ApplicationEvent 事件，向发送 ApplicationEventType.APPLICATION_LOG_HANDLING_INITED 事件，告知处理器日志聚合服务初始化完成。

```null

  private void initApp(final ApplicationId appId, String user,
      Credentials credentials, ContainerLogsRetentionPolicy logRetentionPolicy,
      Map<ApplicationAccessType, String> appAcls,
      LogAggregationContext logAggregationContext) {
    ApplicationEvent eventResponse;
    try {
      
      verifyAndCreateRemoteLogDir(getConfig());
      
      initAppAggregator(appId, user, credentials, logRetentionPolicy, appAcls,
          logAggregationContext);
      
      eventResponse = new ApplicationEvent(appId,
          ApplicationEventType.APPLICATION_LOG_HANDLING_INITED);
    } catch (YarnRuntimeException e) {
      LOG.warn("Application failed to init aggregation", e);
      eventResponse = new ApplicationEvent(appId,
          ApplicationEventType.APPLICATION_LOG_HANDLING_FAILED);
    }
    
    this.dispatcher.getEventHandler().handle(eventResponse);
  }
```

第 1 件事比较简单，主要是是判断 HDFS 聚合日志的根目录是否存在，由于目录一般都存在，这一块主要是读 HDFS 请求。我们主要来看看 initApp() 方法做的第 2 件事，可以看到第 3 件事是发送 ApplicationEventType.APPLICATION_LOG_HANDLING_INITED 表示日志聚合服务初始化完成，包括创建作业在 HDFS 的日志聚合目录和启动日志聚合线程。所以基本可以知道第 2 件事的 initAppAggregator() 是会创建作业日志聚合目录，并启动日志聚合线程，具体的我们来看代码。

这段代码其实主要做了两件事：

1.  调用 createAppDir() 方法执行 HDFS 写请求为作业创建日志聚合的目录，即 `hdfs://nameservice/tmp/logs/<user>/logs/` 目录，这里的写逻辑如果成功则只调用一次，一般是由第一个 Container 创建（即作业的 ApplicationMaster Container），其他 Container 只执行 HDFS 读请求判断该目录是否存在即可。
2.  通过 threadPool 线程池创建每个作业在 NM 节点的日志聚合线程，异步处理本地日志的上传，该线程池大小由参数 `yarn.nodemanager.logaggregation.threadpool-size-max` 控制，默认大小为 100.

```null

  protected void initAppAggregator(final ApplicationId appId, String user,
      Credentials credentials, ContainerLogsRetentionPolicy logRetentionPolicy,
      Map<ApplicationAccessType, String> appAcls,
      LogAggregationContext logAggregationContext) {

    
    final UserGroupInformation userUgi =
        UserGroupInformation.createRemoteUser(user);
    if (credentials != null) {
      userUgi.addCredentials(credentials);
    }

    
    final AppLogAggregator appLogAggregator =
        new AppLogAggregatorImpl(this.dispatcher, this.deletionService,
            getConfig(), appId, userUgi, this.nodeId, dirsHandler,
            getRemoteNodeLogFileForApp(appId, user), logRetentionPolicy,
            appAcls, logAggregationContext, this.context,
            getLocalFileContext(getConfig()));
    if (this.appLogAggregators.putIfAbsent(appId, appLogAggregator) != null) {
      throw new YarnRuntimeException("Duplicate initApp for " + appId);
    }
    
    YarnRuntimeException appDirException = null;
    try {
      
      
      createAppDir(user, appId, userUgi);
    } catch (Exception e) {
      appLogAggregator.disableLogAggregation();
      if (!(e instanceof YarnRuntimeException)) {
        appDirException = new YarnRuntimeException(e);
      } else {
        appDirException = (YarnRuntimeException)e;
      }
      appLogAggregators.remove(appId);
      closeFileSystems(userUgi);
      throw appDirException;
    }

    
    
    Runnable aggregatorWrapper = new Runnable() {
      public void run() {
        try {
          appLogAggregator.run();
        } finally {
          appLogAggregators.remove(appId);
          closeFileSystems(userUgi);
        }
      }
    };
    this.threadPool.execute(aggregatorWrapper);
  }
```

至此，从日志聚合服务组件的创建，到为作业初始化 HDFS 聚合日志目录，到启动日志聚合线程，整个日志聚合的调用逻辑已介绍完毕，日志的具体上传逻辑在 AppLogAggregatorImpl 类的 run() 方法开始执行，具体上传这里不做详细介绍，感兴趣可以可以去看看上传行为是如何做的。

在背景介绍中，提到了日志聚合操作存在风险的点主要在读 / 写 HDFS 请求所在的集群 namenode rpc 压力，和固定大小的线程池创建线程的阻塞，代码的修改逻辑也是结合这两个问题诞生的。

-   针对读 / 写 HDFS 请求的 rpc 压力，代码将日志聚合逻辑中与 HDFS 交互的方式全部改为异步处理，不依赖日志聚合读写数据的 HDFS 集群。
-   针对固定大小线程池创建线程可能出现的阻塞情况，代码将这一块修改为生产者 - 消费者模式，聚合日志线程的产生与线程的处理解耦。

## 6.1 读 / 写 HDFS 请求异步

日志聚合服务中与 HDFS 交互有两个地方，一个是读操作，判断 HDFS 上 `/tmp/logs/` 目录是否存在，一个是写操作，创建作业的聚合日志目录 `/tmp/logs/<user>/logs/<appid>/`，这写操作每个作业只执行一次，后续都是读操作，判断该目录是否存在即可。

```null

  private void asyncCreateAppDir(final String user, final ApplicationId appId, final UserGroupInformation userUgi) {
      new Thread() {
        @Override
        public void run() {
          synchronized(this) {
            try {
              
              verifyAndCreateRemoteLogDir(getConfig());
              
              createAppDir(user, appId, userUgi);
            } catch (Exception e) {
              e.printStackTrace();
            }
          }
        }
      }.start();
  }
```

将日志聚合读写 HDFS 请求改为异步后，可能会产生另外一个问题。由于作业日志聚合目录的创建是异步的，而执行日志上传操作也是异步进行的，这里存在着先后顺序，即必须作业的日志聚合目录已经创建完成，上传操作才能正常进行。因此，在具体执行上传操作时，我们对日志聚合目录是否存在添加一层校验，以确保上传前聚合目录必须存在。

```null

  private void doAppLogAggregation() {
    
    while (!this.appFinishing.get() && !this.aborted.get()) {
      synchronized(this) {
        try {
          waiting.set(true);
          if (this.rollingMonitorInterval > 0) {
            wait(this.rollingMonitorInterval * 1000);
            if (this.appFinishing.get() || this.aborted.get()) {
              break;
            }
            uploadLogsForContainers(false);
          } else {
            wait(THREAD_SLEEP_TIME);
          }
        } catch (InterruptedException e) {
          LOG.warn("PendingContainers queue is interrupted");
          this.appFinishing.set(true);
        }
      }
    }

    if (this.aborted.get()) {
      return;
    }

    
    
    checkRemoteDir();

    
    
    uploadLogsForContainers(true);


    
    doAppLogAggregationPostCleanUp();

    this.dispatcher.getEventHandler().handle(
        new ApplicationEvent(this.appId,
            ApplicationEventType.APPLICATION_LOG_HANDLING_FINISHED));
    this.appAggregationFinished.set(true);
  }
```

```null

  
  private  void checkRemoteDir() {
  try {
    userUgi.doAs(new PrivilegedExceptionAction<Object>() {
      @Override
      public Object run() throws Exception {
        FileSystem remoteFS = remoteNodeLogFileForApp.getFileSystem(conf);
        if (!remoteFS.exists(remoteNodeLogFileForApp.getParent())) {
          try {
            FsPermission dirPerm = new FsPermission(APP_DIR_PERMISSIONS);
            remoteFS.mkdirs(remoteNodeLogFileForApp.getParent(), dirPerm);
            FsPermission umask = FsPermission.getUMask(remoteFS.getConf());
            if (!dirPerm.equals(dirPerm.applyUMask(umask))) {
              remoteFS.setPermission(remoteNodeLogFileForApp.getParent(), dirPerm);
            }
          } catch (IOException e) {
            LOG.error("Failed to setup application log directory for " + appId, e);
            throw e;
          }
        }
        return null;
      }});
    } catch (Exception e) {
      throw new YarnRuntimeException(e);
    }
  }
```

## 6.2 聚合日志线程的创建和处理解耦

这一块主要是通过生产者 - 消费者模式，将日志聚合线程的创建和处理解耦，生产的线程由阻塞队列 logAggregatorQueue 维护，具体的线程消费逻辑由独立线程 LauncherLogAggregatorThread 处理，具体代码如下。

```null

  protected void initAppAggregator(final ApplicationId appId, String user,
      Credentials credentials, ContainerLogsRetentionPolicy logRetentionPolicy,
      Map<ApplicationAccessType, String> appAcls,
      LogAggregationContext logAggregationContext) {
      

      
      if (blockNewLogAggr) {
      	return;
      }

      processed = false;

    
    Runnable aggregatorWrapper = new Runnable() {
      public void run() {
        try {
          appLogAggregator.run();
        } finally {
          appLogAggregators.remove(appId);
          closeFileSystems(userUgi);
        }
      }
    };
    
	  logAggregatorQueue.add(aggregatorWrapper);
  }

  
  private BlockingQueue<Runnable> logAggregatorQueue = new LinkedBlockingQueue<Runnable>();
```

```null

    
    
    private volatile boolean stopped = false;
    
    private volatile boolean blockNewLogAggr = false;
    
    private volatile boolean processed = true;
    
    private Object waitForProcess = new Object();
 
  
  private class LauncherLogAggregatorThread implements Runnable {
 
    @Override
    public void run() {
      while (!stopped && !Thread.currentThread().isInterrupted()) {
        processed = logAggregatorQueue.isEmpty();
          if (blockNewLogAggr) {
              synchronized (waitForProcess) {
                  if (processed) {
                      waitForProcess.notify();
                  }
              }
          }
        Runnable toLaunch;
        try {
          
          toLaunch = logAggregatorQueue.take();
          threadPool.execute(toLaunch);
        } catch (InterruptedException e) {
          LOG.warn(this.getClass().getName() + " interrupted. Returning.");
        }
      }
    }
  }
```

服务停止时等待消费队列聚合事件处理完成，然后关闭消费线程和线程池。

```null

 
  @Override
  protected void serviceStop() throws Exception {
    LOG.info(this.getName() + " waiting for pending aggregation during exit");
    blockNewLogAggr = true;
    synchronized (waitForProcess) {
      while (!processed && launcherLogAggregatorThread.isAlive()) {
        waitForProcess.wait(1000);
        LOG.info("Waiting for launcherLogAggregatorThread to process. Thread state is :" + launcherLogAggregatorThread.getState());
      }
    }
 
    this.stopped = true;
    if (launcherLogAggregatorThread != null) {
      launcherLogAggregatorThread.interrupt();
      try {
        launcherLogAggregatorThread.join();
      } catch (InterruptedException ie) {
        LOG.warn(launcherLogAggregatorThread.getName() + " interrupted during join ", ie);
      }
    }
    stopAggregators();
    super.serviceStop();
  }
```

测试集群分为 hadoop-up1 集群和 hadoop-up2 集群，采用 viewfs 模式访问 HDFS，作业提交在 `hadoop-up1` 集群，日志聚合目录 `/tmp/logs/` 挂载在 `hadoop-up2` 集群下，即 `hdfs://hadoop-up2/tmp/logs/` 目录。

## 7.1 NM 日志聚合改造前（原始版本）

作业提交命令：

> hadoop jar /opt/cloudera/parcels/CDH-5.14.4-1.cdh5.14.4.p0.3/jars/hadoop-mapreduce-examples-2.6.0-cdh5.14.4.jar pi -Dmapred.job.queue.name=root.exquery 50 50

### （1）开启 hadoop-up2 集群 HDFS 服务

**结论：** 

作业正常提交，日志正常聚合。

### （2）关闭 hadoop-up2 集群 HDFS 服务

结论：  
作业提交卡住，需等待请求 HDFS 服务超时，作业处于 Accepted 状态卡住，作业的 ApplicationMaster 处于 NEW 状态，该 Container 没有被分配（整个过程卡住大概 3min），直到抛异常触发日志聚合失败（即 ApplicationEventType.APPLICATION_LOG_HANDLING_FAILED）事件，作业的 ApplicationMaster 分配到 Container，作业开始运行，并且日志不聚合。

**现象 1:**

作业提交卡住，Yarn UI 作业的状态为 Accepted 状态，Elapsed 时间大概持续了 3min，说明作业在这段时间一直等待运行，并且用于启动 ApplicationMaster 的 Container 状态为 NEW，没有转换到 Submited 状态，表示 Container 没有运行，Yarn 认为该作业还未提交。这也是线上集群在日志聚合集群 rpc 压力大时会影响作业的提交数和 Container 分配性能下降的原因。  
![](https://img2020.cnblogs.com/blog/1372882/202103/1372882-20210310174325881-1984948013.jpg)

![](https://img2020.cnblogs.com/blog/1372882/202103/1372882-20210310174335394-1907728906.png)

![](https://img2020.cnblogs.com/blog/1372882/202103/1372882-20210310174345251-475906527.png)

**现象 2:**

由于 hadoop-up2 集群 HDFS 服务关闭，分析 NodeManager 执行日志，先打印 `Application failed to init aggregation` 信息，然后打印`LogAggregationService.verifyAndCreateRemoteLogDir()` 方法执行 HDFS 读请求的调用异常，读请求多次重试后抛出 YarnRuntimeException 异常，堆栈信息的调用栈和执行代码都和这一现象吻合。

```null

2021-03-10 09:56:43,444 WARN org.apache.hadoop.yarn.server.nodemanager.containermanager.logaggregation.LogAggregationService: Application failed to init aggregation
org.apache.hadoop.yarn.exceptions.YarnRuntimeException: Failed to check permissions for dir [/tmp/logs]
        at org.apache.hadoop.yarn.server.nodemanager.containermanager.logaggregation.LogAggregationService.verifyAndCreateRemoteLogDir(LogAggregationService.java:205)
        at org.apache.hadoop.yarn.server.nodemanager.containermanager.logaggregation.LogAggregationService.initApp(LogAggregationService.java:336)
        at org.apache.hadoop.yarn.server.nodemanager.containermanager.logaggregation.LogAggregationService.handle(LogAggregationService.java:463)
        at org.apache.hadoop.yarn.server.nodemanager.containermanager.logaggregation.LogAggregationService.handle(LogAggregationService.java:68)
        at org.apache.hadoop.yarn.event.AsyncDispatcher.dispatch(AsyncDispatcher.java:182)
        at org.apache.hadoop.yarn.event.AsyncDispatcher$1.run(AsyncDispatcher.java:109)
        at java.lang.Thread.run(Thread.java:748)
Caused by: java.net.ConnectException: Call From 10-197-1-236/10.197.1.236 to 10-197-1-238:8020 failed on connection exception: java.net.ConnectException: Connection refused; For more details see:  http:
        
        at org.apache.hadoop.hdfs.DistributedFileSystem.getFileStatus(DistributedFileSystem.java:1261)
        at org.apache.hadoop.fs.FilterFileSystem.getFileStatus(FilterFileSystem.java:432)
        at org.apache.hadoop.fs.viewfs.ChRootedFileSystem.getFileStatus(ChRootedFileSystem.java:226)
        at org.apache.hadoop.fs.viewfs.ViewFileSystem.getFileStatus(ViewFileSystem.java:379)
        at org.apache.hadoop.yarn.server.nodemanager.containermanager.logaggregation.LogAggregationService.verifyAndCreateRemoteLogDir(LogAggregationService.java:194)
2021-03-10 09:56:43,445 WARN org.apache.hadoop.yarn.server.nodemanager.containermanager.application.Application: Log Aggregation service failed to initialize, there will be no logs for this application
```

```null


  private void initApp(final ApplicationId appId, String user,
      Credentials credentials, ContainerLogsRetentionPolicy logRetentionPolicy,
      Map<ApplicationAccessType, String> appAcls,
      LogAggregationContext logAggregationContext) {
    ApplicationEvent eventResponse;
    try {
	    verifyAndCreateRemoteLogDir(getConfig());
      initAppAggregator(appId, user, credentials, logRetentionPolicy, appAcls,
          logAggregationContext);
      eventResponse = new ApplicationEvent(appId,
          ApplicationEventType.APPLICATION_LOG_HANDLING_INITED);
    } catch (YarnRuntimeException e) {
      LOG.warn("Application failed to init aggregation", e);
      eventResponse = new ApplicationEvent(appId,
          ApplicationEventType.APPLICATION_LOG_HANDLING_FAILED);
    }
    this.dispatcher.getEventHandler().handle(eventResponse);
  }
```

## 7.2 NM 日志聚合改造后

作业提交命令：

> hadoop jar /opt/cloudera/parcels/CDH-5.14.4-1.cdh5.14.4.p0.3/jars/hadoop-mapreduce-examples-2.6.0-cdh5.14.4.jar pi -Dmapred.job.queue.name=root.exquery 50 50

### （1）开启 hadoop-up2 集群 HDFS 服务

**结论：** 

作业正常提交，日志正常聚合。

### （2）关闭 hadoop-up2 集群 HDFS 服务

结论：  
作业正常提交，日志聚合失败，不影响作业提交和运行。

现象：

-   作业正常执行提交和执行。
-   日志聚合读请求 HDFS 异常（和 7.1 中 NM 日志一致），但不影响作业执行。
-   日志聚合失败，JobHistoryServer 无法查看聚合的日志。  
    ![](https://img2020.cnblogs.com/blog/1372882/202103/1372882-20210310174259939-164697436.png)

1.  [YARN 的 Log Aggregation 原理](https://blog.csdn.net/Androidlushangderen/article/details/90115624) 
    [https://www.cnblogs.com/lemonu/p/14513315.html](https://www.cnblogs.com/lemonu/p/14513315.html) 
    [https://www.cnblogs.com/lemonu/p/14513315.html](https://www.cnblogs.com/lemonu/p/14513315.html)
