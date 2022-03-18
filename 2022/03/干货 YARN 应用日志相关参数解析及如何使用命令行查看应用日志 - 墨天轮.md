# 干货 | YARN 应用日志相关参数解析及如何使用命令行查看应用日志 - 墨天轮
> 版本：
>
> yarn：2.6.0+cdh5.11.0

### 一、前言

对于从事大数据相关工作的朋友来说，在平时应该会跟 yarn 打过不少交道。像 MapReduce on yarn，Spark on yarn，Flink on yarn 等都是需要将应用运行在 yarn 上面的。但是对于应用运行日志的查看，yarn 却不像寻常服务那样方便，确实是有一些门槛的。而今天，我们就来好好梳理运行在 yarn 上面的应用日志相关参数及查看方式，最后以查看 Flink on yarn 日志示例。

### 二、作业运行日志

Container 日志包含 ApplicationMaster 日志和普通 Task 日志等信息，由配置 yarn.nodemanager.log-dirs 管理，这个是应用的本地（nodemanager 节点）日志。

由于作业在 Container 里面运行，应用会随机调度在某一 NodeManager 节点，假如 yarn.nodemanager.log-dirs 配置了多个路径。那么查看某应用日志，就比较繁琐了，你需要先确定 NodeManager 节点，然后找到日志路径，如果日志路径配置多的话，寻找日志比较困难。

### 三、日志聚合

为了解决以上痛点，yarn 为了方便用户，还支持开启日志聚合功能，设置 **yarn.log-aggregation-enable** 为 true ，默认为 false 。日志聚合是 yarn 提供的日志中央化管理功能，收集每个容器的日志并将这些日志移动到文件系统中，比如 HDFS 上，方便用户查看日志。

可能大部分朋友，都会通过执行 yarn logs -applicationId Container−Id 的目录下有该 Container 生成的文件 err、log 和 out 文件。由于作业在 Container 里面运行，应用会随机调度在某一 NodeManager 节点，假如 yarn.nodemanager.log−dirs 配置了多个路径。那么查看某应用日志，就比较繁琐了，你需要先确定 NodeManager 节点，然后找到日志路径，如果日志路径配置多的话，寻找日志比较困难。¨K4K 为了解决以上痛点，yarn 为了方便用户，还支持开启日志聚合功能，设置∗∗yarn.log−aggregation−enable∗∗为 true，默认为 false。日志聚合是 yarn 提供的日志中央化管理功能，收集每个容器的日志并将这些日志移动到文件系统中，比如 HDFS 上，方便用户查看日志。可能大部分朋友，都会通过执行 yarn logs −applicationId ${applicationId} 来查看应用日志。yarn logs -applicationId 命令查看的其实就是聚合后的应用日志，也就是 HDFS 上面的日志，日志目录可由 yarn-site.xml 文件参数配置：

-   yarn.nodemanager.remote-app-log-dir：日志聚合的地址，默认为 /tmp/logs
-   yarn.nodemanager.remote-app-log-dir-suffix：日志聚合的地址后缀，默认为 logs

结合上述两个参数，默认情况下，远程日志目录将在 /tmp/logs/![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211126_a1a40074-4e90-11ec-8658-fa163eb4f6be.png)
{user} 为 yarn 应用执行用户。

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211126_a1d26afe-4e90-11ec-8658-fa163eb4f6be.png)

日志聚合开启后，运行的应用日志是什么时候触发聚合操作呢？运行中还是结束后？我们继续往下看：

我们又找到了 **yarn.nodemanager.log-aggregation.roll-monitoring-interval-seconds** 配置，该配置表示：NodeManager 上传日志文件的频率。默认值为 -1。默认情况下，日志将在应用程序完成时上传。通过设置该配置，可以在应用程序运行时定期上传日志。可以设置的最小滚动间隔秒数为 3600。

