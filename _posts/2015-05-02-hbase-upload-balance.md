---
layout: page
title: HBase数据导入的负载均衡策略
comments: true
---

---

本文阐述HBase数据导入时进行负载均衡的策略，以期达到最大的导入速率，主要为写多读少的业务服务。

---

## 1.问题描述

由于在开始建表时，表只会有一个region，并随着region增大而拆分成更多的region，这些region才能分布在多个regionserver上从而使负载均分。对于写负载很大的业务，如果一开始所有负载都在一个regionserver上，则该regionserver会承受不了而导致数据丢失。因此，有必要在一开始就将HBase的负载均摊到每个regionserver。

## 2.策略

要将负载均摊，可用的方法就是建表时将表分区，将这些分区均匀地放到每个regionserver上，然后客户端在进行写操作的时候，将这些写操作均匀分布到各个分区上。要实现这个策略，需要了解HBase的机制以及Hash技术。

## 3.HBase的机制

1. HBase建表时可以预定义分区（region），即先预设表的起始行键和终止行键，再指定分区的数目，并根据这些参数来将表均匀划分成多个分区。
2. HBase每个region里面的行键都是有序的，按字典序排序并存储。
3. 一个region在拆分后，它的两个子region仍然会停留在父region所在的regionserver上，直至默认的负载均衡机制启动后，才可能被迁移至别的regionserver。
4. 默认的负载均衡机制是根据region数目来均衡的，它会使每个节点的region数目趋向一致。
5. region拆分时，是按照最大的HFile文件对半拆分成两个region的，即在这个最大的HFile文件中，找到能将其对半分割的rowkey A，父region的起始行键至rowkey A的所有行都分给子region A，从rowkey A至父region的终止行键的所有行都分给子region B。
6. HBase可以通过设置起止行键来缩小查询扫描的范围。

（上述几点截至HBase 0.92）

## 4.Hash技术

