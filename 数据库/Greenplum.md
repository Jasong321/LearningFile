# Greenplum架构

## 基础架构

Greenplum是基于Postgres的开源分布式数据库，从拓扑结构上看，它是单机Postgres组成的数据库集群。但不限于此，**Greenplum对外提供统一的数据库接口，让使用者感觉就在使用单机数据库一样，并且，Greenplum对集群处理做了大量优化。**

**从物理拓扑结构来说，Greenplum数据库是典型的Master-Segment结构**，一个Greenplum集群通常是由一个Master节点、一个Standby节点和多个Segment节点组成，节点之间通过高速网络互连。Master是整个数据库的入口，终端用户连接Master执行查询，Standby Master会为Master提供高可用支持。Segment节点是工作节点，数据都存在Segment上，Mirror Segment会为Segment提供高可用支持。

下图中的每个方框就是一个物理机器，**为了能获取物理机器最好的性能，一个节点可以灵活部署多个Segment进程**。在查询过程中，当Master节点接收到用户发起的查询语句时，会进行查询编译、查询优化等操作，生成并行查询计划，并分发到Segment节点执行。Segment执行完毕，会将数据发回Master节点，最终呈现给用户。

<img src="https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibGEpQmvkRNy9pkR1K3v0Xsa0IJ0RhQ9TGicXKm31O6Qvbdf3KONw728Tg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1" alt="图片"  />

**了解了Greenplum的架构，我们再来看看在Greenplum上,数据是如何存储的**。在Greenplum上，每个物理机器上会对应多个磁盘，每个磁盘上会挂载物理磁盘，每个磁盘上会存有多个数据分片。Greenplum上数据的组织会采取如下策略：

- 首先，数据会按照设定的分布策略均匀的分布在各个Segment上。Greenplum支持的分布策略包括哈希分布、随机分布和Greenplum6版本中新增加的复制分布。这个操作我们称为数据的分片。(distribute)
- 然后，对于每个节点上的数据，可以再次通过数据分区将该节点上的数据划分为更小的子集合，从而在查询时代价更小。这个操作我们称为数据的分区。(partition)

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibGShaWZUEibcVCzWiav9Tryxqe3eb2B4d1OiaTnwkWFXdgywWo8XGVc89Lw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**接着,我们来看看Greenplum的服务流程**，下面的例子中，左边是一个Master节点，右边是两个Segment节点。前面提到从拓扑结构来看，Greenplum是由单机Postgres组成的数据库集群，例子中总共有三个Postgres进程。当客户端有请求时，会通过Postgres的libpq协议链接到Greenplum的Master节点。Master上的postmaster进程监听到连接请求后，会创建一个子进程（QD）用于处理该客户端所有的查询请求。QD和原来的master之间会通过共享内存来进行数据的共享。

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibG7qNyjTveVeib7IxXakVtOHibxgBiatsOIhDib51r3HPWu9r8icXJ93ObRYg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

然后是QD与各个Segment间建立连接。QD可以看作为每个Segment的客户端，QD进程会发连接请求到所有的Segment节点上，并通过libpq协议和每个Segment建立连接请求，Segment上的postmaster进程监听到QD的连接请求后，也会创建一个子进程（QE）以处理后续的查询请求。**QD的全称是Query Dispatcher，是一个分发器，而QE是Query Executer，是处理查询的。所有的QD和QE进程会共同来完成客户端发送的查询请求。**

**准备工作做好后，查询是怎么进行的呢？**客户端将查询请求发给Master上的QD进程后，QD进程会对收到的查询进行处理，包括解析原始查询语句、优化器生成分布式查询计划等，并通过libpq协议发送给各个Segment上的QE进程。QD和每个Segment上的QEs会建立interconnect连接。需要注意的是，**libpq主要用于控制命令和结果返回，而 Interconnect用于内部数据通信**。