yarn 更多配置参数可参考：[https://hadoop.apache.org/docs/r2.6.0/hadoop-yarn/hadoop-yarn-common/yarn-default.xml](https://hadoop.apache.org/docs/r2.6.0/hadoop-yarn/hadoop-yarn-common/yarn-default.xml)

### 四、日志清理

#### 1、运行日志

-   **yarn.nodemanager.log.retain-seconds**: 保存在本地节点的日志的留存时间, 默认值是 10800，单位：秒，即 3 小时。**当开启日志聚合功能后，该配置无效。** 
-   **yarn.nodemanager.delete.debug-delay-sec**：默认值为 0，表示在开启日志聚合功能的情况下，应用完成后，进行日志聚合，然后 NodeManager 的 DeletionService 立即删除应用的本地日志。如果想查看应用日志，可以将该属性值设置得足够大（例如，设置为 600 = 10 分钟）以允许查看这些日志。
-   **yarn.nodemanager.delete.thread-count**: NodeManager 用于日志清理的线程数，默认值为 4。

#### 2、远程聚合日志

-   **yarn.log-aggregation.retain-seconds**: 在删除聚合日志之前保留聚合日志的时间。默认值是 -1，表示永久不删除日志。这意味着应用程序的日志聚合所占的空间会不断的增长，从而造成 HDFS 集群的资源过度使用。
-   **yarn.log-aggregation.retain-check-interval-seconds**: 聚合日志保存检查间隔时间，确定多长时间去检查一次聚合日志的留存情况以执行日志的删除。如果设置为 0 或者负值，那这个值就会用聚合日志保存时间的 1/10 来自动配置，默认值是 -1。

### 五、查看 Flink on Yarn 日志

现在以在 yarn 上查看 flink 应用日志为例，由于 flink 应用是实时运行的，所以如果不配置 yarn.nodemanager.log-aggregation.roll-monitoring-interval-seconds 的话，则不会将日志聚合到 HDFS 上，那就需要我们去查看 Container 日志。

1、yarn application -list

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211126_a1f5b4e6-4e90-11ec-8658-fa163eb4f6be.png)

2、yarn applicationattempt -list <ApplicationId>

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211126_a20fe0d2-4e90-11ec-8658-fa163eb4f6be.png)

3、yarn container -list <Application Attempt Id>

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211126_a228d4c0-4e90-11ec-8658-fa163eb4f6be.png)

4、查看对应 Container 日志

上述列表中，Container 启动最早的那个编号是 jobmanager，其余的是 taskmanager 。根据 yarn 配置：yarn.nodemanager.log-dirs，路径为：/data/yarn/container-logs。

前往对应 Host 节点，查看 /data/yarn/container-logs/![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211126_a238f4cc-4e90-11ec-8658-fa163eb4f6be.png)
{Container-Id} 下面的容器日志。

-   jobmanager.log 为 flink 任务管理日志。
-   taskmanager.log 为 flink 任务工作日志。

当然，也有朋友会问，我在 yarn resourceManager UI 上面也可以看到应用日志啊。是的，能看到，但我还是感觉命令行简单，并且你也不能保证每个项目的 yarn 环境，都能访问外网是吧。

所以我上面分享的查到对应的 Container 日志命令，是很有必要掌握的。

### 六、总结

1、本篇文章，以 yarn 2.6.0 版本为例，主要讲解了 yarn 应用日志相关，分为本地 Container 日志和聚合日志。

2、接下来又讲解了 yarn 应用日志的相关参数，比如：日志存储目录、日志聚合相关参数、日志清理相关参数等

3、最后，就以查看 flink on yarn 日志为例，梳理了一下用 yarn 命令如何定位 Container 日志所在主机，如何用命令来查看日志。当然最后也建议大家，尽量学会以命令行的方式查看日志，因为不是每个项目环境的 yarn 都留有外网，而命令行则是我们程序员最后的倔强。 
 [https://www.modb.pro/db/179946](https://www.modb.pro/db/179946)
