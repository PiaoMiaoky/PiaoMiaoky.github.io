---
layout:     post
title:      "Flink-窗口计算"
date:       2023-09-24 23:39:00
author:     "kevinkang"
header-img: "img/post-bg-miui6.jpg"
tags:
    - Flink
    - 窗口计算
    - Window Assigner
    - Window Trigger
    - Window Evictor
---
# 为什么需要窗口计算
> 无界数据集统计，通过一个全局窗口统计，是不现实的
>
> 只有通过一些窗口的范围, 去签订出来一些有界的数据集, 基于这些有界数据集, 去统计出来一些结果, 最终汇总到我们的系统, 这其实是解决无界数据集统计的一种方式

- 每5分钟统计一次，统计当前5分钟以内数据中的最大值，出现次数以及 Sum 值
<br>![img](/img/in-post/post-flink/img_73.png)
- 【窗口计算】: 
  - 对无界数据集进行有界处理的过程
  - 并通过对窗口上统计, 产生对应输出的结果
<br></br>
- 每3个 event 统计一次，统计3个 event 中数字出现最大值？最小值？Sum 值？
<br>![img](/img/in-post/post-flink/img_74.png)
    
## window 应用场景
<br>![img](/img/in-post/post-flink/img_75.png)
- 聚合统计: 对数据进行聚合操作(1分钟、5分钟聚合操作等)，写入到外围数据库中
- 记录合并: 对多个kafka的数据源在一定时间窗口内，进行数据合并(如一些用户行为数据，进行合并，减小下游及es写入压力)，写入到es中
- 双流join: 两条流在窗口上面进行join, 数据量会降低在窗口范围内, 聚合后数据写入到kafka里面去
- Watermark 本身也属于特殊的事件；

## window 抽象概念
<br>![img](/img/in-post/post-flink/img_76.png)
- flink中窗口会抽象成不同的概念
- 数据从dataStream接进来的时候, 会去抽取它的 timeStamp 和 waterMark
  - 我们可以看到, 对timeStamp 的获取
- 对 timeStamp 进行keyBy的操作,生成 keyedSteam 和 DataStream
    - keyedSteam: 把key提取出来, 分成不同的分区, 写入不同的分区
- 对 keyedSteam 进行一个 window 操作, 生成一个 windowedStream
- windowAssigner: 根据我们输入的数据集(数据记录), 将数据记录划分成不同的窗口
  - 控制窗口的类型, 时间类型为窗口、SlidingWindow 滑动窗口、滚动窗口、Session 窗口
- Trigger(可选组件): 控制窗口何时触发
  - 根据不同的窗口类型去选择相应的 window 触发的策略
- Evictor(可选组件): 数据剔除器, 窗口函数计算之前、计算之后, 对满足条件的一些数据进行相应过滤操作
  - 如需要将符合条件的数据，写入到我们的window Function里面, 通过 Evictor 控制, 剔除不需要数据
- window Function(核心组件): 窗口函数, 主要用于对窗口内的数据做计算
  - 包括需要对窗口数据，怎样生成对应的统计结果, 那么所有的统计策略, 以及统计的方法, 都是在 window Function 进行定义
- SideOutput: 与window Function相连, 对数据的输出
  - 可以通过SideOutput Tag去控制数据如何输出到外围, 下游的 DataStream 里面去

## window 编程接口
<br>![img](/img/in-post/post-flink/img_77.png)

# Window 组件介绍
## Window Assigner
- Flink 窗口的骨架结构中有两个必须的两个操作:
  - 使用窗口分配器（WindowAssigner）将数据流中的元素分配到对应的窗口。
  - 当满足窗口触发条件后，对窗口内的数据使用窗口处理函数（Window Function）进行处理，常
    用的 Window Function 有 reduce、aggregate、process。
<br>![img](/img/in-post/post-flink/img_78.png)

## Flink支持的窗口类型
<br>![img](/img/in-post/post-flink/img_79.png)

### Sliding Window（滑动窗口）
<br>![img](/img/in-post/post-flink/img_80.png)
- 滑动窗口以一个步长（Slide）不断向前滑动, 窗口的长度固定
  - Window Size：窗口大小
  - Window Slide：滑动间隔
- 数据可以被重复计算，取决于 Size 和 Slide Time
  - Slide Time < Window Size 数据多个窗口中统计
  - Slide Time > Window Size 数据可能不再任何一个 Window中
