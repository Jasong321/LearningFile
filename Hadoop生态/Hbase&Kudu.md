# HBase

## 基础知识

[快速入门Hbase](https://zhuanlan.zhihu.com/p/145551967?utm_source=wechat_session)

[中文版官方文档](http://abloz.com/hbase/book.html#architecture)

​		HBASE是一个高可靠性、高性能、面向列、可伸缩的分布式存储系统，利用HBASE技术可在廉价PC Server上搭建起大规模结构化存储集群。

　　HBASE的目标是存储并处理大型的数据，更具体来说是仅需使用普通的硬件配置，就能够处理由成千上万的行和列所组成的大型数据。

　　HBASE是Google Bigtable的开源实现，但是也有很多不同之处。比如：Google Bigtable使用GFS作为其文件存储系统，HBASE利用Hadoop HDFS作为其文件存储系统；Google运行MAPREDUCE来处理Bigtable中的海量数据，HBASE同样利用Hadoop MapReduce来处理HBASE中的海量数据；Google Bigtable利用Chubby作为协同服务，HBASE利用Zookeeper作为协同服务。

### NoSql?

HBase是一种 "NoSQL" 数据库. "NoSQL"是一个通用词表示数据库不是RDBMS ，后者支持 SQL 作为主要访问手段。技术上来说, HBase 更像是"数据存储(Data Store)" 多于 "数据库(Data Base)"。因为缺少很多RDBMS特性, 如列类型，第二索引，触发器，高级查询语言等。

然而, HBase 有许多特征同时支持线性化和模块化扩充。 HBase 集群通过增加RegionServers进行扩充。 它可以放在普通的服务器中。例如，如果集群从10个扩充到20个RegionServer，存储空间和处理容量都同时翻倍。 RDBMS 也能很好扩充， 但仅对一个点 - 特别是对一个单独数据库服务器的大小 - 同时，为了更好的性能，需要特殊的硬件和存储设备。 HBase 特性：

- 强一致性读写: HBase 不是 "最终一致性(eventually consistent)" 数据存储. 这让它很适合高速计数聚合类任务。
- 自动分片(Automatic sharding): HBase 表通过region分布在集群中。数据增长时，region会自动分割并重新分布。
- RegionServer 自动故障转移
- Hadoop/HDFS 集成: HBase 支持本机外HDFS 作为它的分布式文件系统。
- MapReduce: HBase 通过MapReduce支持大并发处理， HBase 可以同时做源和目标.
- Java 客户端 API: HBase 支持易于使用的 Java API 进行编程访问.
- Thrift/REST API: HBase 也支持Thrift 和 REST 作为非Java 前端.
- Block Cache 和 Bloom Filters: 对于大容量查询优化， HBase支持 Block Cache 和 Bloom Filters。
- 运维管理: HBase提供内置网页用于运维视角和JMX 度量.

### HBase的使用场景

HBase不适合所有问题.

首先，确信有足够多数据，如果有上亿或上千亿行数据，HBase是很好的备选。 如果只有上千或上百万行，则用传统的RDBMS可能是更好的选择。因为所有数据可以在一两个节点保存，集群其他节点可能闲置。

其次，确信可以不依赖所有RDBMS的额外特性 (e.g., 列数据类型, 第二索引, 事物,高级查询语言等.) 一个建立在RDBMS上应用，如不能仅通过改变一个JDBC驱动移植到HBase。相对于移植， 需考虑从RDBMS 到 HBase是一次完全的重新设计。

第三， 确信你有足够硬件。甚至 HDFS 在小于5个数据节点时，干不好什么事情 (根据如 HDFS 块复制具有缺省值 3), 还要加上一个 NameNode.

HBase 能在单独的笔记本上运行良好。但这应仅当成开发配置。

HBase在HDFS之上提供了**高并发的随机写和支持实时查询**，这是HDFS不具备的。



### HBase的存储机制

HBase是一个面向列的数据库，在表中它由行排序。表模式定义只能列族，也就是键值对。一个表有多个列族以及**每一个列族可以有任意数量的列**。后续列的值连续存储在磁盘上。表中的每个单元格值都具有时间戳。总之，在一个HBase：

- 表是行的集合。

- 行是列族的集合。
- 列族是列的集合。
- 列是键值对的集合。

　　这里的列式存储或者说面向列，其实说的是列族存储，HBase是根据列族来存储数据的。列族下面可以有非常多的列，**列族在创建表的时候就必须指定**。

### RowKey

与nosql数据库一样，row key是用来表示唯一一行记录的**主键**，HBase的数据是按照RowKey的**字典顺序**进行全局排序的，所有的查询都只能依赖于这一个排序维度。访问HBASE table中的行，只有三种方式：

1. 通过单个row key访问；
2. 通过row key的range（正则）
3. 全表扫描

Row key 行键（Row key）可以是任意字符串(最大长度是64KB，实际应用中长度一般为10-1000bytes)，在HBASE内部，row key保存为字节数组。存储时，数据按照Row key的字典序(byte order)排序存储。**设计key时，要充分利用排序存储这个特性，将经常一起读取的行存储放到一起。(位置相关性)**

### Columns Family列族

列簇：HBASE表中的每个列，都归属于某个列族。列族是表的schema的一部分(而列不是)，必须在使用表之前定义。列名都以列族作为前缀。例如courses：history，courses：math 都属于courses这个列族。

在生产环境中要控制列族的个数，以防止出现小文件

### Cell

由{row key，columnFamily，version} 唯一确定的单元。cell中的数据是没有类型的，全部是字节码形式存储。

### TimeStamp

​		HBASE中通过rowkey和columns确定的为一个存储单元称为cell。每个cell都保存着同一份数据的多个版本。版本通过时间戳来索引。时间戳的类型是64位整型。时间戳可以由HBASE(在数据写入时自动)赋值，此时时间戳是精确到毫秒的当前系统时间。时间戳也可以由客户显示赋值。如果应用程序要避免数据版本冲突，就必须自己生成具有唯一性的时间戳。每个cell中，不同版本的数据按照时间倒序排序，即最新的数据排在最前面。

　　为了避免数据存在过多版本造成的管理(包括存储和索引)负担，HBASE提供了两种数据版本回收方式。一是保存数据的最后n个版本，而是保存最近一段时间内的版本(比如最近7天)。用户可以针对每个列族进行设置。



### 命名空间

![image-20211216225241932](https://raw.githubusercontent.com/Jasong321/PicBed/master/202112162252048.png)

1. Table：表，所有的表都是命名空间的成员，即表必属于某个命名空间，如果没有指定，
   则在 default 默认的命名空间中。

2) RegionServer group：一个命名空间包含了默认的 RegionServer Group。
3) Permission：权限，命名空间能够让我们来定义访问控制列表 ACL（Access Control List）。
例如，创建表，读取表，删除，更新等等操作。
4) Quota：限额，可以强制一个命名空间可包含的 region 的数量。

