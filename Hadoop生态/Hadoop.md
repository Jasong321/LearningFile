# Hadoop是什么？

Hadoop是由java语言编写的，在分布式服务器集群上存储海量数据并运行分布式分析应用的开源框架，其核心部件是<font color=red>**HDFS**</font>与<font color=red>**MapReduce**</font>。

​       HDFS是一个分布式文件系统：引入存放文件元数据信息的服务器**Namenode**和实际存放数据的服务器**Datanode**，对数据进行分布式储存和读取。

　　MapReduce是一个计算框架：MapReduce的核心思想是把计算任务分配给集群内的服务器里执行。通过对计算任务的拆分（Map计算/Reduce计算）再根据任务调度器（JobTracker）对任务进行分布式计算。

> HDFS分布式存储，突破了服务器硬盘大小的限制，解决了单台机器无法存储大文件的问题，同时MapReduce分布式计算可以将大数据量的作业先分片计算，最后汇总输出。



# Hadoop 特点

优点：

1、支持超大文件。HDFS存储的文件可以支持TB和PB级别的数据。

2、检测和快速应对硬件故障。数据备份机制，NameNode通过心跳机制来检测DataNode是否还存在。

3、高扩展性。可建构在廉价机上，实现线性（横向）扩展，当集群增加新节点之后，NameNode也可以感知，将数据分发和备份到相应的节点上。

4、成熟的生态圈。借助开源的力量，围绕Hadoop衍生的一些小工具。

缺点：

1、不能做到低延迟。对数据吞吐量做了优化，牺牲了获取数据的延迟。

2、不适合大量的小文件存储。

3、文件修改效率低。**HDFS适合一次写入，多次读取的场景**。



# HDFS

## HDFS框架介绍

硬件错误是常态而不是异常。HDFS可能由成百上千的服务器所构成，每个服务器上存储着文件系统的部分数据。我们面对的现实是构成系统的组件数目是巨大的，而且任一组件都有可能失效，这意味着总是有一部分HDFS的组件是不工作的。因此错误检测和快速、自动的恢复是HDFS最核心的架构目标。

HDFS应用需要一个“一次写入多次读取”的文件访问模型。一个文件经过创建、写入和关闭之后就不需要改变。这一假设简化了数据一致性问题，并且使高吞吐量的数据访问成为可能。Map/Reduce应用或者网络爬虫应用都非常适合这个模型。目前还有计划在将来扩充这个模型，使之支持文件的附加写操作。

一个应用请求的计算，离它操作的数据越近就越高效，在数据达到海量级别的时候更是如此。因为这样就能降低网络阻塞的影响，提高系统数据的吞吐量。将计算移动到数据附近，比之将数据移动到应用所在显然更好。HDFS为应用提供了将它们自己移动到数据附近的接口。

HDFS是Master和Slave的主从结构。主要由NameNode、Secondary NameNode、DataNode构成。



### NameNode 和 DataNode

HDFS采用master/slave架构。一个HDFS集群是由一个Namenode和一定数目的Datanodes组成。<u>Namenode是一个中心服务器，负责管理文件系统的名字空间(namespace)以及客户端对文件的访问。</u>集群中的Datanode一般是一个节点一个，负责管理它所在节点上的存储。HDFS暴露了文件系统的名字空间，用户能够以文件的形式在上面存储数据。<u>从内部看，一个文件其实被分成一个或多个数据块，这些块存储在一组Datanode上。Namenode执行文件系统的名字空间操作，比如打开、关闭、重命名文件或目录。它也负责确定数据块到具体Datanode节点的映射。Datanode负责处理文件系统客户端的读写请求。在Namenode的统一调度下进行数据块的创建、删除和复制。</u>

