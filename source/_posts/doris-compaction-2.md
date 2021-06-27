---
title: Doris Compaction 调优指南（2）
date: 2021-05-16 11:29:31
tags: doris
keywords: doris
categories:
  - doris 最佳实践
---

这是 Doris Compaction 调优系列文章的第二篇。本文介绍了 Doris Compaction 任务的生成逻辑和执行逻辑。并且介绍了相关控制参数。
<!--more-->

本文是 Compaction 调优系列文章的第二篇。在前一篇文章中我们介绍了Compaction的一些基本概念。这里我们回顾下两个重要概念：

1. 每个 BE 节点上的 Compaction 操作都是独立进行的。Compaction 的对象是单个 BE 节点上的全部数据分片。

2. Compaction 分为 Base Compaction(BC) 和 Cumulative Compaction(CC)，由Cumulative Point(CP) 划分，根据一定策略，选择一组rowset进行Compaction。

本文将继续从以下两个方面深入了解 Compaction

1. Compaction 机制是如何挑选数据分片进行 Compaction 的。

2. 对于一个数据分片，Compaction 机制是如何确定哪些数据版本参与 Compaction 的。

## 数据分片选择策略

Compaction 的目的是合并多个数据版本，一是避免在读取时大量的 Merge 操作，二是避免大量的数据版本导致的随机IO。因此，Compaction 策略的重点问题，就是如何选择合适的 tablet，以保证节点上不会出现数据版本过多的数据分片。

### Compaction 分数

一个自然的想法，就是每次都选择数据版本最多的数据分片进行 Compaction。这个策略也是 Doris 的默认策略。这个策略在大部分场景下都能很好的工作。但是考虑到一种情况，就是版本多的分片，可能并不是最频繁访问的分片。而 Compaction 的目的就是优化读性能。那么有可能某一张 “写多读少” 表一直在 Compaction，而另一张 “读多写少” 的表不能及时的 Compaction，导致读性能变差。

因此，Doris 在选择数据分片时还引入了 “读取频率” 的因素。“读取频率” 和 “版本数量” 会根据各自的权重，综合计算出一个 Compaction 分数，分数越高的分片，优先做 Compaction。这两个因素的权重由以下 BE 参数控制（取值越大，权重越高）：

* `compaction_tablet_scan_frequency_factor`：“读取频率” 的权重值，默认为 0。
* 
* `compaction_tablet_compaction_score_factor`：“版本数量” 的权重，默认为 1。

> “读取频率” 的权重值默认为0，即默认仅考虑 “版本数量”  这个因素。

### 生产者与消费者

Compaction 系统是一个 生产者-消费者 模型。由一个生产者线程负责选择需要做 Compaction 的数据分片，而多个消费者负责执行 Compaction 操作。

生产者线程只有一个，会定期扫描所有 tablet 来选择合适的 compaction 对象。因为 Base Compaction 和 Cumulative Compaction 是不同类型的任务，因此目前的策略是每生成 9 个 CC 任务，生成一个 BC 任务。任务生成的频率由以下两个参数控制：

* `cumulative_compaction_rounds_for_each_base_compaction_round`：多少个CC任务后生成一个BC任务。
* 
* `generate_compaction_tasks_min_interval_ms`：任务生成的间隔。

> 这两个参数通常情况下不需要调整。

生产者线程产生的任务会被提交到消费者线程池。因为 Compaction 是一个IO密集型的任务，为了保证 Compaction 任务不会过多的占用IO资源，Doris 限制了每个磁盘上能够同时进行的 Compaction 任务数量，以及节点整体的任务数量，这些限制由以下参数控制：

* `compaction_task_num_per_disk`：每个磁盘上的任务数，默认为2。该参数必须大于等于2，以保证 BC 和 CC 任务各自至少有一个线程。
* 
* `max_compaction_threads`：消费者线程，即Compaction线程的总数。默认为 10。