- 应用非常广泛
  - 每隔5 min 统计前10 min 的总数

### Tumbliing Window（滚动窗口）
<br>![img](/img/in-post/post-flink/img_81.png)
- 滚动窗口下窗口之间之间不重叠, 且窗口长度是固定的
  - 特殊的滑动窗口
  - Window size = Window Slide
  - 应用实例：
    - 每5分钟统计一次用户总的成交额

### Session Window
<br>![img](/img/in-post/post-flink/img_82.png)
- 根据 Session gap 切分不同的窗口
- 当一个窗口在大于 Session gap 的时间内没有接收到新数据时，窗口关闭。 
- Window Size 可变

### Global Window
<br>![img](/img/in-post/post-flink/img_83.png)
- 前面不同类型窗口的基础, 通过添加不同的Trigger实现不同类型窗口
- 只有一个窗口, 没有Trigger触发器
- 这个窗口的触发操作, 由用户自己去指定, 窗口如何切分(由用户指定)

## Flink Stream API 内置窗口
### Predefined Keyed Windows
```java
// Tumbling time window
keyedStream.timeWindow(Time.minutes(1))

// Sliding time window
keyedStream.timeWindow(Time.minutes(1), Time.seconds(10))

// Tumbling count window
keyedStream.countWindow(100)
        
// Sliding count window
keyedStream.countWindow(100, 10)

// Session window
keyedStream.window(EventTimeSessionWindows.withGap(Time. seconds(3))
```

#### Predefined Keyed Windows 实例
```java
DataStream<T> input = ...;

// sliding event-time windows
input.keyBy(<key selector>)
        .window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))
        .<windowed transformation>(<window function>);

// sliding processing-time windows
input.keyBy(<key selector>)
        .window(SlidingProcessingTimeWindows.of(Time.seconds(10), Time.seconds(5)))
        .<windowed transformation>(<window function>);

// sliding processing-time windows offset by -8 hours, 涉及到时区
input.keyBy(<key selector>)
        .window(SlidingProcessingTimeWindows.of(Time.hours(12), Time.hours(1), Time.hours(-8)))
        .<windowed transformation>(<window function>);
```

### Predefined Non-keyed Windows
```java
stream.windowAll(…)…
        
stream.timeWindowAll(Time.seconds(10))…
        
stream.countWindowAll(20, 10)…
```

## Window Trigger
- 触发器（Trigger）决定了何时启动 Window Function 来处理窗口中的数据以及何时将窗口内的数据清理。
  <br>![img](/img/in-post/post-flink/img_84.png)

### Window Trigger 触发机制
<br>![img](/img/in-post/post-flink/img_85.png)
<br>![img](/img/in-post/post-flink/img_86.png)
- 整体介绍
  - 左边为接入的数据, 从12:00～12:10数据,对应waterMark为2分钟
  - 这时候我们会通过一个窗口分配器(window assigner)分配对应的窗口, 调用tumblingEventTimeWindows, 指定窗口大小为5分钟
  - 12:00第一条数据进入窗口,窗口有一个状态的维护,也就是说我们接入的数据,会先维系在窗口的状态里面
    - 基于窗口之上,进行状态的统计
    - 在窗口之上,有一个 EventTimeTrigger,对于其来讲,会去控制我们窗口window Function的计算, 以及结果的输出, 包括window Result的输出
- 数据流程介绍
  - 当第一条数据进来之后,waterMark的时间是11:58 (12:00 - 2min)
  - 当第二条数据进来后, 窗口同样也会将接入的数据进行维护, 我们在窗口统计过程中, 窗口内会进行累加的计算, 累加1和2, 产生状态的统计结果。且waterMark更新为 12：02 对于EventTimeTrigger还是没有触发窗口结果的输出
  - 12:03第三条数据接入, 在窗口内对状态进行存储,这时我们看到, 窗口计算12:00~12:05,状态累计为6,且当前时间未超过上一个事件事件,waterMark不需要进行更新, EventTimeTrigger还是没有触发窗口结果的输出
  - 12:08第四条数据接入, 会把waterMark的事件更新到12:06, 此时 > 12:05 (窗口结束事件), 也就是说我们12:00~12:05这个窗口的数据, 可以进行相应的统计操作, 即 EventTimeTrigger 会对窗口结果输出到 window Result里面。外面的算子就可以接收到12:00~12:05的统计结果6

