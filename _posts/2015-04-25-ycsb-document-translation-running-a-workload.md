---
layout: post
title: YCSB官方文档-运行一个工作负载(中译)
comments: true
---

---

本文是[YCSB-wiki-Running-a-Workload](https://github.com/brianfrankcooper/YCSB/wiki/Running-a-Workload)一文的中文翻译。

---

### 1.概述

运行一个工作负载有6步：

1. 安装要测试的数据库
2. 选择合适的数据库接口层
3. 选择合适的工作负载
4. 选择合适的运行时参数（客户端线程数，目标吞吐量）
5. 装载数据
6. 执行工作负载

这里假定你是在运行单个客户端服务器。这对于小到中型集群（例如10台机器）应该是足够的。对于更大的集群，你可能需要在不同服务器上跑多个客户端来生成足够的工作负载。相似的，在某些情景下使用多客户端时导入速度会更快。如何跑多个客户端，参考[这篇博文](http://lsr1991.github.io/2015/04/26/ycsb-document-translation-running-a-workload-in-parallel/)。

### 2.安装要测试的数据库

第一步是安装你希望测试的数据库系统。这可以是在一台机器或者一个集群上完成，它取决于你希望对哪种配置进行基准测试。

你也必须创建或者安装表/键空间（keyspace）/存储等储存区来存储记录。这方面的细节随着数据库系统的不同而不同，并且取决于你希望运行的工作负载。在YCSB客户端运行之前，表必须已被创建，因为客户端自身不会去请求创建那些表。这是因为对于一些系统，有一个人工建表的步骤，并且对于其他系统，在数据库集群启动之前表就必须建好了。

那些必须要建的表依赖于工作负载。对于CoreWorkload，YCSB客户端将假定有一个表称为“usertable”，它拥有一个可变的模式：可以在运行时按需添加列。这个“usertable”能够被映射到任意一个合适的存储容器。比如，在MySQL中你将“CREATE TABLE”，在Cassandra中你将在它的配置中定义一个keyspace，等等。数据库接口层（在步骤2描述）将接收读写“usertable”记录的请求，并将它们翻译成对你所分配的实际存储的请求。这可能意味着你必须提供信息给数据库接口层，以便它理解底层存储的结构是什么。比如，在Cassandra中，除了keyspaces之外，你还必须定义“column families”。因此，创建一个column family并给它某个名字（比如你可能用“values”）是必要的。然后，数据库存取层将需要知道上面指的是“values”column family，因为字符串“values”可能是作为一个属性被传递的，也可能是在数据库接口层中被硬编码了（以常数数值表示而非变量）。

### 3.选择合适的数据库接口层

数据库接口层是一个java类，它将由YCSB客户端生成的对读，插入，更新，删除和遍历等操作的调用转换为对你的数据库API的调用。这个类是在com.yahoo.ycsb包中的数据库抽象类的一个子类。当你运行YCSB客户端时你将在命令行指定接口层的类的名称。任何在命令行中指定的属性或者参数文件，将会被传递到数据库接口实例中，并且能被用于配置接口层（例如，告诉它你所测试的数据库的主机名）。

YCSB客户端发布时包含一个简单的哑接口层，com.yahoo.ycsb.BasicDB。这个层只是将它本想执行的操作给打印到System.out。它有利于保证客户端操作是正当的，以及调试你的工作负载。

参考使用数据库库以了解在运行YCSB时如何使用客户端的细节。要参考关于一个数据库接口层的实现的更多细节，参考添加一个数据库。

你能够使用`ycsb`命令直接对数据库执行命令。这个客户端使用数据库接口层给数据库发送命令。你能够使用这个客户端来保证数据库层是正常工作的，你的数据库是安装正确的，以及数据库层能够连接上数据库，等等。它也提供了一个面向各种数据库的通用接口，并能够被用于检查数据库中的数据。运行命令行客户端：

```shell
$ ./bin/ycsb shell basic
> help
Commands:
read key [field1 field2 ...] - Read a record
scan key recordcount [field1 field2 ...] - Scan starting at key
insert key name1=value1 [name2=value2 ...] - Insert a new record
update key name1=value1 [name2=value2 ...] - Update a record
delete key - Delete a record
table [tablename] - Get or [set] the name of the table
quit - Quit
```

### 4.选择合适的工作负载
工作负载定义了在装载阶段将被装载到数据库中的数据，以及在事务阶段将被执行的对数据集的操作。

典型的，一个工作负载包括：

- 工作负载java类（com.yahoo.ycsb.Workload的子类）
- 参数文件（Java Properties格式）

因为数据集的属性应该在装载阶段被获知（这样才能构建合适的记录并将其插入），在事务阶段也要被获知（这样那些正确的记录id和字段才能被引用），所以一个单独的属性集会在这两个阶段被共享。因此那个配置文件在两个阶段都被使用。工作负载的java类使用那些属性去插入记录（装载阶段）或者对这些记录执行事务（事务阶段）。选择运行哪一个阶段取决于你运行`ycsb`命令时指定的一个参数。

当你运行YCSB客户端时你在命令行指定了java类和参数文件。客户端会动态地装载你的工作负载类，将参数文件中的属性传递给它（以及任何在命令行指定的额外属性），然后执行那个工作负载。这在装载阶段和事务阶段都会发生，并且这两阶段会使用相同的属性和工作负载逻辑。例如，如果装载阶段创建了包含10个字段的记录，那么事务阶段必须知道有10个字段它能够查询和修改。

CoreWorkload是一个标准工作负载的包，它随着YCSB一起发布，可以被直接使用。特别地，CoreWorkload定义了一个读/插入/更新/遍历等操作的简单混合。每种操作的相对频率在参数文件中被定义，工作负载的其他参数也是如此。因此，通过修改参数文件，各种不同的具体的工作负载可以被执行。需要更多CoreWorkload的细节，参考[这篇博文](http://lsr1991.github.io/2015/04/26/ycsb-document-translation-core-workloads/)。

如果CoreWorkload不能满足你的需求，你可以定义你自己的工作负载，这可以通过创建com.yahoo.ycsb.Workload类的子类来实现。更多细节参考[这篇博文](http://lsr1991.github.io/2015/04/26/ycsb-document-translation-implement-new-workloads/)。

### 5.选择合适的运行时参数
即使工作负载类和参数文件定义了一个特定的工作负载，在运行基准测试时你还是想指定一些额外的设置。当你运行YCSB客户端时命令行提供了这些设置。这些设置包括：

- `-threads`：客户端的线程。默认地，YCSB客户端使用一个工作者线程，但是额外的线程可以被指定。当需要增加对数据库的装载数量时这是经常使用的。
- `-target`：每秒的目标操作数。默认地，YCSB客户端将试图尽可能地执行最多操作。例如，如果每个操作平均使用了100ms，客户端每个工作者线程每秒将执行10个操作。然而，你可以限制每秒的目标操作数。比如，为了生成一条延迟-吞吐量曲线，你可以指定不同的目标吞吐量，以测试每种吞吐量下的延迟。
- `-s`：状态。对于一个运行时间长的工作负载，让客户端报告状态是有用的，这可以让你知道它并没有挂掉，并且给你某些对它的执行过程的想法。通过在命令行指定“-s”，客户端将每10秒输出状态到stderr。

### 6.装载数据
工作负载拥有两个可执行的阶段：装载阶段（定义了被插入的数据）和事务阶段（定义了对数据集的操作）。为了装载数据，你运行YCSB客户端并告诉它执行装载阶段。

例如，考虑基准测试工作负载A（关于标准工作负载的更多细节参考[Core Workload]）。为了装载标准数据集：

`$ ./bin/ycsb load basic -P workloads/workloada`

关于这个命令的一些提示：

- `load`参数告诉客户端执行工作负载的装载阶段。
- `basic`参数告诉客户端使用哑BasicDB层。你也可以在你的参数文件中使用“db”属性指定它（例如，“db=com.yahoo.ycsb.BasicDB”）。
- `-P`参数用于装载属性文件。在这个例子中，我们使用它来装载workloada参数文件。

为了装载HBase的数据集：

`$ ./bin/ycsb load hbase -P workloads/workloada -p columnfamily=family`

关于这个命令的一些提示：

- `load`参数告诉客户端执行工作负载的装载阶段。
- `hbase`参数告诉客户端使用HBase层。
- `-P`参数用于装载属性文件。在这个例子中，我们使用它来装载workloada参数文件。
- `-p`参数被用于设置参数。在这个HBase例子中，我们用它来设置数据库的列。在运行这个命令前，你应该拥有数据库`usertable`，并且它拥有列`family`。
- 在运行这个命令前，确保你已经启动了`Hadoop`和`HBase`。

如果你使用BasicDB，你将会看到对数据库的插入声明语句。如果你使用一个真实的数据库接口层，记录就会被装载到数据库中。

标准的工作负载参数文件创建非常小的数据库；例如，workloada只创建了1000条记录。当要调试你的安装的时候这是有用的。然而，要运行一个实际的基准测试，你将要生成一个远大于此的数据库。比如，想象你想要装载1亿条记录。那么，你将需要覆盖在workloada文件中的默认“recordcount”属性。这可以通过两种方式完成：

- 指定一个新的属性文件，它包含`recordcount`的一个新的值。如果这个文件在命令行的workloada文件之后被指定，它将覆盖workloada中的任何属性。例如，创建一个只有一行的文件“large.dat”：  
```shell
recordcount=1000000000
```  
接着运行客户端：  
```shell
$ ./bin/ycsb load basic -P workloads/workloada -P large.dat
```  
客户端将装载这两个属性文件，但是只会使用最后一个装载文件（如large.dat）的`recordcount`的值。
- 在命令行指定`recordcount`属性的一个新的值。在命令行指定的任意属性都会覆盖属性文件中指定的属性。在这个例子中，运行客户端：  
```shell
$ ./bin/ycsb load basic -P workloads/workloada -p recordcount=100000000
```

一般的，将任何重要的属性存储在一个新的参数文件中而不是在命令行指定它们是好的实践。这让你的基准测试结果变得更可复现；你只需要重用属性文件，而不需要重构你使用过的命令。然而需要注意的是，YCSB客户端会在它执行命令行的命令时将它打印出来，因此如果你将客户端的输出存储到一个数据文件中，你也能从那个文件里面找到执行的命令行。

因为一个大的数据库装载时间长，你可能希望客户端输出状态，以及将任何输出都放到一个数据文件中。因此，你可能得执行下述命令来装载你的数据库：

`$ ./bin/ycsb load basic -P workloads/workloada -P large.dat -s > load.dat`

`-s`参数将要求客户端生成状态报告到stderr。因此，这个命令的输出将是：

```shell
$ ./bin/ycsb load basic -P workloads/workloada -P large.dat -s > load.dat
Loading workload... (might take a few minutes in some cases for large data sets)
Starting test.
0 sec: 0 operations
10 sec: 61731 operations; 6170.6317473010795 operations/sec
20 sec: 129054 operations; 6450.76477056883 operations/sec
...
```

这个状态输出将会帮助你看到装载操作执行的速度（因此你可以估计装载所需要的时间），以及确认装载是在进行中。当装载完成，客户端将报告装载的性能统计信息。这些统计信息和事务阶段的那些统计信息是一样的，因此看下文以便知道如何解释这些统计信息。

### 7.执行工作负载
当数据被装载后，你就可以执行工作负载。这可以通过告诉客户端运行工作负载的事务阶段来完成。要执行工作负载，你可以使用如下命令：

`$ ./bin/ycsb run basic -P workloads/workloada -P large.dat -s > transactions.dat`

在这个调用中主要的区别在于我们使用`run`参数来告诉客户端使用事务阶段而不是装载阶段。如果你使用BasicDB，并且检查了transactions.dat文件，你将看到一堆读和更新的请求，以及对执行过程的统计信息。

典型的，你将想要使用`-threads`和`-target`参数来控制工作负载数量。例如，我们可能想要10个线程，总吞吐量为每秒100个操作（每个线程每秒10个操作）。只要操作的平均时延没有超过100ms，每个线程就能够达到每秒10个操作。通常，你需要足够的线程以使得不会有线程得超过它的速度上限才能达到总吞吐量的需求，否则你得到的吞吐量将会比指定的目标吞吐量要少。对于这个例子来说，我们可以执行：

`$ ./bin/ycsb run basic -P workloads/workloada -P large.dat -s -threads 10 -target 100 > transactions.dat`

注意，在这个例子中我们使用了`-threads 10`命令行参数来指定10个线程，以及`-target 100`命令行参数来指定100ops作为目标吞吐量。可选的，这些值都可以在你的参数文件中被设置，分别使用`threadcount`和`target`属性。例如：

```shell
threadcount=10
target=100
```

在运行最后，客户端会报告性能统计信息输出到stdout。在上述例子中，这些统计信息会被写入transactions.dat文件。默认是生成每种操作类型（读，插入等等）时延的均值，最小值，最大值，95%百分位的时延值，99%百分位的时延值，以及每个操作的返回代码的总数，和每个操作时延的直方图。返回代码通过你的数据库接口层来定义，允许你查看是否在执行工作负载期间有任何错误。对于上述例子，我们可能得到这样的输出：

```shell
[OVERALL],RunTime(ms), 10110
[OVERALL],Throughput(ops/sec), 98.91196834817013
[UPDATE], Operations, 491
[UPDATE], AverageLatency(ms), 0.054989816700611
[UPDATE], MinLatency(ms), 0
[UPDATE], MaxLatency(ms), 1
[UPDATE], 95thPercentileLatency(ms), 1
[UPDATE], 99thPercentileLatency(ms), 1
[UPDATE], Return=0, 491
[UPDATE], 0, 464
[UPDATE], 1, 27
[UPDATE], 2, 0
[UPDATE], 3, 0
[UPDATE], 4, 0
...
```

这个输出说明了：

- 总执行时间是10.11秒
- 平均吞吐量是98.9操作/秒（全部线程）
- 有491个更新操作，以及这些操作的平均时延，最大最小时延，95和99百分位的时延
- 所有491个更新操作返回代码为0（在这个例子中表明成功）
- 464个操作在1ms内完成，27个在1～2ms内完成

读操作也有相似的统计信息。

一个时延的直方图经常是有用的，有时候一个时间序列更有用。要请求一个时间序列，在客户端命令行或者属性文件中指定“measurementtype=timeseries”属性。默认的，客户端会以1000ms为间隔来报告平均时延。你可以使用“timeseries.granularity”属性来为报告指定一个不同的时间粒度。比如：

```shell
$ ./bin/ycsb run basic -P workloads/workloada -P large.dat -s -threads 10 -target 100 -p \
measurementtype=timeseries -p timeseries.granularity=2000 > transactions.dat
```

这将报告一个时间序列，以及每2000ms的平均时延。结果将是：

```shell
[OVERALL],RunTime(ms), 10077
[OVERALL],Throughput(ops/sec), 9923.58836955443
[UPDATE], Operations, 50396
[UPDATE], AverageLatency(ms), 0.04339630129375347
[UPDATE], MinLatency(ms), 0
[UPDATE], MaxLatency(ms), 338
[UPDATE], Return=0, 50396
[UPDATE], 0, 0.10264765784114054
[UPDATE], 2000, 0.026989343690867442
[UPDATE], 4000, 0.0352882703777336
[UPDATE], 6000, 0.004238958990536277
[UPDATE], 8000, 0.052813085033008175
[UPDATE], 10000, 0.0
[READ], Operations, 49604
[READ], AverageLatency(ms), 0.038242883638416256
[READ], MinLatency(ms), 0
[READ], MaxLatency(ms), 230
[READ], Return=0, 49604
[READ], 0, 0.08997245741099663
[READ], 2000, 0.02207505518763797
[READ], 4000, 0.03188493260913297
[READ], 6000, 0.004869141813755326
[READ], 8000, 0.04355329949238579
[READ], 10000, 0.005405405405405406
```

这个输出分别显示了更新和读取操作的时间序列，以及每2000ms的报告数据。一个时间点的报告数据只是前2000ms内的均值。（在这个例子中我们使用了10万个操作和一个10万ops的目标吞吐量来获得一个更有趣的输出）。对于时延测量需要注意一点：客户端测量执行一个对数据库的特定操作的端到端时延。也就是说，它在数据库接口层类中调用一个合适的方法前就开启了一个计时器，当方法返回值时停止该计时器。因此时延包括：接口层里面的执行时间，与数据库服务器间的网络时延，以及数据库的执行时间。它们没有包括限制目标吞吐量而引进的时延。也就是说，如果你指定一个10ops的目标吞吐量（一个线程），那么客户端将只会每100ms执行一个操作。如果这个操作用了12ms，那么客户端会等待额外的88ms之后才执行下一个操作。然而，报告的时延将不包括这个等待时间；12ms，而不是100ms的时延将被报告。
