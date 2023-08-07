---
layout:     post
title:      "Flink集群部署-kubernetes集群模式"
date:       2023-08-07 14:14:00
author:     "kevinkang"
header-img: "img/post-bg-miui6.jpg"
tags:
    - Flink
    - Flink部署与应用
---
## kubernetes集群架构概览
<br>![img](/img/in-post/post-flink/img_31.png)
- Master节点:
  - 负责整个集群的管理，资源管理
  - 运行APIServer，ControllerManager，Scheduler服务
  - 提供ETCD高可用键值存储服务，用来保存Kubernetes集群所有对象的状态信息和网络服务
- Node:
  - 集群操作的单元，Pod运行宿主机
  - 运行业务负载，业务负载会以Pod的形式运行
- Kubelet:
  - 运行在Node节点上，维护和管理该Node上的容器
- Container Runtime:
  - Docker容器运行环境，负责容器的创建和管理
- Pod:
  - 运行在Node节点上，多个相关的Container的组合
  - Kubernetes创建和管理的最小单位

### kubernetes集群主要概念
<br>![img](/img/in-post/post-flink/img_32.png)
- `Replication Controller(RC)`: RC是K8s集群中最早保证Pod高可用的API对象。通过监控运行中的Pod来保证集群中运行指定数目的Pod副本。
- `Service`: Service是对一组提供相同功能的Pods的抽象，并为他们提供一个统一的入口
- `Persistent Volume(PV)`: 容器的数据都是非持久化的，在容器消亡以后数据也跟着丢失，所以Docker提供了Volume机制以便将数据持久化存储
- `ConfigMap`: ConfigMap 用于保存配置数据的键值对，可以用来保存单个属性，也可以用来保存配置文件

### Flink On kubernetes 集群部署 - kubernetes安装
1. Docker & [Kubernetes安装](https://kuboard.cn/install/install-k8s.html)

### Flink On kubernetes 集群部署 - 镜像制作
<br>![img](/img/in-post/post-flink/img_33.png)

### Flink On kubernetes 集群架构 - Session Mode
<br>![img](/img/in-post/post-flink/img_34.png)
1. JobManager Deployment
2. TaskManager Deployment
3. UI Service(Restful API)
4. JobManager Service（Internal）
5. Flink ConfigMap
6. 通过客户端提交Job

### Flink On kubernetes 集群架构 - Per Job Mode
<br>![img](/img/in-post/post-flink/img_35.png)
1. JobManager Deployment
2. TaskManager Deployment
3. UI Service(Restful API)
4. JobManager Service（Internal）
5. Flink ConfigMap
6. User jar通过HostPath指定
7. Package jar to image
8. 不支持单独提交job

### Flink On kubernetes 资源配置
- Flink Conf ConfigMap
  - 用于存储Flink-conf.yaml，log4j-console.properties等配置信息
  - Flink JM和TM deployment 启动时会自动获取配置
- JobManager Service
  - 通过Service Name和Port暴露JobManager服务，让TaskManager能够连接到JobManager
- JobManager Deployment
  - 定义JobManager Pod副本数目，版本等，保证在Pods中至少有一个副本
- TaskManager Deployment
  - 定义TaskManager Pod副本数目，版本等，保证在Pods中至少有一个副本

#### Flink On kubernetes 资源配置-configMap
<br>![img](/img/in-post/post-flink/img_36.png)

#### Flink On kubernetes 资源配置-JobManager Service
<br>![img](/img/in-post/post-flink/img_37.png)

#### Flink On kubernetes 资源配置-Common
<br>![img](/img/in-post/post-flink/img_38.png)

### Flink On kubernetes 集群部署-Session Mode
<br>![img](/img/in-post/post-flink/img_39.png)

### Flink On kubernetes 集群部署-Per Job Mode
<br>![img](/img/in-post/post-flink/img_40.png)

### Flink On kubernetes 集群部署-Native
<br>![img](/img/in-post/post-flink/img_41.png)
- 客户端一次性提交ConfigMap，JobManager，JobManager Deployment等资源描述符以及授权信息
- 通过KubernetesResourceManager管理TaskManager Pod节点的启停
- 基于Fabric8 Kubernetes java客户端


#### Flink On kubernetes 集群部署-Native
<br>![img](/img/in-post/post-flink/img_42.png)
<br>![img](/img/in-post/post-flink/img_43.png)

#### Flink On kubernetes 优势与劣势
- 部署优势
  - 资源管理统一通过Kubernetes 进行管理，提升整体资源利用率
  - 基于Native方式，TaskManager资源按需申请和启动，防止资源浪费
  - Kubernetes副本和重启机制保证JobManager及TaskManager自动恢复
  - 基于容器化部署符合未来的趋势

- 部署劣势
  - Native 模式还需要增强，包括支持节点选择等高级特性
  - 故障排查比较困难，日志查看等
  - 运维成本较高，需要部署Docker和Kubernetes环境


