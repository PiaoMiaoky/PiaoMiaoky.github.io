---
layout:     post
title:      "流式计算以及Flink技术的关键点"
date:       2023-06-27 01:51:00
author:     "kevinkang"
header-img: "img/post-bg-miui6.jpg"
tags:
    - 知乎
    - MIUI
    - Android
---
## Apache Flink核心特性
1. 统一数据处理组件栈,处理不同类型的数据需求(batch,stream,Machine Learning,graph)
<br>![img](/img/in-post/post-flink/img_2.png)
2. 支持事件实践,接入时间,处理时间等时间概念
<br>![img](/img/in-post/post-flink/img.png)
3. 基于轻量级分布式快照(snapshot)实现容错
<br>![img](/img/in-post/post-flink/img_1.png)
4. 支持状态计算、灵活的state-backend(HDFS,内存,RocksDB)
<br>![img](/img/in-post/post-flink/img_3.png)
5. 支持高度灵活的窗口(Window)操作
<br>![img](/img/in-post/post-flink/img_4.png)
6. 带反压的连续流模型
<br>![img](/img/in-post/post-flink/img_5.png)
7. 基于JVM实现独立的内存管理
<br>![img](/img/in-post/post-flink/img_6.png)


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



