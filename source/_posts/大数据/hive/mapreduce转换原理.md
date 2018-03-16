---
title: HIVE运行过程
date: 2018/3/15 20:46:25
category:
- 大数据
- HIVE
tag:
- HIVE
comments: true  
---
## Hive与Hadoop

Hive的执行入口是Driver，执行的SQL语句首先提交到Drive驱动，然后调用compiler解释驱动，最终解释成MapReduce任务去执行。

**![img](http://img.blog.csdn.net/20160123184241450?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)**

## Hive的服务端组件

1. Driver组件：该组件包括：Compiler、Optimizer、Executor,它可以将Hive的编译、解析、优化转化为MapReduce任务提交给Hadoop1中的JobTracker或者是Hadoop2中的SourceManager来进行实际的执行相应的任务。


2. MetaStore组件：存储着hive的元数据信息，将自己的元数据存储到了关系型数据库当中，支持的数据库主要有：Mysql、Derby、支持把metastore独立出来放在远程的集群上面，使得hive更加健壮。元数据主要包括了表的名称、表的列、分区和属性、表的属性（是不是外部表等等）、表的数据所在的目录。**
3. 用户接口：CLI（Command Line Interface)(常用的接口：命令行模式）、Client:Hive的客户端用户连接至Hive Server ,在启动Client的时候，需要制定Hive Server所在的节点，并且在该节点上启动Hive Server、WUI:通过浏览器的方式访问Hive。

## Hive的工作原理

![img](http://img.blog.csdn.net/20160123191344049?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

流程大致步骤为：

1. 用户提交查询等任务给Driver。
2. 编译器获得该用户的任务Plan。
3. 编译器Compiler根据用户任务去MetaStore中获取需要的Hive的元数据信息。
4. 编译器Compiler得到元数据信息，对任务进行编译，先将HiveQL转换为抽象语法树，然后将抽象语法树转换成查询块，将查询块转化为逻辑的查询计划，重写逻辑查询计划，将逻辑计划转化为物理的计划（MapReduce）, 最后选择最佳的策略。
5. 将最终的计划提交给Driver。
6. Driver将计划Plan转交给ExecutionEngine去执行，获取元数据信息，提交给JobTracker或者SourceManager执行该任务，任务会直接读取HDFS中文件进行相应的操作。
7. 获取执行的结果。
8. 取得并返回执行结果。

http://blog.csdn.net/u010738184/article/details/70893161

