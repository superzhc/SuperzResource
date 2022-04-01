# Spark On Yarn如何查看任务日志 - 简书
无论 Flink 还是 Spark 都支持自建集群 (standalone cluster)。但是为了保证稳定性和资源隔离等，生产环境里的任务最好借助资源管理框架(如 Yarn) 运行。任务运行在 yarn 上，查询日志就可能不是很方便，尤其是任务进程异常退出之后。

### JobHistoryServer

yarn 容器退出之后，默认是不保存日志的。所以需要开启 JobHistoryServer，具体方法网上有很多教程。

### 查看运行中 Spark 任务的 Log

运行中的 Spark 任务可以直接通过 spark web ui 查看：

![](https://github.com/superzhc/SuperzResource/blob/main/images/2022-4-1%2010-43-29/0dc8076f-c813-47e2-80e7-229dcad97262.png?raw=true)

Executors

![](https://github.com/superzhc/SuperzResource/blob/main/images/2022-4-1%2010-43-29/3f829431-3d67-4b47-a1e0-4e6d394c26dc.png?raw=true)

Driver Log

### 查看已退出 Spark 任务的 Log

对于已经结束的 yarn 应用，spark 进程已经退出也无法提供 webui 服务。

##### 1. 通过应用的 logs 只能看到 driver 的日志。

![](https://github.com/superzhc/SuperzResource/blob/main/images/2022-4-1%2010-43-29/c81c18e6-24b3-456d-8240-c52d5cdcfff9.png?raw=true)

Application

![](https://github.com/superzhc/SuperzResource/blob/main/images/2022-4-1%2010-43-29/00b03059-160e-4e15-a099-8da4f86864e5.png?raw=true)

spark driver log

##### 2.executor 日志在哪里？

根据[Flink On Yarn 如何查看任务日志](https://www.jianshu.com/p/95f4970dfd63)，我们已经知道了日志的 url 组成方式，这次同理，只要找到`容器名`和`node`就能访问日志了。  
driver 的 url 为：`http://node5:19888/jobhistory/logs/node3:8041/container_1634207619484_0496_01_000001/container_1634207619484_0496_01_000001/root/stderr/?start=0`  
搜索 driver 的日志，找到容器名`container_1634207619484_0496_01_000002`和 host`node3`  

![](https://github.com/superzhc/SuperzResource/blob/main/images/2022-4-1%2010-43-29/b9f5bdb9-607b-498a-9bc8-0f27c9d5c3b7.png?raw=true)

spark executor container

所以，最终我们得到 executor 的 url 为：`http://node5:19888/jobhistory/logs/node3:8041/container_1634207619484_0496_01_000002/container_1634207619484_0496_01_000002/root`  

![](https://github.com/superzhc/SuperzResource/blob/main/images/2022-4-1%2010-43-29/4b89a89f-d1c5-4416-bda2-31c23d5a8267.png?raw=true)

spark executor log

### 总结

运行中的 flink/spark 的日志查看非常容易，因为它们本身都提供了 web ui 服务。但是当任务异常退出之后，flink/spark 进程的结束导致无法提供 web ui 服务。我们利用 job history server 来保留和展示当时的日志。但是 yarn 的 web 只展示了 flink job manager/spark driver 的日志链接，我们需要自己拼接 flink task manager/spark executor 日志链接。

最后我有一个小疑问：文中介绍的 URL 组成是推测出来的，其中第三部分`/container_1634207619484_0505_01_000001/container_1634207619484_0505_01_000001`是两个同样的容器名，这是为什么？希望知道的小伙伴能留言解惑一下。

相关链接：  
[Flink On Yarn 如何查看任务日志](https://www.jianshu.com/p/95f4970dfd63)  
[Spark On Yarn 如何查看任务日志](https://www.jianshu.com/p/c5400b52bb3c) 
 [https://www.jianshu.com/p/c5400b52bb3c](https://www.jianshu.com/p/c5400b52bb3c) 
 [https://www.jianshu.com/p/c5400b52bb3c](https://www.jianshu.com/p/c5400b52bb3c)
