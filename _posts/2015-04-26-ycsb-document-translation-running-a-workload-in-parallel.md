---
layout: page
title: YCSB官方文档-并行运行一个工作负载(中译)
comments: true
---

---

本文是[YCSB-wiki-Running-a-Workload-in-Parallel](https://github.com/brianfrankcooper/YCSB/wiki/Running-a-Workload-in-Parallel)一文的中文翻译。

---

## 1.概述

从多个服务器上运行工作负载的事务阶段是简单明了的——只需要在不同服务器上启动客户端，每个客户端运行相同的工作负载。每个客户端会在完成时生成性能统计信息，你将需要把这些独立的文件聚集为一个结果集。

在一些场景中，使用多服务器来装载数据库是有意义的。在这个例子中，你将想要把记录分区交给所有客户端来装载。通常，YCSB只是装载所有的记录（正如recordcount属性所定义的）。然而，如果你想要把装载分区，你需要额外为每个客户端指定两个其他属性：

- insertstart：起始记录的下标。
- insertcount：插入记录的数目。

这些属性可以在属性文件或者命令行（使用-p选项）被指定。

例如，想象你想要装载1亿条记录（所以recordcount=1000000000）。想象你想要用四个客户端来装载。对第一个客户端：

```shell
insertstart=0
insertcount=25000000
```

对第二个客户端：

```shell
insertstart=25000000
insertcount=25000000
```

对第三个客户端：

```shell
insertstart=50000000
insertcount=25000000
```

对第四个客户端：

```shell
insertstart=75000000
insertcount=25000000
```
