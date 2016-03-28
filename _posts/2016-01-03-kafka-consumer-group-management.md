---
layout: post
title: 管理Kafka的Consumer-Group信息
comments: true
---

---

本文阐述如何查看/删除Kafka的Consumer-Group信息。

---

### 1. 问题描述

由于consumer group的设置会影响到consumer读取数据的行为，因此需要知道如何确定一个group是否是新的，如果是已存在的，那么它会从那个偏移量（即哪一条数据）开始读取。如果想删除一个group，又应该怎么做。

### 2. 解决方法

由于consumer group信息存储在zookeeper中，我们需要通过zookeeper的客户端来查看和设置。

CDH中安装了zookeeper，在网关（安装了Gateway的机器）和安装了zookeeper的节点上都可以使用zookeeper的客户端访问zookeeper。

使用以下命令：

```shell
zookeeper-client -server datanode11:2181
```

进入zookeeper目录后，使用`ls /`可以查看所有znode。

```shell
[zk: datanode11:2181(CONNECTED) 0] ls /
[consumers, hive_zookeeper_namespace_hive, solr, \
storm, 192.168.80.5, controller_epoch, isr_change_notification, \
hbase, zookeeper, admin, config, controller, brokers]
```

可以看到有consumers一项。查看它的内容：

```shell
[zk: datanode11:2181(CONNECTED) 1] ls /consumers
[consumer-linshangzhen, console-consumer-26561, topicZXX-consumer-group1, \
topicZXX1-consumer-group1, topicZXX-consumer-group22, cqlClient, \
topicZXX-consumer-group, lvqiujian-test, consumerWM1, consumerWM3, \
consumerWM2, wm101, wm103, wm100, lxy, consumerLZ, zxy-group-id]
```

随便选择一个consumer group查看：

```shell
[zk: datanode11:2181(CONNECTED) 2] ls /consumers/wm103
[owners, offsets, ids]      
[zk: datanode11:2181(CONNECTED) 3] ls /consumers/wm103/offsets
[testWM103]
[zk: datanode11:2181(CONNECTED) 4] ls /consumers/wm103/offsets/testWM103
[3, 2, 1, 0, 4]
[zk: datanode11:2181(CONNECTED) 5] ls /consumers/wm103/offsets/testWM103/0
[]
[zk: datanode11:2181(CONNECTED) 6] get /consumers/wm103/offsets/testWM103/0
47016415
cZxid = 0xb00374390
ctime = Mon Dec 14 10:31:50 CST 2015
mZxid = 0xb003889de
mtime = Mon Dec 14 14:27:53 CST 2015
pZxid = 0xb00374390
cversion = 0
dataVersion = 925
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 8
numChildren = 0
```

从上面的信息可以看到，consumer group wm103里面存储了offsets，owner，ids，分别对应数据的偏移量，消费的topic，broker的id。查看偏移量可以发现，里面以topic进行区分，进入每个topic，里面又以partition的id进行区分，从上面可以知道topic testWM103有5个partition，查看partitioin 0，可以知道它的最大偏移量是47016415。也就是说，属于consumer group的consumer如果开始接收partition 0的数据，会从偏移量为47016415开始接收。

我们再看一个没有偏移量的consumer group：

```shell
[zk: datanode11:2181(CONNECTED) 1] ls /consumers/lsrtest2
[owners, ids]
```

这个group是以下两种情况之一：刚创建没有读取数据；或者读取数据后没有向zookeeper报告已读到哪一条。

这样的group就可以用来在开发环境做调试，在调试代码中加上对consumer参数的配置`auto.commit.enable=false`就可以使得这个group不向zookeeper报告已读到哪一条，当每次程序启动时，consumer得知zookeeper中没有初始化偏移量（程序输出日志中会显示偏移量为-1），它就会从最新的数据开始读取，这样应用就只会处理在应用启动后producer发送的数据，应用启动前发送的数据都不会处理，这就避免了每次调试数据都会重复读取。

那么，如果zookeeper中创建的consumer太多了想删除怎么办？

进入zookeeper-client，使用以下命令：

```shell
[zk: datanode11:2181(CONNECTED) 3] rmr /consumers/lsrtest
```

可以将该路径递归删除（如果使用delete命令会提示你该路径不为空）。那么consumer group`lsrtest`就被删掉了。

那么如何查看topic中的最大偏移量（即最新消息的偏移量）是多少呢？

使用以下命令：

```shell
./kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list datanode71:9092 --topic topicLSR --partitions 0 --time -1
```

可以查看topicLSR的partition 0的最大偏移量。

使用`./kafka-run-class.sh kafka.tools.GetOffsetShell`可以查看命令的参数说明。

### 3. 关于kafka-console-consumer.sh

使用该脚本可以消费消息，consumer的所有参数它都可以使用，但是在设置`auto.commit.enable=false`之后看起来却并没有生效，因为就算在它消费之后立刻将它关掉，它仍然会向zookeeper报告消费到哪里。

通过实验验证，该程序是在被Ctrl+C时将偏移量报告给zookeeper的。所以即使设置了参数，当它运行时，该参数虽然生效了（不会每分钟报告一次），但是当kill掉的时候它仍然会进行报告。这带来了一个好处是，该客户端不会出现因为时间不够而来不及报告的情况，所以它也总是会从它已经消费的那条消息开始继续消费，从而不需要设置`auto.commit.enable`参数。
