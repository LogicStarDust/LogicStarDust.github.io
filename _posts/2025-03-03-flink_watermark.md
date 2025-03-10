---
title: 【大数据基石】 Flink WaterMark
date: 2021-08-27 14:10:02 +0800
categories: [Flinke]
tags: [基石]
---
基石系列为我自己整理的大数据技术的基础理论和技术，作为整个在大数据行业工作的基础。故名基石，借此巩固自己的基础，希望其成为自我成长的基石。

# Flink 水印和时间机制

谈到flink的水印机制，我们必然要清楚事件时间和处理时间。

- 处理时间（Processing Time）
  - 处理时间时执行处理的机器的系统时间。
  - 使用处理时间可以达到最高性能和最低延迟。
  - 但是因为是分布式应用，具有很大的不确定性（不同的机器之间，数据延迟问题等）。
- 事件时间（Event Time）
  - 事件时间是每个事件实际发生的时间，通常在进入flink之前就嵌入事件内部。
  - 事件时间应用必须指定如何生成事件时间水印。

而事件时间和水印机制，就是我们这篇文章讨论的重点。这些很多参考了数据流模型，数据流模型相关论文和参考资料：

- 数据流模型：大规模无边界无序的数据处理中平衡正确、延迟和成本的方法
  - The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost in Massive-Scale, Unbounded, Out-of-Order Data Processing
  - https://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/43864.pdf
- 流媒体 101：批处理之外的世界（现代数据处理概念的高级介绍）
  - Streaming 101: The world beyond batch
  - https://www.oreilly.com/radar/the-world-beyond-batch-streaming-101/

总结数据流模型和flink时间机制设计的关联有下面几点：

- 事件时间
  - 事件时间的流处理器需要衡量事件时间的进度，比如在窗口操作的时候，需要有通知机制告知什么时间关闭窗口
  - 事件时间可以用于处理历史数据，短时间完成对大量历史数据的处理
- 水印
  - flink中衡量事件时间进度的是水印，水印作为事件流的一部分流通，携带时间戳。比如系统已经进行到7点的水印，那么事件流就不应存在6点59分的事件，存在的话也不会处理
  - 水印对于无序流至关重要，决定系统和开发中在当前时刻应该把内部事件时钟推进到哪一刻
  - 水印的创建：水印会在source或者紧随其后生成，并行的任务独立生成个字的水印
  - 水印的传递和更新：水印会流经过算子的时候更新算子的事件时间
  - 多输入的算子的水印：union或者keyby后的算子，消费多个输入流，算子的事件事件取决于输入流的最小水印
- 迟到数据
  - 迟到数据默认被丢弃
  - 可以支持迟到事件容忍时长，默认0，窗口的关闭在会在超过容忍时长后再关闭，把迟到时间的数据纳入窗口计算
  - flink的旁路输出功能，可以单独输出迟到的数据
