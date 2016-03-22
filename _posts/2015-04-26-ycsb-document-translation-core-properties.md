---
layout: page
title: YCSB官方文档-核心属性(中译)
comments: true
---

---

本文是[YCSB-wiki-Core-Properties](https://github.com/brianfrankcooper/YCSB/wiki/Core-Properties)一文的中文翻译。

---

## 1.核心YCSB属性

所有工作量文件可以指定以下属性：

- workload：要使用的工作量类（例如com.yahoo.ycsb.workloads.CoreWorkload）
- db：要使用的数据库类。可选地，这在命令行可以指定（默认：com.yahoo.ycsb.BasicDB）
- exporter：要是用的测量结果的输出类（默认：com.yahoo.ycsb.measurements.exporter.TextMeasurementsExporter）
- exportfile：用于替代stdout的输出文件路径（默认：未定义/输出到stdout）
- threadcount：YCSB客户端的线程数。可选地，这可以在命令行指定（默认：1）
- measurementtype：支持的测量结果类型有直方图和时间序列（默认：直方图）

## 2.核心工作量包属性

和核心工作量构造器一起使用的属性文件可以指定以下属性的值：

- fieldcount：一条记录中的字段数（默认：10）
- fieldlength：每个字段的大小（默认：100）
- readallfields：是否应该读取所有字段（true）或者只有一个字段（false）（默认：true）
- readproportion：读操作的比例（默认：0.95）
- updateproportion：更新操作的比例（默认：0.05）
- insertproportion：插入操作的比例（默认：0）
- scanproportion：遍历操作的比例（默认：0）
- readmodifywriteproportion：读-修改-写一条记录的操作的比例（默认：0）
- requestdistribution：选择要操作的记录的分布——均匀分布（uniform）、Zipfian分布（zipfian）或者最近分布（latest）（默认：uniform）
- maxscanlength：对于遍历操作，最大的遍历记录数（默认：1000）
- scanlengthdistribution：对于遍历操作，要遍历的记录数的分布，在1到maxscanlength之间（默认：uniform）
- insertorder：记录是否应该有序插入（ordered），或者是哈希顺序（hashed）（默认：hashed）
- operationcount：要进行的操作数数量
- maxexecutiontime：最大的执行时间（单位为秒）。当操作数达到规定值或者执行时间达到规定最大值时基准测试会停止。
- table：表的名称（默认：usertable）
- recordcount：装载进数据库的初始记录数（默认：0）

## 3.测量结果属性

这些属性被应用于每一个测量结果类型：

**直方图**

- histogram.buckets：直方图输出的区间数（默认：1000）

**时间序列**

- timeseries.granularity：时间序列输出的粒度（默认：1000）