<br>![img](/img/in-post/post-flink/img_87.png)
- 整体介绍
  - 如果想让窗口每隔两分钟就触发一次结果输出呢？而不是依赖 waterMark > window endTime来控制结果输出
  - 这时候可以选择使用 ContinuousEventTimeTrigger, 需要指定一个事件间隔,设置为2分钟, 即eventTime每隔2分钟就会触发一次窗口统计
- 数据流程介绍
  - 比如说对于第二条数据接入后, 12:04-12:00 = 4min > 2min, 对于 ContinuousEventTimeTrigger就会触发一次窗口的输出, 对于12:00~12:05的结果就会进行一次output
  - 第三条数据12:03接入进来后, 此时waterMark无需更新, 且事件时间间隔 < 2min, 不会触发窗口的计算
  - 第三条数据12:08接入进来后, 此时waterMark更新到12:06, 且12:08-12:04 = 4min > 2min, 对于
    - 【update mode】ContinuousEventTimeTrigger就会触发一次窗口的输出, 此时是一种 update mode,会把前面的输出结果进行一次更新 结果为 1+2+3 = 6 对于第二个窗口12:05~12:10, 此时也满足, 也会进行窗口的输出
    - <br>![img](/img/in-post/post-flink/img_88.png)
    - <br>![img](/img/in-post/post-flink/img_89.png)
    - 【append mode】对于在append mode下, 每一次ContinuousEventTimeTrigger触发了窗口结果的统计输出时, 是一种append模式, window result中的状态, 是不断在append
      - 在第一次触发之后产生的结果、第二次触发产生的结果, 都会append到window下游的算子中, 如果此时下游不清楚产生的结果, 就可能会把结果进行累加, 即(3+6), 但其实这并不是我们想要的结果, 导致下游的算子统计上就会有一些问题
        - append mode问题处理：flink中提出了puringTrigger的接口, 可以在 ContinuousEventTimeTrigger 之上进行一次封装,达到的一个效果是, 可以对我们窗口里面的状态进行清除。
        - 即窗口触发完成后, 会对window状态进行清理, 从而保证下游接收到的window Result可以有一个正确的结果
        - 在12:03以及12:08的两台数据来说,此时12:00~12:05两次都是3,因为对状态进行了清理, 保证下游累加正确性

### Window Trigger 总结
- 通常情况下WindowAssigner 中已经默认包含相应的Window Trigger，且适用于大部分场景；
- 所有基于Event Time 时间的 WindowAssigner 默认Trigger 都是 EventTimeTrigger；
- GlobalWindow默认为 NeverTrigger，也就是不触发，因此使用 GlobalWindow 必须指定 CustomTrigger，否则窗口永远不触发；
- 通过 WindowStream 提供的 trigger() 方法能够接受新的 Trigger 实现，但是会覆盖WindowAssigner 自带的默认 Triigger；


## Window Evictor
- 数据清除器（Evictor）
- 可选组件
  <br>![img](/img/in-post/post-flink/img_90.png)
- processing Function 的前后, 都可以使用Window Trigger

### Window Evictor 多种介绍
<br>![img](/img/in-post/post-flink/img_91.png)

### Window Evictor 使用
```java
windowStream.evictor(CountEvictor.of(5))

windowStream.evictor(TimeEvictor.of(Time.of(5, TimeUnit.SECONDS)))

windowStream.evictor(DeltaEvictor.of(threshold, new DeltaFunction<Tuple2<String,
Integer>>() {
@Override
public double getDelta(Tuple2<String, Integer> oldDataPoint, Tuple2<String,
Integer> newDataPoint) {
return newDataPoint.f1 - oldDataPoint.f1;
}}, evictAfter),0,null))
```

### Window Evictor 接口定义
```java
public interface Evictor<T, W extends Window> extends Serializable {
  /**
   * Optionally evicts elements. Called before windowing function.
   */
  void evictBefore(Iterable<TimestampedValue<T>> elements, int size, W window, EvictorContext evictorContext);
  /**
   * Optionally evicts elements. Called after windowing function.
   */
  void evictAfter(Iterable<TimesstampedValue<T>> elements, int size, W window, EvictorContext evictorContext);
  /**
   * A context object that is given to {@link Evictor} methods.
   */
  interface EvictorContext {
    /**
     * Returns the current processing time.
     */
    long getCurrentProcessingTime();
    MetricGroup getMetricGroup();
    /**
     * Returns the current watermark time.
     */
    long getCurrentWatermark();
  } 
}
```