![图片](https://mmbiz.qpic.cn/mmbiz_gif/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibGkIu5CquhrEQaXM1h3sEPmjWkYB3HdGIUDC8CnNPrFwibGqlV9WqCnzA/640?wx_fmt=gif&tp=webp&wxfrom=5&wx_lazy=1)

每个QE进程会执行分配给它的查询子任务，并将结果返回给QD，**QE之间也是是通过Interconnect交互数据，没有libpq链接（QE之间通信为什么不再用libpq?1.因为libpq主要是用来传控制信息，不是用来传数据的，都放在一起传输会造成堵塞。）**。QE和QD之间通过Libpq协议进行状态更新和管理，包括错误处理等。最终QD会将收集到的查询结果进行汇总，并通过libpq返回给客户端。

![图片](https://mmbiz.qpic.cn/mmbiz_gif/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibGttLZg3PWgPaJX21Y9rArZyh3sdnso1Bkch79QzgB1SwMEsIpp6veIQ/640?wx_fmt=gif&tp=webp&wxfrom=5&wx_lazy=1)

在执行比较复杂的查询时，比如先select两个表然后再将两个表做JOIN运算，此时整个查询就被分为了两层，第一层数据读取出来，再进行中间的数据交换，第二层会将相同的链接键的值放在一起进行JOIN，最后再将JOIN的结果返回给客户端。PostgreSQL是进程模型，为每个查询创建单个进程进行处理。为了提高CPU利用率，Greenplum实现了Segment内算子之间的查询并行化。用户查询在单个Segment上可以有多个QE，QE之间通过Interconnect交互数据。

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibGrqnqjsFibmoWJ0QuHvSibOBuBfib2xuOe4UWPjnTmTnMViaNB4LLRbPodg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



## 存储管理

教科书里经常会提到存储金字塔。在金字塔分布中，越往上走容量越小，但越来越快，越往下走，容量越大，但越来越慢。内存以上的存储是易失存储，很快但容易丢失数据，这就是易失存储。而非易失存储相对慢，但是不易丢失数据。

<img src="https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibG55m6cnxBQkgibWnlEgvOjJbLtc8MawU7TZdL6mpKocSda6TK3tohSXg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1" alt="图片"  />

在写程序的时候，我们会通过cache或者buffer来访问文件。 **Greenplum的进程中通过共享缓冲区域来作为中间的内存buffer。**Beckend进程会和中间这一层直接打交道。而共享内存中的buffer会和磁盘文件做交换。

<img src="https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibG3B2vXSaG6ibdcStoib5wibicDWiabK44zpRgCduyJTpeXvuJ3xMzicHGYB7A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1" alt="图片"  />

那么在共享缓冲区里会发生什么呢？Greenplum是按块来组织数据，图中的虚线方块就是共享内存区。映射表会通过块号去找到共享缓冲块对应的块。如果发现内容是无效的会通过下层的文件操作将数据加载到共享缓冲块中。如果有效，就会直接读取。

Greenplum会将变长数据在每个文件块内从后往前存储，在页头存的是定长的指针，指向了块尾部真正数据的块内偏移。通过指针可以快速的查找第n个数据。中间区域是空闲区域，数据会从后往前的生长，Item Id会从前往后生长，中间区域会保留连续的空间减少碎片。

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibGic6sxeqQrN2MjYPF7ChRV14NFnStyM6YfFXzkEOljOytic7ib7Gol3wtQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**对于Greenplum，读和读是可以同时进行的，读和写也是可以同时进行的。**为了让读和写可以同时进行，Greenplum提供了多版本的控制机构MVCC（多版本并发控制），来提高并发度。当插入数据时，会保存两个隐藏字段：xmin和xmax，分别表示插入事务和删除事务的编号。

在下面的例子里，图（c)对B0数据执行了删除，于是在对于该数据增加了xmax值18。如果是事务17在运行中，由于事务17在事务18之前，因此对于这个操作是看不见的，因此B0值对于事务17仍然可见，事务17不会认为数据已经被删除。而对于事务19来说，B0就已经被删除了。而UPDATE会拆成两个操作：DELETE和INSERT。例如例子中的图（e)。而**如果写和写同一个表中同时进行，并且涉及同一元组，就会根据隔离级别进行处理。**当被删除的元组对于当前所有的运行事务都不可见了，VACUUM会回收磁盘空间。

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibGWt2pjic42eho2bc7HEzxVEdqvF4RhcC8dLyK2zjpzuIea2ibKtKQgcUw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 查询执行

现在，对于元组的增删查改都可以支持了，但这远远不够，**为了方便用户的使用和提供更为强大好用的功能，Greenplum提供了执行引擎**。在执行查询时，SQL语句经过解析器，会将字符串的SQL语句解析成结构化的树，通过优化器做出最有效的执行计划，并有执行器执行生成查询结果。

<img src="https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibGVh1gYy5qxW32ibWPaAFPsvkwR4RqNR3Pu7yibTKGQibBJljYgpzc3WkicQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1" alt="图片"  />

