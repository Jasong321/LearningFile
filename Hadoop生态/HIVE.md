# Hadoop是什么？

Hadoop是由java语言编写的，在分布式服务器集群上存储海量数据并运行分布式分析应用的开源框架，其核心部件是<font color=red>**HDFS**</font>与<font color=red>**MapReduce**</font>。

​       HDFS是一个分布式文件系统：引入存放文件元数据信息的服务器**Namenode**和实际存放数据的服务器**Datanode**，对数据进行分布式储存和读取。

　　MapReduce是一个计算框架：MapReduce的核心思想是把计算任务分配给集群内的服务器里执行。通过对计算任务的拆分（Map计算/Reduce计算）再根据任务调度器（JobTracker）对任务进行分布式计算。



