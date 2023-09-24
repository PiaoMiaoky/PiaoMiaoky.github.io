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

# Watermark 的生成方式
- Two Styles of Watermark Generation
  - Periodic Watermarks: Based on Event Time
    - 最大的 event time - 固定时间延迟，产生waterMark
  - Punctuated Watermarks: Based on something in the event stream
    - 事件流中通过固定的信息，生成waterMark

## Timestamp Assign 与 Watermark Generate
```java
// @Deprecated from 1.11
public SingleOutputStreamOperator<T> assignTimestampsAndWatermarks(AssignerWithPunctuatedWatermarks<T> timestampAndWatermarkAssigner)
public SingleOutputStreamOperator<T> assignTimestampsAndWatermarks(AssignerWithPeriodicWatermarks<T> timestampAndWatermarkAssigner)

// Add From 1.11
public SingleOutputStreamOperator<T> assignTimestampsAndWatermarks(WatermarkStrategy<T> watermarkStrategy)
```
## flink 1.11 版本之前
```java
// @Deprecated from 1.11
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
DataStream<MyEvent> stream = env.readFile(
        myFormat, myFilePath, FileProcessingMode.PROCESS_CONTINUOUSLY, 100,
        FilePathFilter.createDefaultFilter(), typeInfo);

// Timestamp与waterMark设定
DataStream<MyEvent> withTimestampsAndWatermarks = stream
        .filter( event -> event.severity() == WARNING )
        .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessGenerator());

withTimestampsAndWatermarks
        .keyBy( (event) -> event.getGroup() )
        .timeWindow(Time.seconds(10))
        .reduce( (a, b) -> a.add(b) )
        .addSink(...);
```

### PeriodicWatermarks 定义
<br>![img](/img/in-post/post-flink/img_71.png)

### PunctuatedWatermarks 定义
<br>![img](/img/in-post/post-flink/img_72.png)
- 通过事件中携带的waterMark的标签确定是否生成

### Source Functions with Timestamps and Watermarks
- 直接在Source Functions 中生成event time 和 waterMark，传递到SourceContext，携带到Flink中，不需要在DataStream API中生成
- (大多数情况：通过前面两种方式，在DataStream API中生成)
```java
@Override
public void run(SourceContext<MyType> ctx)throws Exception{
        while(/* condition */){
            MyType next=getNext();
            ctx.collectWithTimestamp(next,next.getEventTimestamp());
            if(next.hasWatermarkTime()){
                ctx.emitWatermark(new Watermark(next.getWatermarkTime()));
            }
        }
}
```

## flink 1.11版本之后 引入Watermark Strategies 介绍
- 希望把1.11版本之前waterMark的两种生成策略进行统一

```java
public interface WatermarkStrategy<T> extends TimestampAssignerSupplier<T>, WatermarkGeneratorSupplier<T>{
    /**
     * Instantiates a {@link TimestampAssigner} for assigning timestamps according to this
     * strategy.
     */
    @Override
    TimestampAssigner<T> createTimestampAssigner(TimestampAssignerSupplier.Context context);
    /**
     * Instantiates a WatermarkGenerator that generates watermarks according to this strategy.
     */
    @Override
    WatermarkGenerator<T> createWatermarkGenerator(WatermarkGeneratorSupplier.Context context);
}
```

### Using Watermark Strategies
```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

DataStream<MyEvent> stream = env.readFile(
        myFormat, myFilePath, FileProcessingMode.PROCESS_CONTINUOUSLY, 100,
        FilePathFilter.createDefaultFilter(), typeInfo);

// timeStamp 与 waterMark 设定
DataStream<MyEvent> withTimestampsAndWatermarks = stream
        .filter( event -> event.severity() == WARNING )
        .assignTimestampsAndWatermarks(WatermarkStrategy
        .<Tuple2<Long, String>>forBoundedOutOfOrderness(Duration.ofSeconds(20))
        .withTimestampAssigner((event, timestamp) -> event.f0));

withTimestampsAndWatermarks
        .keyBy( (event) -> event.getGroup() )
        .timeWindow(Time.seconds(10))
        .reduce( (a, b) -> a.add(b) )
        .addSink(...);
```

### Watermark Strategies and the Kafka Connector
```java
// 在数据源连接器直接指定
FlinkKafkaConsumer<MyType> kafkaSource = new FlinkKafkaConsumer<>("myTopic",
        schema, props);
        kafkaSource.assignTimestampsAndWatermarks(
        WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(20)));
        
DataStream<MyType> stream = env.addSource(kafkaSource);
```

### Writing WatermarkGenerators
```java
public interface WatermarkGenerator<T> {
    /**
     * Called for every event, allows the watermark generator to examine and remember the
     * event timestamps, or to emit a watermark based on the event itself.
     */
    void onEvent(T event, long eventTimestamp, WatermarkOutput output);
    /**
     * Called periodically(周期), and might emit a new watermark, or not.
     *
     * <p>The interval in which this method is called and Watermarks are generated
     * depends on {@link ExecutionConfig#getAutoWatermarkInterval()}.
     */
    void onPeriodicEmit(WatermarkOutput output);
}
```

### Periodic WatermarkGenerator
```java
public class BoundedOutOfOrdernessGenerator implements WatermarkGenerator<MyEvent> {
    private final long maxOutOfOrderness = 3500; // 3.5 seconds
    private long currentMaxTimestamp;
    @Override
    public void onEvent(MyEvent event, long eventTimestamp, WatermarkOutput output) {
        currentMaxTimestamp = Math.max(currentMaxTimestamp, eventTimestamp);
    }
    @Override
    public void onPeriodicEmit(WatermarkOutput output) {
        // emit the watermark as current highest timestamp minus the out-of-orderness bound
        output.emitWatermark(new Watermark(currentMaxTimestamp - maxOutOfOrderness - 1));
    } 
}
```

### Punctuated WatermarkGenerator
```java
public class PunctuatedAssigner implements WatermarkGenerator<MyEvent> {
    @Override
    public void onEvent(MyEvent event, long eventTimestamp, WatermarkOutput output) {
        if (event.hasWatermarkMarker()) {
            output.emitWatermark(new Watermark(event.getWatermarkTimestamp()));
        } }
    @Override
    public void onPeriodicEmit(WatermarkOutput output) {
    // don't need to do anything because we emit in reaction to events above
    } 
}
```

# Watermark 总结
- (Un)comfortably bounded by fixed delay (固定延迟的边界)
  - too slow: results are delayed 
  - too fast: some data is late
- Heuristic(启发式)
  - allow windows to produce results as soon as meaningfully possible, and then continue
    with updates during the allowed lateness interval(允许windows尽快产生有意义的结果，然后继续 在允许的延迟间隔内进行更新)