![HDFS 架构](https://hadoop.apache.org/docs/r1.0.4/cn/images/hdfsarchitecture.gif)

### 文件系统的名字空间（NameSpace）

HDFS支持传统的层次型文件组织结构。用户或者应用程序可以创建目录，然后将文件保存在这些目录里。文件系统名字空间的层次结构和大多数现有的文件系统类似：用户可以创建、删除、移动或重命名文件。当前，HDFS不支持用户磁盘配额和访问权限控制，也不支持硬链接和软链接。但是HDFS架构并不妨碍实现这些特性。

Namenode负责维护文件系统的名字空间，任何对文件系统名字空间或属性的修改都将被Namenode记录下来。应用程序可以设置HDFS保存的文件的副本数目。文件副本的数目称为文件的副本系数，这个信息也是由Namenode保存的。

### Secondary NameNode

它并非NameNode的热备，当NameNode挂掉的时候，它并不能马上替换NameNode并提供服务。

1. 辅助NameNode，分担其工作量。
2. 定期合并Fsimage和Edits，并推送给NameNode。
3. 在紧急情况下，可辅助恢复NameNode。



### NN和2NN

NameNode启动：

1、第一次启动namenode格式化后，创建fsimage和edits文件。如果不是第一次启动，直接加载编辑日志和镜像文件到内存。

2、客户端对元数据进行增删改的请求。

3、namenode记录操作日志，更新滚动日志。

4、namenode在内存中对数据进行增删改

2NN启动：

1、Secondary NameNode询问namenode是否需要checkpoint。直接带回namenode是否检查结果。

2、Secondary NameNode请求执行checkpoint。

3、namenode滚动正在写的edits日志

4、将滚动前的编辑日志和镜像文件拷贝到Secondary NameNode

5、Secondary NameNode加载编辑日志和镜像文件到内存，并合并。

6、生成新的镜像文件fsimage.chkpoint

7、拷贝fsimage.chkpoint到namenode

8、namenode将fsimage.chkpoint重新命名成fsimage

#### Fsimage和Edits解析

namenode被格式化之后，将在/opt/module/hadoop-2.7.2/data/tmp/dfs/name/current目录中产生如下文件

>fsimage_0000000000000000000
>
>fsimage_0000000000000000000.md5
>
>seen_txid
>
>VERSION

1、Fsimage文件：HDFS文件系统元数据的一个永久性的检查点，其中包含HDFS文件系统的所有目录和文件idnode的序列化信息。

2、Edits文件：存放HDFS文件系统的所有更新操作的路径，文件系统客户端执行的所有写操作首先会被记录到edits文件中。  

3、seen_txid文件保存的是一个数字，就是最后一个edits_的数字

4、每次Namenode启动的时候都会将fsimage文件读入内存，并从00001开始到seen_txid中记录的数字依次执行每个edits里面的更新操作，保证内存中的元数据信息是最新的、同步的，可以看成Namenode启动的时候就将fsimage和edits文件进行了合并。

#### CheckPoint时间设置

1、通常情况下，SecondaryNameNode每隔一小时执行一次。

[hdfs-default.xml]

```xml
<property>
  <name>dfs.namenode.checkpoint.period</name>
  <value>3600</value>
</property>
```

2、一分钟检查一次操作次数，当操作次数达到1百万时，SecondaryNameNode执行一次。。

[hdfs-default.xml]

```xml
<property>
  <name>dfs.namenode.checkpoint.txns</name>
  <value>1000000</value>
<description>操作动作次数</description>
</property>

<property>
  <name>dfs.namenode.checkpoint.check.period</name>
  <value>60</value>
<description> 1分钟检查一次操作次数</description>
</property>
```



#### SecondaryNameNode目录结构

Secondary NameNode用来监控HDFS状态的辅助后台程序，每隔一段时间获取HDFS元数据的快照。

在/opt/module/hadoop-2.7.2/data/tmp/dfs/namesecondary/current这个目录中查看SecondaryNameNode目录结构。

>edits_0000000000000000001-0000000000000000002
>
>fsimage_0000000000000000002
>
>fsimage_0000000000000000002.md5
>
>VERSION

SecondaryNameNode的namesecondary/current目录和主namenode的current目录的布局相同。

好处：在主namenode发生故障时（假设没有及时备份数据），可以从SecondaryNameNode恢复数据。

#### 集群安全模式

Namenode启动时，首先将映像文件（fsimage）载入内存，并执行编辑日志（edits）中的各项操作。一旦在内存中成功建立文件系统元数据的映像，则创建一个新的fsimage文件和一个空的编辑日志。此时，namenode开始监听datanode请求。但是此刻，namenode运行在安全模式，即namenode的文件系统对于客户端来说是只读的。

系统中的数据块的位置并不是由namenode维护的，而是以块列表的形式存储在datanode中。在系统的正常操作期间，namenode会在内存中保留所有块位置的映射信息。在安全模式下，各个datanode会向namenode发送最新的块列表信息，namenode了解到足够多的块位置信息之后，即可高效运行文件系统。

如果满足“最小复本条件”，namenode会在30秒钟之后就退出安全模式。所谓的最小复本条件指的是在整个文件系统中99.9%的块满足最小复本级别（默认值：dfs.replication.min=1）。在启动一个刚刚格式化的HDFS集群时，因为系统中还没有任何块，所以namenode不会进入安全模式。

基本语法：

集群处于安全模式，不能执行重要操作（写操作）。集群启动完成后，自动退出安全模式。

```shell
bin/hdfs dfsadmin -safemode get		（功能描述：查看安全模式状态）
bin/hdfs dfsadmin -safemode enter  	（功能描述：进入安全模式状态）
bin/hdfs dfsadmin -safemode leave	（功能描述：离开安全模式状态）
bin/hdfs dfsadmin -safemode wait	（功能描述：等待安全模式状态）
```



### DataNode

#### 工作机制

1、一个数据块在datanode上以文件形式存储在磁盘上，包括两个文件，一个是数据本身，一个是元数据包括数据块的长度，块数据的校验和，以及时间戳。

2、DataNode启动后向namenode注册，通过后，周期性（1小时）的向namenode上报所有的块信息。

3、心跳是每3秒一次，心跳返回结果带有namenode给该datanode的命令如复制块数据到另一台机器，或删除某个数据块。如果超过10分钟没有收到某个datanode的心跳，则认为该节点不可用。

4、集群运行中可以安全加入和退出一些机器

#### 数据完整性

1、当DataNode读取block的时候，它会计算checksum。

2、如果计算后的checksum，与block创建时值不一样，说明block已经损坏。

3、client读取其他DataNode上的block。

4、datanode在其文件创建后周期验证checksum。

掉线时限参数设置：

datanode进程死亡或者网络故障造成datanode无法与namenode通信，namenode不会立即把该节点判定为死亡，要经过一段时间，这段时间暂称作超时时长。HDFS默认的超时时长为10分钟+30秒。如果定义超时时间为timeout，则超时时长的计算公式为：
$$
timeout = 2 * dfs.namenode.heartbeat.recheck-interval + 10 * dfs.heartbeat.interval。
$$
而默认的dfs.namenode.heartbeat.recheck-interval 大小为5分钟，dfs.heartbeat.interval默认为3秒。

需要注意的是hdfs-site.xml 配置文件中的heartbeat.recheck.interval的单位为毫秒，dfs.heartbeat.interval的单位为秒。

```xml
<property>
    <name>dfs.namenode.heartbeat.recheck-interval</name>
    <value>300000</value>
</property>
<property>
    <name> dfs.heartbeat.interval </name>
    <value>3</value>
</property>

```



### 数据复制

HDFS被设计成能够在一个大集群中跨机器可靠地存储超大文件。它将每个文件存储成一系列的数据块，除了最后一个，所有的数据块都是同样大小的。为了容错，文件的所有数据块都会有副本。每个文件的数据块大小和副本系数都是可配置的。应用程序可以指定某个文件的副本数目。副本系数可以在文件创建的时候指定，也可以在之后改变。HDFS中的文件都是一次性写入的，并且严格要求在任何时候只能有一个写入者。

Namenode全权管理数据块的复制，它周期性地从集群中的每个Datanode接收心跳信号和块状态报告(Blockreport)。接收到心跳信号意味着该Datanode节点工作正常。块状态报告包含了一个该Datanode上所有数据块的列表。

![HDFS Datanodes](https://hadoop.apache.org/docs/r1.0.4/cn/images/hdfsdatanodes.gif)

#### 副本存放：最最开始的一步

副本的存放是HDFS可靠性和性能的关键。优化的副本存放策略是HDFS区分于其他大部分分布式文件系统的重要特性。这种特性需要做大量的调优，并需要经验的积累。HDFS采用一种称为机架感知(rack-aware)的策略来改进数据的可靠性、可用性和网络带宽的利用率。目前实现的副本存放策略只是在这个方向上的第一步。实现这个策略的短期目标是验证它在生产环境下的有效性，观察它的行为，为实现更先进的策略打下测试和研究的基础。

大型HDFS实例一般运行在跨越多个机架的计算机组成的集群上，不同机架上的两台机器之间的通讯需要经过交换机。在大多数情况下，同一个机架内的两台机器间的带宽会比不同机架的两台机器间的带宽大。

通过一个机架感知的过程，Namenode可以确定每个Datanode所属的机架id。一个简单但没有优化的策略就是将副本存放在不同的机架上。这样可以有效防止当整个机架失效时数据的丢失，并且允许读数据的时候充分利用多个机架的带宽。这种策略设置可以将副本均匀分布在集群中，有利于当组件失效情况下的负载均衡。但是，因为这种策略的一个写操作需要传输数据块到多个机架，这增加了写的代价。

在大多数情况下，副本系数是3，HDFS的存放策略是将一个副本存放在本地机架的节点上，一个副本放在同一机架的另一个节点上，最后一个副本放在不同机架的节点上。这种策略减少了机架间的数据传输，这就提高了写操作的效率。机架的错误远远比节点的错误少，所以这个策略不会影响到数据的可靠性和可用性。于此同时，因为数据块只放在两个（不是三个）不同的机架上，所以此策略减少了读取数据时需要的网络传输总带宽。在这种策略下，副本并不是均匀分布在不同的机架上。三分之一的副本在一个节点上，三分之二的副本在一个机架上，其他副本均匀分布在剩下的机架中，这一策略在不损害数据可靠性和读取性能的情况下改进了写的性能。

#### 副本选择

为了降低整体的带宽消耗和读取延时，HDFS会尽量让读取程序读取离它最近的副本。如果在读取程序的同一个机架上有一个副本，那么就读取该副本。如果一个HDFS集群跨越多个数据中心，那么客户端也将首先读本地数据中心的副本。



> 本章节主要介绍了，HDFS的数据备份策略。用户可以配置文件存储时备份的多少。HDFS通过机架感知的策略来存放备份文件，在保证数据可靠性和读取性能的前提下，改进了写的性能。此外，当数据所在节点出现故障时，HDFS会尽量让程序读取离它最近的副本。



### 文件系统元数据的持久化

Namenode上保存着HDFS的名字空间。对于任何对文件系统元数据产生修改的操作，Namenode都会使用一种称为EditLog的事务日志记录下来。例如，在HDFS中创建一个文件，Namenode就会在Editlog中插入一条记录来表示；同样地，修改文件的副本系数也将往Editlog插入一条记录。Namenode在本地操作系统的文件系统中存储这个Editlog。整个文件系统的名字空间，包括数据块到文件的映射、文件的属性等，都存储在一个称为FsImage的文件中，这个文件也是放在Namenode所在的本地文件系统上。

Namenode在内存中保存着整个文件系统的名字空间和文件数据块映射(Blockmap)的映像。这个关键的元数据结构设计得很紧凑，因而一个有4G内存的Namenode足够支撑大量的文件和目录。当Namenode启动时，它从硬盘中读取Editlog和FsImage，将所有Editlog中的事务作用在内存中的FsImage上，并将这个新版本的FsImage从内存中保存到本地磁盘上，然后删除旧的Editlog，因为这个旧的Editlog的事务都已经作用在FsImage上了。这个过程称为一个检查点(checkpoint)。在当前实现中，检查点只发生在Namenode启动时，在不久的将来将实现支持周期性的检查点。

Datanode将HDFS数据以文件的形式存储在本地的文件系统中，它并不知道有关HDFS文件的信息。它把每个HDFS数据块存储在本地文件系统的一个单独的文件中。Datanode并不在同一个目录创建所有的文件，实际上，它用试探的方法来确定每个目录的最佳文件数目，并且在适当的时候创建子目录。在同一个目录中创建所有的本地文件并不是最优的选择，这是因为本地文件系统可能无法高效地在单个目录中支持大量的文件。当一个Datanode启动时，它会扫描本地文件系统，产生一个这些本地文件对应的所有HDFS数据块的列表，然后作为报告发送到Namenode，这个报告就是块状态报告。

### 磁盘数据错误，心跳检测和重新复制

每个Datanode节点周期性地向Namenode发送心跳信号。网络割裂可能导致一部分Datanode跟Namenode失去联系。Namenode通过心跳信号的缺失来检测这一情况，并将这些近期不再发送心跳信号Datanode标记为宕机，不会再将新的IO请求发给它们。任何存储在宕机Datanode上的数据将不再有效。Datanode的宕机可能会引起一些数据块的副本系数低于指定值，Namenode不断地检测这些需要复制的数据块，一旦发现就启动复制操作。在下列情况下，可能需要重新复制：某个Datanode节点失效，某个副本遭到损坏，Datanode上的硬盘错误，或者文件的副本系数增大。

### 集群均衡

HDFS的架构支持数据均衡策略。如果某个Datanode节点上的空闲空间低于特定的临界点，按照均衡策略系统就会自动地将数据从这个Datanode移动到其他空闲的Datanode。当对某个文件的请求突然增加，那么也可能启动一个计划创建该文件新的副本，并且同时重新平衡集群中的其他数据。这些均衡策略目前还没有实现。



### 数据完整性

从某个Datanode获取的数据块有可能是损坏的，损坏可能是由Datanode的存储设备错误、网络错误或者软件bug造成的。HDFS客户端软件实现了对HDFS文件内容的校验和(checksum)检查。当客户端创建一个新的HDFS文件，会计算这个文件每个数据块的校验和，并将校验和作为一个单独的隐藏文件保存在同一个HDFS名字空间下。当客户端获取文件内容后，它会检验从Datanode获取的数据跟相应的校验和文件中的校验和是否匹配，如果不匹配，客户端可以选择从其他Datanode获取该数据块的副本。



### 元数据磁盘错误

FsImage和Editlog是HDFS的核心数据结构。如果这些文件损坏了，整个HDFS实例都将失效。因而，Namenode可以配置成支持维护多个FsImage和Editlog的副本。任何对FsImage或者Editlog的修改，都将同步到它们的副本上。这种多副本的同步操作可能会降低Namenode每秒处理的名字空间事务数量。然而这个代价是可以接受的，因为即使HDFS的应用是数据密集的，它们也非元数据密集的。当Namenode重启的时候，它会选取最近的完整的FsImage和Editlog来使用。

Namenode是HDFS集群中的单点故障(single point of failure)所在。如果Namenode机器故障，是需要手工干预的。目前，自动重启或在另一台机器上做Namenode故障转移的功能还没实现。

### 数据组织

#### 数据块

HDFS被设计成支持大文件，适用HDFS的是那些需要处理大规模的数据集的应用。这些应用都是只写入数据一次，但却读取一次或多次，并且读取速度应能满足流式读取的需要。HDFS支持文件的“一次写入多次读取”语义。一个典型的数据块大小是64MB。因而，HDFS中的文件总是按照64M被切分成不同的块，每个块尽可能地存储于不同的Datanode中。

#### Staging

客户端创建文件的请求其实并没有立即发送给Namenode，事实上，在刚开始阶段HDFS客户端会先将文件数据缓存到本地的一个临时文件。应用程序的写操作被透明地重定向到这个临时文件。当这个临时文件累积的数据量超过一个数据块的大小，客户端才会联系Namenode。Namenode将文件名插入文件系统的层次结构中，并且分配一个数据块给它。然后返回Datanode的标识符和目标数据块给客户端。接着客户端将这块数据从本地临时文件上传到指定的Datanode上。当文件关闭时，在临时文件中剩余的没有上传的数据也会传输到指定的Datanode上。然后客户端告诉Namenode文件已经关闭。此时Namenode才将文件创建操作提交到日志里进行存储。如果Namenode在文件关闭前宕机了，则该文件将丢失。

上述方法是对在HDFS上运行的目标应用进行认真考虑后得到的结果。这些应用需要进行文件的流式写入。如果不采用客户端缓存，由于网络速度和网络堵塞会对吞估量造成比较大的影响。这种方法并不是没有先例的，早期的文件系统，比如AFS，就用客户端缓存来提高性能。为了达到更高的数据上传效率，已经放松了POSIX标准的要求。

#### 流水线复制

当客户端向HDFS文件写入数据的时候，一开始是写到本地临时文件中。假设该文件的副本系数设置为3，当本地临时文件累积到一个数据块的大小时，客户端会从Namenode获取一个Datanode列表用于存放副本。然后客户端开始向第一个Datanode传输数据，第一个Datanode一小部分一小部分(4 KB)地接收数据，将每一部分写入本地仓库，并同时传输该部分到列表中第二个Datanode节点。第二个Datanode也是这样，一小部分一小部分地接收数据，写入本地仓库，并同时传给第三个Datanode。最后，第三个Datanode接收数据并存储在本地。因此，<font color=red>Datanode能流水线式地从前一个节点接收数据，并在同时转发给下一个节点，数据以流水线的方式从前一个Datanode复制到下一个。</font>

## 存储空间回收

### 文件的删除和恢复

当用户或应用程序删除某个文件时，这个文件并没有立刻从HDFS中删除。实际上，HDFS会将这个文件重命名转移到/trash目录。只要文件还在/trash目录中，该文件就可以被迅速地恢复。文件在/trash中保存的时间是可配置的，当超过这个时间时，Namenode就会将该文件从名字空间中删除。删除文件会使得该文件相关的数据块被释放。注意，从用户删除文件到HDFS空闲空间的增加之间会有一定时间的延迟。

只要被删除的文件还在/trash目录中，用户就可以恢复这个文件。如果用户想恢复被删除的文件，他/她可以浏览/trash目录找回该文件。/trash目录仅仅保存被删除文件的最后副本。/trash目录与其他的目录没有什么区别，除了一点：在该目录上HDFS会应用一个特殊策略来自动删除文件。目前的默认策略是删除/trash中保留时间超过6小时的文件。将来，这个策略可以通过一个被良好定义的接口配置。

### 减少副本系数

当一个文件的副本系数被减小后，Namenode会选择过剩的副本删除。下次心跳检测时会将该信息传递给Datanode。Datanode遂即移除相应的数据块，集群中的空闲空间加大。同样，在调用setReplication API结束和集群中空闲空间增加间会有一定的延迟。



## HDFS读写流程

### HDFS中的block、packet、chunk

1、block
这个大家应该知道，文件上传前需要分块，这个块就是block，一般为128MB，当然你可以去改，不顾不推荐。因为**块太小：寻址时间占比过高。块太大：Map任务数太少，作业执行速度变慢。**它是最大的一个单位。

2、packet

packet是第二大的单位，它是client端向DataNode，或DataNode的PipLine之间传数据的基本单位，默认64KB。

3、chunk

chunk是最小的单位，它是client向DataNode，或DataNode的PipLine之间进行数据校验的基本单位，默认512Byte，因为用作校验，故每个chunk需要带有4Byte的校验位。所以实际每个chunk写入packet的大小为516Byte。由此可见真实数据与校验值数据的比值约为128 : 1。（即64*1024 / 512）

### HDFS写流程

1、跟NameNode通信请求上传文件，NameNode检查目标文件是否已经存在，父目录是否已经存在

2、NameNode返回是否可以上传

3、Client先对文件进行切分，请求第一个block该传输到哪些DataNode服务器上

4、NameNode返回3个DataNode服务器DataNode 1，DataNode 2，DataNode 3

5、Client请求3台中的一台DataNode 1(网络拓扑上的就近原则，如果都一样，则随机挑选一台DataNode)上传数据（本质上是一个RPC调用，建立pipeline）,DataNode 1收到请求会继续调用DataNode 2,然后DataNode 2调用DataNode 3，将整个pipeline建立完成，然后逐级返回客户端

6、Client开始往DataNode 1上传第一个block（先从磁盘读取数据放到一个本地内存缓存），以packet为单位。写入的时候DataNode会进行数据校验，它并不是通过一个packet进行一次校验而是以chunk为单位进行校验（512byte）。DataNode 1收到一个packet就会传给DataNode 2，DataNode 2传给DataNode 3，DataNode 1每传一个pocket会放入一个应答队列等待应答

7、当一个block传输完成之后，Client再次请求NameNode上传第二个block的服务器.

![img](https://raw.githubusercontent.com/Jasong321/PicBed/master/699090-20190626155745864-1227676006.png?token=AL2KPOIQOU3USHPWPEXLMNLBMLPRK)

### HDFS读流程

1、与NameNode通信查询元数据，找到文件块所在的DataNode服务器

2、挑选一台DataNode（网络拓扑上的就近原则，如果都一样，则随机挑选一台DataNode）服务器，请求建立socket流

3、DataNode开始发送数据(从磁盘里面读取数据放入流，以packet（一个packet为64kb）为单位来做校验)

4、客户端以packet为单位接收，先在本地缓存，然后写入目标文件

![img](https://raw.githubusercontent.com/Jasong321/PicBed/master/1460000013767517?token=AL2KPOJENLAIRGLPAK3QJJLBMLPRM)



### 网络拓扑

![image-20211006113631777](https://raw.githubusercontent.com/Jasong321/PicBed/master/image-20211006113631777.png?token=AL2KPOIAP5US3IFEHORLNQ3BMLPQY)

### 机架感知

1、第一个副本在client所处的节点上。如果客户端在集群外，随机选一个。

2、第二个副本和第一个副本位于相同机架，随机节点。

3、第三个副本位于不同机架，随机节点。

![image-20211006113741887](https://raw.githubusercontent.com/Jasong321/PicBed/master/image-20211006113741887.png?token=AL2KPOPQ3XUSJKF4FSIZOXDBMLPQU)

# MapReduce

## 介绍

MapReduce是一种编程模型，用于大规模数据集（大于1TB）的并行运算。概念"Map（映射）"和"Reduce（归约）"，和它们的主要思想，都是从函数式编程语言里借来的，还有从矢量编程语言里借来的特性。它极大地方便了编程人员在不会分布式并行编程的情况下，将自己的程序运行在分布式系统上。 当前的软件实现是指定一个Map（映射）函数，用来把一组键值对映射成一组新的键值对，指定并发的Reduce（归约）函数，用来保证所有映射的键值对中的每一个共享相同的键组。

## 作业运行流程

![img](https://raw.githubusercontent.com/Jasong321/PicBed/master/8aab5880-d171-30f7-91d6-aaacba2d03ce.jpg?token=AL2KPOIKKOI4K6TBYRPNBW3BMLPRE)

![img](https://raw.githubusercontent.com/Jasong321/PicBed/master/e1090dee-ee98-30d1-ad55-2f88f774fa73.jpg?token=AL2KPOL5FAVN3CUAC3BYB5TBMLPQW)

### Map端流程

1、每个输入分片会让一个map任务来处理，默认情况下，以HDFS的一个块的大小（默认为64M）为一个分片，当然我们也可以设置块的大小。map输出的结果会暂且放在一个环形内存缓冲区中（该缓冲区的大小默认为100M，由io.sort.mb属性控制），当该缓冲区快要溢出时（默认为缓冲区大小的80%，由io.sort.spill.percent属性控制），会在本地文件系统中创建一个溢出文件，将该缓冲区中的数据写入这个文件。

2、在写入磁盘之前，线程首先根据reduce任务的数目将数据划分为相同数目的分区，也就是一个reduce任务对应一个分区的数据。这样做是为了避免有些reduce任务分配到大量数据，而有些reduce任务却分到很少数据，甚至没有分到数据的尴尬局面。其实分区就是对数据进行hash的过程。然后对每个分区中的数据进行排序，如果此时设置了Combiner，将排序后的结果进行Combine操作，这样做的目的是让尽可能少的数据写入到磁盘。

3、当map任务输出最后一个记录时，可能会有很多的溢出文件，这时需要将这些文件合并。合并的过程中会不断地进行排序和combine操作，目的有两个：1.尽量减少每次写入磁盘的数据量；2.尽量减少下一复制阶段网络传输的数据量。最后合并成了一个已分区且已排序的文件。为了减少网络传输的数据量，这里可以将数据压缩，只要将mapred.compress.map.out设置为true就可以了。

4、将分区中的数据拷贝给相对应的reduce任务。有人可能会问：分区中的数据怎么知道它对应的reduce是哪个呢？其实map任务一直和其父TaskTracker保持联系，而TaskTracker又一直和JobTracker保持心跳。所以JobTracker中保存了整个集群中的宏观信息。只要reduce任务向JobTracker获取对应的map输出位置就ok了哦。

### Reduce端流程

1、Reduce会接收到不同map任务传来的数据，并且每个map传来的数据都是有序的。如果reduce端接受的数据量相当小，则直接存储在内存中（缓冲区大小由mapred.job.shuffle.input.buffer.percent属性控制，表示用作此用途的堆空间的百分比），如果数据量超过了该缓冲区大小的一定比例（由mapred.job.shuffle.merge.percent决定），则对数据合并后溢写到磁盘中。

2、随着溢写文件的增多，后台线程会将它们合并成一个更大的有序的文件，这样做是为了给后面的合并节省时间。其实不管在map端还是reduce端，MapReduce都是反复地执行排序，合并操作，现在终于明白了有些人为什么会说：排序是hadoop的灵魂。

3、合并的过程中会产生许多的中间文件（写入磁盘了），但MapReduce会让写入磁盘的数据尽可能地少，并且最后一次合并的结果并没有写入磁盘，而是直接输入到reduce函数。

### Partition分区

分区标明了数据应该去往哪个ReduceTask。

### 排序

排序是MapReduce最重要的操作之一

MapTask和ReduceTask均会对数据按照key进行排序，该操作属于Hadoop的默认行为。任何应用程序中的数据均会被排序，而不管逻辑上是否需要。

默认排序是按照字典顺序排序，且实现该排序的方法是快速排序。

### Combiner合并

对MapTask局部的运行结果进行汇总。它本质上也是在做reduce操作。它是一个可选流程。

例：

3个（a,1）合并成（a,3）

它帮助减少IO。

### Shuffle

Map方法之后，Reduce方法之前的数据处理过程称之为Shuffle，目的是对数据进行分区排序。

![image-20211010193624051](https://raw.githubusercontent.com/Jasong321/PicBed/master/202110102051542.png?token=AL2KPOMSUISQQSEXONBYPCDBMLRBA)

## MapReduce进程

一个完整的MapReduce程序包含3个进程。

1. MrAppMaster:负责整个程序的过程调度及状态协调
2. MapTask:Map阶段的数据处理流程
3. ReduceTask:Reduce阶段的数据处理流程

