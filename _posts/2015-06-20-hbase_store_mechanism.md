---
layout: page
title: HBase的数据存储机制
---

#### 本文说明
本文分三部分。第一部分是资料的整理和总结。第二部分是查考资料时的阅读摘要。若要把握整体流程请看第一部分，若要查考细节请看第二部分。第三部分是参考文献。

## 第一部分 总结

#### 1.客户端如何访问HBase

什么是HBase客户端？HBase客户端即一个HTable对象。

客户端对hbase的访问都是通过zookeeper集群做中介的。客户端程序通过hbase-site.xml文件中的配置得知zookeeper的连接地址，然后通过zookeeper知道集群中各个regionserver的连接地址。客户端程序运行时会先检查java程序的classpath，如果包含hbase-site.xml的路径，则读取hbase-site.xml获取zookeeper的连接地址，否则从操作系统的CLASSPATH中读取hbase-site.xml，从而获取zookeeper的连接地址。

#### 2.一次put操作的执行流程

确定执行put操作服务器的位置：由于put操作是一次RPC，它需要被传送到执行该操作的服务器（reigonserver）上，因此需要确定服务器的连接地址，准确地说是需要确定put操作中要修改的数据所在region的位置。客户端通过zookeeper可以获知所有region的位置（即所在服务器的连接地址）并将信息缓存下来，当缓存为空时，客户端获取region位置的具体过程为：客户端访问zookeeper，获取拥有-ROOT-表的服务器的地址，将信息缓存下来，然后访问该服务器获取-ROOT-表，-ROOT-表中记录着拥有.META.表的服务器的地址，客户端又将这些信息缓存下来，然后继续访问拥有.META.表的服务器，获取.META.表，.META.表中记录着所有region的信息，包括起始行键、所属regionserver的地址等关键信息，客户端又将这些信息缓存下来。从缓存的这些信息中，客户端就可以知道put操作应该交给哪一个region，发往哪一个regionserver服务器。直到某一个信息失效（访问时找不到对应内容），客户端才会重新访问对应服务器获取新的信息。从上述看来，客户端缓存的信息有三个：-ROOT-表的位置，.META.表的位置，region的位置。最差的情况是在执行put操作时缓存的三个信息全部失效，那么客户端必须先执行3次访问才能确定所有信息失效，再执行3次访问才能将所有信息更新，总共需要6次网络往返请求。

put操作的发送：put操作被客户端封装在一个keyvalue对象中，由RPC送往目标regionserver，由regionserver上的对应region对象接收，接着region对象将该keyvalue对象写入日志文件HLog中（作为HLog一个K,V类型的数据单元中的value），然后region对象再将keyvalue对象送入memstore内存中，接着RPC完成，客户端程序继续运行。

#### 3.HBase如何将收到的数据写入磁盘

刷写：如同之前的put操作，其他修改操作（delete，increment）都被封装成一个keyvalue对象存储在memstore内存中。每一个region会为对应表的所有列族各创建一个store对象，每一个store对象会被分配一个memstore，所有的memstore有一个大小上限，由hbase.hregion.memstore.flush.size控制。当memstore大小超过上限时，memstore中的数据将被刷写到磁盘（_memstore在刷写时还能被写入吗？_），同时生成一个存储这些数据的文件，这个文件以HFile的格式存储在HDFS上。

存储：HFile文件被分割成多个HDFS文件块分布在HDFS集群上。

#### 4.HBase如何管理存储的数据

合并：由于memstore大小上限一般较小，在数据不断刷写到磁盘后会产生许多文件。为了使文件数量不至于太多（_还有其他原因？_），region会使用合并机制将小文件合并成大文件。合并的方式有两种，一种是minor compaction，一种是major compaction。前者只会合并一定数量的小文件，后者则会将所有文件合并，两者触发的条件不一样，执行频率较高的是前者。

拆分：当合并后的文件超过阈值时，会触发region的拆分，父region会分成两个大小为原来一半的子region。阈值由hbase.hregion.max.filesize控制的。


## 第二部分 阅读摘要

#### RPC

RPC的概念（以下内容翻译自维基百科“remote procedure call”条目[2]）：在计算机科学中，一个远程过程调用（RPC）是一个进程间通信，它允许一个计算机程序在另一个地址空间（通常在共享网络的另一台计算机上）调用并执行一个子程序或者过程，它不需要程序员为这次远程通信明确地用代码写出相应的细节。这就是说，无论子程序是在本地执行还是在远程机器上执行，程序员实质上写的是相同的代码。当所讨论的软件使用了面向对象原理时，RPC被称为“远程调用”或者“远程方法调用”。

