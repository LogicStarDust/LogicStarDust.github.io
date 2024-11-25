---
title: 【大数据基石】MapReduce On Yarn 内存管理和设置
date: 2021-08-10 01:54:02 +0800
categories: [大数据,yarn,mapreduce]
tags: [基石]
---
基石系列为我自己整理的大数据技术的基础理论和技术，作为整个在大数据行业工作的基础。故名基石，借此巩固自己的基础，希望其成为自我成长的基石。

## 一、内存模型
首先是yarn的简单架构：
![yarn](http://logic.stardust.wang/assets/img/posts_img/yarn.png)
		在一个NodeManager中，可以启动多个Container,每个Container内部再去执行Map Task或Reduce Task，有一个Container作为本次任务的AM(Application Master)管理、协调和追踪整个任务的执行。

* Container: 逻辑上看是一个“容器”，具有内存和cpu两种关键的资源，是一种资源单元。其实际是一个**进程**，由这个**进程启动各种子进程以完成交给自己的工作，比如作为AM或者child（执行MapReduce Task）。**
* Application Master(AM): 是Container运行的一种子进程，负责协调作业的运行
* Map/Reduce Task: 是Container运行的一种子进程，负责执行map或reduce具体的task

## 二、控制参数


<table>
<tr>
    <td>配置对象</td>
    <td>参数项</td>
    <td>旧版参数</td>
    <td>默认值</td>
    <td>所在配置文件</td>
    <td>是否可在任务中配置</td>
</tr>
<tr>
    <td rowspan="2">Map Task</td>
    <td>mapreduce.map.java.opts</td>
    <td>mapred.map.child.java.opts</td>
    <td>-Xmx200m</td>
    <td>mapred-site.xml</td>
    <td>1</td>
</tr>
<tr>
    <td>mapreduce.map.memory.mb</td>
    <td>mapred.job.map.memory.mb</td>
    <td>1024</td>
    <td>mapred-site.xml</td>
    <td>1</td>
</tr>
<tr>
    <td rowspan="2">Reduce Task</td>
    <td>mapreduce.reduce.java.opts</td>
    <td>mapred.reduce.child.java.opts</td>
    <td>-Xmx200m</td>
    <td>mapred-site.xml</td>
    <td>1</td>
</tr>
<tr>
    <td>mapreduce.reduce.memory.mb</td>
    <td>mapred.job.reduce.memory.mb</td>
    <td>1024</td>
    <td>mapred-site.xml</td>
    <td>1</td>
</tr>
<tr>
    <td>MapReduce Task</td>
    <td>mapred.child.java.opts</td>
    <td>-</td>
    <td>-</td>
    <td>mapred-site.xml</td>
    <td>1</td>
</tr>
<tr>
    <td rowspan="3">NodeManage</td>
    <td>yarn.nodemanager.resource.memory-mb</td>
    <td>-</td>
    <td>8192</td>
    <td>yarn-default.xml</td>
    <td>0</td>
</tr>
<tr>
    <td>yarn.nodemanager.vmem-pmem-ratio</td>
    <td>-</td>
    <td>2.1</td>
    <td>yarn-default.xml</td>
    <td>0</td>
</tr>
<tr>
    <td>yarn.nodemanager.vmem-check-enable</td>
    <td>-</td>
    <td>true</td>
    <td>yarn-default.xml</td>
    <td>0</td>
</tr>
</table>


<br>

## 三、参数原理

* mapreduce.map.java.opts 和 mapreduce.map.memory.mb


  * mapreduce.map.java.opts： **是启动JVM（mapreduce task）的时候传给JVM的参数，进程使用超过则报告OOM**
  * mapreduce.map.memory.mb ：**是Container的内存上限，这个值作为NodeManager监控Container进程的依据，当超过一定比例的时候，kill掉Container**
  * 设置这两个参数的目的是因为Container可以运行其他的子进程，前者控制JVM的用量，后者控制Container的用量。

* mapreduce.reduce.java.opts 和 mapred.job.reduce.memory.mb


  * 和map的控制方式相同。

* mapred.child.java.opts


  * 这个是为了兼容旧版本的参数，旧版本map和reduce的设置是不分开，统一由这个控制，注意控制的是jvm。
  * 优先级：**mapreduce.map.java.opts > mapred.child.java.opts > -Xmx200m**

* yarn.nodemanager.vmem-pmem-ratio 和 yarn.nodemanager.vmem-check-enabled


  * 这里是NodeManager的控制参数，NodeManager运行在主机上，负责分配和监控Container的运行，当其超过限制资源量一定比例的时候，kill掉Container
  * yarn.nodemanager.vmem-pmem-ratio：这个是控制Container进程使用的虚拟内存大小的:`[JVM申请内存]*[ratio]=最大虚拟内存大小`,**一旦Container的虚拟内存超过的最大值，NodeManager会直接kill掉Container**
  * yarn.nodemanager.vmem-check-enabled：NodeManager是否周期检查虚拟内存，就是上面说到的监控行为。

* 监控过程：


  * ContainerMonitor 就是上述所说的 NodeManager 中监控每个 Container 内存使用情况的 monitor，它是一个独立线程。ContainerMonitor 获得单个 Container 内存（包括物理内存和虚拟内存）使用情况的逻辑如下：

    * Monitor 每隔 3 秒钟就更新一次每个 Container 的使用情况；更新的方式是：

      1. 查看 /proc/pid/stat 目录下的所有文件，从中获得每个进程的所有信息；

      2. 根据当前 Container 的 pid 找出其所有的子进程，并返回这个 Container 为根节点，子进程为叶节点的进程树；在 Linux 系统下，这个进程树保存在 ProcfsBasedProcessTree 类对象中；

      3. 然后从 ProcfsBasedProcessTree 类对象中获得当前进程 (Container) 总虚拟内存量和物理内存量。

  * ![container_monitor](http://logic.stardust.wang/assets/img/posts_img/container_monitor.png)

