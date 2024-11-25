---
title: 【大数据基石】 HBase 指南
date: 2021-08-27 14:10:02 +0800
categories: [HBase]
tags: [基石]
---
基石系列为我自己整理的大数据技术的基础理论和技术，作为整个在大数据行业工作的基础。故名基石，借此巩固自己的基础，希望其成为自我成长的基石。

## 〇、常用
|索引|内容|
|----|----|
| 官方文档                                      | https://hbase.apache.org/book.html#arch.overview |
| HBase空间分析｜排查解决HBase目录空间占用异常  | https://www.modb.pro/db/57372                    |
| spark采用快照（snapshot）的方式读取hbase数据  |                                                  |
|spark直接使用hfile大量写入数据||
|spark使用自定义rdd读取rowkey头部加盐的hbase表||

## 一、简介
HBase是一个分布式的、面向列的开源数据库，该技术来源于 Fay Chang 所撰写的Google论文“Bigtable：一个结构化数据的分布式存储系统”。就像Bigtable利用了Google文件系统（File System）所提供的分布式数据存储一样，HBase在Hadoop之上提供了类似于Bigtable的能力。HBase是Apache的Hadoop项目的子项目。HBase不同于一般的关系数据库，它是一个适合于非结构化数据存储的数据库。另一个不同的是HBase基于列的而不是基于行的模式。

- 强一致性读/写：HBase 不是“最终一致”的数据存储。这使得它非常适合于诸如高速计数器聚合之类的任务。
- 自动分片：HBase 表通过区域分布在集群上，区域会随着数据的增长自动拆分和重新分布。
- 自动 RegionServer 故障转移
- Hadoop/HDFS 集成：HBase 支持开箱即用的 HDFS 作为其分布式文件系统。
- MapReduce：HBase 支持通过 MapReduce 进行大规模并行处理，将 HBase 用作源和接收器。
- Java 客户端 API：HBase 支持易于使用的 Java API 进行编程访问。
- Thrift/REST API：HBase 还支持非 Java 前端的 Thrift 和 REST。
- 块缓存和布隆过滤器：HBase 支持用于大容量查询优化的块缓存和布隆过滤器。
- 运营管理：HBase 为运维观察和 JMX 指标提供内置web服务。



## 二、架构

1. 数据模型
2. 

## 三、使用

## 四、维护

## 五、总结