举个例子，假设一个 BE 节点配置了3个数据目录（即3块磁盘），每个磁盘上的任务数配置为2，总线程数为5。则同一时间，最多有5个 Compaction 任务在进行，而每块磁盘上最多有2个任务在进行。并且最多有3个 BC 任务在进行，因为每块盘上会自动预留一个线程给CC任务。

另一方面，Compaction 任务同时也是一个内存密集型任务，因为其本质是一个多路归并排序的过程，每一路是一个数据版本。如果一个 Compaction 任务涉及的数据版本很多，则会占用更多的内存，如果仅限制任务数，而不考虑任务的内存开销，则有可能导致系统内存超限。因此，Doris 在上述任务个数限制之外，还增加了一个任务配额限制：

* `total_permits_for_compaction_score`：Compaction 任务配额，默认 10000。

每个 Compaction 任务都有一个配额，其数值就是任务涉及的数据版本数量。假设一个任务需要合并100个版本，则其配额为100。当正在运行的任务配额总和超过配置后，新的任务将被拒绝。

三个配置共同决定了节点所能承受的 Compaction 任务数量。

## 数据版本选择策略

一个 Compaction 任务对应的是一个数据分片（Tablet）。消费线程拿到一个 Compaction 任务后，会根据 Compaction 的任务类型，选择 tablet 中合适的数据版本（Rowset）进行数据合并。下面分别介绍 Base Compaction 和 Cumulative Compaction 的数据分片选择策略。

### Base Compaction

前文说过，BC 任务是增量数据和基线数据的合并任务。并且只有比 Cumulative Point（CP） 小的数据版本才会参与 BC 任务。因此，BC 任务的数据版本选取策略比较简单。

首先，会选取所有版本在 0 到 CP之间的 rowset。然后根据以下几个配置参数，判断是否启动一个 BC 任务：

* `base_compaction_num_cumulative_deltas`：一次 BC 任务最小版本数量限制。默认为5。该参数主要为了避免过多 BC 任务。当数据版本数量较少时，BC 是没有必要的。
* 
* `base_compaction_interval_seconds_since_last_operation`：第一个参数限制了当版本数量少时，不会进行 BC 任务。但我们需要避免另一种情况，即某些 tablet 可能仅会导入少量批次的数据，因此当 Doris 发现一个 tablet 长时间没有执行过 BC 任务时，也会触发 BC 任务。这个参数就是控制这个时间的，默认是 86400，单位是秒。

> 以上两个参数通常情况下不需要修改，在某些情况下如何需要想尽快合并基线数据，可以尝试改小 base_compaction_num_cumulative_deltas 参数。但这个参数只会影响到 “被选中的 tablet”。而 “被选中” 的前提是这个 tablet 的数据版本数量是最多的。

### Cumulative Compaction

CC 任务只会选取版本比 CP 大的数据版本。其本身的选取策略也比较简单，即从 CP 版本开始，依次向后选取数据版本。最终的数据版本集合由以下参数控制：

* `min_cumulative_compaction_num_singleton_deltas`：一次 CC 任务最少的版本数量限制。这个配置是和`cumulative_size_based_compaction_lower_size_mbytes` 配置同时判断的。即如果版本数量小于阈值，并且数据量也小于阈值，则不会触发 CC 任务。以避免躲过不比较的 CC 任务。默认是5。

* `max_cumulative_compaction_num_singleton_deltas`：一次 CC 任务最大的版本数量限制。以防止一次 CC 任务合并的版本数量过多，占用过多资源。默认是1000。

* `cumulative_size_based_compaction_lower_size_mbytes`：一次 CC 任务最少的数据量，和min_cumulative_compaction_num_singleton_delta 同时判断。默认是 64，单位是 MB。

简单来说，默认配置下，就是从 CP 版本开始往后选取 rowset。最少选5个，最多选 1000 个，然后判断数据量是否大于阈值即可。

