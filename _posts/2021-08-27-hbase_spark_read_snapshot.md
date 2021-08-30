---
title: 【解决方案】 spark采用快照（snapshot）的方式读取HBase数据
date: 2021-08-27 01:10:02 +0800
categories: [解决方案,HBase,Spark]
tags: [ETL]
---
## 一、简介

HBase的数据大量读取可以采用scan的方式，但是这种方式流量是经过RegionServer处理的，如果数据量较大就会影响线上业务，甚至gc导致RS挂掉。

之前我们集群提供了采用scan的方式导出的工具，但是使用者不了解机制，一次性导出大量数据，甚至并发若干个任务导出数据，导致RS过长GC、网络IO被打满，从而影响业务。

这里提供通过HBase Snapshot（固定数据）后，直接读取HFile的方式，绕过RS，从而避免影响线上业务和提高导出效率。我比较熟悉spark，这里就用spark开发。

## 二、实现

#### 前提：

- 快照功能是打开状态：hbase.snapshot.enabled 这个配置项是true,配置文件为hbase-site.xml
- 创建一个快照：`hbase> snapshot 'myTable', 'snapshot_test'`

#### 具体spark的流程：

2. 初始化一个org.apache.hadoop.conf.Configuration，填入HBase相关的zookeeper等集群配置。
3. 使用上一步的配置对象初始化org.apache.hadoop.mapreduce.Job对象
4. 设置TableSnapshotInputFormat的输入参数，
5. 使用newAPIHadoopRDD方法，获取读取HBase后的RDD
6. 把读取HBase RDD(格式为(ImmutableBytesWritable,Result),解析成可读的文本格式

详细代码如下：

```scala
package com.test.hbase.get

import com.test.hbase.get.logger

import org.apache.hadoop.hbase.mapreduce.{TableInputFormat, TableSnapshotInputFormat}
import org.apache.hadoop.mapreduce.Job
import org.apache.spark.sql.SparkSession
import org.apache.hadoop.fs.Path
import org.apache.hadoop.conf.Configuration
import org.apache.hadoop.hbase._
import org.apache.hadoop.hbase.client.Scan
import org.apache.hadoop.hbase.protobuf.ProtobufUtil
import org.apache.hadoop.hbase.util.{Base64, Bytes}

object getHBaseBySnapshot {

    // 获取扫描器
  def getScanStr: String = {
    val scan = new Scan()
    // scan.set....  各种过滤
    scan.setStartRow(Bytes.toBytes("88f000109bc0dee8025c"))
    scan.setStopRow(Bytes.toBytes("88f000109bc13e1c0385"))
    val proto = ProtobufUtil.toScan(scan)
    Base64.encodeBytes(proto.toByteArray)
  }

  // 构造 Hbase 配置信息,这里根据需要填写
  def getHbaseConf: Configuration = {
    val conf: Configuration = HBaseConfiguration.create()
    conf.set("hbase.zookeeper.quorum", "xxx1:2181,xxx2:2181")
    conf.set("hbase.zookeeper.property.clientPort", "2181")
    conf.set("zookeeper.znode.parent", "/xxxx")
    conf.set("mapred.task.timeout", "1")
    conf.set("hbase.rpc.timeout", "1800000")
    conf.set("hbase.client.operation.timeout", "1800000")
    conf.set("hbase.client.scanner.timeout.period", "1800000")
    // 设置查询的表名
    conf.set(TableInputFormat.INPUT_TABLE, "xxx")
    conf.set("fs.defaultFS","hdfs://xxxx:9000")
    conf.set("hbase.rootdir","hdfs://xxx:9000/hbase")
    conf.set(TableInputFormat.SCAN, getScanStr)
    conf
  }

  def getRes(result: org.apache.hadoop.hbase.client.Result): String = {
    val rowkey = Bytes.toString(result.getRow)
    // 通过列簇和列名获取数据
    val name = Bytes.toString(result.getValue("cf".getBytes, "name".getBytes))
    println(rowkey + "---" + name)
    name
  }

  def main(args: Array[String]): Unit = {
    val sparkSession = SparkSession.builder
      .getOrCreate()
    val sc = sparkSession.sparkContext

    // 1.2. 初始化任务 注意如果使用spark-shell会报错，请用spark-submit测试
    //    val job = Job.getInstance(getHbaseConf)
    lazy val job = Job.getInstance(getHbaseConf)

    // 快照名
    val snapshotName = "snapshot_test"
    // 3. 注意第三个参数路径是：快照恢复的临时目录。
    // 		当前用户应该对这个目录有写权限，并且这不应该是 rootdir 的子目录。作业完成后，可以删除restoreDir。
    TableSnapshotInputFormat.setInput(job, snapshotName, new Path(s"hdfs://xx:9000/tmp/hbase_snapshow_tmp"))
	// 4. 使用newAPIHadoopRDD方法，获取读取HBase后的RDD
    val hbaseRDD = sc.newAPIHadoopRDD(job.getConfiguration, classOf[TableSnapshotInputFormat],
      classOf[org.apache.hadoop.hbase.io.ImmutableBytesWritable],
      classOf[org.apache.hadoop.hbase.client.Result])
    // action触发操作,把读取HBase RDD(格式为(ImmutableBytesWritable,Result),解析成可读的文本格式
    hbaseRDD.map(_._2).take(10).map(getRes)
  }

}
```

## 三、其他

1. 快照创建

   - 每隔固定时间使用命令创建：
     ```shell
     #直接创建
     hbase> snapshot 'myTable', 'snapshot_test_20210821'
     #可以使用SKIP_FLUSH跳过flush,这样快照中不包含RS内存中的数据，这样快照速度会更快
     hbase> snapshot 'mytable', 'snapshot123', {SKIP_FLUSH => true}
     ```
   - 每次任务执行前创建

2. 快照清理  

   - 定时任务清理快照

   - 创建快照的时候添加过期时间：

     ```shell
     # 确认是否开启自动删除
     hbase> snapshot_cleanup_enabled
     # 开启自动删除
     hbase> snapshot_cleanup_switch true
     # 创建快照24小时后自动删除
     hbase> snapshot 'mytable', 'snapshot1234', {TTL => 86400}
     ```


