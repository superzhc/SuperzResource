# hadoop之yarn详解（命令篇） - 一寸HUI - 博客园
* * *

本篇主要对 yarn 命令进行阐述

## 一、yarn 命令概述

\[root@lgh ~]# yarn -help 
Usage: yarn \[--config confdir] COMMAND where COMMAND is one of:
  resourcemanager -format-state-store   deletes the RMStateStore
  resourcemanager                       run the ResourceManager
                                        Use -format-state-store for deleting the RMStateStore.
                                        Use -remove-application-from-state-store <appId> for removing application from RMStateStore.
  nodemanager                           run a nodemanager on each slave
  timelineserver                        run the timeline server
  rmadmin                               admin tools
  version                               print the version
  jar <jar> run a jar file
  application                           prints application(s)
                                        report/kill application
  applicationattempt                    prints applicationattempt(s)
                                        report
  container                             prints container(s) report
  node                                  prints node report(s)
  queue                                 prints queue information
  logs                                  dump container logs
  classpath                             prints the class path needed to get the Hadoop jar and the
                                        required libraries
  daemonlog get/set the log level for each
                                        daemon
  top                                   run cluster usage tool
 or
  CLASSNAME                             run the class named CLASSNAME

使用语法：

yarn \[--config confdir] COMMAND \[--loglevel loglevel] \[GENERIC_OPTIONS] \[COMMAND_OPTIONS]

\--config confdir        #覆盖默认的配置目录，默认为 ${HADOOP_PREFIX}/conf. --loglevel loglevel     #覆盖日志级别。有效的日志级别为 FATAL，ERROR，WARN，INFO，DEBUG 和 TRACE。默认值为 INFO。
GENERIC_OPTIONS         #多个命令支持的一组通用选项
COMMAND COMMAND_OPTIONS #以下各节介绍了各种命令及其选项

## 二、命令详解

### 2.1、application

使用语法：yarn  application \[options] #打印报告，申请和杀死任务

\-appStates <States>         #与 - list 一起使用，可根据输入的逗号分隔的应用程序状态列表来过滤应用程序。有效的应用程序状态可以是以下之一：ALL，NEW，NEW_SAVING，SUBMITTED，ACCEPTED，RUNNING，FINISHED，FAILED，KILLED -appTypes <Types>           #与 - list 一起使用，可以根据输入的逗号分隔的应用程序类型列表来过滤应用程序。 -list                       #列出 RM 中的应用程序。支持使用 - appTypes 来根据应用程序类型过滤应用程序，并支持使用 - appStates 来根据应用程序状态过滤应用程序。 -kill <ApplicationId> #终止应用程序。 -status <ApplicationId>     #打印应用程序的状态。

### 2.2、applicationattempt

使用语法：yarn applicationattempt \[options] #打印应用程序尝试的报告

\-help                    #帮助 -list <ApplicationId>    #获取到应用程序尝试的列表，其返回值 ApplicationAttempt-Id 等于 <Application Attempt Id>
\-status <Application Attempt Id>    #打印应用程序尝试的状态。

### 2.3、classpath

使用语法：yarn classpath #打印需要得到 Hadoop 的 jar 和所需要的 lib 包路径

### 2.4、container

使用语法：yarn container \[options] #打印 container(s) 的报告

\-help                            #帮助 -list <Application Attempt Id> #应用程序尝试的 Containers 列表 -status <ContainerId>            #打印 Container 的状态

### 2.5、jar

使用语法：yarn jar <jar> \[mainClass] args... #运行 jar 文件，用户可以将写好的 YARN 代码打包成 jar 文件，用这个命令去运行它。

### 2.6、logs

使用语法：yarn logs -applicationId <application ID> \[options] #转存 container 的日志。

\-applicationId <application ID> #指定应用程序 ID，应用程序的 ID 可以在 yarn.resourcemanager.webapp.address 配置的路径查看（即：ID） -appOwner <AppOwner> #应用的所有者（如果没有指定就是当前用户）应用程序的 ID 可以在 yarn.resourcemanager.webapp.address 配置的路径查看（即：User） -containerId <ContainerId> #Container Id -help                              #帮助 -nodeAddress <NodeAddress>         #节点地址的格式：nodename:port （端口是配置文件中: yarn.nodemanager.webapp.address 参数指定）

### 2.7、node

使用语法：yarn node \[options] #打印节点报告

\-all             #所有的节点，不管是什么状态的。 -list             #列出所有 RUNNING 状态的节点。支持 - states 选项过滤指定的状态，节点的状态包含：NEW，RUNNING，UNHEALTHY，DECOMMISSIONED，LOST，REBOOTED。支持 --all 显示所有的节点。 -states <States> #和 - list 配合使用，用逗号分隔节点状态，只显示这些状态的节点信息。 -status <NodeId> #打印指定节点的状态。

### 2.8、queue

使用语法：yarn queue \[options] #打印队列信息

\-help     #帮助 -status  #<QueueName>    打印队列的状态

### 2.9、daemonlog

使用语法：

yarn daemonlog -getlevel &lt;host:httpport> <classname>  
yarn daemonlog -setlevel &lt;host:httpport> <classname> <level>