下面的例子中的SQL语句对两个表进行了连接操作。**在Greenplum中，执行器是通过迭代器的方式来执行查询计划的，即自顶向下，每次一条元组的方式。**每个执行器节点向上提供下一条元组，向下获取下一条元组。一条语句可能存在多条查询计划，比如前面讲到的顺序扫描和索引扫描，查询优化器会足够聪明选择代价最小的执行计划。在例子中，最下面是对两个表的扫描操作，扫描结束后，为了执行Join，需要将Bars表按照名字进行元组的重分布，让具有连接条件的Bars元组和Sells元组能够汇聚在一起。由于Sells已经按照bar分布，所以这里不需要再对Sells做重分布。最后，做完投影运算后，需要把结果汇聚到QD节点，这是由最顶层的Gather Motion来完成的。

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibGbDICXBoiaGEOMnjkhLqU7BZbq1VmK8k9T4NNqbHMSnd8f8kNfebYtMg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibGRZDPePASQqKicYDeMDwice4wvxfHueZ0z1VJibHVpotrEFwlOazd1icmOA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**如果采取索引扫描，一个存在的问题是索引执行的元组在文件中，导致磁盘的随机访问。**

**一种解决办法是Greenplum Cluster操作。**Cluster操作会按照索引的顺序将文件中的元组重新排序，这样按照索引扫描顺序时，就可以按照顺序的方式访问磁盘。

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibGnN15UD3mhS1db10W52ronF4GyYWeHGMvU73icuxcP3Hricy2BznKqibbg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**另外一种解决办法，特别是如果是多个条件的情况下，可以基于位图来扫描**。在下面的例子中，“date”中有一个索引，“beer”有一个索引，根据查询条件可以获得每个条件的位图信息，位图中记录了哪些元组是否满足查询条件，满足查询条件的元组用位图中的1来表示。两个查询条件各自对应的位图，通过位图AND操作得到最终待扫描位图。基于位图扫描就可以用顺序访问的方式来访问文件。

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibG5SFPlDQtkVRNjwG9W3lGOvG3v8spEBiaUbOfw6UVIIIPyMqq193qHNw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**Greenplum里元组的连接（JOIN）操作主要有三种，第一种是Nested Loop Join**，和之前说的文件存储类似，即两个循环叠加起来，将内外的扫描匹配，返回结果。这里可能的一个变种是，内层循环有可能用到索引而不是顺序扫描，让执行效率更好。

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibGTchTcWmIHfH2TNVEtsPH8e0NH5XhSHD9otjBEyEIQjC9jSOlw667aQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**第二种就是Merge Join**。Merge Join有两个阶段，第一阶段是根据连接条件将待连接的元组分别排序。然后在第二阶段基于排好序的元组进行归并操作。

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibGgPp2mNES91Pzc3mQ6MqjicwmZLoONt7yDcIaQ9LvsnFTLIqAZX3Bp5A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibGMtic8fLAKFzb15Fa6FqIESVAYibnCjykLOYus2dTAC4dlBQmO8VJBNCw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**第三种方式是Hash Join**。在Hash Join时，一般是将一个表作为查找表，另一个小表作成Hash表。Hash表如果足够小，可以放在内存里。每次查找时，对查找表进行扫描，看看当前元组是否在Hash表中有匹配项，有的话，直接返回，没有就跳过。但是存在的问题是，如果Hash表如果过大怎么办？需要将哪些元组存放到外存中？Outter待匹配的元组发现需要匹配的Hash表元组在外存怎么处理？

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibGZiav3v2uyTwubl4E9VgYSrRJV4tE8m5KDKhD1MYp3dcicCM9mLniaSa0Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 物理Join

在多表联合查询的时候，如果我们查看它的执行计划，就会发现里面有多表之间的连接方式。多表之间的连接有三种方式：Nested Loops，Hash Join 和 Sort Merge Join.具体适用哪种类型的连接取决于

- 当前的优化器模式 （ALL_ROWS 和 RULE）
- 取决于表大小
- 取决于连接列是否有索引
- 取决于连接列是否排序

1、**NESTED LOOP:嵌套循环连接**

Nested loops 工作方式是循环从一张表中读取数据(驱动表outer table)，然后访问另一张表（被查找表 inner table,通常有索引）。驱动表中的每一行与inner表中的相应记录JOIN。类似一个嵌套的循环。

对于被连接的数据子集较小的情况，嵌套循环连接是个较好的选择。在嵌套循环中，内表被外表驱动，外表返回的每一行都要在内表中检索找到与它匹配的行，因此整个查询返回的结果集不能太大（大于1 万不适合），要把返回子集较小表的作为外表，而且在内表的连接字段上一定要有索引。

2、**SORT MERGE JOIN:排序合并连接**

Merge Join 是先将关联表的关联列各自做排序，然后从各自的排序表中抽取数据，到另一个排序表中做匹配。

因为merge join需要做更多的排序，所以消耗的资源更多。 通常来讲，能够使用merge join的地方，hash join都可以发挥更好的性能,即散列连接的效果都比排序合并连接要好。然而如果行源已经被排过序，在执行排序合并连接时不需要再排序了，这时排序合并连接的性能会优于散列连接。