RPC中信息的传递：一个RPC操作是被客户端初始化——客户端向一个已知的远程服务器发送一条请求信息，请求执行一个指定的过程并提供了参数。该远程服务器向客户端发送一个应答，然后服务器上的应用继续执行客户端上的程序。当服务器处理这次调用时，除非客户端向服务器发送一个同步请求（比如一个XHTTP调用），客户端会被锁住（它会一直等待直到服务器完成处理，然后才重新运行）。

RPC与本地调用的区别：一个重要的区别在于远程调用可能因为不可预测的网络问题而失败。另外，调用者一般得在不知道远程过程是否真地被启动的情况下处理这类失败。

一次RPC操作的过程：
1. 客户端调用客户端存根。这个调用是本地过程调用，它把参数都以正常方式放入堆栈中。 
2. 客户端存根将参数打包进一个消息中并告诉系统将消息发出。打包参数的过程叫做封送处理（marshalling）。
3. 客户端的本地操作系统将消息从客户端机器发送至服务器机器上。
4. 服务器上的操作系统将收到的报文送到服务器存根中。
5. 服务器存根从消息中取出参数。
6. 最后，服务器存根调用服务器过程。服务器回复也是遵照相同的流程。

#### put操作[1.p78]

HBase客户端API中的一个put操作都是一个RPC操作。一个RPC操作在LAN网络中大约花1毫秒的时间（这个时间不包括传输数据的时间，即与传输的数据大小无关，是固定时间开销），即一个服务器1秒只能处理1000次RPC操作，即1000次put操作。当应用程序每秒存储上千行数据时就会出现问题。

#### 减少RPC操作次数的方法[1.p78]

将多次修改操作批量提交到服务器，具体为：开启客户端缓冲区，即调用函数setAutoFlush(false)，这样每一次客户端执行put操作都会将修改的数据存储到客户端进程的内存中，当需要在特定时刻将多个操作数据一起发送至服务器时，调用函数flushCommits()即可，这时只会进行一次RPC操作。但调用flushCommits()通常是不必要的，因为当缓冲区大小达到设定的值（默认是2MB）时，客户端会隐式调用刷写命令。

#### 隐式刷写和显式刷写[1.p80]

显式刷写即用户代码调用flushCommits()；隐式刷写即客户端不通过用户代码，自己调用flushCommits()，这在以下三种情况下会触发：
- 用户调用put()并且缓冲区超出限制；
- 用户调用setWriteBufferSize()并且缓冲区超出限制；
- 用户调用HTable类的close()方法。

#### HTable、HTablePool类及其实例[1.p68;1.p191]

HTable类及其实例：HBase主要客户端接口是由HTable类提供的，它位于org.apache.hadoop.hbase.client包中。创建HTable实例需要扫描.META.表并执行一些操作，非常耗时（数秒）。因此对于同一线程的多次请求，应复用同一个HTable实例。

HTablePool类及其实例：为什么使用HTablePool类？因为HTable类不是线程安全（_什么是线程安全？_）的，所以在多线程环境中复用HTable实例会出现问题，于是得为每一个线程创建一个HTable实例。

#### 客户端与服务器的连接[1.p194~195]

每个HTable实例（即客户端）都要与服务器建立连接。HBase内部的连接以键值对的形式存储。一个configuration实例作为一个键。即如果客户端使用同一个configuration实例，则它们共享一个连接，这可以使客户端初始化连接的开销（因为不用重复执行初始化）。共享连接的缺点是必须要用户显式关闭连接，连接才会断开。

#### HBase中RPC的实现[1.p197]

客户端与服务器、服务器与服务器之间的通信，都使用到了Hadoop RPC框架。这个框架中需要远程方法中的参数都实现Writable接口，因此，HBase的API提供的大多数类（尤其是与管理功能相关的所有类）都实现了Hadoop Writable接口。这要求各个有通信需求的客户端、服务器上的HBase的jar包都是相同的，以确保Writable的实现相同。

#### 管理功能API[1.p208]

概念范围：包括表、列族相关操作，以及HBaseAdmin类提供的API。这类API的实例不应该长期保留。

抛出的异常：IOException：客户端与服务器通信异常；InterruptedException：执行过程中的干扰异常

#### B+树与LSM树[1.p301]

区别：它们利用硬盘的方式不同。对于数据的插入或者更新这类操作，B+树需要对数据进行查找后执行相应操作，其插入或更新的速度受限于硬盘的寻道速率，LSM树则通过排序和合并文件来执行操作，其插入或更新的速度受限于硬盘的连续传输能力（_什么是连续传输能力？_）。

