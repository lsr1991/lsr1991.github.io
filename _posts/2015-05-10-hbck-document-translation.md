---
layout: post
title: hbck官方文档(中译)
comments: true
---

---

本文是[HBase官方文档-hbck in depth](http://hbase.apache.org/0.94/book/hbck.in.depth.html)一文的中文翻译。

---

## 0. 目录

1.运行hbck检测不一致性

2.不一致性

3.局部修复

4.region重叠修复

4.1.特例：Meta没有被正确分配

4.2.特例：HBase版本文件丢失

4.3.特例：Root和META失效（corrupt）

4.4.特例：离线拆分父Region

HBaseFsck（hbck）是用于检查region一致性和表完整性问题，修复失效HBase的一个工具。它有两种基本的工作模式——一个是“只读”的一致性检查模式，一个是多阶段“读-写”修复模式。

### 1. 运行hbck检测不一致性
为了检查你的HBase集群是否失效，对你的HBase集群运行hbck：

`./bin/hbase hbck`

在命令的输出末尾它会打印OK或者告诉你不一致性（INCONSISTENCIES）的数量。你也可能想要运行几次hbck因为一些不一致性可能是暂时的（例如集群在启动或者一个region在拆分）。操作上你可能要定期运行hbck并在它重复报告不一致性时告警（如，通过nagios）。运行一次hbck将会报告一系列不一致性以及受影响的表，region的简要描述。使用`-details`选项将报告更多的细节，包括所有表的所有分区列表。

`./bin/hbase hbck -details`

如果你只是想知道一些表是否失效，你可以限制hbck只对特定表检查不一致性。例如，下面的命令只会尝试检查表TableFoo和TableBar。这样做的好处是hbck将更快运行完。

`./bin/hbase hbck TableFoo TableBar`

### 2. 不一致性
如果在几次运行之后，不一致性仍然被报告，你可能遇到了一个失效。这些应该是少有的，但当它们发生时，更新版本的HBase包括了拥有自动修复选项的hbck工具。

有两种不变性被破坏时会造成HBase中的不一致性：

- 不变性一：HBase的region一致性。这个不变性被满足的条件是：所有region都被明确地分配并部署在一个regionserver上，并且存储这个状态的所有位置其内容都是一致的。
- 不变性二：HBase的表完整性。这个不变性被满足的条件是：对于每一张表，每一个可能的rowkey都可以被明确地解析到一个region上。

修复工作通常有三个阶段——一个“只读”的信息收集阶段用于检测不一致性，一个表完整性修复阶段用于重组表完整性不变性，最后是一个region一致性修复阶段用于重组region一致性不变性。从0.90版本开始，hbck能够从可能的表完整性问题的一个子集中检测出region的一致性问题报告。它也能够自动修复最常见的不一致性，即region分配和部署一致性问题。这个修复通过`-fix`命令行选项执行。修复时会将在错误的服务器或者多个regionserver上打开的region关闭，也将将没有打开的region分配给regionserver。

从HBase 0.90.7，0.92.2，和0.94.0版本开始，几个新的命令行选项被引进以辅助修复失效的HBase。这个hbck有时也称为“uberhbck”。每个uberhbck特定的版本与主版本号相同的HBase是兼容的（0.90.7的uberhbck可以修复0.90.4的HBase）。然而，版本号`<=0.90.6`以及版本号`<=0.92.1`可能需要重启master或者失效备援到一个备用master。

### 3. 局部修复
当修复一个失效的HBase时，最好是最先修复最低风险的不一致性。这些通常是region一致性修复——局部的单个region修复，它只会修改内存中的数据，暂时的zookeeper数据，或者修补META表中的漏洞。Region一致性要求HBase实例拥有region在HDFS中的数据的状态（.regioninfo文件），region在.META.表中的行状态，以及region在regionserver，master上的分配和部署状态都是一致的。

修复region一致性的选项包括：

- `-fixAssignments`（与0.90版本的`-fix`选项是等同的）修复未分配，不正确的分配或者被分配到多个服务器的region。
- `-fixMeta`从META表中移除在HDFS上不存在的region信息，并将在HDFS上存在的region但没有在META中的region信息添加到META中。

修复部署和分配的问题你可以运行下面命令：

`$ ./bin/hbase hbck -fixAssignments`

修复部署和分配问题同时修复不正确的meta数据你可以运行以下命令：

`$ ./bin/hbase hbck -fixAssignments -fixMeta`

有几类表完整性问题是低风险的修复。

最低风险的两类是退化（`startkey==endkey`）的region和反向的region（`starkey>endkey`）。这些会被自动处理，工具会把region的数据移到一个临时目录（/hbck/XXXX）。第三种低风险类别是hdfs上的region漏洞。这可以通过以下选项修复：

- `-fixHdfsHoles`选项可以在文件系统上生成一个新的空region。如果漏洞被检测出来，你可以用这个选来并且使用`-fixMeta`和`-fixAssignments`选项来使新的region一致。

`$ ./bin/hbase hbck -fixAssignments -fixMeta -fixHdfsHoles`

因为这是一个常见的操作，我们已经加入了一个-repairHoles选项，它与上述命令是等同的。

`$ ./bin/hbase hbck -repairHoles`

如果不一致性在执行完这几步后仍然存在，你最有可能是存在表完整性问题，这些问题与无归属region或者重叠region有关。

### 4. Region重叠修复
表完整性问题会需要处理重叠性的修复。这是一个更有风险的操作因为它需要对文件系统进行修改，需要做一些决定，以及可能需要一些手动的步骤。这些修复最好是分析一下`hbck -details`的输出信息，这样你才能将修复只定位在检测所发现的问题上。因为这是有风险的，所以应该使用保护措施来限制修复的范围。**注意**：这是一个相对新的并且只是对在线但空闲的HBase（无读/写）测试过。如果在活跃的生产环境中使用应评估风险！这些修复被破坏的表完整性选项包括：

- `-fixHdfsOrphans`选项用于将一个缺失了region元数据文件（.regioninfo文件）的region目录重分配。
- `-fixHdfsOverlaps`用于修复重叠的region

当修复重叠region时，一个region在文件系统上的数据能够通过两种方式修改：1)通过合并region到一个大的region；2)通过让region的数据移到一个备用目录下来使region下线，目录中的数据之后还能修复。将大量region合并在技术上将是正确的但是将导致一个极其大的region，它需要一个开销巨大的压缩和拆分操作。在这些情况下，可能更好的做法是让与最多region重叠的那些region下线（很可能是范围最大的那些region），这样才能使合并在一个更合理的范围里执行。因为这些下线的region已经被列在HBase的本地目录下并以HFile格式存储，它们可以通过HBase的bulk load机制来恢复。默认的保护阈值是有争议的。这些选项让你覆盖默认阈值并启用最大region下线的特性。

