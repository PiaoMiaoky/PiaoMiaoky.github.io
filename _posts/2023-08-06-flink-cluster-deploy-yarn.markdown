---
layout:     post
title:      "Flink集群部署-yarn集群模式"
date:       2023-08-07 10:14:00
author:     "kevinkang"
header-img: "img/post-bg-miui6.jpg"
tags:
    - Flink
    - Flink部署与应用
---
## Yarn集群架构
<br>![img](/img/in-post/post-flink/img_27.png)
- ResourceManager (NM):
  - 负责处理客户端请求
  - 监控NodeManager
  - 启动和监控ApplicationMaster
  - 资源的分配和调度
- NodeManager:
  - 管理单个Worker节点上的资源
  - 处理来自ResourceManager的命令
  - 处理来自ApplicationMaster的命令
  - 汇报资源状态
- ApplicationMaster:
  - 负责数据的切分
  - 为应用申请计算资源，并分配给Task
  - 任务的监控与容错
  - 运行在Worker节点上
- Container:
  - 资源抽象，封装了节点上的多维度资源，如CPU，内存，网络资源等

### Flink On Yarn集群部署 - Session模式
<br>![img](/img/in-post/post-flink/img_28.png)
1. 多JobManager共享Dispatcher和YarnResourceManager
2. 支持Native模式，TM动态申请

### Flink On Yarn集群部署 - Per-Job模式
<br>![img](/img/in-post/post-flink/img_29.png)
1. 单个JobManager独享Dispatcher和YarnResourceManager
2. ApplicationMaster与Flink Master节点处在同一个Container

### Flink On Yarn 优势与劣势
- 主要优势:
  - 数据平台无缝对接(Hadoop 2.4+)
  - 部署集群与任务提交都非常简单
  - 资源管理统一通过Yarn管理，提升整体资源利用率
  - 基于Native方式，TaskManager资源按需申请和启动，防止资源浪费
  - 容错保证借助于Hadoop Yarn提供的自动failover机制，能保证JobManager，TaskManager节点异常恢复

- 主要劣势:
  - 尤其是网络资源的隔离，Yarn 做的还不够完善
  - 离线和实时作业同时运行相互干扰等问题需要重视
  - Kerberos 认证超期问题导致的 Checkpoint 无法持久化

### Flink On Yarn 部署步骤要点
1. 下载安装包，并解压到单台节点指定路径
2. Hadoop Yarn 版本需要在2.4.1以上，并具备HDFS文件系统
3. 该节点需要配置HADOOP_CONF_DIR环境变量，并指向Hadoop客户端配置路径
4. 下载Hadoop依赖包配置
5. Flink 1.11版本后将不再在主版本中支持Flink-shaded-hadoop-2-uber包，用户需要自己指定HADOOP_CLASSPATH，或者继续到以下地址下载Hadoop依赖包[下载地址](https://flink.apache.org/downloads.html#additional-components)

#### Flink On Yarn 启动命令
<br>![img](/img/in-post/post-flink/img_30.png)