### Hbase数据模型

#### 逻辑模型

![image-20211218150601186](https://raw.githubusercontent.com/Jasong321/PicBed/master/202112181506314.png)

#### 物理模型

![image-20211218150825470](https://raw.githubusercontent.com/Jasong321/PicBed/master/202112181508562.png)

相同RowKey，一个列族在一个region里只会有一个文件，如果一个列族有多个文件一定是放在不同的region里的



## HBase原理

### Hbase架构

![image-20211218141135035](https://raw.githubusercontent.com/Jasong321/PicBed/master/202112181411299.png)

从图中可以看出 Hbase 是由 Client、Zookeeper、Master、HRegionServer、HDFS 等
几个组件组成，下面来介绍一下几个组件的相关功能：
1）Client
Client 包含了访问 Hbase 的接口，另外 Client 还维护了对应的 cache 来加速 Hbase 的访问，比如 cache 的.META.元数据的信息。
2）Zookeeper
HBase 通过 Zookeeper 来做 master 的高可用、RegionServer 的监控、元数据的入口以及集群配置的维护等工作。

具体工作如下：

- 通过 Zoopkeeper 来保证集群中只有 1 个 master 在运行，
- 如果 master 异常，会通过竞争机制产生新的 master 提供服务通过 Zoopkeeper 来监控 RegionServer 的状态，
- 当 RegionSevrer 有异常的时候，通过回调的形式通知 Master RegionServer 上下线的信息

- 通过 Zoopkeeper 存储元数据的统一入口地址

3）Hmaster
master 节点的主要职责如下：

- 为 RegionServer 分配 Region
- 维护整个集群的负载均衡
- 维护集群的元数据信息
- 发现失效的 Region，并将失效的 Region 分配到正常的 RegionServer 上
- 当 RegionSever 失效的时候，协调对应 Hlog 的拆分
- master不参加读写流程，只有DML需要master

4）HregionServer

- HregionServer 直接对接用户的读写请求，是真正的“干活”的节点。它的功能概括如下：
- 管理 master 为其分配的 Region
- 处理来自客户端的读写请求
- 负责和底层 HDFS 的交互，存储数据到 HDFS
- 负责 Region 变大以后的拆分
- 负责 Storefile 的合并工作

5）HDFS

HDFS 为 Hbase 提供最终的底层数据存储服务，同时为 HBase 提供高可用（Hlog 存储在HDFS）的支持，具体功能概括如下：
提供元数据和表数据的底层分布式存储服务



### 读流程

![image-20211216225608982](https://raw.githubusercontent.com/Jasong321/PicBed/master/202112162256077.png)

1）Client 先访问 zookeeper，从 meta 表读取 region 的位置，然后读取 meta 表中的数据。meta 中又存储了用户表的 region 信息；
2）根据 namespace、表名和 rowkey 在 meta 表中找到对应的 region 信息；
3）找到这个 region 对应的 regionserver；
4）查找对应的 region；
5）先从 MemStore 找数据，如果没有，再到 BlockCache 里面读；
6）BlockCache 还没有，再到 StoreFile 上读(为了读取的效率)；
7）如果是从 StoreFile 里面读取的数据，不是直接返回给客户端，而是先写入 BlockCache，再返回给客户端。

8）同时会读内存和HDFS中的数据，HDFS的数据会放在读缓存，取时间戳最大的

### 写流程

![image-20211216234225847](https://raw.githubusercontent.com/Jasong321/PicBed/master/202112162342947.png)

1）Client 向 HregionServer 发送写请求；
2）HregionServer 将数据写到 HLog（write ahead log）。为了数据的持久化和恢复；
3）HregionServer 将数据写到内存（MemStore）；
4）反馈 Client 写成功。

> 注：从源码上看，在内存中创建了WAL，往WAL里写数据，先不同步，开始往内存中写数据，写完了再同步WAL到HDFS。任何一步出错就会回滚

### [Hbase的两种缓存结构](https://blog.csdn.net/weixin_42712704/article/details/97089891)

#### MemStore

1、其中MemStore称为写缓存
2、HBase执行写操作首先会将数据写入MemStore，并顺序写入HLog，
//代码中这样，我们的理解为 先顺序写入HLog 再将数据写入MemStore
3、等满足一定条件后统一将MemStore中数据刷新到磁盘，这种设计可以极大地提升HBase的写性能。
4、MemStore对于读性能也至关重要，假如没有MemStore，读取刚写入的数据就需要从文件中通过IO查找，这种代价显然是昂贵的！

#### BlockCache

1、BlockCache称为读缓存
2、HBase会将一次文件查找的Block块缓存到Cache中，以便后续同一请求或者邻近数据查找请求，可以直接从内存中获取，避免昂贵的IO操作。

### 数据Flush过程

1、当 MemStore 数据达到阈值（默认是 128M，老版本是 64M），将数据刷到硬盘，将内存
中的数据删除，同时删除 HLog 中的历史数据；

2、将数据存储到HDFS中

3、在HLOG中做标记点

### 数据合并过程

1、当数据块达到 4 块，Hmaster 触发合并操作，Region 将数据块加载到本地，进行合并；

2、当合并的数据超过 256M，进行拆分，将拆分后的 Region 分配给不同的 HregionServer
管理；

3、当HregionServer宕机后，将HregionServer上的hlog拆分，然后分配给不同的HregionServer加载，修改.META.；

4、注意：HLog 会同步到 HDFS。

5、小合并不会删除“删除类型”的数据，大合并会

### 数据删除机制

Flush和Major Compact会把非最新的数据删除掉。只有Major Compact会删除删除标记的数据。

Flush：可以删除内存中的数据，磁盘中的数据不操作
Major Compact：删除需要被删掉的数据

### 数据切分

默认情况下，每个table起初只有一个Region，随着数据的不断写入，region会自动进行拆分。刚拆分时，两个子Regiond都位于当前的Region Server，但出于负载均衡的考虑，Hmaster有可能会将某个Region转移给其他的Region Server。

Region Split时机：

0.94版本之前：hbase.hregion.max.filesize，当某个region的某个列族超过这个大小会进行region切分，默认的大小是10G。

0.94版本之后：当一个region中某个store下所有StoreFile的总大小超过
$$
Min(R^2*hbase.hregion.memstore.flush.size,hbase.hregion.max.filesize)
$$
该Region就会进行拆分，其中R为当前Region Server中属于该table的region个数，hbase.hregion.memstore.flush.size默认128M。

> 数据自动切分有可能产生数据倾斜的问题，需要在建表的时候做好预分区



## Hbase操作

### Shell命令

1、启动shell

```bash
bin/hbase shell 
```

2、查看有那些表

```bash
list
```

3、创建表

```sql
create 'student','info'
```

4、插入数据到表

```bash
put 'student','1001','info:sex','male'
```

5、扫描查看表数据

```bash
scan 'student'
scan 'student',{STARTROW=>'1001',STOPROW=>'1001'}
```

6、查看表结构

```sql
describe 'student
```

7、更新指定字段的数据

```bash
put 'student','1001','info:name','Nick'
```

8、查看指定ROW、指定列族的数据

```bash
get 'student','1001'
get 'student','1001','info:name'
```

9、统计表数据行数

```sql
count 'student'
```

10、删除某个rowkey的全部数据

```bash
deleteall 'student','1001'
```

11、删除某rowkey的一行数据

```bash
delete 'student','1002','info:sex'
```

12、清空表数据

```sql
truncate 'student'
```

13、删除表

首先要让表进入disable状态

```bash
disable 'student'
```

然后才能drop该表

```sql
drop 'student'
```

14、变更表信息

将info列族的数据存放3个版本

```bash
alter 'student',{NAME=>'info',VERSIONS=>3}
get 'student','1001',{COLUMN=>'info:name',VERSIONS=3}
```

## Hbase调优

### 预分区

每一个region维护着StartRow和EndRow，如果加入的数据符合某个Region维护的RowKey范围，则该数据交给这个Region维护。那么依照这个原则，我们可以将数据所要投放的分区提前大致的规划好，以提高Hbase性能。

手动设定预分区

```bash
create 'staff1','info','partition1',SPLITS =>['1000','2000','3000','4000'];
```

### Rowkey设计原则

1、散列性：均匀分布在不同的分区里

2、唯一性

3、长度：70-100位

如何设计Rowkey?

首先要设计预分区

接着可以使用生成随机数、hash、散列值+字符串反转（时间戳）+字符串拼接等方法

如何保证相同的信息存放在相同的分区里？

比如说要把相同的手机号取模，放在模对应的分区里

### 内存优化

HBase 操作过程中需要大量的内存开销，毕竟 Table 是可以缓存在内存中的，一般会分

配整个可用内存的 70%给 HBase 的 Java 堆。但是不建议分配非常大的堆内存，因为 GC 过

程持续太久会导致 RegionServer 处于长期不可用状态，一般 16~48G 内存就可以了，如果因

为框架占用内存过高导致系统内存不足，框架一样会被系统服务拖死。

### 基础优化

1、允许在 HDFS 的文件中追加内容

hdfs-site.xml、hbase-site.xml

```bash
属性：dfs.support.append
解释：开启 HDFS 追加同步，可以优秀的配合 HBase 的数据同步和持久化。默认值为 true。
```

2、优化 DataNode 允许的最大文件打开数

hdfs-site.xml

```bash
属性：dfs.datanode.max.transfer.threads
解释：HBase 一般都会同一时间操作大量的文件，根据集群的数量和规模以及数据动作，
设置为 4096 或者更高。默认值：4096
```

3、优化延迟高的数据操作的等待时间

hdfs-site.xml

```bash
属性：dfs.image.transfer.timeout
解释：如果对于某一次数据操作来讲，延迟非常高，socket 需要等待更长的时间，建议把
该值设置为更大的值（默认 60000 毫秒），以确保 socket 不会被timeout掉。
```

4、优化数据的写入效率

mapred-site.xml

```bash
属性：
mapreduce.map.output.compress
mapreduce.map.output.compress.codec
解释：开启这两个数据可以大大提高文件的写入效率，减少写入时间。第一个属性值修改为true，第二个属性值修改为org.apache.hadoop.io.compress.GzipCodec 或者其他压缩方式。
```

5、设置 RPC 监听数量

hbase-site.xml

```bash
属性：hbase.regionserver.handler.count
解释：默认值为 30，用于指定 RPC 监听的数量，可以根据客户端的请求数进行调整，读写请求较多时，增加此值。
```

6、优化 HStore 文件大小

hbase-site.xml

```bash
属性：hbase.hregion.max.filesize
解释：默认值 10737418240（10GB），如果需要运行 HBase 的 MR 任务，可以减小此值，因为一个 region 对应一个 map 任务，如果单个 region 过大，会导致 map 任务执行时间过长。该值的意思就是，如果 HFile 的大小达到这个数值，则这个 region 会被切分为两个 Hfile。
```

7、优化 hbase 客户端缓存

hbase-site.xml

```bash
属性：hbase.client.write.buffer
解释：用于指定 HBase 客户端缓存，增大该值可以减少 RPC 调用次数，但是会消耗更多内存，反之则反之。一般我们需要设定一定的缓存大小，以达到减少 RPC 次数的目的。
```

8、指定 scan.next 扫描 HBase 所获取的行数

hbase-site.xml

```bash
属性：hbase.client.scanner.caching
解释：用于指定 scan.next 方法获取的默认行数，值越大，消耗内存越大。
```

9、flush、compact、split 机制

当 MemStore 达到阈值，将 Memstore 中的数据 Flush 进 Storefile；compact 机制则是把 flush出来的小文件合并成大的 Storefile 文件。split 则是当 Region 达到阈值，会把过大的 Region一分为二。

涉及属性:

即：128M 就是 Memstore 的默认阈值

```bash
hbase.hregion.memstore.flush.size：134217728
```

即：这个参数的作用是当单个 HRegion 内所有的 Memstore 大小总和超过指定值时，flush 该HRegion 的所有 memstore。RegionServer 的 flush 是通过将请求添加一个队列，模拟生产消费模型来异步处理的。那这里就有一个问题，当队列来不及消费，产生大量积压请求时，可能会导致内存陡增，最坏的情况是触发 OOM

```bash
hbase.regionserver.global.memstore.upperLimit：0.4
hbase.regionserver.global.memstore.lowerLimit：0.38
```

即：当 MemStore 使用内存总量达到 hbase.regionserver.global.memstore.upperLimit 指定值时，将会有多个 MemStores flush 到文件中，MemStore flush 顺序是按照大小降序执行的，直到刷新到 MemStore 使用内存略小于 lowerLimit
