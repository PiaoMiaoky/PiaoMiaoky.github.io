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

<!--sec data-title="标题1" data-id="section0" data-show=true ces-->
内容部分1；
<button class="section" target="section1" show="显示下一部分" hide="隐藏下一部分"></button>
<!--endsec-->
<!--sec data-title="标题2" data-id="section1" data-show=true ces-->
## Flink On Yarn部署说明
### 环境要求
- Apache Hadoop 2.4.1及以上
- HDFS (Hadoop Distributed File System) 环境
- Hadoop依赖包

### 环境配置
- 下载和解压安装包（参考Standalone模式）
- 配置HADOOP_CONFIG_DIR环境变量

 ```bash
 $ vim /etc/profile
# 添加如下环境变量信息：
export JAVA_HOME=/usr/java/jdk1.8.0_241
export HADOOP_CONF_DIR=/home/flink-training/hadoop-conf
export HADOOP_CLASSPATH=/opt/cloudera/parcels/CDH/lib/hadoop:/opt/cloudera/parcels/CDH/lib/hadoop-yarn:/opt/cloudera/parcels/CDH/lib/hadoop-hdfs
export PATH=$PATH:$JAVA_HOME/bin
 ```
- 如果HADOOP_CLASSPATH配置后，作业执行还报Hadoop依赖找不到错误，可以到如下地址下载，并放置在lib路径中：
 ```
$ cd lib/
$ wget https://repo.maven.apache.org/maven2/org/apache/flink/flink-shaded-hadoop-2-uber/2.7.5-10.0/flink-shaded-hadoop-2-uber-2.7.5-10.0.jar
```

### 基于Session Mode部署
```bash
./bin/yarn-session.sh -tm 1028 -s 8
```
可配置的运行参数如下：
```bash
Usage:
   Optional
     -D <arg>                        Dynamic properties
     -d,--detached                   Start detached
     -jm,--jobManagerMemory <arg>    Memory for JobManager Container with optional unit (default: MB)
     -nm,--name                      Set a custom name for the application on YARN
     -at,--applicationType           Set a custom application type on YARN
     -q,--query                      Display available YARN resources (memory, cores)
     -qu,--queue <arg>               Specify YARN queue.
     -s,--slots <arg>                Number of slots per TaskManager
     -tm,--taskManagerMemory <arg>   Memory per TaskManager Container with optional unit (default: MB)
     -z,--zookeeperNamespace <arg>   Namespace to create the Zookeeper sub-paths for HA mode
```

### Attach to an existing Session
```bash
$ ./bin/yarn-session.sh -id application_1463870264508_0029
```

### 提交Flink作业到指定Session集群
```bash
#上传测试文件
hdfs dfs -put ./data-set/frostroas.txt /flink-training
# 运行Flink程序
./bin/flink run ./examples/batch/WordCount.jar --input hdfs://node02:8020/flink-training/frostroad.txt --output hdfs://node02:8020/flink-training/wordcount-result.txt
```

### 停止集群服务
方式1：
```bash
echo "stop" | ./bin/yarn-session.sh -id application_1597152309776_0008
```
方式2：Yarn Kill命令
```bash
# 找到作业对应的ApplicationID
$ yarn application list
# 指定Kill命令
$ yarn application kill application_1597152309776_0008
```
### 基于Per-Job Mode部署
直接运行如下命令即可提交作业：
```bash
./bin/flink run -m yarn-cluster ./examples/batch/WordCount.jar
```
Detach 模式：
```bash
./bin/flink run -m yarn-cluster -d ./examples/batch/WordCount.jar
```

### 基于Application Mode部署

#### 1. 通过从本地上传Dependencies和User Application Jar
```bash
./bin/flink run-application -t yarn-application \
    -Djobmanager.memory.process.size=2048m \
    -Dtaskmanager.memory.process.size=4096m \
    ./MyApplication.jar
```
#### 2. 通过从HDFS获取Dependencies和本地上传User Application Jar
```bash
./bin/flink run-application -t yarn-application \
    -Djobmanager.memory.process.size=2048m \
    -Dtaskmanager.memory.process.size=4096m \
    -Dyarn.provided.lib.dirs="hdfs://node02:8020/flink-training/flink-1.11.1" \
    /home/flink-training/cluster-management/flink-on-yarn-1.11.1/examples/streaming/TopSpeedWindowing.jar
```

#### 3. 通过指定yarn.provided.lib.dirs参数部署，将Flink Binary包和Application Jar都同时从HDFS上获取
```bash
./bin/flink run-application -t yarn-application \
    -Djobmanager.memory.process.size=2048m \
    -Dtaskmanager.memory.process.size=4096m \
    -Dyarn.provided.lib.dirs="hdfs://node02:8020/flink-training/flink-1.11.1" \
    hdfs://node02:8020/flink-training/flink-1.11.1/examples/streaming/TopSpeedWindowing.jar
```

### 高可用配置
<!--endsec-->