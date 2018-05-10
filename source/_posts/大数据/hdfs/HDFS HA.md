---
title: HDFS HA
date: 2018/5/10 17:08:48
category:
- 大数据
- hdfs
tag:
- hdfs
comments: true  
---

# 5 Hdfs HA

背景：hdfs集群SPOF，主要在两种情况下：

1. 突发事件如断电等
2. 系统升级

在同一个集群中运行两个(以及两个以上)冗余的NameNodes。这样可以在机器崩溃或者为计划维护的情况下快速转移到新的NameNode

可通过Quorum Journal Manager或NFS 实现

## 架构

只有一个活动namenode负责处理客户端请求，其他namenode仅用来快速恢复

所有的节点通过JournalNodes(jns)通信，active namenode会把所有日志记录到JNs上，Standby node会不断监控并更新editlog，active namenode故障时，Standby node会确保读取了所有日志，才切换到active namenode

为了提供快速故障转移，还需要备用节点具有关于集群中块位置的最新信息。datanode配置了所有的NameNodes的位置，并发送块位置信息和心跳。

为防止“split-brain scenario”，JNs只允许有一个writer

