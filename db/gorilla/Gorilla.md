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


## 介绍(1)

大规模互联网服务的目标是即使在出现以外故障的情况下，也能为用户保持高可用性和响应性，随着这些服务发展到全球，服务已经运行在数千台机器上了。

运营这些大规模服务的一个重要要求是准确监测底层系统的健康状态和性能，并在出现问题时快速识别和诊断它们。

- 写主导。数百个服务暴露的数据项，写入速率很容易超过千万条数据每秒点数。读取的速率通常比写入速率低几个量级。

- 状态转化。 通过服务升级、配置更新、网络中断导致的重大状态转换问题。**希望TSDB可以在较短的时间窗口内获取细粒度的聚合结果**。在十几秒/几十秒内发现状态转换是非常重要的。我们可以后续可以通过自动化的方式在问题广泛传播前修复问题。
```
  Thus, we wish for our TSDB to support fine-grained aggregations over short-time windows
 ```

- 高可用。网络分区或不同数据中心断开，在给定的数据中心内操作的系统应该也能够将数据写入到本地。

- 故障容忍度。将所有写入复制到多个区域，以免出现单点故障。

**Gorilla设计的insight是 监控系统的用户不会关心单个数据点，而是强调聚合分析。系统不存储用户信息，ACID不是TSDB的核心要求。 新的数据点比旧的数据点更有价值(更能及时发现问题)
Gorilla优化即使在遇到故障时也保持写入和读取的高可用性，代价是会丢掉少量数据。**

挑战来自高数据插入率、总数据量、实时聚合和可靠性要求。同时监控数据库85%的查询来自近26小时内收集的数据。进一步的分析使得我们确认我们使用内存数据库替换基于磁盘的数据库。
更长远来看，可以将Gorilla这个内存数据库看做是基于磁盘持久存储的cache。我们可以获得内存系统的速度，也可以获得磁盘的持久性。

**2015年，Facebook就存储了20亿个time series，其中每秒约添加1200万个sample/point,这代表每天超过1万亿点，每个sample/point 占16个bytes，那么存储就会占16TB.
我们使用XOR浮点压缩技术，会压缩到平均每个sample/point到1.37bytes,缩小了12x。**

我们在不同的数据中心部署多个Gorilla实例，并将数据流式传输到每一个实例来满足可用性要求，而不试图保证一致性(consistency)。


## 背景和要求(2)

### ODS(2.1)

ODS (Operational Data Store): 操作型数据存储。 ODS 在facebook monitor中非常重要的一部分(partition)。

ODS由一个时序数据库(TSDB)、一个查询服务、一个监测和告警服务。

#### 监控系统读取性能问题

在2013年时，HBase时间序列无法扩展处理未来的读取负载。虽然交互式图表的平均读取延迟是可以接受的，但第90分位的查询时间已经增加到几秒，从而阻碍了我们的自动化。
几千个time series的查询也需要10多秒才能完成，导致用户需要自我审查查询的使用方法。
```
Larger queries executing over sparse datasets would timeout as the HBase data store was tuned to prioritize writes.
由于 HBase 数据存储已调整为优先写入，因此在稀疏数据集上执行的较大查询将超时。
```
Facebook的hive也不合适，因为有ODS相比，它的查询延迟已经比ODS高出几个数量级，而查询效率和延迟是我们重点关注点。
**查询延迟和效率是主要关注点。**


【下面的描述针对读写cache并不能满足当前的监控场景?】 -cache miss [可以看陈浩的cache 文章]
「读缓存」我们于是将注意力转向内存缓存，ODS已经使用了一个简单的读取缓存。它主要针对多个仪表盘共享相同时间序列的图表系统，困难的场景是当仪表盘查询最近的数据点，在缓存中丢失，然后直接向Hbase
数据存储发出请求。
「写缓存」我们还考虑了基于独立的Memcache的直写缓存，但是拒绝了它，因为将新数据添加到已经存在的time series中需要一个读写周期(read/write cycle)，但是对于Memcache服务的流量会很高。

结论： 我们需要一个更有效(efficient)的解决方案。


### Gorilla requirements(2.2)

下面列出Gorilla应该具备哪些能力：
```bigquery
- 可存储20亿个time series

- 每秒1200w 个point/sample; 每分钟 7亿+ point/sample

- 峰值每秒超过4w查询

- 在1毫秒内读取成功

- 支持多每15s读取一次

- 两个不在同一位置内存的副本(用于灾备)

- 单个服务宕机依然保持可用(高可用)

- 能够快速扫描所有内存数据

- 支持每年至少两倍数据的增长
```

在Section 3 会简单比较其他TSDB实现方式。

在Section 4会介绍Gorilla的实现细节。首先会讨论在4.1讨论新的timestamp和data value的压缩计划(schemes)。在4.4中会介绍Gorilla在区域性灾难中的高可用。

在Section 5启用了Gorilla的新工具，并在Section 6 介绍了开发和部署Gorilla的相关经验。



## TSDB之间的比较(3)  / 与之前的方案进行比对

由于Gorilla一开始就被设计为所有数据存储在内存中，因此内存结构也不同于现有TSDB。
但是如果将Gorilla看做是其他磁盘TSDB之前的中间存储，那么Gorilla可以看做是任何TSDB的直写缓存(修改相对简单)。 **Gorilla的摄取速度和水平扩展与现有方案类似。**

### OpenTSDB (3.1)

OpenTSDB是基于HBase，非常接近我们使用长期数据的时候的HBase Storage Layer，两者有类似的表结构，在优化和水平扩展方面基本类似。**我们发现更高效的监控工具所需要的查询量要比基于磁盘的存储支持的查询量要大。**

与OpenTSDB不同，ODS HBase会将旧数据做rollup aggregation(汇总聚合)来节省空间,旧数据会失去部分精度。OpenTSDB则不会。我们觉得旧数据价值较低，减少的长时间查询和空间成本，值得精度损失。

OpenTSDB也有更丰富的数据模型来标识time series。


### Whisper、Graphite(3.2)


### InfluxDB(3.3)





## 实现原理/数据模型(4)




## Gorilla的新工具(5)



## 经验(experience) (6)



## 未来工作开展(7)



## 结论(8)