Hash技术（又称散列，哈希）可以将一个大的集合（数集，字符串集等）映射到另一个相对小的数集。
它有不同的算法和实现。针对不同的数据类型采用不同的方法。对于正整数，通常使用除留余数法，即取模，除数M
通常是质数，这是为了充分利用key的每一位数字的信息，如果是合数，则可能导致我们无法均匀散列散列值。
举个例子，M=10^k，当key分布符合某些特征，如大部分key倒数第二位为0，那么所有key的散列值都会集中在10以内。
对于浮点数，常用的散列法是先把数转换成二进制表示，再使用除留余数法。除留余数法的实现可以使用除法`key mod M`，也可以使用
位操作`key & (M-1)`。对于字符串，常用的散列法是加法乘法混合、位移法，具体可以参考[这篇博文](http://blog.csdn.net/eaglex/article/details/6310727)。

Hash技术的一个主要应用是快速查找，如哈希表。

快速查找的目标是查找某个元素是否存在，或者以某个元素为key来查找它所对应的value，这两种都可以使用Hash技术，这里以前者为例。前者需要在一个数组中查找某个元素是否存在，若存在，则返回1，若不存在，则返回0，也可顺带将新元素存储到数组中。将要查找（存储）的元素集合通过哈希函数映射（接近一一映射）到一个数组的下标数集，来使得查找元素时可以用哈希函数来计算它在数组中的下标来访问元素，这样的查找时间复杂度为O(1)。这种查找要关注的问题是一一映射，如果不同元素映射到相同下标的话就会产生冲突，冲突的概率取决于元素集合（无重复元素）的大小。因此需要根据元素集合的大小来确定合适的数组大小，以避免过多地发生冲突。为了确保发生冲突后不会引起错误，解决方案1是拉链法，即采用数组+链表来存储元素，即一个数组元素是一个链表的头指针，这样可以将拥有相同哈希值的不同元素存储到这个链表中，查找时先找到对应的链表，再将查找元素与链表中所有的元素一一比对，以确定是否存在这个元素；解决方案2是线性探测法，即只使用一个数组来存储元素，当发现要被存储的元素与其他已存储元素的散列值相同，在数组中已经被占坑了，那么就把数组下标加1，检查该位置是否为空，是则存进去，否则继续检测。在查找时，如果散列值所对应的位置不为空，但存储的元素不是要被查找的那个，那么使用数组下标下移继续查找，直到遇到空元素或者相同的元素，遇到空元素说明哈希表中不存在被查找元素。

快速查找的性能取决于冲突的频率，key的散列值分布越均匀，越不容易出现冲突。常用的除留余数法（key mod M），这种方法的缺点在于当M越来越接近元素集合大小N时，散列值的分布将越来越不均匀。（参考维基百科[哈希函数](http://en.wikipedia.org/wiki/Hash_function#Hashing_uniformly_distributed_data)）

在redis数据库技术中，它的实现是使用crc16(key) mod 16384来将key均匀映射到16384个slot上。

## 5.均衡负载

### 5.1.方案1

在建表时预定义region，region数目与regionserver相同，然后将散列值集合为region数目的N倍（N=1,2,3,...）均匀分配给各个region，如散列值集合为0～15，region总数为8，则可以按照0~1，2～3，4～5将散列值区间依次分配给8个region。在对HBase进行插入操作时，可以把记录中的某个特征用来计算散列值。计算出散列值之后将其作为前缀添加到原始rowkey中构成新的rowkey，这样，散列值为1的记录就会被插入到0～1区间的region了，从而负载会均摊到各个regionserver。随着region的拆分和迁移，写负载仍然会近似均匀地分布在所有regionserver。

数据上传之后要接受查询，单条查询时可以根据记录的特征来计算散列值，并组成新的rowkey，来查询记录；范围查询也是如此。但是，如果查询的rowkey中没有提供用于计算散列值的特征，那么就得按照散列值集合的大小（按照上述例子的话就是16）开启对应数目（即16）的线程用于扫描所有分区（因为HTable实例不是线程安全的，因此不同线程必须使用不同HTable，可以使用HTable pool）。用单个线程的话，也可用一个循环来扫描。

### 5.2.方案2

同样预定义region，设置好每个region起始行键和终止行键，总的范围是1～N，N为regionserver的数目或者倍数。插入记录时也是对rowkey加前缀，但是该前缀采用非散列的方法，即在开始上传记录时，设置一个计数器，每次来一个记录后计数器加一，然后取计数器的值为前缀，当计数器的值达到N时重置为1，这样可以确保各个regionserver的写入负载是均匀分布的。在这里称行键前缀范围`[i, i+1](0<i<N)`的所有记录为一个逻辑分区。

这样会有一个明显的缺点，在上传时，客户端需要跟N个regionserver保持连接，在查询时，无论是连续区域查询，还是单条查询，都需要访问N个逻辑分区，这意味着与N个regionserver都要建立连接。当集群规模较大（即N较大）时，连接开销是巨大的。因此，这种方案只适用于小规模集群（N小于20）。在小集群情况下，直接让客户端遍历所有分区时，如果采取多线程查询，启动线程数为k（k小于等于N），不同线程的查询负载会落到不同regionserver上。也就是说，原本一个regionserver执行一次操作就能完成的查询，采取方案后每个regionserver都要执行一次操作才能完成。

如果要在大集群上部署，可以采用分布式客户端集群，即每个上传/查询客户端只会上传/查询属于一个逻辑分区的数据。对于查询客户端集群，可以使用主从结构，即一个主客户端解析和分配查询任务，每个从客户端与regionserver位于同一个节点上，负责对一个逻辑分区进行查询后返回结果给主客户端，主客户端汇总结果后返回给用户。主客户端可以使用pull的方法（如同hadoop中的reducer），开启少数线程进行归并。在不使用分布式客户端集群时，因为客户端只能启动少数扫描线程（因为每个线程中会有一个独立的连接和返回的结果，如果开启与逻辑分区相同数目的线程，客户端会占用巨大的网络和内存资源），所有逻辑分区的查询并发数取决于客户端启动的线程数，导致HBase集群节点数多的时候性能下降。而使用分布式客户端集群时，主客户端和从客户端都是自定义的，仅仅需要让主客户端给每个从客户端提交查询任务后等待即可，从客户端完成查询后将结果写入本地磁盘，并通知主客户端结果已经完成，主客户端再用少数线程从从客户端那里拉取结果（可以使用nfs）后归并。

### 5.3.前缀范围N的确定

N的值在建表时就确定下来，它取决于regionserver的数目，以便使大量的写负载可以均摊到所有的regionserver上，从而最大化写入速度。同时它也带来了读负载的倍化。因此，需要在这两者间作权衡，如果写负载均摊到部分而非所有regionserver上就可以满足写入速度需求，同时要有相当量的读负载，那么就尽量不再增加N，以使读取速度满足需求。

### 5.4.节点的增加

如果业务的写负载增大，即使N值足以把负载均摊到所有regionserver也无法满足写入速度的需求，那么就需要增加节点，同时N值也会改变。而无论以上哪种方案，如果是需要遍历所有分区的查询，客户端都需要在查询时知道rowkey前缀的范围N。同时，客户端在导入时也需要知道N值已经改变。考虑到在增加节点时服务不能中断，即客户端不能够修改代码，则需要客户端能够在写入时自行修改N值，在查询时自行获取N值。对于写入模块，由于N值是建表时就已经确定，所以节点增加后，只能在创建下一个表（业务需求是按小时建表）时才能修改N值，方法可以是通过HBaseAdmin API获取集群信息以根据regionserver数目来修改N值，并在表中加入一行，rowkey = totalPartition，col = N。对于查询模块，则直接从表中获取该值就行。

### 5.5.自动负载均衡的问题

HBase有自动负载均衡机制，它可能会使得正在执行写入的表的各个region无法被均匀地分布到所有regionserver上，因为HBase中已有许多region，而HBase自动负载均衡机制只关注region的数目而不关注region的负载，因此它并不知道哪些region有大量的写负载，因此在自动负载均衡机制启动后，它对所有region都一视同仁，那么就有可能造成部分或所有写负载的region都被集中到一个regionserver上。解决方案1是关闭自动负载均衡机制，并使用客户端在每次建表之前手动执行负载均衡。解决方案2是修改HBase的源码，定制自己的自动负载均衡机制，如让它按照regionserver的负载去均衡而不是按regionserver拥有的region数目。

> 补充引用：0.94之后的新的负载均衡策略

> StochasticLoadBalancer负载均衡器首先会根据每个region server上的region个数作决定要不要进行rebalance，具体方法是算出所有server的平均region个数，然后根据配置项hbase.regions.slop产生一个区间[floor(average * (1-slop)), ceil(average * (1+slop))]，配置项默认0.2，如果region 个数最多的region server不比右区间大，并且region个数最少的region server不比左区间小，则说明region个数比较平均，就不进行rebalance，直接退出，等待下次调度。否则，计算当前集群状态的cost值，这个cost值的计算会考虑到移动region的成本、region 本地化策略、region count分布、每个server上table的分布等做一个加权平均。然后一共迭代computedMaxSteps次，次数由配置项hbase.master.balancer.stochastic.maxSteps和hbase.master.balancer.stochastic.stepsPerRegion、当前集群的region个数，server个数共同决定。每次迭代，都会随机选择一种pick region的策略，一共有三种，分别为RandomRegionPicker，LoadPicker和LocalityBasedPicker。随机选定一个picker策略后，这个picker就会从集群中选出两个用于相互交换的region或者选出一个用于迁移到其他server的region，然后更新集群状态的数据结构，重新计算当前集群状态的cost值，如果发现新的cost比原来的小，则说明，这种region的交换或者迁移是有效的。每次迭代都是基于上次的成果，总共做computedMaxSteps。最后产生出一系列的plan，每个plan就是交换region或者迁移region。对于所有的表都做一次，把所有的plan都放入AssignmentManager的regionsPlans中。然后对于每个plan，都调用assignmentManager.balance(plan)，这个函数会调用unassign()方法，首先在zk上为这个region创建/hbase/region-in-transition/region_encoded_name节点，节点内容为这个原来在某个server上的region处于closing状态了，然后给这个region原来所在的server发送close region命令对region进行卸载，随后再调用public void assign(HRegionInfo region, boolean setOfflineInZK)给region的目标region server发送open region的命令，目标region server是从regionPlans中查到的。最后删除zk上的节点。其中，每次做完一个plan后都会检查是否时间到了。

参考:
   
- [1](http://www.cnblogs.com/niurougan/p/3975433.html)  Hbase负载均衡流程以及源码 - albeter - 博客园
- [2](http://www.cnblogs.com/foxmailed/p/3899574.html)  HBase 负载均衡 - emailed - 博客园

### 5.6.方案1还是方案2？

方案1：
   
- 缺点
   - 在插入/查询时都引入了计算散列值这一步骤，会增加一些程序的开销（可能影响不大）
   - 散列函数不好实现，需要测试数据在散列后分布是否均匀。如果不均匀，将有可能造成某个regionserver负载达到上限，无法使负载均衡带来的写入速度最大化
- 注意的地方
   - 使用散列来分区，这是参考redis的解决方案，但是因为redis没有HBase所使用的LVM树，它并没有行键排序和连续区域查询的功能，而是纯粹的单条随机访问，它的查询可以直接使用key来计算散列值，所以使用散列来分区是合适的。而HBase因为有了行键排序的机制，所以利用这种机制可以进行连续区域的查询，业务在进行存储模型的设计时也会利用这个特点，因此不合理地使用散列可能会将这个特点破坏掉，但这可以在设计时避免。举个例子，当key是时间时，某次查询需要查询08:00:00~08:50:00的数据，这时，08:00:00的key映射到逻辑分区1，08:00:01的key映射到逻辑分区2，08:00:02映射到逻辑分区3，对于每个key，都需要计算它的逻辑分区号，这本来可以用一个scan操作来查询，但实际上需要使用3000次get操作。因此，要避免使用那些值顺序是有意义的特征作为散列key。
   - 散列key必须可以由rowkey得出，否则仍然需要遍历所有逻辑分区。
- 优点
   - 相对于方案2，不用遍历所有逻辑分区，即使集群规模很大，使用一个查询客户端即可完成查询。

方案2：

- 缺点
   - 查询时需要遍历所有逻辑分区。
- 优点
   - 可以保证任意时间点写负载都是均匀分布到各个regionserver的
   - 不会给程序带来额外的计算开销。

集群规模的影响：

- 大规模集群
   - 大规模集群情况下，单个regionserver导入速率总是比较小的，即使方案1可能无法均匀分布负载，但热点regionserver也不会因此挂掉
- 小规模集群
   - 集群规模小，又要求导入速率高，几乎每个regionserver都需要达到导入速率的上限。这种情况下，只要有一个regionserver成为热点，就可能直接挂掉。因此，需要通过测试来在两种方案之间做选择。测试的内容为方案1是否能均匀分布负载（波动范围有多大），以及客户端计算散列带来的开销（内存与CPU使用）是否有较大影响（相比方案2）。如果方案1可以均匀分布负载，并且计算散列带来开销不大，那么就可以用方案1代替方案2，因为方案1客户端查询时不用遍历所有逻辑分区。

## 6.HBase的region拆分策略（补）

以下资料截至0.94版本。参见[这里](http://hortonworks.com/blog/apache-hbase-region-splitting-and-merging/)。

有三种策略：

- ConstantSizeRegionSplitPolicy：在0.94之前只有这个策略。当region中的一个store（对应一个columnfamily的一个storefile）超过了配置参数`hbase.hregion.max.filesize`时拆分成两个，该配置参数默认为10GB。region拆分线是最大storefile的中间rowkey。
- IncreasingToUpperBoundRegionSplitPolicy：0.94默认策略。拆分阈值是`Min (R^2 * “hbase.hregion.memstore.flush.size”, “hbase.hregion.max.filesize”)`，其中R是一张表中位于同一个regionserver的region的数目。`hbase.hregion.memstore.flush.size`默认是128MB，后一个参数默认是10GB。因此，如果region没有预拆分，默认的行为是，表的第一个region会位于一个regionserver上，然后当达到1^2*128MB=128MB，region被拆分成2个，它们仍然位于同一个regionserver上，因此，随着region的增加，拆分阈值也增加：128MB、512MB、1152MB、2GB、3.2GB、4.6GB、6.2GB，依次类推，直至达到9个region之后，拆分阈值就恒定为10GB。这可以控制一个regionserver拥有的region个数在小数据量的时候不会太少，在大的数据量时候不会太多。region拆分线是最大storefile的中间rowkey。
- KeyPrefixRegionSplitPolicy：你可以配置用来对你的rowkey进行分组所依赖的前缀长度，然后该策略保证region的拆分线不会在一组拥有相同前缀的rowkey的中间，也就是说，拆分后的region中，相同前缀的rowkey会一直位于同一个region上。其他的拆分策略与IncreasingToUpperBoundRegionSplitPolicy是一样的。

如何配置策略？

通过配置参数`hbase.regionserver.region.split.policy`来配置，这是全局的。可以针对单独的表进行配置，用表的HTableDescriptor的setValue()可以配置。另外，后一种配置还可以使用自定义的拆分策略类。
