---
title: 【大数据基石】MapReduce On Yarn 推荐配置
date: 2021-08-11 19:54:02 +0800
categories: [大数据,yarn,MapReduce,hive]
tags: [基石,调优]
---
基石系列为我自己整理的大数据技术的基础理论和技术，作为整个在大数据行业工作的基础。故名基石，借此巩固自己的基础，希望其成为自我成长的基石。

## 一、简介
在集群的维护中，尤其是yarn计算平台。发现有了每个人配置执行参数的时候，使参数过大或过小,或者历史遗留脚本配置不合理，造成了集群利用率低下或任务失败过多。
这里根据对历史任务的使用总结结合yarn参数本身的执行原理和集群资源情况，给出MapReduce相关的推荐配置，针对的任务主要是mr和hive on mr。
其中自己开发mr的话对mr本身理解较深入的或者hive sql有特殊需求的，可以自己根据内存使用情况自定义参数。

适用版本：Hadoop 2.7.3
## 二、推荐配置

这些配置遵循的原则：

* 常用的有些配置是旧Hadoop集群的配置，大部分以“mapred.”开头，这里不针对说明和使用，一切以新的配置名为准。
  * 新旧配置关系：https://hadoop.apache.org/docs/r2.7.3/hadoop-project-dist/hadoop-common/DeprecatedProperties.html
* 本集群一个CPU core大约配1.5-2g可以达到集群最大的利用率，大部分配置以此为准。
* 参数官方解释以及默认值：https://hadoop.apache.org/docs/r2.7.3/hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml
* mapreduce的详解可以参考：https://blog.csdn.net/aijiudu/article/details/72353510

下表是mapred-site.xml建议项目：

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
mapreduce.map.output.compress.codec| DefaultCodec |com.hadoop.compression.lzo.LzoCodec|map阶段输出压缩类,推荐lzo,因为机器支持，现在使用默认的，请确定集群支持压缩格式
mapreduce.reduce.shuffle.parallelcopies|5|50|reduce同时可以从几个map拉数据
yarn.app.mapreduce.am.job.reduce.rampup.limit|0.5|0.2|在maps task已经完成，启动reduce task的比率。减少启动比例，防止浪费
mapreduce.job.reduce.slowstart.completedmaps|0.05|0.5|map task完成多少比例时候可以给reduce分配资源。增大比例，防止浪费
mapreduce.reduce.shuffle.input.buffer.percent|0.7|0.6|reduce用来存放从map拉取的数据的内存可以使用设置的reduce的jvm内存的比例

主要的修改：

* 合理配置化map、reduce、application Master的容器内存、jvm内存比例(默认太小，配比不适合集群)
* 增大map输出的环形缓冲区大小和同时合并文件数量，增大reduce端的同时拉取map输出的并发量。提高shuffle效率。
* 开启map端输出的压缩，降低磁盘和网络io，提供cpu利用率
* 调整map全部完成前，reduce启动的时机和数量，防止reduce浪费资源

## 三、样例配置文件
```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

<!-- Put site-specific property overrides in this file. -->

<configuration>
  <property>
     <name>mapreduce.framework.name</name>
     <value>yarn</value>
  </property>

  <property>
     <name>mapreduce.jobhistory.address</name>
     <value>根据需要配置</value>
  </property>

  <property>
     <name>mapreduce.jobhistory.webapp.address</name>
     <value>根据需要配置</value>
  </property>

  <property>
     <name>mapreduce.jobhistory.intermediate-done-dir</name>
     <value>根据需要配置</value>
  </property>

  <property>
      <name>mapreduce.jobhistory.done-dir</name>
      <value>根据需要配置</value>
  </property>
  <property>
    <name>mapreduce.map.memory.mb</name>
    <value>1536</value>
  </property>
  <property>
    <name>mapreduce.map.java.opts</name>
    <value>-Xmx1280M</value>
  </property>
  <property>
    <name>mapreduce.reduce.memory.mb</name>
    <value>2048</value>
  </property>
  <property>
    <name>mapreduce.reduce.java.opts</name>
    <value>-Xmx1536M</value>
  </property>
  <property>
    <name>yarn.app.mapreduce.am.resource.mb</name>
    <value>6144</value>
  </property>
  <property>
    <name>yarn.app.mapreduce.am.command-opts</name>
    <value>-Xmx6000m</value>
  </property>
  <property>
    <name>mapreduce.task.io.sort.mb</name>
    <value>512</value>
  </property>
  <property>
    <name>mapreduce.task.io.sort.factor</name>
    <value>100</value>
  </property>
  <property>
    <name>mapreduce.reduce.shuffle.parallelcopies</name>
    <value>50</value>
  </property>
    <property>
    <name>yarn.app.mapreduce.am.job.reduce.rampup.limit</name>
    <value>0.2</value>
  </property>
  <property>
    <name>mapreduce.job.reduce.slowstart.completedmaps</name>
    <value>0.5</value>
  </property>
  <property>
    <name>mapreduce.map.output.compress</name>
    <value>true</value>
    <final>true</final>
  </property>
  <property>
    <name>mapreduce.map.output.compress.codec</name>
    <value>com.hadoop.compression.lzo.LzoCodec</value>
    <final>true</final>
  </property>
    <property>
    <name>mapreduce.reduce.shuffle.input.buffer.percent</name>
    <value>0.6</value>
  </property>
</configuration>

```

## 四、其他说明

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
