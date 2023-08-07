---
layout:     post
title:      "Flink集群部署-standalone集群模式"
date:       2023-06-28 02:14:00
author:     "kevinkang"
header-img: "img/post-bg-miui6.jpg"
tags:
    - Flink
---
## Standalone集群
<br>![img](/img/in-post/post-flink/img_23.png)
- 集群特点:
  - 分布式多台物理主机部署
  - 依赖于java 8或者java 11 jdk环境
  - 仅支持Session模式提交Job
  - 支持高可用配置(Master主备)

### Standalone (单机) 集群部署
<br>![img](/img/in-post/post-flink/img_24.png)
1. JobManager和TaskManager全部在一台机器上运行
2. 支持Linux和Mac OS X上部署,windows机器需要安装Cygwin或WSL环境
3. 依赖java 8或者java 11 jdk环境
4. 仅适合本地测试,不适用于生产环境
5. 仅支持session模式提交job
6. 不支持高可用

#### Standalone (单机) 集群部署步骤
<br>![img](/img/in-post/post-flink/img_25.png)
1. 下载安装Flink安装包或者源码编译生成
2. 解压安装包
3. 启动Flink集群

#### Standalone (多机) 集群部署步骤
<br>![img](/img/in-post/post-flink/img_26.png)