- `-maxMerge <n>`重叠region合并的最大数目
- `-sidelineBigOverlaps`如果超过maxMerge的region是重叠的，这个选项将让那些与最多region重叠的region下线
- `-maxOverlapsToSideline <n>`如果将大的重叠region下线，最多下线的region数量。

因为你通常只是想要让表被修复，你可以使用这个选项来开启所有修复选项：

- `-repair`包括了所有region一致性选项以及漏洞修复表完整性的选项。

最后，有保护措施来限制只对指定表进行修复。例如下面的命令将只是试图检查并修复TableFoo和TableBar。

`$ ./bin/hbase hbck -repair TableFoo TableBar`

**4.1 特例：Meta没有正确分配**

有一些特例是hbck可以处理的。有时meta的region被不一致地分配或者部署。在这个情况下有一个特殊的选项`-fixMetaOnly`来试图修复meta分配。

`$ ./bin/hbase hbck -fixMetaOnly -fixAssignments`

**4.2 特例：HBase版本文件缺失**

HBase在文件系统上的数据需要一个版本文件用于启动。如果这个文件丢失，你可以使用`-fixVersionFile`选项来生成一个新的HBase版本文件。这假定你使用的hbck的版本与HBase集群的版本是对应的。

**4.3 特例：Root和META失效**

最惨的失效场景就是ROOT或者META失效然后HBase无法启动。这种情况下你可以使用OfflineMetaRepair工具来创建新的ROOT和METAregions和表。这个工具假定HBase是离线的。它会从文件系统中HBase的主目录下寻找并装载尽可能多的region元数据文件。如果region元数据文件拥有正确的表完整性，它就会将原来的root和meta表目录给下线，并重建这两个表来指向那些region目录和数据。

`$ ./bin/hbase org.apache.hadoop.hbase.util.hbck.OfflineMetaRepair`

注意：这个工具不如uberhbck聪明但是可以用来辅助uberhbck可以完成的修复。如果工具成功修复你应该能够启动hbase并且在必要时运行在线修复。

**4.4 特例：离线拆分父region**

当一个region被拆分后，离线的父节点将会自动被清除。有时候，在父节点被清空之前，子region会被再次拆分。HBase能够按正确的顺序清除父节点。然而，有时有一些离线拆分的父节点会逗留。它们存在于META和HDFS上，并且未部署。但是HBase无法清除它们。在这种情况下，你可以使用`-fixSplitParents`选项来在META中重新设置它们为在线并且不拆分。因此，如果修复重叠region的选项被使用时，hbck能够将它们和其他regions进行合并。

这个选项通常不应该使用，它没有被包含在`-fixAll`里面。
