  ---
  title: 【大数据基石】 Spark 指南
  date: 2021-08-27 14:10:02 +0800
  categories: [Spark]
  tags: [基石]
  ---

基石系列为我自己整理的大数据技术的基础理论和技术，作为整个在大数据行业工作的基础。故名基石，借此巩固自己的基础，希望其成为自我成长的基石。

## 一、简介
Apache Spark 是专为大规模数据处理而设计的快速通用的计算引擎。Spark是UC Berkeley AMP lab (加州大学伯克利分校的AMP实验室)所开源的类Hadoop MapReduce的通用并行框架，Spark，拥有Hadoop MapReduce所具有的优点；但不同于MapReduce的是——Job中间输出结果可以保存在内存中，从而不再需要读写HDFS，因此Spark能更好地适用于数据挖掘与机器学习等需要迭代的MapReduce的算法。
参考资料：

|索引|地址|
|----|----|
|官方文档|https://archive.apache.org/dist/spark/docs/3.2.0/|

## 二、Spark的组成部分

依据官方网站，spark从使用角度来看，有以下几个部分：

1. 概述
   - spark的整体介绍、下载、例子以及各种开发维护的索引
2. 开发指南
   - 快速开始
   - **RDD编程**
   - **Spark sql、DataFrame和Datasets编程**
   - Structured Streaming编程
   - **Spark Streaming编程**
   - 机器学习库编程
   - 图形计算库编程
   - R语言开发
   - python语言开发
3. API文档
   - Scala、Java、Python、R、SQL（以及内置函数）
4. 部署
   - 部署指南
   - 提交应用
   - spark 独立集群模式
   - Mesos集群模式
   - YARN集群模式
   - K8S集群模式
5. 更多
   - 配置
   - 监控
   - 调优
   - 作业调度
   - 安全
   - 硬件配置
   - 迁移指南
   - 编译spark、加入spark贡献、第三方项目

## 三、整体架构
spark是运行在集群上的应用，本质上是一组集群上的独立的进程。这组进程依赖SparkContext（sc）进行协调。sc是用户在主程序（驱动程序）中初始化的。

本质上，sc为了协调整个spark应用，是通过连接到集群管理器上（如spark独立集群、mesos、yarn、k8s），这些管理器为应用分配资源，然后建立连接，成功以后，会在节点上启动执行器executor。executor是负责计算和存储数据的进程，sc会将应用的业务代码发送到executor，进行执行。

![JVM 类加载](assets/img/posts_img/cluster-overview.png)

