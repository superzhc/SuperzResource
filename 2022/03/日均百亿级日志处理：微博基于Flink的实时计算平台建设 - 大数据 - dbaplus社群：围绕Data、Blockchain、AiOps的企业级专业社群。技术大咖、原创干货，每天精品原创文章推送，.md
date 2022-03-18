# 日均百亿级日志处理：微博基于Flink的实时计算平台建设 - 大数据 - dbaplus社群：围绕Data、Blockchain、AiOps的企业级专业社群。技术大咖、原创干货，每天精品原创文章推送，每周线上技术分享，每月线下技术沙龙。
作者介绍

**吕永卫，**微博广告资深数据开发工程师，实时数据项目组负责人。

**黄鹏，**微博广告实时数据开发工程师，负责法拉第实验平台数据开发、实时数据关联平台、实时算法特征数据计算、实时数据仓库、实时数据清洗组件开发工作。

**林发明，**微博广告资深数据开发工程师，负责算法实时特征数据计算、实时数据关联平台、实时数据仓库、FlinkStream 组件开发工作。

**崔泽峰，**微博广告资深数据开发工程师，负责实时算法特征数据计算、实时任务管理平台、FlinkStream 组件、FlinkSQL 扩展开发工作。

**引言**

是随着微博业务线的快速扩张，微博广告各类业务日志的数量也随之急剧增长。传统基于 Hadoop 生态的离线数据存储计算方案已在业界形成统一的默契，但受制于离线计算的时效性制约，越来越多的数据应用场景已从离线转为实时。微博广告实时数据平台以此为背景进行设计与构建，目前该系统已支持日均处理日志数量超过百亿，接入产品线、业务日志类型若干。

**一、技术选型**

相比于 Spark，目前 Spark 的生态总体更为完善一些，且在机器学习的集成和应用性暂时领先。但作为下一代大数据引擎的有力竞争者 - Flink 在流式计算上有明显优势，Flink 在流式计算里属于真正意义上的单条处理，每一条数据都触发计算，而不是像 Spark 一样的 Mini Batch 作为流式处理的妥协。Flink 的容错机制较为轻量，对吞吐量影响较小，而且拥有图和调度上的一些优化，使得 Flink 可以达到很高的吞吐量。而 Strom 的容错机制需要对每条数据进行 ack，因此其吞吐量瓶颈也是备受诟病。

这里引用一张图来对常用的实时计算框架做个对比。

