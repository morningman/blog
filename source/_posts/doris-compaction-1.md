---
title: Doris Compaction 调优指南（1）
date: 2021-05-10 12:37:31
tags: doris
keywords: doris
categories:
  - doris 最佳实践
---

这是 Doris Compaction 调优系列文章的第一篇，本文主要介绍Compaction相关的基础知识。
<!--more-->

## 什么是 Compaction

Doris 的数据写入模型使用了 LSM-Tree 类似的数据结构。数据都是以追加（Append）的方式写入磁盘的。这种数据结构可以将随机写变为顺序写。这是一种面向写优化的数据结构，他能增强系统的写入吞吐，但是在读逻辑中，需要通过 Merge-on-Read 的方式，在读取时合并多次写入的数据，从而处理写入时的数据变更。

Merge-on-Read 会影响读取的效率，为了降低读取时需要合并的数据量，基于 LSM-Tree 的系统都会引入后台数据合并的逻辑，以一定策略定期的对数据进行合并。Doris 中这种机制被称为 Compaction。

Doris 中每次数据写入会生成一个数据版本。Compaction的过程就是讲多个数据版本合并成一个更大的版本。Compaction 可以带来以下好处：

1. 使数据更加有序

    每个数据版本内的数据是按主键有序的，但是版本之间的数据是无序的。Compaction后形成的大版本将多个小版本的数据变成有序数据。在有序数据中进行数据检索的效率更高。

2. 消除数据变更

    数据都是以追加的方式写入的，因此 Delete、Update 等操作都是写入一个标记。Compaction 操作可以处理这些标记，进行真正的数据删除或更新，从而在读取时，不再需要根据这些标记来过滤数据。

3. 增加数据聚合度

    在聚合模型下，Compaction 能进一步聚合不同数据版本中相同 key 的数据行，从而增加数据聚合度，减少读取时需要实时进行的聚合计算。


关于 Compaction 的详细介绍，可以参阅 [【Doris全面解析】Doris Compaction机制解析](https://mp.weixin.qq.com/s?__biz=Mzg5MDEyODc1OA==&mid=2247485136&idx=1&sn=a10850a61f2cb6af42484ba8250566b5&chksm=cfe016c9f8979fdf100776d9103a7960a524e5f16b9ddc6220c0f2efa84661aaa95a9958acff&scene=21&token=260549987&lang=zh_CN#wechat_redirect)。


## Compaction 的问题

用户可能需要根据实际的使用场景来调整 Compaction 的策略，否则可能遇到如下问题：

1. Compaction 速度低于数据写入速度

    在高频写入场景下，短时间内会产生大量的数据版本。如果 Compaction 不及时，就会造成大量版本堆积，最终严重影响写入速度。

2. 写放大问题

    Compaction 本质上是将已经写入的数据读取后重写写回的过程，这种数据重复写入被称为写放大。一个好的Compaction策略应该在保证效率的前提下，尽量降低写放大系数。过多的 Compaction 会占用大量的磁盘IO资源，影响系统整体效率。

Doris 中用于控制Compaction的参数非常多。本文尝试以下方面，介绍这些参数的含义以及如果通过调整参数来适配场景。

* 数据版本是如何产生的，哪些因素影响数据版本的产出。
* 为什么需要 Base 和 Cumulative 两种类型的 Compaction。
* Compaction 机制是如何挑选数据分片进行 Compaction 的。
* 对于一个数据分片，Compaction 机制是如何确定哪些数据版本参与 Compaction 的。
* 在高频导入场景下，可以修改哪些参数来优化 Compaction 逻辑。
* Compaction 相关的查看和管理命令。

## 数据版本的产生

首先，用户的数据表会按照分区和分桶规则，切分成若干个数据分片（Tablet）存储在不同 BE 节点上。每个 Tablet 都有多个副本（默认为3副本）。Compaction 是在每个 BE 上独立进行的，Compaction 逻辑处理的就是一个 BE 节点上所有的数据分片。

前文说到，Doris的数据都是以追加的方式写入系统的。Doris目前的写入依然是以微批的方式进行的，每一批次的数据针对每个 Tablet 都会形成一个 rowset。而一个 Tablet 是由多个Rowset 组成的。每个 Rowset 都有一个对应的起始版本和终止版本。对于新增Rowset，起始版本和终止版本相同，表示为 [6-6]、[7-7] 等。多个Rowset经过 Compaction 形成一个大的 Rowset，起始版本和终止版本为多个版本的并集，如 [6-6]、[7-7]、[8-8] 合并后变成 [6-8]。

Rowset 的数量直接影响到 Compaction 是否能够及时完成。那么一批次导入会生成多少个 Rowset 呢？这里我们举一个例子：

假设集群有3个 BE 节点。每个BE节点2块盘。只有一张表，2个分区，每个分区3个分桶，默认3副本。那么总分片数量是（2 * 2 * 3）12 个，如果均匀分布在所有节点上，则每个盘上2个tablet。假设一次导入涉及到其中一个分区，则一次导入总共产生6个Rowset，即平均每块盘产生一个 Rowset。（这里仅考虑数据完全均匀分布的情况下，实际情况中，可能多个 Tablet 集中在某一块磁盘上。）

从上面的例子我们可以得出，rowset的数量直接取决于表的分片数量。举个极端的例子，如果一个Doris集群只有3个BE节点，但是有9000个分片。那么一次导入，每个BE节点就会新增3000个rowset，则至少要进行3000次compaction，才能处理完所有的分片。所以：

合理的设置表的分区、分桶和副本数量，避免过多的分片，可以降低Compaction的开销。

## Base & Cumulative Compaction

Doris 中有两种 Compaction 操作，分别称为 Base Compaction(BC) 和 Cumulative Compaction(CC)。BC 是将基线数据版本（以0为起始版本的数据）和增量数据版本合并的过程，而CC是增量数据间的合并过程。BC操作因为涉及到基线数据，而基线数据通常比较大，所以操作耗时会比CC长。

如果只有 Base Compaction，则每次增量数据都要和全量的基线数据合并，写放大问题会非常严重，并且每次 Compaction 都相当耗时。因此我们需要引入 Cumulative Compaction 来先对增量数据进行合并，当增量数据合并后的大小达到一定阈值后，再和基线数据合并。这里我们有一个比较通用的 Compaction 调优策略：

`在合理范围内，尽量减少 Base Compaction 操作。`

BC 和 CC 之间的分界线成为 Cumulative Point（CP），这是一个动态变化的版本号。比CP小的数据版本会只会触发 BC，而比CP大的数据版本，只会触发CC。

<div style="width:70%;margin:auto">{% asset_img cumulative_point.png "cumulative point" %}</div>

整个过程有点类似 2048 小游戏：只有合并后大小足够，才能继续和更大的数据版本合并。

<div style="width:70%;margin:auto">{% asset_img 2048.png 2048 %}</div>

## 未完待续

本文介绍了 Doris Compaction 的一些基础概念，在接下来的文章中，我们将详细介绍影响 Compaction 触发的参数，以及如何调整这些参数。

<div class="tip">
    测试用
</div>
