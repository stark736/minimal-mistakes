---
title: Elasticsearch 内存那点事
tags:
    - Elasticsearch
---

今天，我们不谈世界上最好的语言，来聊一聊 Elasticsearch。使用 ELK 栈也有一段时间了，踩了不少坑，今天就给大家分享一个。

先铺垫一下背景，ELK 做为日志收集服务相信大家都不陌生。我最开始选型 ELK 也并没有脱离它最擅长的场景 -- 收集业务上报日志。对上报后的日志，我们需要统计加工，最后生成统计的图表数据。Elasticsearch 作为日志的存储服务，我发现其同样支持聚合查询，于是决定使用它作为统计的数据源。我们的填坑之旅就从此开始了。

开始的开始，一切都很美好。Elasticsearch 很好了完成了我们交给它的任务，直到那一天，统计正在激烈运行的凌晨，我们遇到了 OOM 事件。

## OOM

什么是 OOM？很简单，Out of Memory。在执行统计查询的过程中，Elasticsearch 突然就崩溃了，然后给我们留下这样的话：

```
Caused by: java.lang.OutOfMemoryError: Java heap space
```

当时，为了节省资源，Elasticsearch 服务与其他数据库公用一台机器，为其分配的内存也只有可怜的8G。我们的第一反应肯定是内存不够，于是果断决定，迁移新机器，独立部署！

## 内存分配

买了16G的机器，给 Elasticsearch 分配了12G的内存，把数据迁移到新机器。想着日子应该可以这样平静地过下去了吧。然而，当天晚上，现实就给我们上了一课。平时只需要2个小时统计任务，当天足足执行了10多个小时才结束。期间，磁盘IO和负载告警不断。

是我对内存的分配有问题吗？查阅官方手册后发现还是真有不正确的地方。

> ##### No more than 50% of available RAM
> Lucene makes good use of the filesystem caches, which are managed by the kernel. Without enough filesystem cache space, performance will suffer. Furthermore, the more memory dedicated to the heap means less available for all your other fields using doc values.
> ##### No more than 32 GB
> If the heap is less than 32 GB, the JVM can use compressed pointers, which saves a lot of memory: 4 bytes per pointer instead of 8 bytes.

原来 Elasticsearch 限制的内存大小是 JAVA 堆空间的大小。并不包含 Lucene 使用的文件缓存。而堆空间占用的越高，Lucene 可以使用的缓存空间就越小，磁盘换读的频率越高，查询的效率自然就降了下来。至于不要超过32G，没有踩到这个坑，小小的庆幸一下。

## 垃圾回收

重新分配好内存就解决问题了吗？当然没有，16G内存只能分配8G，和迁移前的机器配置一样，OOM 异常还是会出现。看来有必要重新认识为什么会出现 OOM。打开 Elasticsearch 的日志，我留意到这些信息：

```
[2017-03-06T00:09:15,151][INFO ][o.e.m.j.JvmGcMonitorService] [GnqLs1r] [gc][old][98590][55] duration [7.9s], collections [1]/[8.1s], total [7.9s]/[34.7s], memory [7.7gb]->[7.1gb]/[7.9gb], all_pools {[young] [492.6mb]->[1.6mb]/[532.5mb]}{[survivor] [66.5mb]->[0b]/[66.5mb]}{[old] [7.1gb]->[7.1gb]/[7.3gb]}
```

这是 Elasticsearch 在进行内存回收时记录的日志。我们解析下日志可以得到以下信息：

日志内容  | 解析
------------- | -------------
`duration [7.9s]`  | 垃圾回收器消耗的总时长为7.9秒
`memory [7.7gb]->[7.1gb]/[7.9gb]`  | 内存使用从7.7G回收到了7.1G，总共可用内存为7.9G
`[young] [492.6mb]->[1.6mb]/[532.5mb]` | 新生代内存回收情况
`[survivor] [66.5mb]->[0b]/[66.5mb]` | 中生代内存回收情况
`[old] [7.1gb]->[7.1gb]/[7.3gb]` | 老生代内存回收情况

我们注意到，内存回收已经到达瓶颈，可回收的内存只占可用内存的1/8，这种情况随着查询的进行还会更加严重。这就是产生 OOM 异常的原因。

另外，老生代是内存占用的主力军，说明查询的跨度大，内存更迭频率高，大量的内存被移入老生代区域。

## 解决方案

到此，我们找到了 OOM 产生的原因。为了找寻解决方案，我模拟了统计工具的执行环境，查看当时 Elasticsearch 的内存使用分布。发现，几乎所有占用的内存都是 Segment Memory，该内存是为了提高查询速度，Lucene 将索引部分加载到内存中。这意味着，查询的数据量决定了内存的使用。

再来看下，我们执行统计需要的数据量。目前，我们每天统计的日志量在2~3G，同时查询的天数最多有30天。面对这样量级的查询，需要装载的索引大小超过我们给它分配的8G，也是极有可能的。

分析到此，当下解决问题最好的方案：加内存。将内存扩容到32G，并且分配16G给 Elasticsearch，统计时出现 OOM 异常就暂时消失了。

## 结语

写了这么多，你肯定发现了，这并不是一篇严谨的解决问题的思路分享。因为其中夹杂了很多我自己的臆断，逻辑上并没有做到严丝合缝。而之所以分享出来，是因为有一些结论，我认为比解决上述的问题更为重要：

- Elasticsearch 其实并不适合做稍大数据量的日志统计，这有 JVM 内存的处理方式的锅。
- 最开始决定使用 Elasticsearch，并且根据其提供的接口判断其能胜任统计的工作，实在是有些草率。这种被新技术冲昏头脑带来的失误，提醒我：对于新技术，需要更加深入的了解，对于其带来的风险，需要做更加深刻的评估。
- 有的时候，经过挣扎后使用的解决方案并不显得高深莫测。这其中，看似没有差别的决策，蕴含的却是知其所以然的道理。技术本如此，不断迭代方能更加厚实。

问题到最后，其实并没彻底解决。未来，可能会使用集群分片查询，也可能会迁移到更好的统计平台，无论如何，这次的收获都会帮助我把下一次的决策做得更好。