适用情况：

1. 不等价关联(>,<,>=,<=,<>)
2. 用在没有索引，并且数据已经排序的情况
3. 无法使用hashjoin

3、**HASH JOIN:散列连接**

优化器使用两个表中较小的表（通常是小一点的那个表或数据源）利用连接键（JOIN KEY）在内存中建立散列表，将列数据存储到hash列表中，然后扫描较大的表，同样对JOIN KEY进行HASH后探测散列表，找出与散列表匹配的行。需要注意的是：如果HASH表太大，无法一次构造在内存中，则分成若干个partition，写入磁盘的temporary segment，则会多一个写的代价，会降低效率。

这种方式适用于较小的表完全可以放于内存中的情况，这样总成本就是访问两个表的成本之和。但是在表很大的情况下并不能完全放入内存，这时优化器会将它分割成若干不同的分区，不能放入内存的部分就把该分区写入磁盘的临时段，此时要有较大的临时段从而尽量提高I/O 的性能。

适用情况：

两个表数据量相差很大的情况

## 事务与日志

如果Greenplum在修改文件的过程中进程挂了，如何保证数据的一致性呢？数据库课程中经常提到一个经典问题就是：从A账户向B账户转账100，如果A账户减少了100后系统重启崩溃，此时会不会发生A减了100，而B账户没有加100？**Greenplum通过事务来保证操作的原子性。**

<img src="https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibGaTOXickQmAlPNv0HdBeEicjAibSbL3l6gVYpDKc0MoVtAJ3PlxeWUSSEA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1" alt="图片"  />

另外一个问题是事务间的隔离性问题，一个事务处理转账，另外一个事务给银行账户加息（这里是2%的利息）。如果转账事务和加息事务同时进行，如果按照图中的错误序列进行操作就会出现问题，最后会发现少了2块钱！利用事务的隔离性就可以解决这类头疼的问题。

<img src="https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibGREiaf19OmaVrZu758Es1zbia3gdvVzgiaT1fLh8oDp6td4tiaOvibwWicBaQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1" alt="图片"  />

每次写数据时，会先改内存，再写磁盘。在下面的例子中，A先在内存中被修改成了23，提交后，如果系统挂了，此时A的修改就丢了，重启时就会发现A依旧等于12，因为修改还没有来得及写回磁盘。当然，要求每次修改都写磁盘可以防止这类问题发生，但是会有效率问题。**Greenplum提供的日志功能就能很好的解决这类问题**。日志详细记录了对于数据库的修改流程。日志记录是按照顺序访问的，并且提供了逻辑时间线的概念，效率相对于磁盘的随机访问会高很多。

<img src="https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibG0ATJuiaE1JNTLcibroPgqianEk9SibWys7IP76icz790P2AlfPPFW7sNlMw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1" alt="图片"  />

前面讲到，Greenplum中，数据是存在多个Segment上的，因此要确保在写入数据时，所有Segments上的数据要么都要写入成功，要么都写入不成功。不能接受的是，部分成功，部分不成功的情况，出现数据的不一致状态。**这里需要提到一个经典的算法：两阶段提交算法。**顾名思义，该算法包含两个阶段，在第一阶段，做prepare，让所有节点投票是否都可以提交，如果所有节点回复为yes时，便会在第二阶段进行提交。否则，只要有一个节点不回复yes，全部节点进行回滚。

<img src="https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibGhhib3aIZ9wRJbPBaply2rjicybg6arVD8m46MULwsHCCZwtwm1gpPicJw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1" alt="图片"  />

下面看看错误处理的情况：



1. 第一种情况是错误发生在第一阶段，有节点不能提交，后续操作只需在第二阶段回滚即可。

   <img src="https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibG6Xf3QTDFPCUiblDX473nRRemvghRqRnHQibKibb5PYDqaU4ZsWormSaiaQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1" alt="图片"  />

2. 如果在第一阶段所有节点都回复了Yes，但是第二阶段出现问题，此时DTM管理器就会对失败的节点进行继续提交，直到成功。所有的提交信息都会被DTM存入日志，通过这些信息可以恢复出事务的状态信息，以便找出下一步需要做什么。

   <img src="https://mmbiz.qpic.cn/mmbiz_png/iaZJdHJXMOBdicIv4icFZib39G7JPibo3I0ibG3PibLTamRribf6fD1fribyQ9ddPK7uR4LUK6keg4uWWxEWtGtKRxXuDMA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1" alt="图片"  />

# 内核解析