![](https://dbaplus.cn/uploadfile/2019/1023/20191023105507381.png)

Flink 特点

Flink 是一个开源的分布式实时计算框架。Flink 是有状态的和容错的，可以在维护一次应用程序状态的同时无缝地从故障中恢复；它支持大规模计算能力，能够在数千个节点上并发运行；它具有很好的吞吐量和延迟特性。同时，Flink 提供了多种灵活的窗口函数。

**1）状态管理机制**

Flink 检查点机制能保持 exactly-once 语义的计算。状态保持意味着应用能够保存已经处理的数据集结果和状态。

![](https://dbaplus.cn/uploadfile/2019/1023/20191023105525230.png)

**2）事件机制**

Flink 支持流处理和窗口事件时间语义。事件时间可以很容易地通过事件到达的顺序和事件可能的到达延迟流中计算出准确的结果。

![](https://dbaplus.cn/uploadfile/2019/1023/20191023105541886.png)

**3）窗口机制**

Flink 支持基于时间、数目以及会话的非常灵活的窗口机制（window）。可以定制 window 的触发条件来支持更加复杂的流模式。

![](https://dbaplus.cn/uploadfile/2019/1023/20191023105557184.png)

**4）容错机制**

Flink 高效的容错机制允许系统在高吞吐量的情况下支持 exactly-once 语义的计算。Flink 可以准确、快速地做到从故障中以零数据丢失的效果进行恢复。

![](https://dbaplus.cn/uploadfile/2019/1023/20191023105617242.png)

**5）高吞吐、低延迟**

Flink 具有高吞吐量和低延迟（能快速处理大量数据）特性。下图展示了 Apache Flink 和 Apache Storm 完成分布式项目计数任务的性能对比。

![](https://dbaplus.cn/uploadfile/2019/1023/20191023105634395.png)

**二、架构演变**

初期架构

初期架构仅为计算与存储两层，新来的计算需求接入后需要新开发一个实时计算任务进行上线。重复模块的代码复用率低，重复率高，计算任务间的区别主要是集中在任务的计算指标口径上。

在存储层，各个需求方所需求的存储路径都不相同，计算指标可能在不通的存储引擎上有重复，有计算资源以及存储资源上的浪费情况。并且对于指标的计算口径也是仅局限于单个任务需求里的，不通需求任务对于相同的指标的计算口径没有进行统一的限制于保障。各个业务方也是在不同的存储引擎上开发数据获取服务，对于那些专注于数据应用本身的团队来说，无疑当前模式存在一些弊端。

![](https://dbaplus.cn/uploadfile/2019/1023/20191023105654129.png)

后期架构

随着数据体量的增加以及业务线的扩展，前期架构模式的弊端逐步开始显现。从当初单需求单任务的模式逐步转变为通用的数据架构模式。为此，我们开发了一些基于 Flink 框架的通用组件来支持数据的快速接入，并保证代码模式的统一性和维护性。在数据层，我们基于 Clickhouse 来作为我们数据仓库的计算和存储引擎，利用其支持多维 OLAP 计算的特性，来处理在多维多指标大数据量下的快速查询需求。在数据分层上，我们参考与借鉴离线数仓的经验与方法，构建多层实时数仓服务，并开发多种微服务来为数仓的数据聚合，指标提取，数据出口，数据质量，报警监控等提供支持。

![](https://dbaplus.cn/uploadfile/2019/1023/20191023105714853.png)

**整体架构分为五层：** 

**1）接入层：** 接入原始数据进行处理，如 Kafka、RabbitMQ、File 等。

**2）计算层：** 选用 Flink 作为实时计算框架，对实时数据进行清洗，关联等操作。

**3）存储层：**  对清洗完成的数据进行数据存储，我们对此进行了实时数仓的模型分层与构建，将不同应用场景的数据分别存储在如 Clickhouse,Hbase,Redis,Mysql 等存储。服务中，并抽象公共数据层与维度层数据，分层处理压缩数据并统一数据口径。

**4）服务层：** 对外提供统一的数据查询服务，支持从底层明细数据到聚合层数据 5min/10min/1hour 的多维计算服务。同时最上层特征指标类数据，如计算层输入到 Redis、Mysql 等也从此数据接口进行获取。

**5）应用层：** 以统一查询服务为支撑对各个业务线数据场景进行支撑。

**监控报警：** 对 Flink 任务的存活状态进行监控，对异常的任务进行邮件报警并根据设定的参数对任务进行自动拉起与恢复。根据如 Kafka 消费的 offset 指标对消费处理延迟的实时任务进行报警提醒。

**数据质量：** 监控实时数据指标，对历史的实时数据与离线 hive 计算的数据定时做对比，提供实时数据的数据质量指标，对超过阈值的指标数据进行报警。

**三、数据处理流程**

1、整体流程

整体数据从原始数据接入后经过 ETL 处理, 进入实时数仓底层数据表，经过配置化聚合微服务组件向上进行分层数据的聚合。根据不同业务的指标需求也可通过特征抽取微服务直接配置化从数仓中抽取到如 Redis、ES、Mysql 中进行获取。大部分的数据需求可通过统一数据服务接口进行获取。

![](https://dbaplus.cn/uploadfile/2019/1023/20191023105731839.png)

2、问题与挑战

原始日志数据因为各业务日志的不同，所拥有的维度或指标数据并不完整。所以需要进行实时的日志的关联才能获取不同维度条件下的指标数据查询结果。并且关联日志的回传周期不同，有在 10min 之内完成 95% 以上回传的业务日志，也有类似于激活日志等依赖第三方回传的有任务日志，延迟窗口可能大于 1 天。并且最大日志关联任务的日均数据量在 10 亿级别以上，如何快速处理与构建实时关联任务的问题首先摆在我们面前。对此我们基于 Flink 框架开发了配置化关联组件。对于不同关联日志的指标抽取，我们也开发了配置化指标抽取组件用于快速提取复杂的日志格式。以上两个自研组件会在后面的内容里再做详细介绍。

**1）回传周期超过关联窗口的日志如何处理?**

对于回传晚的日志，我们在关联窗口内未取得关联结果。我们采用实时 + 离线的方式进行数据回刷补全。实时处理的日志我们会将未关联的原始日志输出到另外一个暂存地 (Kafka)，同时不断消费处理这个未关联的日志集合，设定超时重关联次数与超时重关联时间，超过所设定任意阈值后，便再进行重关联。离线部分，我们采用 Hive 计算昨日全天日志与 N 天内的全量被关联日志表进行关联，将最终的结果回写进去，替换实时所计算的昨日关联数据。

**2）如何提高 Flink 任务性能？**

**① Operator Chain**

为了更高效地分布式执行，Flink 会尽可能地将 operator 的 subtask 链接（chain）在一起形成 task。每个 task 在一个线程中执行。将 operators 链接成 task 是非常有效的优化：它能减少线程之间的切换，减少消息的序列化 / 反序列化，减少数据在缓冲区的交换，减少了延迟的同时提高整体的吞吐量。

Flink 会在生成 JobGraph 阶段，将代码中可以优化的算子优化成一个算子链（Operator Chains）以放到一个 task（一个线程）中执行，以减少线程之间的切换和缓冲的开销，提高整体的吞吐量和延迟。下面以官网中的例子进行说明。

![](https://dbaplus.cn/uploadfile/2019/1023/20191023105749761.png)

图中，source、map、\[keyBy|window|apply]、sink 算子的并行度分别是 2、2、2、2、1，经过 Flink 优化后，source 和 map 算子组成一个算子链，作为一个 task 运行在一个线程上，其简图如图中 condensed view 所示，并行图如 parallelized view 所示。算子之间是否可以组成一个 Operator Chains，看是否满足以下条件：

-   上下游算子的并行度一致；
-   下游节点的入度为 1；
-   上下游节点都在同一个 slot group 中；
-   下游节点的 chain 策略为 ALWAYS；
-   上游节点的 chain 策略为 ALWAYS 或 HEAD；
-   两个节点间数据分区方式是 forward；
-   用户没有禁用 chain。

**② Flink 异步 IO**

流式计算中，常常需要与外部系统进行交互。而往往一次连接中你那个获取连接等待通信的耗时会占比较高。下图是两种方式对比示例：

![](https://dbaplus.cn/uploadfile/2019/1023/20191023105809524.png)

图中棕色的长条表示等待时间，可以发现网络等待时间极大地阻碍了吞吐和延迟。为了解决同步访问的问题，异步模式可以并发地处理多个请求和回复。也就是说，你可以连续地向数据库发送用户 a、b、c 等的请求，与此同时，哪个请求的回复先返回了就处理哪个回复，从而连续的请求之间不需要阻塞等待，如上图右边所示。这也正是 Async I/O 的实现原理。

**③ Checkpoint 优化**

Flink 实现了一套强大的 checkpoint 机制，使它在获取高吞吐量性能的同时，也能保证 Exactly Once 级别的快速恢复。

首先提升各节点 checkpoint 的性能考虑的就是存储引擎的执行效率。Flink 官方支持的三种 checkpoint state 存储方案中，Memory 仅用于调试级别，无法做故障后的数据恢复。其次还有 Hdfs 与 Rocksdb，当所做 Checkpoint 的数据大小较大时，可以考虑采用 Rocksdb 来作为 checkpoint 的存储以提升效率。

其次的思路是资源设置，我们都知道 checkpoint 机制是在每个 task 上都会进行，那么当总的状态数据大小不变的情况下，如何分配减少单个 task 所分的的 checkpoint 数据变成了提升 checkpoint 执行效率的关键。

最后，增量快照. 非增量快照下，每次 checkpoint 都包含了作业所有状态数据。而大部分场景下，前后 checkpoint 里，数据发生变更的部分相对很少，所以设置增量 checkpoint，仅会对上次 checkpoint 和本次 checkpoint 之间状态的差异进行存储计算，减少了 checkpoint 的耗时。

**3）如何保障任务的稳定性?**

在任务执行过程中，会遇到各种各样的问题，导致任务异常甚至失败。所以如何做好异常情况下的恢复工作显得异常重要。

**① 设定重启策略**

Flink 支持不同的重启策略，以在故障发生时控制作业如何重启。集群在启动时会伴随一个默认的重启策略，在没有定义具体重启策略时会使用该默认策略。如果在工作提交时指定了一个重启策略，该策略会覆盖集群的默认策略。

默认的重启策略可以通过 Flink 的配置文件 flink-conf.yaml 指定。配置参数 restart-strategy 定义了哪个策略被使用。

常用的重启策略：

-   固定间隔 (Fixed delay)；
-   失败率 (Failure rate)；
-   无重启 (No restart)。

**② 设置 HA**

Flink 在任务启动时指定 HA 配置主要是为了利用 Zookeeper 在所有运行的 JobManager 实例之间进行分布式协调. Zookeeper 通过 leader 选取和轻量级一致性的状态存储来提供高可用的分布式协调服务。

**③ 任务监控报警平台**

在实际环境中，我们遇见过因为集群状态不稳定而导致的任务失败。在 Flink1.6 版本中，甚至遇见过任务出现假死的情况，也就是 Yarn 上的 job 资源依然存在，而 Flink 任务实际已经死亡。为了监测与恢复这些异常的任务，并且对实时任务做统一的提交、报警监控、任务恢复等管理，我们开发了任务提交与管理平台。通过 Shell 拉取 Yarn 上 Running 状态与 Flink Job 状态的列表进行对比，心跳监测平台上的所有任务，并进行告警、自动恢复等操作。

![](https://dbaplus.cn/uploadfile/2019/1023/20191023105834726.png)

**④ 作业指标监控**

Flink 任务在运行过程中，各 Operator 都会产生各自的指标数据，例如，Source 会产出 numRecordIn、numRecordsOut 等各项指标信息，我们会将这些指标信息进行收集，并展示在我们的可视化平台上。指标平台如下图：

![](https://dbaplus.cn/uploadfile/2019/1023/20191023105851896.png)

**⑤ 任务运行节点监控**

我们的 Flink 任务都是运行在 Yarn 上，针对每一个运行的作业，我们需要监控其运行环境。会收集 JobManager 及 TaskManager 的各项指标。收集的指标有 jobmanager-fullgc-count、jobmanager-younggc-count、jobmanager-fullgc-time、jobmanager-younggc-time、taskmanager-fullgc-count、taskmanager-younggc-count、taskmanager-fullgc-time、taskmanager-younggc-time 等，用于判断任务运行环境的健康度，及用于排查可能出现的问题。监控界面如下：

![](https://dbaplus.cn/uploadfile/2019/1023/20191023105909336.png)

**四、数据关联组件**

1、如何选择关联方式？

**1）Flink Table**

从 Flink 的官方文档，我们知道 Flink 的编程模型分为四层，sql 是最高层的 api, Table api 是中间层，DataSteam/DataSet Api 是核心，stateful Streaming process 层是底层实现。

![](https://dbaplus.cn/uploadfile/2019/1023/20191023105926815.png)

刚开始我们直接使用 Flink Table 做为数据关联的方式，直接将接入进来的 DataStream 注册为 Dynamic Table 后进行两表关联查询，如下图：

![](https://dbaplus.cn/uploadfile/2019/1023/20191023110035313.png)

但尝试后发现在做那些日志数据量大的关联查询时往往只能在较小的时间窗口内做查询，否则会超过 datanode 节点单台内存限制，产生异常。但为了满足不同业务日志延迟到达的情况，这种实现方式并不通用。

**2）Rocksdb**

之后，我们直接在 DataStream 上进行处理，在 CountWindow 窗口内进行关联操作，将被关联的数据 Hash 打散后存储在各个 datanode 节点的 Rocksdb 中，利用 Flink State 原生支持 Rocksdb 做 Checkpoint 这一特性进行算子内数据的备份与恢复。这种方式是可行的，但受制于 Rocksdb 集群物理磁盘为非 SSD 的因素，这种方式在我们的实际线上场景中关联耗时较高。

**3）外部存储关联**

如 Redis 类的 KV 存储的确在查询速度上提升不少，但类似广告日志数据这样单条日志大小较大的情况，会占用不少宝贵的机器内存资源。经过调研后，我们选取了 Hbase 作为我们日志关联组件的关联数据存储方案。

为了快速构建关联任务，我们开发了基于 Flink 的配置化组件平台，提交配置文件即可生成数据关联任务并自动提交到集群。下图是任务执行的处理流程。

示意图如下：

![](https://dbaplus.cn/uploadfile/2019/1023/20191023110110705.png)

下图是关联组件内的执行流程图：

![](https://dbaplus.cn/uploadfile/2019/1023/20191023110123957.png)

2、问题与优化

**1）加入 Interval Join**

随着日志量的增加，某些需要进行关联的日志数量可能达到日均十几亿甚至几十亿的量级。前期关联组件的配置化生成任务的方式的确解决了大部分线上业务需求，但随着进一步的关联需求增加，Hbase 面临着巨大的查询压力。在我们对 Hbase 表包括 rowkey 等一系列完成优化之后，我们开始了对关联组件的迭代与优化。

第一步，减少 Hbase 的查询。我们使用 Flink Interval Join 的方式，先将大部分关联需求在程序内部完成，只有少部分仍需查询的日志会去查询外部存储 (Hbase). 经验证，以请求日志与实验日志关联为例，对于设置 Interval Join 窗口在 10s 左右即可减少 80% 的 hbase 查询请求

**① Interval Join 的语义示意图**

![](https://dbaplus.cn/uploadfile/2019/1023/20191023110142810.png)

-   数据 JOIN 的区间 - 比如时间为 3 的 EXP 会在 IMP 时间为\[2, 4]区间进行 JOIN；
-   WaterMark - 比如图示 EXP 一条数据时间是 3，IMP 一条数据时间是 5，那么 WaterMark 是根据实际最小值减去 UpperBound 生成，即：Min(3,5)-1 = 2；
-   过期数据 - 出于性能和存储的考虑，要将过期数据清除，如图当 WaterMark 是 2 的时候时间为 2 以前的数据过期了，可以被清除。

**② Interval Join 内部实现逻辑**

![](https://dbaplus.cn/uploadfile/2019/1023/20191023110202984.png)

**③ Interval Join 改造**

因 Flink 原生的 Intervak Join 实现的是 Inner Join, 而我们业务中所需要的是 Left Join，具体改造如下：

-   取消右侧数据流的 join 标志位；
-   左侧数据流有 join 数据时不存 state。

**2）关联率动态监控**

在任务执行中，往往会出现意想不到的情况，比如被关联的数据日志出现缺失，或者日志格式错误引发的异常，造成关联任务的关联率下降严重。那么此时关联任务虽然继续在运行，但对于整体数据质量的意义不大，甚至是反向作用。在任务进行恢复的时，还需要清除异常区间内的数据，将 Kafka Offset 设置到异常前的位置再进行处理。

故我们在关联组件的优化中，加入了动态监控，下面示意图：

![](https://dbaplus.cn/uploadfile/2019/1023/20191023110219141.png)

-   关联任务中定时探测指定时间范围 Hbase 是否有最新数据写入，如果没有，说明写 Hbase 任务出现问题，则终止关联任务；
-   当写 Hbase 任务出现堆积时，相应的会导致关联率下降，当关联率低于指定阈值时终止关联任务；
-   当关联任务终止时会发出告警，修复上游任务后可重新恢复关联任务，保证关联数据不丢失。

**五、数据清洗组件**

为了快速进行日志数据的指标抽取，我们开发了基于 Flink 计算平台的指标抽取组件 Logwash。封装了基于 Freemaker 的模板引擎做为日志格式的解析模块，对日志进行提取，算术运算，条件判断，替换，循环遍历等操作。

下图是 Logwash 组件的处理流程：

![](https://dbaplus.cn/uploadfile/2019/1023/20191023110237903.png)

组件支持文本与 Json 两种类型日志进行解析提取，目前该清洗组件已支持微博广告近百个实时清洗需求，提供给运维组等第三方非实时计算方向人员快速进行提取日志的能力。

配置文件部分示例：

![](https://dbaplus.cn/uploadfile/2019/1023/20191023110253736.png)

**六、FlinkStream 组件库**

Flink 中 DataStream 的开发，对于通用的逻辑及相同的代码进行了抽取，生成了我们的通用组件库 FlinkStream。FlinkStream 包括了对 Topology 的抽象及默认实现、对 Stream 的抽象及默认实现、对 Source 的抽象和某些实现、对 Operator 的抽象及某些实现、Sink 的抽象及某些实现。任务提交统一使用可执行 Jar 和配置文件，Jar 会读取配置文件构建对应的拓扑图。

1、Source 抽象

对于 Source 进行抽象，创建抽象类及对应接口，对于 Flink Connector 中已有的实现，例如 kafka,Elasticsearch 等，直接创建新 class 并继承接口，实现对应的方法即可。对于需要自己去实现的 connector，直接继承抽象类及对应接口，实现方法即可。目前只实现了 KafkaSource。

2、Operator 抽象

与 Source 抽象类似，我们实现了基于 Stream 到 Stream 级别的 Operator 抽象。创建抽象 Operate 类，抽象 Transform 方法。对于要实现的 Transform 操作，直接继承抽象类，实现其抽象方法即可。目前实现的 Operator，直接按照文档使用。如下：

![](https://dbaplus.cn/uploadfile/2019/1023/20191023110311614.png)

3、Sink 抽象

针对 Sink，我们同样创建了抽象类及接口。对 Flink Connector 中已有的 Sink 进行封装。目前可通过配置进行数据输出的 Sink。目前以实现和封装的 Sink 组件有：Kafka、Stdout、Elasticsearch、Clickhouse、Hbase、Redis、MySQL。

4、Stream 抽象

创建 Stream 抽象类及抽象方法 buildStream，用于构建 StreamGraph。我们实现了默认的 Stream，buildStream 方法读取 Source 配置生成 DataStream，通过 Operator 配置列表按顺序生成拓扑图，通过 Sink 配置生成数据写出组件。

5、Topology 抽象

对于单 Stream，要处理的逻辑可能比较简单，主要读取一个 Source 进行数据的各种操作并输出。对于复杂的多 Stream 业务需求，比如多流 Join，多流 Union、Split 流等，因此我们多流业务进行了抽象，产生了 Topology。在 Topology 这一层可以对多流进行配置化操作。对于通用的操作，我们实现了默认 Topology，直接通过配置文件就可以实现业务需求。对于比较复杂的业务场景，用户可以自己实现 Topology。

6、配置化

我们对抽象的组件都是可配置化的，直接通过编写配置文件，构造任务的运行拓扑结构，启动任务时指定配置文件。

-   正文文本框 Flink Environment 配置化，包括时间处理类型、重启策略，checkpoint 等；
-   Topology 配置化，可配置不同 Stream 之间的处理逻辑与 Sink；
-   Stream 配置化，可配置 Source，Operator 列表，Sink。

配置示例如下：

run_env:

  timeCharacteristic: "ProcessingTime" #ProcessingTime\\IngestionTime\\EventTime

  restart: # 重启策略配置

    type: # noRestart, fixedDelayRestart, fallBackRestart, failureRateRestart

  checkpoint: # 开启 checkpoint

    type: "rocksdb" # 

streams:

  impStream:  #粉丝经济曝光日志

    type: "DefaultStream"

    config:

      source:

        type: "Kafka011" # 源是 kafka011 版本

        config:

        parallelism: 20

      operates:

        \-

          type: "StringToMap"

          config:

        \-

          type: "SplitElement"

          config:

        ...

        \-

          type: "SelectElement"

          config:

transforms:

  \-

    type: "KeyBy"

    config:

  \-

    type: "CountWindowWithTimeOut"  #Window 需要和 KeyBy 组合使用

    config:

  \-

    type: "SplitStream"

    config:

  \-

    type: "SelectStream"

    config:

sink:

  \-

    type: Kafka

    config:

  \-

    type: Kafka

    config:

7、部署

在实时任务管理平台，新建任务，填写任务名称，选择任务类型（Flink）及版本，上传可执行 Jar 文件，导入配置或者手动编写配置，填写 JobManager 及 TaskManager 内存配置，填写并行度配置，选择是否重试，选择是否从 checkpoint 恢复等选项，保存后即可在任务列表中启动任务，并观察启动日志用于排查启动错误。

![](https://dbaplus.cn/uploadfile/2019/1023/20191023110338747.png)

**七、FlinkSQL 扩展**

SQL 语言是一门声明式的，简单的，灵活的语言，Flink 本身提供了对 SQL 的支持。Flink1.6 版本和 1.8 版本对 SQL 语言的支持有限，不支持建表语句，不支持对外部数据的关联操作。因此我们通过 Apache Calcite 对 Flink SQL API 进行了扩展，用户只需要关心业务需求怎么用 SQL 语言来表达即可。

1、支持创建源表

扩展了支持创建源表 SQL，通过解析 SQL 语句，获取数据源配置信息，创建对应的 TableSource 实例，并将其注册到 Flink environment。示例如下：

![](https://dbaplus.cn/uploadfile/2019/1023/20191023110356691.png)

2、支持创建维表

使用 Apache Calcite 对 SQL 进行解析，通过维表关键字识别维表，使用 RichAsyncFunction 算子异步读取维表数据，并通过 flatMap 操作生成关联后的 DataStream，然后转换为 Table 注册到 Flink Environment。示例如下：

![](https://dbaplus.cn/uploadfile/2019/1023/20191023110413201.png)

3、支持创建视图

使用 sqlQuery 方法，支持从上一层表或者视图中创建视图表，并将新的视图表注册到 Flink Environment。创建语句需要按照顺序写，比如 myView2 是从视图 myView1 中创建的，则 myView1 创建语句要在 myView2 语句前面。如下：

![](https://dbaplus.cn/uploadfile/2019/1023/20191023110516259.png)

4、支持创建结果表

支持创建结果表，通过解析 SQL 语句，获取配置信息，创建对应的 AppendStreamTableSink 或者 UpsertStreamTableSink 实例，并将其注册到 Flink Environment。示例如下：

![](https://dbaplus.cn/uploadfile/2019/1023/20191023110458409.png)

5、支持自定义 UDF

支持自定义 UDF 函数，继承 ScalarFunction 或者 TableFunction。在 resources 目录下有相应的 UDF 资源配置文件，默认会注册全部可执行 Jar 包中配置的 UDF。直接按照使用方法使用即可。

6、部署

部署方式同 FlinkStream 组件。

**八、实时数据仓库的构建**

为了保证实时数据的统一对外出口以及保证数据指标的统一口径，我们根据业界离线数仓的经验来设计与构架微博广告实时数仓。

1、分层概览

数据仓库分为三层，自下而上为：数据引入层（ODS，Operation Data Store）、数据公共层（CDM，Common Data Model）和数据应用层（ADS，Application Data Service）

![](https://dbaplus.cn/uploadfile/2019/1023/20191023110536777.png)

**数据引入层（ODS，Operation Data Store）：** 将原始数据几乎无处理的存放在数据仓库系统，结构上与源系统基本保持一致，是数据仓库的数据准。

**数据公共层（CDM，Common Data Model，又称通用数据模型层）：** 包含 DIM 维度表、DWD 和 DWS，由 ODS 层数据加工而成。主要完成数据加工与整合，建立一致性的维度，构建可复用的面向分析和统计的明细事实表，以及汇总公共粒度的指标。

**公共维度层（DIM）：** 基于维度建模理念思想，建立整个企业的一致性维度。降低数据计算口径和算法不统一风险。

公共维度层的表通常也被称为逻辑维度表，维度和维度逻辑表通常一一对应。

**公共汇总粒度事实层（DWS，Data Warehouse Service）：** 以分析的主题对象作为建模驱动，基于上层的应用和产品的指标需求，构建公共粒度的汇总指标事实表，以宽表化手段物理化模型。构建命名规范、口径一致的统计指标，为上层提供公共指标，建立汇总宽表、明细事实表。

公共汇总粒度事实层的表通常也被称为汇总逻辑表，用于存放派生指标数据。

**明细粒度事实层（DWD，Data Warehouse Detail）：** 以业务过程作为建模驱动，基于每个具体的业务过程特点，构建最细粒度的明细层事实表。可以结合企业的数据使用特点，将明细事实表的某些重要维度属性字段做适当冗余，也即宽表化处理。

明细粒度事实层的表通常也被称为逻辑事实表。

**数据应用层（ADS，Application Data Service）：存放**数据产品个性化的统计指标数据。根据 CDM 与 ODS 层加工生成。

2、详细分层模型

![](https://dbaplus.cn/uploadfile/2019/1023/20191023110553773.png)

对于原始日志数据，ODS 层几乎是每条日志抽取字段后进行保留，这样便能对问题的回溯与追踪。在 CDM 层对 ODS 的数据仅做时间粒度上的数据压缩，也就是在指定时间切分窗口里，对所有维度下的指标做聚合操作，而不涉及业务性的操作。在 ADS 层，我们会有配置化抽取微服务，对底层数据做定制化计算和提取，输出到用户指定的存储服务里。 
 [https://dbaplus.cn/news-73-2790-1.html](https://dbaplus.cn/news-73-2790-1.html)
