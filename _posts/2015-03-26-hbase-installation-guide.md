---
layout: post
title: 如何安装HBase
comments: true
---

---

本文说明如何安装部署HBase-0.94.27/Hadoop-1.2.1。

---

## 1.准备环境与资源

### 1.1.安装jdk
安装jdk(java development kit)-1.6版本。通过命令`java -version`查看有无安装jdk，若显示有版本号如下：

```shell
java version "jdk1.6.0_02"
```

说明已经安装1.6.0_02版本的jdk，可以跳过安装jdk这一步。

这里要注意的是，centos安装时可以选择安装java平台（一般会选择安装），如果安装了java平台，执行上述命令后会显示：

```shell
java version "1.7.0_75"
```

区别在于是否有“jdk”字样。如果没有jdk字样，或者连java版本信息都没有，则需要下载安装jdk。

到[Oracle官网](http://www.oracle.com/technetwork/java/javase/downloads/java-archive-downloads-javase6-419409.html#jdk-6u45-oth-JPR)上下载jdk-1.6版本，其中有两种格式，一种是jdk-6u45-linux-i586-rpm.bin，一种jdk-6u45-linux-i586.bin。下载前面那种。以.bin为后缀的文件是自解压文件，运行它之后会自动解压。

下载之后到文件所在的目录下执行该文件，即

```shell
./jdk-6u45-linux-i586-rpm.bin
```

待其自解压结束后，会出现文件`jdk-6u45-linux-i586.rpm`，之后使用root账户（执行命令`su`）执行命令：

```shell
rpm -ivh jdk-6u45-linux-i586.rpm
```

若安装成功会显示类似“XXX is already installed”的信息。

安装成功后，jdk的安装目录位于`/usr/java`下。可以看到该目录下有`jdk1.6.0_45`。

则jdk的安装路径为`/usr/java/jdk1.6.0_45`，稍后安装Hadoop和HBase时均会使用到这个路径。

使用完root账户后，记得退出：执行命令`exit`。

### 1.2.下载hadoop
到[Hadoop官网](http://mirrors.hust.edu.cn/apache/hadoop/common/hadoop-1.2.1/)下载Hadoop-1.2.1版本，其中有多种格式，下载`hadoop-1.2.1-bin.tar.gz`格式。

下载之后，为了方便管理，自己创建一个目录用来放Hadoop和HBase的安装文件，执行命令：

```shell
mkdir ~/software
```

将下载的文件放到这个目录下，然后到这个目录下解压该文件：

```shell
tar xzf hadoop-1.2.1-bin.tar.gz
```

解压后可以看到`hadoop-1.2.1`目录，该目录即为Hadoop安装目录。

### 1.3.下载hbase
到[HBase官网](http://mirrors.cnnic.cn/apache/hbase/)下载`hbase-0.94.27.tar.gz`。放到Hadoop安装文件所在的目录下。

执行加压命令：

```shell
tar xzf hbase-0.94.27.tar.gz
```

解压后可以看到`hbase-0.94.27`目录。

### 1.4.配置环境变量
首先确认一下Hadoop和HBase的安装目录位于哪个目录下，这里假设是/home/lsr/software。

为了能快捷地使用Hadoop和HBase的命令，使用root账户在/etc/profile文件中最下方添加以下语句：

```shell
export PATH=/home/lsr/software/hadoop-1.2.1/bin:/home/lsr/software/hbase-0.94.27/bin:$PATH
```

注意要将其中的安装目录/home/lsr/software替换为你自己的安装目录，同时要确认hadoop和hbase的版本号是否正确，若与上述不一样则要把上述语句修改为自己的版本。

修改完该文件后保存。并执行命令`source /etc/profile`使该修改生效。

然后验证修改是否生效：在终端中输入`had`，按tab键，看`hadoop`命令是否能补齐，是则说明hadoop环境变量修改成功；然后在终端中输入`hb`，按tab键，看`hbase`命令是否能补齐，是则说明HBase环境变量修改成功。

### 1.5.配置ssh免密码登录功能

由于在Hadoop中namenode需要用ssh远程登录datanode，因此需要安装ssh服务并配置免密码登录功能，以免namenode远程登录datanode时还需要手动输入密码。因为centos在安装时已经默认安装ssh了，所以这里跳过安装ssh的步骤。

首先切换到root账户。然后执行下面四条命令：

```shell
ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa
cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

然后尝试是否可以免密码登录：

```shell
ssh localhost
```

若没有提示输入密码，则配置成功。

## 2.配置HDFS

### 2.1.说明
HDFS是hadoop的一部分，是一个分布式文件系统，HBase也是以它作为底层的文件系统。在启动HBase之前需要确保HDFS正常运行。

在这里，需要简单介绍一下Hadoop。Hadoop包括三个部分：
 - HDFS：它负责保证分布式存储以及存储的容错机制，包括namenode和datanode守护进程。
 - MapReduce编程模型：它提供一种处理分布式数据集的编程方式。
 - jobtracker/tasktracker守护进程：它们负责保证客户端程序的正常运行。

在Hadoop集群部署中，一个主机往往只会有一个HDFS的守护进程（Daemon），即namenode或datanode，同样地，一个主机也只会有一个jobtracker或tasktracker守护进程。一般地，datanode守护进程和tasktracker位于同一个主机，namenode和jobtracker位于同一个主机。

在集群中，而这些主机也会拥有与其功能对应的域名namenode，datanode1，datanode2，...，datanodeN。在一个不连接互联网的集群中，域名往往就是主机名。比如，主机namenode上运行的守护进程即为NameNode和JobTracker，它被称为主节点，主机datanode1上运行的守护进程即为DataNode和TaskTracker，它被称为从节点。

除了上面的守护进程外，还会有一个守护进程叫secondarynamenode，它用于对namenode中的数据做备份，以防止namenode崩溃时集群无法继续运行和恢复。因此这个守护进程也必须与namenode位于不同主机上，同时它也往往不会与datanode位于同一主机上。因此，一般地，最小的Hadoop集群必须拥有三台主机。

现在只有一台主机，整个Hadoop集群的所有守护进程都会运行在这台主机上，为了模拟真正的集群，则要将这台主机看成多台主机，方法是为这个主机添加多个域名。因为这台主机同时扮演了主节点、从节点和备份节点，所以它要有三个域名来代表这三个角色身份，即namenode、datanode1和secondarynamenode。

### 2.2.配置域名
要配置/etc/hosts文件使得这两个域名能被解析，查看/etc/hosts文件，一般形式如下：

```shell
127.0.0.1     localhost     localhost.localdomain
::1     localhost     localhost.localdomain
```

切换到root账户，将该文件修改为：

```shell
127.0.0.1     datanode1     namenode     secondarynamenode     localhost     localhost.localdomain     lsr1991
::1     datanode1     namenode     secondarynamenode     localhost     localhost.localdomain     lsr1991
```

注意，务必按上述形式修改文件，不可将datanode1添加到行尾。因为这会决定之后zookeeper是否能成功启动。其中，lsr1991为你当前的主机名。

### 2.3.配置Hadoop
因为后面可能会需要编写MapReduce程序，所以把Hadoop各个部分都配置好。

进入Hadoop解压目录`hadoop-1.2.1`，里面有`conf`目录，该目录下的文件即为Hadoop的配置文件。

需要修改六个文件，各文件的配置如下：

1.core-site.xml

```shell
...
<configuration>
    <property>
        <name>fs.default.name</name>
        <value>hdfs://namenode:9000</value>
    </property>
</configuration>
```

2.hdfs-site.xml（HDFS配置文件）

```shell
...
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <name>dfs.name.dir</name>
        <value>/hdfs/data/dfs/nn</value>
    </property>
    <property>
        <name>dfs.data.dir</name>
        <value>/hdfs/data/dfs/dn</value>
    </property>
</configuration>
```

3.mapred-site.xml（MapReduce配置文件）

```shell
...
<configuration>
    <property>
        <name>mapred.job.tracker</name>
        <value>namenode:9001</value>
    </property>
    <property>
        <name>mapreduce.jobtracker.staging.root.dir</name>
        <value>/user</value>
    </property>
</configuration>
```

4.masters（配置secondarynamenode的网络地址）

将文件中的localhost替换为

```shell
secondarynamenode
```

5.slaves（配置从节点的网络地址）

将localhost替换为

```shell
datanode1
```

6.hadoop-env.sh（配置Hadoop运行的环境）

里面内容较多，这里直接说明如何修改。

将`# export JAVA_HOME=/usr/java/jdk1.6.0_02`一行的#号去掉，并将`/usr/java/jdk1.6.0_02`修改为你的jdk安装路径，如果之前是按照本教程安装jdk，那么路径就是`/usr/java/jdk1.6.0_45`。

接着将`# export HADOOP_PID_DIR=/var/hadoop/pids`一行的#号去掉。

修改完成。

所有配置文件修改结束。

### 2.4.测试Hadoop是否正常启动
切换到root账户，执行命令`hadoop namenode -format`创建一个HDFS并格式化，成功的话显示的信息最下方形如：

```shell
Shutting down NameNode at lsr1991/127.0.0.1
```

接着执行命令`start-all.sh`，会看到各个守护进程相继启动。等启动结束后，执行命令`jps`，会显示出已启动的守护进程，如下：

```shell
5780 Jps
3756 SecondaryNameNode
4875 JobTracker
3517 NameNode
3632 DataNode
4991 TaskTracker
```

其中第一列是进程号，可见上述提到的Hadoop中的所有守护进程都启动了。如果有一个没有启动，则需要查看其日志文件以确定没有启动的原因。

日志文件位于Hadoop安装目录下的logs目录中。如NameNode的启动日志名称形式如下：

```shell
hadoop-lsr-namenode-lsr1991.log
```

其中，lsr为用户名，lsr1991为主机名。

接着查看web页面是否可访问。在浏览器中输入namenode:50070可以查看HDFS的web页面，输入namenode:50030可以查看MapReduce作业的web页面。

如果上述测试均通过，说明Hadoop已经正常启动。

### 2.5.测试Hadoop是否正常运行
虽然Hadoop已经启动，但无法确定客户端是否可执行相应的操作，如上传文件到HDFS，提交MapReduce作业等等。

首先需要说明一点，本台主机模拟了一个Hadoop集群，同时它也模拟了一个客户端。可以认为，root账户就是集群的管理员（因为他启动了集群），他拥有操作和配置集群的权限，而非root账户则是集群的用户，他们只可以使用集群来存储文件和运行自己的MapReduce作业，他们能够访问的HDFS上的目录只有他们自己的目录，别人的目录他们都不能访问。而一个新的用户想要使用集群时需要先去找集群管理员获取访问权限，比如用户lsr想要使用集群，他去找集群管理员root，那么root就会在HDFS上为他创建一个专属目录，创建的过程如下：

1.切换到root账户

`su`

2.创建lsr用户的目录

`hadoop fs -mkdir /user/lsr`

3.更改目录所有者

`hadoop fs  -chown -R lsr:lsr /user/lsr`

操作完毕。

这样，用户lsr就可以访问HDFS上的目录/user/lsr了。

为了测试是否能访问，执行以下操作：

1.从root账户切换到lsr账户

`exit`

2.在本地目录下创建一个文件

`touch a`

3.将该文件上传到HDFS上

`hadoop fs -put a /user/lsr`（注意，此命令中的/user/lsr是HDFS上的路径而不是本地文件系统上的路径）

4.查看文件是否被上传到了HDFS上

`hadoop fs -ls /user/lsr`，若显示有a文件，说明上传成功。

5.将文件a从HDFS上下载到本地目录，并命名为b

`hadoop fs -get ./a b`，然后查看本地当前目录，看是否有文件b，若有说明下载成功。

上述测试若通过，说明用户lsr可以访问HDFS了。

为了测试是否能运行MapReduce作业（以wordcount程序为例，wordcount程序用于统计文件中各个单词出现的次数），执行一下操作：

1.从root账户切换到lsr账户（已是lsr账户则不用切换）

`exit`

2.到Hadoop安装目录下

`cd /home/lsr/software/hadoop-1.2.1`

3.创建MapReduce作业所需要的数据

`vim wordcount.txt`

在里面添加：

```shell
hello hello hi hi
hi
no hello
no hi
world
```

然后保存

4.将数据上传到HDFS上

`hadoop fs -put wordcount.txt /user/lsr`

5.查看数据是否上传成功

`hadoop fs -ls /user/lsr`

6.运行作业

`hadoop jar hadoop-examples-1.2.1.jar wordcount /user/lsr/wordcount.txt /user/lsr/output`

其中，/user/lsr/output为运行作业的结果输出路径，该路径不能是已存在的路径。

7.看输出信息有无报错，在web页面（namenode:50030可以查看作业运行的情况）

8.查看输出结果是否正确

到web页面（namenode:50070）上浏览HDFS上的数据，根据路径/user/lsr/output找到输出目录，其中以`part-r-00000`形式命名的文件即为输出结果，点击可打开查看，正确的统计结果为

```shell
hello     3
hi     4
no     2
world     1
```

若上述测试通过，说明用户lsr可以运行MapReduce作业。

至此，可以认为Hadoop能正常运行。

## 3.配置HBase

### 3.1.修改配置文件

到HBase的安装目录下，可以看到`conf`目录，进入该目录，修改以下配置文件：

1.hbase-site.xml

```shell
...
<configuration>
     <property>
        <name>hbase.rootdir</name>
        <value>hdfs://namenode:9000/hbase</value>
        <description>The directory shared by RegionServers.</description>
     </property>
     <property>
        <name>dfs.replication</name>
        <value>1</value>
         <description>The replication count for HLog and HFile storage. Should not be greater             than HDFS datanode count.</description>
     </property>
     <property>
        <name>hbase.cluster.distributed</name>
        <value>true</value>
        <description>The mode the cluster will be in. Possible values are
        false: standalone and pseudo-distributed setups with managed Zookeeper
        true: fully-distributed with unmanaged Zookeeper Quorum (see hbase-env.sh)
        </description>
     </property>
     <property>
        <name>hbase.zookeeper.quorum</name>
        <value>datanode1</value>
     </property>
</configuration>
```

2.hbase-env.sh

里面内容较多，这里直接说明如何修改。

将`# export JAVA_HOME=/usr/java/jdk1.6.0_02`一行的#号去掉，并将`/usr/java/jdk1.6.0_02`修改为你的jdk安装路径，如果之前是按照本教程安装jdk，那么路径就是`/usr/java/jdk1.6.0_45`。

接着将`# export HBASE_PID_DIR=/var/hadoop/pids`一行的#号去掉。

然后将`# export HBASE_MANAGES_ZK=true`一行的#号去掉。

修改完成。

3.regionservers

将里面的`localhost`替换为`datanode1`。

所有配置文件修改结束。

### 3.2.测试HBase是否能正常启动
切换到root账户后，执行命令：

`start-hbase.sh`

会看到zookeeper，master，regionserver相继启动。通常来讲，Hadoop集群和HBase集群是部署在一起的，NameNode守护进程和HMaster守护进程位于同一台主机，DataNode守护进程和Regionserver守护进程位于同一台主机。另外，HBase还需要一个zookeeper集群来做协调工作，如果没有独立的zookeeper集群，那么HBase会在启动时自行启动一个zookeeper集群，集群的各个主机由配置文件中的`hbase.zookeeper.quorum`参数指定，上述配置中指定为`datanode1`，即HBase在启动时，会在datanode1上启动一个zookeeper的守护进程，叫做HQuorumPeer。

用`jps`命令查看已启动的守护进程：

```shell
6605 HQuorumPeer
6808 HRegionServer
3756 SecondaryNameNode
4875 JobTracker
6665 HMaster
7058 Jps
3517 NameNode
3632 DataNode
4991 TaskTracker
```

可以看到HQuorumPeer，HRegionServer，HMaster，即HBase的所有守护进程均已启动。

HBase也有web页面，namenode:60010可以查看master节点的情况。datanode1:60030可以查看regionserver节点的情况。
若页面均正常显示，说明HBase已正常启动。

若不能正常启动，需要查看日志文件，HBase的日志文件位于它的安装目录下的logs目录中。

### 3.3.测试HBase是否能正常运行
使用lsr账户，执行命令`hbase shell`打开shell客户端。

在客户端中执行以下命令：

1.创建表

`create 'test','cf'`

2.查看表

`list`

3.插入一行

`put 'test','row1','cf:a','value1'`

4.往上面那行中插入一列

`put 'test','row1','cf:b','value2'`

5.再插入一行

`put 'test','row2','cf:a','value3'`

6.扫描表

`scan 'test'`

7.读取一行

`get 'test','row1'`

8.删除一行

`delete 'test','row1'`

在master节点的web页面中，可以看到被创建的表。

在客户端中继续执行以下命令：

9.使表不可用

`disable 'test'`，这时在master节点的web页面中可以看到该表的online region数目由1变为0，说明此时不能对该表进行插入/删除/更新行/列的操作了。

10.删除表

`drop 'test'`

此时查看web页面会发现表test已经没了。

若上述测试能正常执行，说明HBase有正常运行。

至此，HBase安装部署完毕。

### 3.4.停止HBase
切换到root账户，执行命令`stop-hbase.sh`。

若需要停止Hadoop，继续执行命令`stop-all.sh`。

## 4.其他

### 4.1.只启动HDFS
因为对HBase来说，HDFS是必须的，MapReduce不是必须的，因此启动Hadoop时可以只启动HDFS，这样集群里的守护进程就只有NameNode和DataNode，没有JobTracker和TaskTracker。

启动HDFS命令为`start-dfs.sh`。

需要注意的是，必须先启动HDFS后才能启动HBase。

### 4.2.参考链接
http://abloz.com/hbase/book.html#distributed

### 4.3.安装过程中出现的问题

- **Q1**

启动Hadoop时，SecondaryNameNode守护进程显示在主机namenode而不是secondarynamenode上启动。

- **A1**

原因是conf/master文件配置错误。该文件是写运行SecondaryNameNode的主机，不是运行NameNode的主机。运行NameNode的主机即为运行Hadoop/HDFS启动命令的主机。

> 使用root账户启动hadoop，用lsr账户作为客户端进行访问，出现以下问题：

- **Q2**

无法写入。执行`hadoop fs -mkdir /user/lsr`操作失败，提示错误：

```shell
mkdir: org.apache.hadoop.security.AccessControlException: Permission denied: user=lsr, access=WRITE, inode="":root:supergroup:rwxr-xr-x
```

- **A2**

这是因为lsr账户没有对inode""操作的权限。需要先授予权限，具体步骤为：

```shell
# 切换到root账户
su
# 创建lsr账户的目录
hadoop fs -mkdir /user/lsr
# 更改目录所有者
hadoop fs  -chown -R lsr:lsr /user/lsr
```

然后lsr账户就能操作该目录了，比如`hadoop fs -put wordcount.txt .`。

- **Q3**

无法跑作业。执行`hadoop jar xxx.jar wordcount /user/lsr/wordcount.txt /user/lsr/output`操作失败，提示错误：

```shell
15/03/27 11:40:13 ERROR security.UserGroupInformation: PriviledgedActionException as:lsr cause:org.apache.hadoop.security.AccessControlException: org.apache.hadoop.security.AccessControlException: Permission denied: user=lsr, access=WRITE, inode="mapred":root:supergroup:rwxr-xr-x
```

- **A3**

网上解决方法（见[参考链接](http://www.blogjava.net/paulwong/archive/2012/10/03/388988.html)）是修改conf/mapred-site.xml文件：

```shell
# 添加下面几行
<property>
<name>mapreduce.jobtracker.staging.root.dir</name>
<value>/user</value>
</property>
```

然后重新启动mapred进程，`stop-mapred.sh`接着`start-mapred.sh`。

之后仍然出现错误：

```shell
15/03/27 12:00:48 ERROR security.UserGroupInformation: PriviledgedActionException as:lsr cause:org.apache.hadoop.ipc.RemoteException: org.apache.hadoop.mapred.SafeModeException: JobTracker is in safe mode
```

网上解决方法为（见[参考链接](http://blog.csdn.net/swuteresa/article/details/9467879)）使jobtracker离开安全模式，即执行命令`hadoop dfsadmin -safemode leave`。

问题解决，可用lsr账户正常跑wordcount作业。

- **Q4**

安装HBase后，启动时无法启动zookeeper和HMaster，master的日志文件中说没能找到zookeeper，查看zookeeper日志XXX.out，里面报出错误：

```shell
java.io.IOException: Could not find my address: lsr1991 in list of ZooKeeper quorum servers
at org.apache.hadoop.hbase.zookeeper.HQuorumPeer.writeMyID(HQuorumPeer.java:140)
at org.apache.hadoop.hbase.zookeeper.HQuorumPeer.main(HQuorumPeer.java:61)
```

- **A4**

查找资料（见[参考资料](http://wiki.apache.org/hadoop/Hbase/Troubleshooting#A9)）原因名字解析问题，报出的错误中，“my address: lsr1991”中的“lsr1991”是主机的名称，HBase试图在某台机器上启动一个zookeeper，但是那台机器无法在hbase.zookeeper.quorum配置中找到自己的名称。解决办法是将hbase.zookeeper.quorum配置中的名称改为错误信息中“my address”之后的主机名称。

测试发现，修改/etc/hosts文件中127.0.0.1对应的第一个主机名（从左到右），错误信息中的“my address”之后的主机名也会相应改变为修改值。因此，可以将/etc/hosts文件127.0.0.1对应的第一个主机名改成hbase.zookeeper.quorum配置中的名称（如quorum1）。

然后问题解决，可以正常启动zookeeper和hmaster。

- **Q5**

在安装完hadoop和hbase之后，可以成功运行。当主机重启之后，对hdfs执行格式化`hadoop namenode -format`，之后启动hdfs`start-dfs.sh`，发现namenode没有启动，查看日志发现错误如下：

```shell
ERROR org.apache.hadoop.hdfs.server.namenode.FSNamesystem: FSNamesystem initialization failed.ERROR org.apache.hadoop.hdfs.server.namenode.FSNamesystem: FSNamesystem initialization failed.
java.io.IOException: NameNode is not formatted. 
at org.apache.hadoop.hdfs.server.namenode.FSImage.recoverTransitionRead(FSImage.java:330) 
at org.apache.hadoop.hdfs.server.namenode.FSDirectory.loadFSImage(FSDirectory.java:100)
```

- **A5**

原因不明，无法复现。但是有解决方法：将hdfs的名字空间目录删除，再执行`hadoop namenode -format`，就可以启动namenode了。hdfs的名字空间目录默认位于hadoop的tmp目录下，当以某用户（如root）的身份将hdfs格式化之后，会在tmp目录下生成`hadoop-root/dfs/`目录，该目录下有`name`和`data`两个目录。但是这个名字空间目录是可以配置的，在配置文件`hdfs-site.xml`中，参数`dfs.name.dir`和`dfs.data.dir`可以设置名字空间目录。另外，hadoop的tmp目录也是可以配置的，默认该目录是`/tmp`，centos 6.X系统，该目录中的文件如果10天内没有访问，就会被删除。

