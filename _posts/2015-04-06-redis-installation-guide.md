---
layout: post
title: 如何安装Redis
comments: true
---

---

本文说明如何安装部署Redis-3.0.0。

---

### 1. 单节点

#### 1.1. 下载
官网下载最新版本。目前是redis-3.0.0.tar.gz。

将压缩包上传到服务器。

#### 1.2. 解压

```shell
tar xzf redis-3.0.0.tar.gz
```

#### 1.3. 安装

```shell
cd redis-3.0.0
# 编译
make
# 将执行命令放到/usr/local/bin目录下，使得命令可直接使用
make install
```

#### 1.4. 测试

```shell
# 该步骤可选非必须
make test
```

#### 1.5. 启动

```shell
redis-server
```

会看到输出信息：

```shell
811:C 10 Apr 19:58:16.355 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
811:M 10 Apr 19:58:16.357 # Server started, Redis version 3.0.0
...其他日志信息
811:M 10 Apr 19:58:16.357 * The server is now ready to accept connections on port 6379
...其他日志信息
```

这种启动只为了开发和测试，如果要部署在生产环境中，需要指定配置文件。使用安装目录（即解压目录）中的redis.conf文件作为配置文件的模版。

指定配置文件启动redis：`redis-server /path/to/your/redis.conf`。

#### 1.6. 测试是否正常工作
Redis提供一个命令行工具来给Redis发送命令，叫redis-cli。

```shell
redis-cli
```

它可以带参数，这样只能执行一次命令，不带参数则启动交互模式。参数--help可以看用法。

```shell
127.0.0.1:6379> set mykey somevalue
OK
127.0.0.1:6379> get mykey
"somevalue"
127.0.0.1:6379>
```

Redis可以正常工作了。

