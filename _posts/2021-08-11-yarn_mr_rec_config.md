---
title: 【大数据基石】MapReduce On Yarn 推荐配置
date: 2021-08-11 19:54:02 +0800
categories: [大数据,yarn,MapReduce,hive]
tags: [基石,调优]
---
基石系列为我自己整理的大数据技术的基础理论和技术，作为整个在大数据行业工作的基础。故名基石，借此巩固自己的基础，希望其成为自我成长的基石。

## 一、简介
在集群的维护中，尤其是yarn计算平台。发现有部分人对mr过程执行了解并不透彻，照成了每个人配置执行参数的时候，按照自己风格，使参数过大或过小，造成了集群利用率低下或任务失败过多。
这里根据对历史任务的使用总结结合yarn参数本身的执行原理，给出MapReduce相关的推荐配置，针对的任务主要是**mr和hive on mr**。
其中自己开发mr的话对mr本身理解较深入的或者hive sql有特殊需求的，可以自己根据内存使用情况自定义参数。

适用版本：Hadoop 2.7.3
## 二、推荐配置

这些配置遵循的原则：

* 常用的有些配置是旧Hadoop集群的配置，大部分以“mapred.”开头，这里不针对说明和使用，一切以新的配置名为准。
  * 新旧配置关系：https://hadoop.apache.org/docs/r2.7.3/hadoop-project-dist/hadoop-common/DeprecatedProperties.html
* 本集群一个CPU core大约配1.5-2g可以达到集群最大的利用率，大部分配置以此为准。
* 参数官方解释以及默认值：https://hadoop.apache.org/docs/r2.7.3/hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml
* mapreduce的详解可以参考：https://blog.csdn.net/aijiudu/article/details/72353510

下表是mapred-site.xml建议项目（http://logic.stardust.wang/asset/file/mapred-site.rec.xml）：

参数项|默认|推荐配置|说明
----|----|----|----
mapreduce.map.memory.mb|1024|1536|Map任务，容器内存设1.5G
mapreduce.map.java.opts|-Xmx200m|-Xmx1280M|Map任务，JVM最大堆内存为1.25G，剩下256M用于其他
mapreduce.reduce.memory.mb|1024|2048|Reduce任务，容器内存设2G
mapreduce.reduce.java.opts|-Xmx200m|-Xmx1536M|Map任务，JVM最大堆内存为1.5G，剩下512M用于其他
yarn.app.mapreduce.am.resource.mb|1536|6144|Application Master所在容器使用的最大堆内存为6G
yarn.app.mapreduce.am.command-opts|-Xmx1024m|-Xmx6000m|Application Master所启动的JVM的最大堆内存为6000M
mapreduce.task.io.sort.mb|100|512|排序合并文件，环形缓冲区内存大小，减少磁盘溢写
mapreduce.task.io.sort.factor|10|100|一次排序合并文件流（溢写到磁盘的文件）的数量，加快合并
mapreduce.map.output.compress|false|true|map阶段输出是否压缩，节省io
mapreduce.map.output.compress.codec| DefaultCodec |com.hadoop.compression.lzo.LzoCodec|map阶段输出压缩类
mapreduce.reduce.shuffle.parallelcopies|5|50|reduce同时可以从几个map拉数据
yarn.app.mapreduce.am.job.reduce.rampup.limit|0.5|0.2|在maps task已经完成，启动reduce task的比率。减少启动比例，防止浪费
mapreduce.job.reduce.slowstart.completedmaps|0.05|0.5|map task完成多少比例时候可以给reduce分配资源。增大比例，防止浪费
mapreduce.reduce.shuffle.input.buffer.percent|0.7|0.6|reduce用来存放从map拉取的数据的内存可以使用设置的reduce的jvm内存的比例


## 三、其他说明

### 1. 针对group by数据倾斜的优化
```shell
# 是否开启mapper端聚合
set hive.map.aggr=true;

# 是否开启，如果数据倾斜，是否优化group by为两个MR job
#该配置会触发hive增加额外的mr过程，随机化key后进行聚合操作得到中间结果，再对中间结果执行最终的聚合操作。
#count(distinct)操作比较特殊，无法进行中间的聚合操作，因此该参数对有count(distinct)操作的sql不适用。
set hive.groupby.skewindata=true;
# 用于map端聚合的hashtable最大可用内存，如果超过该内存比例，将flush到磁盘
set hive.map.aggr.hash.force.flush.memory.threshold;

# 可以用于mapper端hatable的内存比例
# hive.map.aggr.hash.percentmemory (Default: 0.5) – Percent of total map task memory that can be used for hash table.
# 如果hashtable大小/输入行数 大于该阈值，那么停止hash聚合,转为sort-based aggregation
# hive.map.aggr.hash.min.reduction (Default: 0.5)
# 每隔多少行，检测hashtable大小和input row比例是否超过阈值
# hive.groupby.mapaggr.checkinterval

```