\-getlevel &lt;host:httpport> <classname>            #打印运行在 &lt;host:port> 的守护进程的日志级别。这个命令内部会连接 http&#x3A;//&lt;host:port>/logLevel?log=<name>
\-setlevel &lt;host:httpport> <classname> <level>    #设置运行在 &lt;host:port> 的守护进程的日志级别。这个命令内部会连接 http&#x3A;//&lt;host:port>/logLevel?log=<name>

### 2.10、nodemanager

使用语法：yarn nodemanager #启动 nodemanager

### 2.11、proxyserver

使用语法：yarn proxyserver #启动 web proxy server

### 2.12、resourcemanager

使用语法：yarn resourcemanager \[-format-state-store] #启动 ResourceManager

\-format-state-store     # RMStateStore 的格式. 如果过去的应用程序不再需要，则清理 RMStateStore， RMStateStore 仅仅在 ResourceManager 没有运行的时候，才运行 RMStateStore

### 2.13、rmadmin

使用语法： # 运行 Resourcemanager 管理客户端

![](https://common.cnblogs.com/images/copycode.gif)

yarn rmadmin \[-refreshQueues]
              \[-refreshNodes]
              \[-refreshUserToGroupsMapping] 
              \[-refreshSuperUserGroupsConfiguration]
              \[-refreshAdminAcls] 
              \[-refreshServiceAcl]
              \[-getGroups \[username]]
              \[-transitionToActive \[--forceactive] \[--forcemanual] <serviceId>]
              \[-transitionToStandby \[--forcemanual] <serviceId>]
              \[-failover \[--forcefence] \[--forceactive] <serviceId1> <serviceId2>]
              \[-getServiceState <serviceId>]
              \[-checkHealth <serviceId>]
              \[-help \[cmd]]

![](https://common.cnblogs.com/images/copycode.gif)

![](https://common.cnblogs.com/images/copycode.gif)

\-refreshQueues    #重载队列的 ACL，状态和调度器特定的属性，ResourceManager 将重载 mapred-queues 配置文件 -refreshNodes     #动态刷新 dfs.hosts 和 dfs.hosts.exclude 配置，无需重启 NameNode。
                  \#dfs.hosts：列出了允许连入 NameNode 的 datanode 清单（IP 或者机器名）
                  \#dfs.hosts.exclude：列出了禁止连入 NameNode 的 datanode 清单（IP 或者机器名）
                  \#重新读取 hosts 和 exclude 文件，更新允许连到 Namenode 的或那些需要退出或入编的 Datanode 的集合。 -refreshUserToGroupsMappings            #刷新用户到组的映射。 -refreshSuperUserGroupsConfiguration    #刷新用户组的配置 -refreshAdminAcls                       #刷新 ResourceManager 的 ACL 管理 -refreshServiceAcl                      #ResourceManager 重载服务级别的授权文件。 -getGroups \[username]                   #获取指定用户所属的组。 -transitionToActive \[–forceactive] \[–forcemanual] <serviceId> #尝试将目标服务转为 Active 状态。如果使用了–forceactive 选项，不需要核对非 Active 节点。如果采用了自动故障转移，这个命令不能使用。虽然你可以重写–forcemanual 选项，你需要谨慎。 -transitionToStandby \[–forcemanual] <serviceId> #将服务转为 Standby 状态. 如果采用了自动故障转移，这个命令不能使用。虽然你可以重写–forcemanual 选项，你需要谨慎。 -failover \[–forceactive] <serviceId1> <serviceId2>  #启动从 serviceId1 到 serviceId2 的故障转移。如果使用了 - forceactive 选项，即使服务没有准备，也会尝试故障转移到目标服务。如果采用了自动故障转移，这个命令不能使用。 -getServiceState <serviceId> #返回服务的状态。（注：ResourceManager 不是 HA 的时候，时不能运行该命令的） -checkHealth <serviceId> #请求服务器执行健康检查，如果检查失败，RMAdmin 将用一个非零标示退出。（注：ResourceManager 不是 HA 的时候，时不能运行该命令的） -help \[cmd]                                         #显示指定命令的帮助，如果没有指定，则显示命令的帮助。

![](https://common.cnblogs.com/images/copycode.gif)

### 2.14、scmadmin

使用语法：yarn scmadmin \[options]  #运行共享缓存管理客户端

\-help            #查看帮助 -runCleanerTask    #运行清理任务

### 2.15、 sharedcachemanager

使用语法：yarn sharedcachemanager #启动共享缓存管理器

### 2.16、timelineserver

使用语法：yarn timelineserver   # 启动 timelineserver

**更多 hadoop 生态文章见：[ hadoop 生态系列](https://www.cnblogs.com/zsql/p/11560374.html)**

**参考：  
[https://hadoop.apache.org/docs/r2.7.7/hadoop-yarn/hadoop-yarn-site/YarnCommands.html](https://hadoop.apache.org/docs/r2.7.7/hadoop-yarn/hadoop-yarn-site/YarnCommands.html)** 
 [https://www.cnblogs.com/zsql/p/11636348.html#\_label1_12](https://www.cnblogs.com/zsql/p/11636348.html#_label1_12)

 [https://www.cnblogs.com/zsql/p/11636348.html#\_label1_12](https://www.cnblogs.com/zsql/p/11636348.html#_label1_12)
