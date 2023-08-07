---
layout:     post
title:      "Flink的核心特性、集群架构"
date:       2023-06-27 01:51:00
author:     "kevinkang"
header-img: "img/post-bg-miui6.jpg"
tags:
    - Flink
    - 流式计算
    - Flink部署与应用

---

## Apache Flink集群架构、部署模式
1. 集群架构组成
<br>![img](/img/in-post/post-flink/img_7.png)
   1. JobManager:管理节点，每个集群至少有一个,管理整个集群计算资源,Job管理与调用执行,以及Checkpoint协调
   2. TaskManager:每个集群有多个TM,负责计算资源提供
   3. Client:本地执行应用main()方法解析JobGraph对象,并最终将JobGraph提交到JobManager运行,同时监控Job执行状态

### JobManager
<br>![img](/img/in-post/post-flink/img_8.png)
1. checkpoint Coordinator-安全点检查
2. JobGraph -> Execution Graph
   客户端提交的JobGraph是逻辑描述，会转化为物理描述，提交task
3. Task部署与调度
4. RPC通信(Actor System)
5. Job接收(Job Dispatch)
6. 集群资源管理(resourceManager)
7. TaskManager注册与管理

### TaskManager
<br>![img](/img/in-post/post-flink/img_9.png)
1. Task Execution
2. Network Manager
   1. TaskManager节点间数据交互，基于netty实现网络通信，netstat网络通信站
3. Shuffle Environment管理
   1. 数据进行groupBy&分组时,数据在不同taskManager会进行交互
4. Rpc通信(Actor System)
   1. 节点与节点间通信
5. heartbeat with JobManager And RM
6. Data Exchange
7. Memory Management
   1. 进行序列化与反序列化使用。任务提交过来后，会接受外围传输进来的一些数据，这些数据会在Memory Management申请存储内存单元
8. Register To RM
   1. TaskManager启动后，向JobManager进行注册
9. Offer Slots To JobManager
   1. 当task提交后，并且在TaskSlot申请资源完成后

### Client
<br>![img](/img/in-post/post-flink/img_10.png)
1. Application的main()方法执行
2. JobGraph Generate
3. Execution Environment管理
4. Job提交与执行
   1. 提交JobGraph
5. Dependency Jar ship
   1. 把jobGraph涉及到的依赖包通过RPC ship到JobManager上
6. RPC with JobManager
7. 集群部署(Cluster deploy)

### JobGraph(底层为有向无环图)
<br>![img](/img/in-post/post-flink/img_11.png)
<br>streamGraph 到 JobGraph转换过程
<br>![img](/img/in-post/post-flink/img_12.png)
1. 通过有向无环图(Dag)方式表达用户程序
2. 不同接口程序的抽象表达
   1. DataStream Api
   2. DataSet Api
   3. Flink SQL
   4. Table Api
3. 客户端和集群之间的Job描述载体
4. 节点(Vertices),result参数
5. Flink 1.11之前只能在Client中生成

## 集群部署模式
### 集群部署模式对比
<div>
   <blockquote>根据以下两种条件将集群部署模式分为三种类型
   </blockquote>
</div>

1. 集群的生命周期和资源隔离
2. 根据程序main()方法执行在Client还是JobManager

#### Session Mode
- 共享JobManager和TaskManager,所有提交的Job都在一个Runtime中运行
#### Per-Job Mode
- 独享JobManager与TaskManager,好比为每个Job单独启动一个Runtime
#### Application Mode(1.11版本提出)
- Application的main()运行在Cluster上,而不在客户端
- 每个Application对应一个Runtime,Application中可以包含多个Job

### Session集群运行模式
<br>![img](/img/in-post/post-flink/img_13.png)
- Session集群运行模式:
  - JobManager与TaskManager共享
  - 客户端通过RPC或者Rest Api连接集群的管理节点(master JobManager)
  - Deployer 需要上传依赖的 Dependences jar
  - Deployer 需要生成JobGraph,并提交到管理节点
  - JobManager的生命周期不受提交的Job影响,会长期运行
- Session运行模式优点
  - 资源充分共享,提升资源利用率
  - Job在Flink Session集群中管理,运维简单
- Session运行模式缺点
  - 资源隔离相对较差
  - 非Native类型部署,TaskManager不易扩展,Slot计算资源伸缩性差

### Per-Job运行模式
<br>![img](/img/in-post/post-flink/img_14.png)
- Per-Job运行模式:
   - 单个Job独享JobManager与TaskManager
   - TaskManager中的Slot资源根据Job指定
   - Deployer 需要上传依赖的 Dependences jar
   - 客户端Deployer 生成JobGraph,并提交到管理节点
   - JobManager的生命周期和Job生命周期绑定
- Per-Job运行模式优点
   - Job与Job之间资源隔离充分
   - 资源根据Job需要进行申请,TaskManager Slots数量可以不同
- Per-Job运行模式缺点
   - 资源相对比较浪费,JobManager需要消耗资源
   - Job管理完全交给ClusterManagement,管理复杂

#### Session集群与Per-Job类型集群问题
为什么不直接把这些工作交给JobManager？从而减轻Client压力
<br>![img](/img/in-post/post-flink/img_15.png)

### Application Mode集群运行模式
<br>![img](/img/in-post/post-flink/img_16.png)
- Application Mode类型集群模式(1.11版本):
   - 每个Application对应一个JobManager,且一个Application可以运行多个Job
   - 客户端无需将Dependencies 上传到JobManager,仅负责管理Job的提交和管理
   - main()方法运行在JobManager中,将JobGraph的生成放在集群上运行,客户端压力降低
- Application Mode类型集群模式优点
   - 有效降低带宽消耗和客户端负载
   - Application间实现资源隔离,Application中实现资源共享
- Application Mode类型集群模式缺点
   - 功能太新,未经过生产验证
   - 仅支持yarn和Kubernetes 

## 集群部署-Cluster Management支持
<br>![img](/img/in-post/post-flink/img_17.png)
<div>
   <blockquote>Flink支持以下资源管理器部署集群
   </blockquote>
</div>

1. standalone
2. hadoop yarn
3. Apache Mesos
4. Docker
5. Kubernetes

### Flink集群部署对比
<br>![img](/img/in-post/post-flink/img_18.png)

### Native集群部署
1. 当在ClusterManagement上启动Session集群时,只启动JobManager实例,不启动TaskManager
<br>![img](/img/in-post/post-flink/img_19.png)
2. 当提交Job-1后根据Job的资源申请,动态启动TaskManager满足计算需求
<br>![img](/img/in-post/post-flink/img_20.png)
3. 继续提交Job-2、Job-3时后,再次向ClusterManagement中申请TaskManager资源
<br>![img](/img/in-post/post-flink/img_21.png)

总结:
<br>![img](/img/in-post/post-flink/img_22.png)
- Session集群根据实际提交的Job,动态申请和启动TaskManager计算资源
- 支持Native部署模式的有yarn,kubernetes,Mesos资源管理器
- standalone不支持Native部署