关于Redis数据结构可以参考：[十五分钟介绍 Redis数据结构](http://blog.nosqlfan.com/html/3202.html?ref=rediszt)。

关于Redis命令，可以参考：[中文官方网站-命令](http://www.redis.cn/commands.html)。

#### 1.7. 停止

```shell
redis-cli shutdown
```

#### 1.8. Redis持久化
参考文档[Redis 持久化](http://www.redis.cn/topics/persistence.html)

- **主要概念**
   - 持久化有RDB和AOF两种方式。
   - RDB能够在指定时间间隔对数据进行快照存储，快照的数据范围可以指定。
   - AOF记录每次对服务器的写操作，在服务器重启的时候重新执行这些命令来恢复原始数据。
   - 可以同时开启两种持久化方式，这样在重启时会优先载入AOF的数据集，因为它会比RDB完整。
   - 在恢复大数据集时，RDB方式会更快。
   - AOF可以将数据丢失的风险降到最小，但相同数据集其体积会大于RDB。
   - 需要在redis.conf中对两种持久化方式进行配置。

- **如何配置**
   - 默认配置下，Redis只会按一定规则将数据持久化，如果需要在重启之后加载所有数据，则要在重启前使用`save`命令将数据手动持久化，或者在重启前使用`shutdown`命令将数据库关闭（这样Redis会在退出之前将数据持久化）。默认配置中的RDB和AOF在下面提到。
   - 手动配置AOF。可以在默认配置文件redis.conf中找到`appendonly no`一行，前者为变量，后者为值，以空格隔开。将值由no改为yes，即可打开AOF持久化。
   - 手动配置RDB。在redis.conf配置文件中有`save 900 1`，`save 300 10`，`save 60 10000`三行，第一行表示每900秒（即15分钟）至少有1个键被改动时自动保存一次数据集。若要关闭RDB，将这三行注释掉或者添加一行`save ""`。

#### 1.9. 内存的配置
由于随着数据增多，内存会无限增大，超过上限会造成数据丢失，因此需要对内存进行配置以给予用户提醒。

- **相关设置**
    - 下述配置中，安装centos的服务器默认是没有配置前两条的，**修改时必须征求集群管理员的意见**。
    - 确保设置Linux内核`overcommit memory setting`为1。向`/etc/sysctl.conf`添加`vm.overcommit_memory = 1`然后重启，或者运行命令`sysctl vm.overcommit_memory=1`以便立即生效。这个配置会影响到fork()后台保存（见[FAQ](http://www.redis.cn/topics/faq.html)）。
    - 确保禁用Linux内核特性transparent huge pages，它对内存使用和延迟有非常大的负面影响。通过命令`echo never > sys/kernel/mm/transparent_hugepage/enabled`来完成。
    - 确保你的系统设置了一些swap（我们建议和内存一样大）。如果linux没有swap并且你的redis实例突然消耗了太多内存，或者Redis由于内 存溢出会当掉，或者Linux内核OOM Killer会杀掉Redis进程。
    - 设置一个明确的maxmemory参数来限制你的实例，以便确保实例会报告错误而不是当接近系统内存限制时失败
    - 如果你对一个写频繁的应用使用redis，当向磁盘保存RDB文件或者改写AOF日志时，redis可能会用正常使用内存2倍的内存。额外使用的内存和保存期间写修改的内存页数量成比例，因此经常和这期间改动的键的数量成比例。确保相应的设置内存的大小。
- **maxmemory配置**
    - 在redis.conf配置文件中，有`# maxmemory <bytes>`一行，<bytes>即为redis可使用的最大内存字节数。还有`# maxmemory-policy noeviction`一行，它指定了超过配置的内存上限时redis会执行什么策略，`noeviction`策略会在使用内存超过`maxmemory`的值时禁止客户端进行任何使用更多内存的操作（会抛出错误），但仍可执行读操作。当当前实例有slaves节点时，需要将`maxmemory`设置小一点，以使服务器有额外的内存作为对slaves的输出缓冲区。

可以设置的策略有：

```shell
# volatile-lru -> 依照LRU算法移除过期Set的key
# allkeys-lru -> 依照LRU算法移除任一key
# volatile-random -> 移除任一有着过期Set的key
# allkeys-random -> 移除任一key
# volatile-ttl -> 移除最近的过期key (minor TTL)
```

配置文件中的`# maxmemory-samples 5`一行可以设置LRU和minor TTL算法的准确性。

#### 1. 10.对Redis的配置
Redis可以通过命令进行运行时配置，无需重启。但升级程序则需要重启。若要保证服务停止时还能正常接受请求，则需要有额外的实例来同步主实例。

#### 1.11. 正式安装
上面的安装仅为测试和开发，如果要投入生产环境，需要进行正式安装。

- **日志文件的路径**
在配置文件中，修改以下参数

```shell
logfile /path/to/your/log
```

### 2. 集群

#### 2.1. 概述

![img1](https://raw.githubusercontent.com/lsr1991/lsr1991.github.io/master/image/2015-04-06-redis-installation-guide-1.png)

各个节点通过一个服务频道直接相连，彼此对等。节点间通信协议是二进制的。客户端可与多个节点通信。节点无法代理查询。节点间并不是全连接的，它们会与其他人交换自己所知道的信息，像路由表更新一样。

Hash slots：keyspace被分为4096个hash slots。不同的节点会拥有这些hash slots的一个子集。一个key通过crc16(key) mod NUMBER_SLOTS来计算出它属于哪个slot。

集群中的备份通过master和slave的形式来进行。master是服务节点，slave是备份节点。拥有不同数据的slave和master节点可以位于同一个物理机器，节点的分配通过redis-trib集群管理程序来控制，以使得备份位于不同物理机器上。

- **哑客户端**
    - 客户端查询时节点会负责计算查询的key位于哪个slot，然后告诉客户端这个slot的节点地址，客户端再去拥有被查询key的节点上查询并获取value。
- **智能客户端**
    - 客户端会从节点获取集群的信息（每个节点拥有什么slot），然后自己计算出key所在的slot并找到节点，对那个节点进行查询。
- **客户端的选择**
    - 已有的客户端代码就是一个哑客户端，而一个智能客户端将与所有节点保持连接，并当遇到节点转移的错误时更新缓存的集群信息表。通常这种模式是水平可扩展的，智能客户端的时延较短。当客户端与节点连接较多时，需要共享这个客户端对象。
- **重分区**
    - 要将一个slot 7从C迁移到新增的另一个节点D上，首先会将插入操作都移到D上，然后对C的slot 7使用migrate命令将数据迁移到D上，在数据全都迁移完之前C和D都会接受查询请求。
- **容错**
    - 所有节点会彼此交换信息，信息中包含对可能失败的节点的判断，当有两个节点同时认为某个节点失败时，这个节点会被集群认为是失败的，所有节点都将放弃它。被放弃的节点即使恢复了也无法连接集群，会被告知关闭ASAP服务。使它重新加入集群的办法是手动运行redis-trib。
- **redis-trib**
    - 被用于创建一个新的集群，当你启动N个空节点。
    - 被用于检查集群是否一致。如果有hash slot同时位于两个节点则会进行修复。
    - 被用于增加新的节点到集群，无论是已有的master节点的slave，还是要用于重分区以减少其他节点负载的空节点。


#### 2.2. 部署redis节点
将一个已编译好的redis包拷贝到所有节点上。将src目录下的redis-*的可执行文件（不带.o和.c后缀）都拷贝到节点的/usr/bin/目录下。

#### 2.3. 部署配置文件
修改每个节点的配置文件中以下参数：

```shell
# 指定节点使用的端口号，被用于接受客户端访问
port 7000
cluster-enabled yes
cluster-config-file /path/to/your/redis-dir/nodes.conf
cluster-node-timeout 5000
appendonly yes
```

#### 2.4. 启动各个节点

```shell
redis-server /path/to/your/redis.conf
```

启动之后，用`ps aux | grep redis`命令查看，可以看到redis进程：

```shell
root 22567 0.0 0.0 137420 9276 ? Ssl 22:26 0:00 redis-server *:7005 [cluster]
```

**注意**，如果有`redis-server *:6379`一行说明是之前启动过的单机redis，一个`redis-server`命令只会启动一个进程。

#### 2.5. 创建集群
创建集群通过一个ruby程序`redis-trib.rb`来完成，这个程序位于redis安装目录下src目录中。它不依赖于当前主机中是否有redis进程，因此可以在redis集群之外的主机上运行（该主机需要在网络上能连接上集群各节点）。

运行命令：

```shell
./redis-trib.rb create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 \
127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005
```

可以创建一个三个Master节点三个slave节点的集群。

**注意**：指定各个节点的网络地址不能使用主机名，只能用ip，否则会出现错误如下：

```shell
/usr/lib64/ruby/gems/1.8/gems/redis-3.2.1/lib/redis/client.rb:113:in `call': ERR Invalid node address specified (Redis::CommandError)
from /usr/lib64/ruby/gems/1.8/gems/redis-3.2.1/lib/redis.rb:2556:in `method_missing'
from /usr/lib64/ruby/gems/1.8/gems/redis-3.2.1/lib/redis.rb:37:in `synchronize'
from /usr/lib/ruby/1.8/monitor.rb:242:in `mon_synchronize'
from /usr/lib64/ruby/gems/1.8/gems/redis-3.2.1/lib/redis.rb:37:in `synchronize'
from /usr/lib64/ruby/gems/1.8/gems/redis-3.2.1/lib/redis.rb:2555:in `method_missing'
from ./redis-trib.rb:621:in `join_cluster'
from ./redis-trib.rb:619:in `each'
from ./redis-trib.rb:619:in `join_cluster'
from ./redis-trib.rb:854:in `create_cluster_cmd'
from ./redis-trib.rb:1075:in `send'
from ./redis-trib.rb:1075
```

当创建集群失败后，各节点仍然会生成nodes.conf的配置文件，因此若直接重新创建集群，会出现以下错误：

```shell
/usr/lib64/ruby/gems/1.8/gems/redis-3.2.1/lib/redis/client.rb:113:in `call': ERR Slot 16011 is already busy (Redis::CommandError)
from /usr/lib64/ruby/gems/1.8/gems/redis-3.2.1/lib/redis.rb:2556:in `method_missing'
from /usr/lib64/ruby/gems/1.8/gems/redis-3.2.1/lib/redis.rb:37:in `synchronize'
from /usr/lib/ruby/1.8/monitor.rb:242:in `mon_synchronize'
from /usr/lib64/ruby/gems/1.8/gems/redis-3.2.1/lib/redis.rb:37:in `synchronize'
from /usr/lib64/ruby/gems/1.8/gems/redis-3.2.1/lib/redis.rb:2555:in `method_missing'
from ./redis-trib.rb:203:in `flush_node_config'
from ./redis-trib.rb:602:in `flush_nodes_config'
from ./redis-trib.rb:601:in `each'
from ./redis-trib.rb:601:in `flush_nodes_config'
from ./redis-trib.rb:851:in `create_cluster_cmd'
from ./redis-trib.rb:1075:in `send'
from ./redis-trib.rb:1075
```

原因是已经有Slot被分配给之前创建失败的集群节点。因此若要重新创建集群，必须把各个节点的nodes.conf文件给删除掉，再重新创建。

创建成功后会出现下面信息：

```shell
>>> Creating cluster
Connecting to node 192.168.80.17:7000: OK
Connecting to node 192.168.80.18:7001: OK
Connecting to node 192.168.80.19:7002: OK
Connecting to node 192.168.80.21:7003: OK
Connecting to node 192.168.80.22:7004: OK
Connecting to node 192.168.80.25:7005: OK
>>> Performing hash slots allocation on 6 nodes...
Using 3 masters:
192.168.80.25:7005
192.168.80.22:7004
192.168.80.21:7003
Adding replica 192.168.80.19:7002 to 192.168.80.25:7005
Adding replica 192.168.80.18:7001 to 192.168.80.22:7004
Adding replica 192.168.80.17:7000 to 192.168.80.21:7003
S: 46f9fc202a05117c254ff5ef8258643c424cdee5 192.168.80.17:7000
replicates 6fce57eea7342f2644642480cadaad24dd39aede
S: 5f22e2686948fee6ad4e0dc4d0d0698ecf2a8d64 192.168.80.18:7001
replicates 575237ad3771be258c9eef83909baddb96698130
S: b1f5bc54e4917da6f6ed878225519fd33cbbf207 192.168.80.19:7002
replicates d1ad37beca1c1c6067c4755d7ef22b6360d6d341
M: 6fce57eea7342f2644642480cadaad24dd39aede 192.168.80.21:7003
slots:10923-16383 (5461 slots) master
M: 575237ad3771be258c9eef83909baddb96698130 192.168.80.22:7004
slots:5461-10922 (5462 slots) master
M: d1ad37beca1c1c6067c4755d7ef22b6360d6d341 192.168.80.25:7005
slots:0-5460 (5461 slots) master
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join...
>>> Performing Cluster Check (using node 192.168.80.17:7000)
S: 46f9fc202a05117c254ff5ef8258643c424cdee5 192.168.80.17:7000
replicates 6fce57eea7342f2644642480cadaad24dd39aede
M: 5f22e2686948fee6ad4e0dc4d0d0698ecf2a8d64 192.168.80.18:7001
slots:5461-10922 (5462 slots) master
replicates 575237ad3771be258c9eef83909baddb96698130
S: b1f5bc54e4917da6f6ed878225519fd33cbbf207 192.168.80.19:7002
replicates d1ad37beca1c1c6067c4755d7ef22b6360d6d341
M: 6fce57eea7342f2644642480cadaad24dd39aede 192.168.80.21:7003
slots:10923-16383 (5461 slots) master
M: 575237ad3771be258c9eef83909baddb96698130 192.168.80.22:7004
slots:5461-10922 (5462 slots) master
M: d1ad37beca1c1c6067c4755d7ef22b6360d6d341 192.168.80.25:7005
slots:0-5460 (5461 slots) master
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

- **关于运行该脚本出现的问题及解决办法**

`redis-trib.rb`依赖rubygems和redis两个gem包。需要先安装这两个包。

首先是rubygems，可到网上下载tar包。解压后进入目录运行`ruby setup.rb`。但是会出现错误：

```shell
ERROR: While executing gem ... (Gem::DocumentError) ERROR: RDoc documentation generator not installed: no such file to load -- rdoc/rdoc
```

这是因为在centos 6.5中，若安装操作系统时已经安装了ruby，那么是ruby 1.8.7版本（`ruby -v`），这个版本默认没有带`rdoc`。

因此需要运行命令`sudo gem install rdoc`安装rdoc，但是直接执行此命令会出现错误：

```shell
ERROR: Error installing rdoc: ERROR: Failed to build gem native extension. /usr/bin/ruby -r extconf.rb mkmf.rb can't find header files for ruby at /usr/lib/ruby/ruby.h
```

解决方法是安装ruby-devel包，`sudo yum install ruby-devel`。

另外，rdoc安装还会报以下错误：

```shell
Errno::ECONNRESET: Connection reset by peer - SSL_connect (https://api.rubygems.org/quick/Marshal.4.8/rdoc-4.2.0.gemspec.rz)
```

这是因为gem的远程库链接被墙了，得用国内的。通过以下命令修改并安装rdoc。

```shell
$ gem sources --remove https://rubygems.org/
$ gem sources -a https://ruby.taobao.org/
$ gem sources -l
*** CURRENT SOURCES ***

https://ruby.taobao.org
# 请确保只有 ruby.taobao.org
$ gem install rdoc
```

安装完rdoc之后，rubygems就可以正常安装。

之后是安装redis的gem包，运行命令`sudo gem install redis`。

如果没有安装这个redis包，会报出错误：

```shell
/usr/lib/ruby/site_ruby/1.8/rubygems/core_ext/kernel_require.rb:54:in `gem_original_require': no such file to load -- redis (LoadError)
```

只要redis的gem包安装后错误就会消失。

至此，`redis-trib.rb`可以正常运行。


#### 2.6. 测试
客户端可以使用`redis-cli`，但是该客户端不适合与java程序整合，因此要用主流的`Jedis`客户端。在集群支持上，`Jedis`客户端版本`2.6.2`无法兼容redis`3.0.0-beta*`版本，因此需要安装`redis-3.0.0-stable`版本。








