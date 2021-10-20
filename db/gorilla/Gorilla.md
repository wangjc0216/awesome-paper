# Gorilla 


## 背景及目的

[Prometheus Storage v2](https://github.com/prometheus-junkyard/tsdb) 是基于Gorilla paper来实现的。设计的相关思路可参考[Writing a Time Series Database from Scratch](https://fabxc.org/tsdb/).
所以阅读Gorilla对于了解Prometheus Storage 的实现是非常重要的。

互联网时代，一家公司会部署大量服务，要保证他们的高可用，需要对它们进行7*24小时的监控。 给他们提供监控通常需要千万/s的测量。
一个有效的办法就是将这些测量(measurements)的存储和查询放在TSDB中。

## 概述

TSDB设计中的一个关键挑战就是如何 在效率、可扩展性、可靠性(efficiency, scalability, and reliability.)保持平衡。

**在文中，我们介绍Gorilla，memory TSDB。**

我们不需要关心单点，而是强调趋势(聚合分析，aggregate analysis)，最近的数据的价值是高于老的数据的，新数据对于更快监测和诊断持续问题的根本原因是更有价值的。

Gorilla在高性能读/写方面做了优化。

写优化方向： **会在面对失败时，在写入路径下可能丢失少量数据作为代价。**

读优化方向： 积极利用压缩技术( delta-of-delta timestamps /   XOR’d floating point), 可将数据保存在内存中。相比Hbase， 存储空间减小了10x，查询延迟减少73x。

## 实现原理/数据模型



## 与之前的方案进行比对