CC 任务还有一个重要步骤，就是在合并任务结束后，设置新的 Cumulative Point。CC 任务合并完成后，会产生一个合并后的新的数据版本，而我们要做的就是判断这个新的数据版是 “晋升” 到 BC 任务区，还是依然保留在 CC 任务区。举个例子：

假设当前 CP 是 10。有一个 CC 任务合并了 [10-13] [14-14] [15-15] 后生成了 [10-15] 这个版本。如果决定将 [10-15] 版本移动到 BC 任务区，则需修改 CP 为 15，否则 CP 保持不变，依然为 10。

CP 只会增加，不会减少。 以下参数决定了是否更新 CP：

* `cumulative_size_based_promotion_ratio`：晋升比率。默认 0.05。

* `cumulative_size_based_promotion_min_size_mbytes`：最小晋升大小，默认 64，单位 MB。

* `cumulative_size_based_promotion_size_mbytes`：最大晋升大小，默认 1024，单位 MB。

以上参数比较难理解，这里我们先解释下 “晋升” 的原则。一个 CC 任务生成的 rowset 的晋升原则，是其数据大小和基线数据的大小在 “同一量级”。这个类似 2048 小游戏，只有相同的数字才能合并形成更大的数字。而上面三个参数，就是用于判断一个新的rowset是否匹配基线数据的数量级。举例说明：

在默认配置下，假设当前基线数据（即所有 CP 之前的数据版本）的数据量为 10GB，则晋升量级为 （10GB * 0.05）512MB。这个数值大于 64 MB 小于 1024 MB，满足条件。所以如果 CC 任务生成的新的 rowset 的大小大于 512 MB，则可以晋升，即 CP 增加。而假设基线数据为 50GB，则晋升量级为（50GB * 0.05）2.5GB。这个数值大于 64 MB 也大于 1024 MB，因此晋升量级会被调整为 1024 MB。所以如果 CC 任务生成的新的 rowset 的大小大于 1024 MB，则可以晋升，即 CP 增加。

从上面的例子可以看出，`cumulative_size_based_promotion_ratio` 用于定义 “同一量级”，0.05 即表示数据量大于基线数据的 5% 的 rowset 都有晋升的可能，而 `cumulative_size_based_promotion_min_size_mbytes` 和 `cumulative_size_based_promotion_size_mbytes` 用于保证晋升不会过于频繁或过于严格。

> 这三个参数会直接影响 BC 和 CC 任务的频率，尤其在高频导入场景下需要适当调整。我们会在后续文章中举例说明。

## 其他 Compaction 参数和注意事项

还有一些参数和 Compaction 相关，在某些情况下需要修改：

* `disable_auto_compaction`：默认为 false，修改为 true 则会禁止 Compaction 操作。该参数仅在一些调试情况，或者 compaction 异常需要临时关闭的情况下才需使用。

### Delete 灾难

通过 `DELETE FROM` 语句执行的数据删除操作，在 Doris 中也会生成一个数据版本用于标记删除。这种类型的数据版本比较特殊，我们成为 “删除版本”。删除版本只能通过 Base Compaction 任务处理。因此在在遇到删除版本时，Cumulative Point 会强制增加，将删除版本移动到 BC 任务区。因此数据导入和删除交替发生的场景通常会导致 Compaction 灾难。比如以下版本序列：

```
[0-10]
[11-11] 删除版本
[12-12]
[13-13] 删除版本
[14-14]
[15-15] 删除版本
[16-16]
[17-17] 删除版本
...
```

在这种情况下，CC 任务几乎不会被触发（因为CC任务只能选择一个版本，而无法处理删除版本），所有版本都会交给 Base Compaction 处理，导致 Compaction 进度缓慢。目前Doris还无法很好的处理这种场景，因此需要在业务上尽量避免。

## 未完待续

本文介绍了 Doris Compaction 任务的生成逻辑和执行逻辑。并且介绍了相关控制参数。接下来的文章，将通过一些具体场景来介绍调整 Compaction 参数的思路，以满足业务需求。