---
layout:     post
title:      "Flink中时间概念"
date:       2023-09-19 23:39:00
author:     "kevinkang"
header-img: "img/post-bg-miui6.jpg"
tags:
    - Flink
    - eventTime
    - processingTime
---
# 不同的时间概念
<br>![img](/img/in-post/post-flink/img_60.png)
- eventTime-事件时间:
    - 与事件发生的具体时间相关
- processingTime-处理时间:
    - 与具体发生的事件无关，仅关注何时处理
    
## Flink 基于processingTime处理数据
<br>![img](/img/in-post/post-flink/img_61.png)
- 特点：
  - 基于机器本地时间进行处理
  - 处理结果不固定，可能产生不一致结果
  - 统计结果不能复现

## Flink 基于eventTime处理数据
<br>![img](/img/in-post/post-flink/img_62.png)
- 特点：
  - 基于事件发生时间进行处理
  - 统计结果可复现，有保障
  - 需要一些额外配置，如对于事件时间的抽取
  - 在乱序的情况下，如何达到数据一致性保障

# 不同时间类型
<br>![img](/img/in-post/post-flink/img_63.png)
- 共分为四种时间类型
  - 事件事件：eventTime，在数据中有记录，后面操作进行提取
  - 处理时间：window processingTime

## 设定 Stream 中 TimeCharacteristic
```java
StreamExecutionEnvironment env =
        StreamExecutionEnvironment.getExecutionEnvironment();
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
// alternatively: 
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime); 
// env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime);
```