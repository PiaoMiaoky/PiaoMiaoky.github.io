---
layout:     post
title:      "Flink适用场景介绍"
date:       2023-06-27 01:51:00
author:     "kevinkang"
header-img: "img/post-bg-miui6.jpg"
tags:
    - Flink
    - Apache Flink介绍
---
## Apache Flink介绍
### 大数据处理计算模式
1. 批量计算-(batch)
   1. MapReduce
   2. Apache Spark
   3. Hive
   4. Flink
   5. Pig
2. 流式计算(stream)
   1. Storm
   2. Spark Streaming
   3. Apache Flink
   4. Samza
3. 图计算(graph)
   1. Giraph（Facebook）
   2. Graphx（Spark）
   3. Gelly (Flink)
4. 交互计算(interactive)
   1. Presto
   2. Impala
   3. Druid
   4. Drill

### 流计算与批计算对比
1. 数据实效性
2. 数据特征
3. 应用场景
4. 运行方式

### 流式计算将成为主流
1. 数据处理时延要求越来越高
2. 流式处理计算日趋成熟
3. 批计算带来的计算和存储成本
4. 批计算本身就是一种特殊的流计算，批和流本身就是相辅相成的

### 使用流计算的场景
1. 实时监控：
   1. 用户行为预警
2. 实时报表
   1. 双11活动直播大屏
   2. 对外数据产品-生意参谋
   3. 数据化运营
3. 流数据分析
   1. 实时计算相关指标反馈及时调整决策
   2. 内容投放
4. 实时数据仓库
   1. 数据实时清洗、归并、结构化
   2. 数仓的补充和优化


### 流计算框架和产品
1. 第一类-商业级流计算平台
2. 开源流式计算框架

### 为什么是Flink
![img](/img/in-post/post-flink/steam-compute-framework-diff-img.png)
1. 低延迟-毫秒级延迟
2. 高吞吐-每秒千万级吞吐
3. 准确性-Exactly-once语义
4. 易用性-SQL/Table Api/DataStream Api

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