优缺点：B+树的优点是它保证了查询的速度，缺点是无法适应写入数据规模较大，速率较高的情况。LSM树的优点是能处理大量的数据，能保证稳定的数据插入速率，在查询连续键的记录时只会引发一次磁盘寻道（大大减少了磁盘寻道的时间），_缺点是？_

#### 临时目录文件[1.309]

`.tmp`：用于合并重写文件

`recovered.edits`：用于WAL回放时写入文件

`splits`：用于region拆分时创建子region文件结构

#### region拆分的过程[1.p310~311]

1. 父region目录创建splits目录
2. 关闭父region，不接受请求
3. splits目录中创建子region的目录
4. 将子region目录移到表目录下
5. .META.表中父region状态更新，记录父region和子region的信息
6. 在子region目录下创建父region存储数据的引用文件（文件名中散列值与父region相同）
7. regionserver打开两个子region，更新.META.表中子region的信息，使其上线
8. 将父region存储文件异步重写到子region中.tmp目录下，边合并边重写
9. 生成的重写后的文件自动取代引用文件
10. 父region在.META.表中被删除，它所有的文件在硬盘中被删除
11. master被告知拆分的情况。

#### storefile合并（compaction）[1.p311]
   
合并的种类：
- minor合并：将最后生成的几个文件重写到一个更大的文件中
- major合并：将所有文件压缩成一个文件

触发合并的事件：压缩检查并符合一定条件

触发压缩检查的事件：
- memstore刷写磁盘后
- shell命令或API调用compaction
- hbase异步线程，以固定周期执行检查

何时发生major合并：
- shell命令或API调用major_compaction
- 压缩检查后发现从上次运行major合并到现在的时间超过时限（时限由hbase.hregion.majorcompaction控制）
- 压缩检查后触发minor合并，并且合并包含了所有存储文件，minor合并可能被提升为major合并

何时发生minor合并：
- 压缩检查后发现符合大小的文件数量超过阈值并且不发生major合并（文件大小和数量阈值由hbase.hstore.compaction.min.size和hbase.hstore.compaction.min控制）

#### HFile[1.p313]

HFile以文件的形式存储在HDFS上，即某一个regionserver上的某一个region的文件不一定存储在本地，而是被分割成HDFS的块分布在HDFS所在的集群上。HFile包含多个块，块大小为64KB（可设置），块可分为meta块、trailer块、data块、fileinfo块等种类，其中data块的组成如下：

| Magic | k,v | k,v | ... | k,v |
| ----- | --- | --- | --- | --- |
   
上表中(k,v)的数据格式如下（说明：斜体字段的值为不定长） 
 
| keylength | valuelength | rowlength | _row_ | colFlength | _colF_ | _col_ | ts  | keytype | _value_ |
| --------- | ----------- | --------- | ----- | ---------- | ------ | ----- | --- | ------- | ------- | 

#### 修改数据的过程[1.p317]
  
每个put、delete、increment操作都被封装成一个keyvalue对象，使用rpc的方式送往对应regionserver，由相应的region对象接收，并写入日志，然后放入memstore中

#### WAL[1.p317~319]

概念：预写日志（write-ahead log），由于hbase更新数据时将数据放入内存，等其增长到一定大小才永久性写入磁盘，因此会有数据丢失的风险。预写日志就是为了在内存中数据丢失时能够将数据恢复。日志文件存放在磁盘中。
日志文件的格式：见下表

| 1 | 2 | 3 | ... | N |
| --- | --- | --- | --- | --- |
| k:HLogKey | k:HLogKey | k:HLogKey | ... | k:HLogKey |
| v:WALEdit | v:WALEdit | v:WALEdit | ... | v:WALEdit |

| HLogKey | WALEdit |
| ------- | ------- |
| region名和表名         | 修改操作的keyvalue对象 |
| 序列号                 |
| 修改被写入日志的时间戳 |
| 集群ID                 |

#### 日志滚动[1.p320]

概念：旧日志文件关闭，开始使用新日志文件

触发条件：
- 达到块大小的一定比例：块大小由hbase.regionserver.hlog.blocksize控制，一定比例由hbase.regionserver.logroll.multipler控制
- 达到一定周期：周期由hbase.regionserver.logroll.period控制
- 旧日志被删除的触发条件：达到一定周期，检查存储文件中最大的序列号A，若日志文件最大序列号比A小，该日志文件会被移到.oldlogs目录下 

#### 客户端缓存的服务器信息[1.p328]
   
- -ROOT-表的位置
- -ROOT-表内容
- .META.表内容


## 第三部分 参考文献

[1]HBase权威指南，Lars George，人民邮电出版社，2013.10

[2]http://en.wikipedia.org/wiki/Remote_procedure_call