先整理一下B树和B+树的概念，参考[此博文](https://blog.csdn.net/v_JULY_v/article/details/6530142)。



## 存储结构

### B树

B树不是二叉树，是多叉树
就像在上图中看到，一个节点如果有n个关键字，那么这个节点就会有n+1个子树
这点很好理解，就像节点key = 2 , 那么我们<2 , >2的两部分就构成了它的子树
特性 ：

1. 关键字集合分布在整颗树中；
2. 任何一个关键字出现且只出现在一个结点中；
3. 搜索有可能在非叶子结点结束；
4. 其搜索性能等价于在关键字全集内做一次二分查找；

![image-20211115225752409](https://raw.githubusercontent.com/Jasong321/PicBed/master/202111152257010.png)

以查找29为例，下面模拟一下查询步骤：

1. 根据根结点指针找到文件目录的根磁盘块1，将其中的信息导入内存。【磁盘IO操作 1次】    

2. 此时内存中有两个文件名17、35和三个存储其他磁盘页面地址的数据。根据算法我们发现：17<29<35，因此我们找到指针p2。
3. 根据p2指针，我们定位到磁盘块3，并将其中的信息导入内存。【磁盘IO操作 2次】    

4. 此时内存中有两个文件名26，30和三个存储其他磁盘页面地址的数据。根据算法我们发现：26<29<30，因此我们找到指针p2。

5. 根据p2指针，我们定位到磁盘块8，并将其中的信息导入内存。【磁盘IO操作 3次】    

6. 此时内存中有两个文件名28，29。根据算法我们查找到文件名29，并定位了该文件内存的磁盘地址。

### B+树

一棵m阶的B+树和m阶的B树的异同点在于：

1. 所有的叶子结点中包含了全部关键字的信息，及指向含有这些关键字记录的指针，且叶子结点本身依关键字的大小自小而大的顺序链接。 (而B 树的叶子节点并没有包括全部需要查找的信息)
2. **所有的非终端结点可以看成是索引部分**，结点中仅含有其子树根结点中最大（或最小）关键字。 (而B 树的非终节点也包含需要查找的有效信息)

为什么说B+树比B 树更适合实际应用中操作系统的文件索引和数据库索引？

1、B+-tree的磁盘读写代价更低

B+树的内部结点并没有指向关键字具体信息的指针。因此其内部结点相对B 树更小。如果把所有同一内部结点的关键字存放在同一盘块中，那么盘块所能容纳的关键字数量也越多。一次性读入内存中的需要查找的关键字也就越多。相对来说IO读写次数也就降低了。

2、B+树的查询效率更加稳定

由于非终结点并不是最终指向文件内容的结点，而只是叶子结点中关键字的索引。所以任何关键字的查找必须走一条从根结点到叶子结点的路。所有关键字查询的路径长度相同，导致每一个数据的查询效率相当。

### B-link树

Greenplum中使用B-link数做索引。

![image-20211116223553830](https://raw.githubusercontent.com/Jasong321/PicBed/master/image-20211116223553830.png)

B-Link树并发控制算法总结：

- 解决了朴素算法的2个问题
  - 树根的位置上的锁冲突率较高=>读写均加读锁
  - 在路径下降时加的这些锁，大概率会被马上释放掉=>下降时加读锁，写锁由下向上申请
- 死锁考虑
  - 插入操作中写锁的申请顺序都是由下向上，由左到右，不会发生死锁
  - 查询操作是可能和插入操作出现不同申请顺序（前者由上向下，后者由下向上），但是查询操作每次都是先释放再申请，因此也不会发生死锁

## 索引结构

### 物理结构

Greenplum中的索引都是二级索引（非聚集索引）

- 物理存储上是独立的文件（独立于表中的数据文件）

- 并且也是按分片存储在每个segment上，其索引内容对应segment上的数据分片

  ![image-20211116224401407](https://raw.githubusercontent.com/Jasong321/PicBed/master/image-20211116224401407.png)

- 并发控制：引入了一个moveright操作，利用到了高健和右兄弟指针，用于及时发现节点已经被分裂，如果分裂，所查找的键值一定在有兄弟节点上。如果所扫描的键值>=高健，那么这个键一定在它的右兄弟节点上。
- B-Link树放松了下降过程的锁操作：Insert操作的下降过程中也加读锁
- B-Link树查找：
  - 从根节点开始逐层下降，加读锁
  - 每次下移一层后，都调用moveright操作，检查节点是否分裂
  - 在下移或者右移操作时，都是先释放锁，然后再去申请新锁，即申请新读锁操作时并不持有锁，因此可以避免死锁（和insert操作并发时）
  - 最后到达叶子节点，加读锁，并读取内容
- B-Link树插入：
  - 开始阶段和查找操作一样，逐层下降加读锁并配合moveright，因此并行佳
  - 逐层下降到达叶子节点后，需要将读锁升级为写锁
  - 如果需要节点分裂：
    - 新建右兄弟页面，加写锁，随后将它挂入到B-tree中
    - 随后递归向上插入键值：由下向上为父节点申请写锁，随后插入到父节点的操作，然后再释放下层节点的写锁。

### 逻辑结构

<img src="https://raw.githubusercontent.com/Jasong321/PicBed/master/image-20211116224540413.png" alt="image-20211116224540413" style="zoom: 50%;" />



## 多版本并发控制（MVCC）

### 常用的并发控制技术

1、multi-version Concurrency Control(MVCC)多版本并发控制

2、Strict Two-Phase Locking(S2PL)严格的两阶段锁

3、Optimistic Concurrency Control(OCC)乐观锁



### 事务

事务：是数据库操作的最小工作单元，是作为单个逻辑工作单元执行的一系列操作；这些操作作为一个整体一起向系统提交，要么都执行、要么都不执行；事务是一组不可再分割的操作集合（工作逻辑单元）

#### 事务的四大特性

1、原子性

事务是数据库的逻辑工作单位，事务中包含的各操作要么都做，要么都不做 

2、一致性

事 务执行的结果必须是使数据库从一个一致性状态变到另一个一致性状态。因此当数据库只包含成功事务提交的结果时，就说数据库处于一致性状态。如果数据库系统 运行中发生故障，有些事务尚未完成就被迫中断，这些未完成事务对数据库所做的修改有一部分已写入物理数据库，这时数据库就处于一种不正确的状态，或者说是 不一致的状态。

3、隔离性

一个事务的执行不能其它事务干扰。即一个事务内部的操作及使用的数据对其它并发事务是隔离的，并发执行的各个事务之间不能互相干扰。

4、持续性

也称永久性，指一个事务一旦提交，它对数据库中的数据的改变就应该是永久性的。接下来的其它操作或故障不应该对其执行结果有任何影响。 

#### 事务的并发问题

1、脏读：事务A读取了事务B更新的数据，然后B回滚操作，那么A读取到的数据是脏数据

2、不可重复读：事务 A 多次读取同一数据，事务 B 在事务A多次读取的过程中，对数据作了更新并提交，导致事务A多次读取同一数据时，结果不一致

3、幻读：一个事务在读取数据时，另一个事务插入了数据，导致上个事务第二次读取数据时，数据不一致。

>  小结：不可重复读的和幻读很容易混淆，不可重复读侧重于修改，幻读侧重于新增或删除。解决不可重复读的问题只需锁住满足条件的行，解决幻读需要锁表



#### 事务的四种隔离级别

| 事务隔离级别                 | 脏读 | 不可重复读 | 幻读 |
| ---------------------------- | ---- | ---------- | ---- |
| 读未提交（read-uncommitted） | 是   | 是         | 是   |
| 不可重复读（read-committed） | 否   | 是         | 是   |
| 可重复读（repeatable-read）  | 否   | 否         | 是   |
| 串行化（serializable）       | 否   | 否         | 否   |



### MVCC概念

每个写操作都会创建数据项目的新版本，同时保留旧版本 

这种方法允许Greenplum在读写同时进行的情况下任然能提供高并发特性

MVCC最大的特点是读操作不会阻塞写操作，写操作也不会阻塞读操作

<img src="https://raw.githubusercontent.com/Jasong321/PicBed/master/202111281132801.png" alt="image-20211128110707646" style="zoom:50%;" />

#### Heap表页面布局

![image-20211128114637145](https://raw.githubusercontent.com/Jasong321/PicBed/master/202111281146288.png)

- 数据文件是由很多相同大小(32k)的block组成，block分为两部分，前面是指针信息，后面是元组，中间为空的。指针从前往后增长，元组从后往前增长
- 指针主要有下面4个信息：
  - Xmin：创建Tuple的事务ID
  - Xmax：删除Tuple的事务ID，有时用于行锁
  - Cid：事务内的查询命令编号，用户跟踪事务内部的可见性
  - Ctid：指向像一个版本tuple的指针，由两个成员blocknumber:offset做成（用于update）

#### 快照

MVCC的快照用于控制那个元组对于当前查询可见

在Read-Commited的隔离级别，每个查询开始时生成快照。在Repeatable-Read额隔离级别，在每个事务开始时生成快照

快照理论上是一个正在运行的事务列表

Greenplum用快照来判断一个事务是否已经提交

- Xmin：所有小于Xmin的事务都已经提交
- Running：正在执行的事务列表
- Xmax：所有大于等于Xmax的事务都未提交

![image-20211128170948629](https://raw.githubusercontent.com/Jasong321/PicBed/master/202111281709744.png)

> 可见：元组Xmin<快照Xmin & (元组Xmax>快照Xmax | 元组Xmax为空 | 元组Xmax正在Running)

<img src="https://raw.githubusercontent.com/Jasong321/PicBed/master/202111281723623.png" alt="image-20211128172342489" style="zoom:200%;" />

<img src="https://raw.githubusercontent.com/Jasong321/PicBed/master/202111281725658.png" alt="image-20211128172531528" style="zoom:200%;" />

![image-20211128174801498](https://raw.githubusercontent.com/Jasong321/PicBed/master/202111281748603.png)

![image-20211128174447539](https://raw.githubusercontent.com/Jasong321/PicBed/master/202111281744677.png)

![image-20211128174651811](https://raw.githubusercontent.com/Jasong321/PicBed/master/202111281746919.png)

![image-20211128174933819](https://raw.githubusercontent.com/Jasong321/PicBed/master/202111281749936.png)

并发update情况下，等上一个update完成update，再执行下一个。

<img src="https://raw.githubusercontent.com/Jasong321/PicBed/master/202111281757692.png" alt="image-20211128175712569" style="zoom:200%;" />

事务有可能失败，就需要回滚。<font color=red>回滚时Xmax并不做修改</font>。

![image-20211128180043981](https://raw.githubusercontent.com/Jasong321/PicBed/master/202111281800141.png)

有一张pg_clog的表存放事务的状态。

为了提升效率，做了一个优化，用infomask标记事务执行状态。通过这个字段，每个事务生命周期只需要访问一次snapshot和commitlog就可以确定可见性，不再需要重复访问。

![image-20211128181300507](https://raw.githubusercontent.com/Jasong321/PicBed/master/202111281813635.png)

Xmax还可以用作行锁的标记

![image-20211128181406620](https://raw.githubusercontent.com/Jasong321/PicBed/master/202111281814753.png)

## 清理

在更新元组时会创建一个新的元组，所以旧的元组需要清理。在删除元组时，只会标记Xmax，不会立即删除元组。

### 何时执行清理操作

1、在查询操作访问到某个页面时，会清理这个页面

2、通过vacuum操作来清理

![image-20211128231755701](https://raw.githubusercontent.com/Jasong321/PicBed/master/202111282317962.png)

## Heap Only Tuple

当频繁的update，索引会大量膨胀，为了解决这个问题引入了Heap Only Tuple(HOT)

Greenplum使用链式更新机制（HOT）来避免插入新的索引项，用旧Tuple的c_tid指向新Tuple

使用HOT机制的条件：更新不涉及索引列，新旧元组在同一个页面内

![image-20211128232631065](https://raw.githubusercontent.com/Jasong321/PicBed/master/202111282326164.png)

![image-20211128232746000](https://raw.githubusercontent.com/Jasong321/PicBed/master/202111282327122.png)

![image-20211128232845920](https://raw.githubusercontent.com/Jasong321/PicBed/master/202111282328033.png)

![image-20211128232959113](https://raw.githubusercontent.com/Jasong321/PicBed/master/202111282329224.png)

![image-20211128233044255](https://raw.githubusercontent.com/Jasong321/PicBed/master/202111282330362.png)

HOT链的收尾节点保留，中间的节点都是可以清理的

>清理只发生在对于任何执行中的事务都不可见的tuple上
>
>HOT发生在处于单个页面，并有相同的索引值的tuple上
>
>多数的清理工作由单页清理操作完成，但是单页清理只涉及Heap Page
>
>Vacuum则会进行更加彻底的清理，包括tuple,item,index



## 分布式锁和两阶段提交协议

### 事务的属性

数据库用什么办法来实现事务的四大属性？

| 属性         | 含义                                         | 数据库系统的实现              |
| :----------: | :------------------------------------------: | :---------------------------: |
| Atomic<br>原子性 | 事务中的操作要么全部正确执行，要么完全不执行 | WAL,分布式事务:两阶段提交协议 |
| Consistency<br>一致性 | 数据库系统必须保证事务的执行使得数据库从一个一致性状态转移到另一个一致性状态。（满足完整性约束） | 实现对A、I、D三个属性的支持 |
| Isolation<br>隔离性 | 多个事务并发地执行，对每个事务来说，它并不会感知系统中有其他事务在同时执行 | MVCC、2PL、OCC |
| Durability | 一个事务在提交之后，该事务对数据库的改变是持久的。 | WAL+存储管理 |
|              |                                              |                               |

![image-20211129224553636](https://raw.githubusercontent.com/Jasong321/PicBed/master/202111292245837.png)

### 缓冲区管理策略

![image-20211129225748432](https://raw.githubusercontent.com/Jasong321/PicBed/master/202111292257594.png)

1、Force/No-Force

Force：事务提交时，所修改的页面**必须强制**刷回到持久存储中

No-Force：事务提交时，所修改的页面**不需要强制**刷回到持久存储中

2、Steal/No-Steal

Steal：**允许**Buffer Pool里未提交事务所修改的脏页刷回到持久存储

No-Steal：**不允许**Buffer Pool里未提交事务所修改的脏页刷回到持久存储

3、Force策略的问题

对持久存储器进行频繁的随机写操作，性能下降

4、No-Steal策略的问题

不允许未提交事务的脏页换出，系统的并发量不高

5、No-Force/Steal有更好的性能，但是怎么保证事务的原子性和持久性？

- No-Force：事务提交，所修改的数据页没有刷回至持久存储，如果发生断点或者系统崩溃
- Steal：Buffer Pool中未提交的事务所修改的脏页刷回到持久存储，如果发生断电或者系统崩溃

用日志的方式解决这个问题：

- No-Force -> Redo Log<br>事务提交时，数据页不需要刷回持久存储，为了保证持久性，先把Redo Log写入日志文件。Redo Log记录修改数据对象的新值
- Steal -> Undo Log<br>允许Buffer Pool未提交事务所修改的脏页刷回到持久存储，为了保证原子性，先把Undo Log写入日志文件。Undo Log记录修改数据对象的旧值

![image-20211129232208618](https://raw.githubusercontent.com/Jasong321/PicBed/master/202111292322793.png)

![image-20211202222325932](https://raw.githubusercontent.com/Jasong321/PicBed/master/202112022223142.png)

通过Log Buffer把原来Buffer Pool的随机写操作变成了顺序写操作（日志满了就写入/事务提交就写入），从而提升了写入的速度。

### Greenplum和PostgreSql采用的策略

1、Steal + No-force

2、redo log，没有undo log，事务回滚不需要做undo操作，因为PG采用的是MVCC，更新操作不是in-place update,而是重新创建tuple，可见性判断

思考：

1、Mysql同样采用MVCC，事务恢复时候为什么需要undo log?

![image-20211202224707934](https://raw.githubusercontent.com/Jasong321/PicBed/master/202112022247503.png)

> Greenplum中对表的修改会把历史的数据完整的保留下来，而在Mysql中存的是数据的差异变化。

### 两阶段提交协议

![image-20211202230231667](https://raw.githubusercontent.com/Jasong321/PicBed/master/202112022302827.png)

![image-20211202230430216](https://raw.githubusercontent.com/Jasong321/PicBed/master/202112022304379.png)

两阶段提交协议需要处理的故障

1、参与者故障

参与者恢复后，根据日志记录来决定重做或撤销事务T，是否有<Ready T>记录？是否有<Commit T>或者<Rollback T>，如果没有，可以询问参与者

2、协调者故障

如果协调者发生故障，参与者必须决定提交或者撤销事务，在某些情况下，参与者并不知道是否提交事务，所以必须等协调者从失败中恢复

#### PostgreSql的两阶段提交

![image-20211204170819314](https://raw.githubusercontent.com/Jasong321/PicBed/master/202112041708825.png)

问题：

1、协调者向参与者发prepare之后，参与者完成prepare相应操作，在发送ready之前，会把日志落盘，那参与者申请的锁会不会释放?

> 答：通过select * from pg_locks，会观察到，这个事务申请的RowExclusive锁还在pg_lock里，所以它并没有释放。因为这个事务还没有完成。

2、在PG里，执行完PREPARE语句之后，此时把数据库停掉（或者杀掉所有数据库进程）再启动起来，会发现pg_locks里，prepared事务所申请的还在pg_lock表里。既然pg_locks是一个内存的数据结构，记录各个backend进程申请的锁，那数据库重启后，为什么已经prepared事务申请的锁仍在pg_lock表呢？

> 答：当执行prepare时候，PG会把该事务的lock信息当做prepare日志记录的一部分记录在日志文件(xlog)里。当数据库重新启动，会读这个日志文件的日志记录，把锁还原到pg_lock表里。
>
> 1. StartupXlog函数发现XLOG_XACT_PREPARE日志记录进行redo，调用函数recreateTwoPhaseFile将该日志记录中的信息放到pg_twophase目录下的文件里，每一个prepared事务对应一个文件
> 2. StartupXlog函数调用recoverPreparedTransaction函数读取pg_twophase目录下的文件并进行相关操作，为该事务重新获取锁
> 3. 恢复成功后，删掉pg_twophase目录下的文件

#### Greenplum实现分布式事务与并发控制

![image-20211204174429054](https://raw.githubusercontent.com/Jasong321/PicBed/master/202112041744388.png)
