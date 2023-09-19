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
# 基于eventTime处理数据
- 使用 Event-Time 处理过程中， Flink 系统需要知道
  - 每个 StreamElement 的 EventTime 时间戳
    - 接入的数据何时可以触发统计计算（Watermark）
      - 窗口 12:00 – 12:10 窗口的数据全部被接收完毕
    
# Watermark
<br>![img](/img/in-post/post-flink/img_64.png)
- Watermark 用于标记 Event-Time 的前进过程；
- Watermark 跟随 DataStream Event-Time 变动，并自身携带 TimeStamp； 
- Watermark 用于表明所有较早的事件已经（可能）到达； 
- Watermark 本身也属于特殊的事件；

## 完美的 Watermark
<br>![img](/img/in-post/post-flink/img_65.png)
- 数据流按照顺序的时候，就能够得到完美的 Watermark

## 有界乱序事件下的 Watermark
<br>![img](/img/in-post/post-flink/img_66.png)
- 当 Event 无序时，通常会认为它们有一定的无序性

## 迟到事件
<br>![img](/img/in-post/post-flink/img_67.png)
- Elements where timestamp <= currentWatermark are late
  - 示例中eventTime 19 < 20, 19被标记为迟到事件
  - 在进行窗口计算时，就不会把19纳入到窗口的统计范围，waterMark 20以前的事件，都会被纳入到窗口的统计范围里

## 并行中的waterMark
<br>![img](/img/in-post/post-flink/img_68.png)
- 白色框：事件，由事件id和eventTime组成
- 灰色圆框-map：operator, 每个operator上面会有一个eventTime,其实是一个TimeService,标记出当前算子的时钟
- 灰色圆框-source：数据接入时, 通过对事件时间抽取, 然后在sourceOperator中产生waterMark (后面的事件时间 > 前面的事件时间)
  - waterMark生成完了之后, 会随着StreamElement 通过上游的算子, 发送给下游的算子, 此时会把waterMark认为是一种特殊的事件, 这时候会伴随着整个数据处理流程, 一直发送到下游算子里面, 一直更新到最终算子里面
  - source产生waterMark，下发到下游算子时，会有以下操作：
    - 更新当前算子里面的时间，29是前面事件产生的waterMark所更新的时间
    - 29会随着waterMark进入进行更新
    - w(17)如果已经在map(2)这个operator里执行完成后, 会把operator里面的时间进行一次update, 这时候operator时间被更新为17


# Watermark 与 Window 之间的关系
<br>![img](/img/in-post/post-flink/img_69.png)
- Watermark in Windowed Grouped Aggregation with Append Mode

<br>![img](/img/in-post/post-flink/img_69.png)
- Watermark in Windowed Grouped Aggregation with Update Mode

# Watermark 使用总结
<br>![img](/img/in-post/post-flink/img_70.png)
- Watermark = Max EventTime – Late Threshold
- Late Threshold 越高，数据处理延时越高
- 启发式更新
- 解决一定范围内的乱序事件
- 窗口触发条件：Current Watermark > Window EndTime
- Watermark 的主要目的是告诉窗口不再会有比当前 Watermark 更晚的数据到达
- Idel Watermark 仅会发生在顺序事件中

