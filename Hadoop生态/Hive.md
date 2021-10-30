# 基本概念

## 什么是Hive

Hive是由Facebook开源用于解决海量结构化日志的数据统计。

它是基于Hadoop的一个数据仓库工具，可以将结构化的数据文件映射为一张表，并提供类SQL的查询功能。

它的本质是将HQL转化为MapReduce程序。

1. Hive处理的程序存储在HDFS
2. Hive分析数据底层的实现是MapReduce
3. 执行程序运行在Yarn上。

##  Hive的优缺点

###  Hive的优点

1. 操作接口采用类 SQL 语法，提供快速开发的能力（简单、容易上手）。

2) 避免了去写 MapReduce，减少开发人员的学习成本。
3) Hive 的执行延迟比较高，因此 Hive 常用于数据分析，对实时性要求不高的场合。
4) Hive 优势在于处理大数据，对于处理小数据没有优势，因为 Hive 的执行延迟比较高。
5) Hive 支持用户自定义函数，用户可以根据自己的需求来实现自己的函数。



### Hive的缺点

1. HQL表达能力有限
   1. 迭代算法无法表达
   2. 数据挖掘不擅长，由于MapReduce数据处理流程的限制，效率更高的算法无法实现
2. Hive效率较低
   1. Hive自动生成的MapReduce作业，通常情况下不够智能化
   2. Hive调优比较困难，粒度较粗



## Hive架构

![image-20211019211919283](https://raw.githubusercontent.com/Jasong321/PicBed/master/202110241734482.png)

1、用户接口：Client

即客户端，支持命令行、JDBC/ODBC，WEBUI

2、元数据：Metastore

元数据包括：表名、表所属的数据库（默认是default）、表的拥有者、列/分区字段、表的类型（是否是外部表）、表的数据所在目录等

3、HADOOP

使用HDFS存储，使用MapReduce框架计算

4、Driver

1. 解析器：将SQL字符串转换成抽象语法书AST,这一步一般都用第三方工具库完成，比如antlr；对AST进行语法分析，比如表是否存在、字段是否存在，SQL语义是否有误。
2. 编译器：将AST编译生成逻辑执行计划。
3. 优化器：对逻辑执行计划进行优化。
4. 执行器：把逻辑执行计划转换成可以运行的物理计划。对于HIVE来说，就是MR/Spark



### 数据更新

由于 Hive 是针对数据仓库应用设计的，而数据仓库的内容是读多写少的。因此，Hive中不建议对数据的改写，所有的数据都是在加载的时候确定好的。而数据库中的数据通常是需 要 经 常 进 行 修 改 的 ， 因 此 可 以 使 用 INSERT INTO … VALUES 添加数据 ， 使用 UPDATE … SET 修改数据。

### 执行

Hive 中大多数查询的执行是通过 Hadoop 提供的 MapReduce 来实现的。而数据库通常有自己的执行引擎。

### 执行延迟

Hive 在查询数据的时候，由于没有索引，需要扫描整个表，因此延迟较高。另外一个导致 Hive 执行延迟高的因素是 MapReduce 框架。由于 MapReduce 本身具有较高的延迟，因此在利用 MapReduce 执行 Hive 查询时，也会有较高的延迟。相对的，数据库的执行延迟较低。当然，这个低是有条件的，即数据规模较小，当数据规模大到超过数据库的处理能力的时候，Hive 的并行计算显然能体现出优势。



## 数据类型

### 基本数据类型

| HIVE数据类型 | JAVA数据类型 |      长度       |
| :----------: | :----------: | :-------------: |
|   TINYINT    |     byte     | 1byte有符号整数 |
|   SMALLINT   |    short     | 2byte有符号整数 |
|     INT      |     int      | 4byte有符号整数 |
|    BIGINT    |     long     | 8byte有符号整数 |
|   BOOLEAN    |   boolean    |    布尔类型     |
|    FLOAT     |    float     |  单精度浮点数   |
|    DOUBLE    |    double    |  双精度浮点数   |
|    STRING    |    string    |    字符类型     |
|  TIMESTAMP   |              |     时间戳      |
|    BINARY    |              |    字节数组     |

对于 Hive 的 String 类型相当于数据库的 varchar 类型，该类型是一个可变的字符串，不过它不能声明其中最多能存储多少个字符，理论上它可以存储 2GB 的字符数。

### 集合数据类型

|  类型  |                             描述                             |                      语法实例                       |
| :----: | :----------------------------------------------------------: | :-------------------------------------------------: |
| STRUCT | 和 c 语言中的 struct 类似，都可以通过“点”符号访问元素内容。例如，如果某个列的数据类型是 STRUCT{first STRING, last STRING},那么第 1 个元素可以通过字段.first 来引用。 | struct()<br>例如 struct<street:string, city:string> |
|  MAP   | MAP 是一组键-值对元组集合，使用数组表示法可以访问数据。例如，如果某个列的数据类型是 MAP，其中键->值对是’first’->’John’和’last’->’Doe’，那么可以通过字段名[‘last’]获取最后一个元素 |           map()<br/>例如 map<string, int>           |
| ARRAY  | 数组是一组具有相同类型和名称的变量的集合。这些变量称为数组的元素，每个数组元素都有一个编号，编号从零开始。例如，数组值为[‘John’, ‘Doe’]，那么第 2 个元素可以通过数组名[1]进行引用。 |           Array()<br/>例如 array<string>            |

Hive 有三种复杂数据类型 ARRAY、MAP 和 STRUCT。ARRAY 和 MAP 与 Java 中的Array 和 Map 类似，而 STRUCT 与 C 语言中的 Struct 类似，它封装了一个命名字段集合，复杂数据类型允许任意层次的嵌套。

### 类型转化

Hive 的原子数据类型是可以进行隐式转换的，类似于 Java 的类型转换，例如某表达式使用 INT 类型，TINYINT 会自动转换为 INT 类型，但是 Hive 不会进行反向转化，例如，某表达式使用 TINYINT 类型，INT 不会自动转换为 TINYINT 类型，它会返回错误，除非使用 CAST 操作。

#### 隐式类型转换规则

1. 任何整数类型都可以隐式地转换为一个范围更广的类型，如 TINYINT 可以转
   换成 INT，INT 可以转换成 BIGINT。
2. 所有整数类型、FLOAT 和 STRING 类型都可以隐式地转换成 DOUBLE
3. TINYINT、SMALLINT、INT 都可以转换为 FLOAT。
4. BOOLEAN 类型不可以转换为任何其它的类型。



## HIVE操作

### 创建表

建表语法：

```sql
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] table_name 
[(col_name data_type [COMMENT col_comment], ...)] 
[COMMENT table_comment] 
[PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)] 
[CLUSTERED BY (col_name, col_name, ...) 
[SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS] 
[ROW FORMAT row_format] 
[STORED AS file_format] 
[LOCATION hdfs_path]
[TBLPROPERTIES (property_name=property_value, ...)]
[AS select_statement]

```

解释说明

1、CREATE TABLE 创建一个指定名字的表。如果相同名字的表已经存在，则抛出异常；用户可以用 IF NOT EXISTS 选项来忽略这个异常。

2、EXTERNAL 关键字可以让用户创建一个外部表，在建表的同时可以指定一个指向实际数据的路径（LOCATION），在删除表的时候，内部表的元数据和数据会被一起删除，而外部表只删除元数据，不删除数据。

3、COMMENT：为表和列添加注释。

4、PARTITIONED BY 创建分区表

5、CLUSTERED BY 创建分桶表

6、SORTED BY 不常用，对桶中的一个或多个列另外排序

7、ROW FORMAT

DELIMITED [FIELDS TERMINATED BY char] [COLLECTION ITEMS 
TERMINATED BY char]

[MAP KEYS TERMINATED BY char] [LINES TERMINATED BY char] 

| SERDE serde_name [WITH SERDEPROPERTIES (property_name=property_value, 
property_name=property_value, ...)]

用户在建表的时候可以自定义 SerDe 或者使用自带的 SerDe。如果没有指定 ROW 
FORMAT 或者 ROW FORMAT DELIMITED，将会使用自带的 SerDe。在建表的时候，用户还需要为表指定列，用户在指定表的列的同时也会指定自定义的 SerDe，Hive 通过 SerDe 确定表的具体的列的数据。

SerDe 是 Serialize/Deserilize 的简称， hive 使用 Serde 进行行对象的序列与反序列化。

8、STORED AS指定存储文件类型

常用的存储文件类型：SEQUENCEFILE（二进制序列文件）、TEXTFILE（文本）、
RCFILE（列式存储格式文件）
<font color=red>如果文件数据是纯文本，可以使用 STORED AS TEXTFILE。如果数据需要压缩，使用 STORED AS SEQUENCEFILE</font>。

9、LOCATION ：指定表在 HDFS 上的存储位置。

10、AS：后跟查询语句，根据查询结果创建表。

11、LIKE允许用户复制现有的表结构，但是不复制数据。

### 分区表

分区表实际上就是对应一个 HDFS 文件系统上的独立的文件夹，该文件夹下是该分区所有的数据文件。Hive 中的分区就是分目录，把一个大的数据集根据业务需要分割成小的数据集。在查询时通过 WHERE 子句中的表达式选择查询所需要的指定的分区，这样的查询效率会提高很多。

创建分区表语法：

```sql
hive (default)> create table dept_partition(
deptno int, dname string, loc string
)
partitioned by (month string)
row format delimited fields terminated by '\t';
```

#### 分区表注意事项

创建二级分区表

```sql
create table dept_partition2( 
deptno int, 
dname string,
loc string
)
partitioned by (month string, day string) 
row format delimited fields terminated by '\t';
```

正常加载数据

```sql
load data local inpath '/opt/module/datas/dept.txt' into table
default.dept_partition2 partition(month='201709', day='13');
```

查询分区数据

```sql
select * from dept_partition2 where month='201709' and day='13';
```

把数据直接上传到分区目录上，让分区表和数据产生关联的三种方式

1、方式一：上传数据后修复

上传数据

```sql
dfs -mkdir -p
/user/hive/warehouse/dept_partition2/month=201709/day=12;

dfs -put /opt/module/datas/dept.txt 
/user/hive/warehouse/dept_partition2/month=201709/day=12;

```

查询数据，查询不到刚上传的数据

```sql
select * from dept_partition2 where month='201709' and day='12';
```

执行修复命令

```sql
msck repair table dept_partition2;
```

再次查询，出现数据

2、方式二：上传数据后添加分区

上传数据

```sql
dfs -mkdir -p
/user/hive/warehouse/dept_partition2/month=201709/day=11;

hive (default)> dfs -put /opt/module/datas/dept.txt 
/user/hive/warehouse/dept_partition2/month=201709/day=11;

```



执行添加分区

```sql
alter table dept_partition2 add partition(month='201709',
day='11');
```

3、创建文件夹后load数据到分区

创建目录

```sql
dfs -mkdir -p
/user/hive/warehouse/dept_partition2/month=201709/day=10;
```

上传数据

```sql
load data local inpath '/opt/module/datas/dept.txt' into table
dept_partition2 partition(month='201709',day='10');
```



### 数据导入

语法：

```sql
load data [local] inpath '/opt/module/datas/student.txt' [overwrite] into table student 
[partition (partcol1=val1,…)];

```

1. load data:表示加载数据
2. local:表示从本地加载数据到 hive 表；否则从 HDFS 加载数据到 hive 表
3. inpath:表示加载数据的路径
4. overwrite:表示覆盖表中已有数据，否则表示追加
5. into table:表示加载到哪张表
6. student:表示具体的表
7. partition:表示上传到指定分区

### 查询

#### 内部排序（Sort By）

Sort By：对于大规模的数据集order by的效率非常低，在很多情况下，并不需要全局排序，此时可以使用sort by。

Sort By为每个reducer产生一个排序文件。每个reducer内部进行排序，对全局结果集来说不是排序。



#### 分区排序（Distribute By）

在有些情况下，我们需要控制某个特定行应该到哪个 reducer，通常是为了进行后续的聚集操作。distribute by 子句可以做这件事。distribute by 类似 MR 中 partition（自定义分区），进行分区，结合 sort by 使用。

对于 distribute by 进行测试，一定要分配多 reduce 进行处理，否则无法看到 distribute by的效果。

```sql
set mapreduce.job.reduces=3;
insert overwrite local directory '/opt/module/datas/distribute￾result' select * from emp distribute by deptno sort by empno desc;
```

1、distribute by 的分区规则是根据分区字段的 hash 码与 reduce 的个数进行模除后，余数相同的分到一个区。

2、Hive 要求 DISTRIBUTE BY 语句要写在 SORT BY 语句之前。



#### Cluster By

当 distribute by 和 sorts by 字段相同时，可以使用 cluster by 方式。

cluster by 除了具有 distribute by 的功能外还兼具 sort by 的功能。但是排序只能是升序排序，不能指定排序规则为 ASC 或者 DESC。

以下2中方法等价：

```sql
select * from emp cluster by deptno;
select * from emp distribute by deptno sort by deptno;
```

#### 分桶及抽样查询

##### 分桶表数据存储

分区提供一个隔离数据和优化查询的便利方式。不过，并非所有的数据集都可形成合理的分区。对于一张表或者分区，Hive 可以进一步组织成桶，也就是更为细粒度的数据范围划分。

分桶是将数据集分解成更容易管理的若干部分的另一个技术。

分区针对的是数据的存储路径；分桶针对的是数据文件。

创建分桶表：

```sql
create table stu_buck(id int, name string)
clustered by(id) 
into 4 buckets
row format delimited fields terminated by '\t';
```

发现还是一个分桶，需要设置一下属性：

```sql
set hive.enforce.bucketing=true;
set mapreduce.job.reduces=-1;
```

分桶规则：

根据结果可知：Hive 的分桶采用对分桶字段的值进行哈希，然后除以桶的个数求
余的方式决定该条记录存放在哪个桶当中

##### 分桶抽样查询

对于非常大的数据集，有时用户需要使用的是一个具有代表性的查询结果而不是全部结果。Hive 可以通过对表进行抽样来满足这个需求。

```sql
select * from stu_buck tablesample(bucket 1 out of 4 on id);
```

tablesample 是抽样语句，语法：TABLESAMPLE(BUCKET x OUT OF y) 。

y 必须是 table 总 bucket 数的倍数或者因子。hive 根据 y 的大小，决定抽样的比例。例如，table 总共分了 4 份，当 y=2 时，抽取(4/2=)2 个 bucket 的数据，当 y=8 时，抽取(4/8=)1/2个 bucket 的数据。

x 表示从哪个 bucket 开始抽取，如果需要取多个分区，以后的分区号为当前分区号加上y。例如，table 总 bucket 数为 4，tablesample(bucket 1 out of 2)，表示总共抽取（4/2=）2 个bucket 的数据，抽取第 1(x)个和第 3(x+y)个 bucket 的数据。
注意：x 的值必须小于等于 y 的值



#### 行转列

CONCAT_WS((separator, str1, str2,...)，它是特殊形式的CONCAT()。第一个参数是剩余参数的分隔符。分隔符可以是与剩余参数一样的字符串。如果分隔符是 NULL，返回值也将为 NULL。这个函数会跳过分隔符参数后的任何 NULL 和空字符串。分隔符将被加到被连接的字符串之间;

COLLECT_SET(col)：函数只接受基本数据类型，它的主要作用是将某字段的值进行去重汇总，产生 array 类型字段。

例子：

```shell
vi constellation.txt
孙悟空 白羊座 A
大海 射手座 A
宋宋 白羊座 B
猪八戒 白羊座 A
凤姐 射手座 A
```

SQL：

```sql
select
	t1.base, 
	concat_ws('|', collect_set(t1.name)) name
from 
    (select 
        name, 
        concat(constellation, ",", blood_type) base 
    from
   		person_info) t1
group by 
	t1.base;
```

查询结果：

```sql
射手座,A 	大海|凤姐
白羊座,A 	孙悟空|猪八戒
白羊座,B 	宋宋
```



#### 列转行

EXPLODE(col)：将 hive 一列中复杂的 array 或者 map 结构拆分成多行。

LATERAL VIEW：LATERAL VIEW udtf(expression) tableAlias AS columnAlias

用于和 split, explode 等 UDTF 一起使用，它能够将一列数据拆成多行数据，在此
基础上可以对拆分后的数据进行聚合。

例子：

```shell
vi movie.txt
《疑犯追踪》 悬疑,动作,科幻,剧情
《Lie to me》 悬疑,警匪,动作,心理,剧情
《战狼 2》 战争,动作,灾难
```

SQL：

```sql
select 
    movie,
    category_name
from 
	movie_info lateral view explode(category) table_tmp as category_name;
```

查询结果：

```sql
《疑犯追踪》悬疑
《疑犯追踪》动作
《疑犯追踪》科幻
《疑犯追踪》剧情
《Lie to me》悬疑
《Lie to me》警匪
《Lie to me》动作
《Lie to me》心理
《Lie to me》剧情
《战狼 2》战争
《战狼 2》动作
《战狼 2》 灾难

```



## Hadoop压缩配置

### 开启Map输出阶段压缩

开启 map 输出阶段压缩可以减少 job 中 map 和 Reduce task 间数据传输量。具体配置如下：

1、开启 hive 中间传输数据压缩功能

```sql
set hive.exec.compress.intermediate=true;
```

2、开启 mapreduce 中 map 输出压缩功能

```sql
set mapreduce.map.output.compress=true;
```

3、设置 mapreduce 中 map 输出数据的压缩方式

```sql
set mapreduce.map.output.compress.codec=
org.apache.hadoop.io.compress.SnappyCodec;
```

### 开启Reduce输出阶段压缩

当 Hive 将 输 出 写 入 到 表 中 时 ， 输 出 内 容 同 样 可 以 进 行 压 缩 。 属 性
hive.exec.compress.output 控制着这个功能。用户可能需要保持默认设置文件中的默认值 false，这样默认的输出就是非压缩的纯文本文件了。用户可以通过在查询语句或执行脚本中设置这个值为 true，来开启输出结果压缩功能。

1、开启hive最终输出数据压缩功能

``` sql
set hive.exec.compress.output=true;
```

2、开启 mapreduce 最终输出数据压缩

```sql
set mapreduce.output.fileoutputformat.compress=true;
```

3、设置 mapreduce 最终数据输出压缩方式

```sql
set mapreduce.output.fileoutputformat.compress.codec =
org.apache.hadoop.io.compress.SnappyCodec;
```

4、设置 mapreduce 最终数据输出压缩为块压缩

```sql
set mapreduce.output.fileoutputformat.compress.type=BLOCK;
```



## 文件存储格式

Hive 支持的存储数据的格式主要有：TEXTFILE 、SEQUENCEFILE、ORC、PARQUET。

### 列式存储和行式存储

#### 行存储的特点

查询满足条件的一整行数据的时候，列存储则需要去每个聚集的字段找到对应的每个列的值，行存储只需要找到其中一个值，其余的值都在相邻地方，所以此时行存储查询的速度更快。

#### 列存储的特点

因为每个字段的数据聚集存储，在查询只需要少数几个字段的时候，能大大减少读取的数据量；每个字段的数据类型一定是相同的，列式存储可以针对性的设计更好的设计压缩算法。

<font color=red>TEXTFILE 和 SEQUENCEFILE 的存储格式都是基于行存储的；</font>

<font color=red>ORC 和 PARQUET 是基于列式存储的。</font>

### TextFile格式

默认格式，数据不做压缩，磁盘开销大，数据解析开销大。可结合 Gzip、Bzip2 使
用，但使用 Gzip 这种方式，hive 不会对数据进行切分，从而无法对数据进行并行操作。

### Orc格式

Orc (Optimized Row Columnar)是 Hive 0.11 版里引入的新的存储格式。

每个 Orc 文件由 1 个或多个 stripe 组成，每个 stripe 一般为HDFS 的块大小，每一个 stripe 包含多条记录，这些记录按照列进行独立存储，对应到Parquet 中的 row group 的概念。每个 Stripe 里有三部分组成，分别是 Index Data，Row 
Data，Stripe Footer：

![image-20211024200549425](https://raw.githubusercontent.com/Jasong321/PicBed/master/202110242005356.png)

1. Index Data：一个轻量级的 index，默认是每隔 1W 行做一个索引。这里做的索引应该只是记录某行的各字段在 Row Data 中的 offset。
2. Row Data：存的是具体的数据，先取部分行，然后对这些行按列进行存储。对每个列进行了编码，分成多个 Stream 来存储。
3. Stripe Footer：存的是各个 Stream 的类型，长度等信息。每个文件有一个 File Footer，这里面存的是每个 Stripe 的行数，每个 Column 的数据类
   型信息等；每个文件的尾部是一个 PostScript，这里面记录了整个文件的压缩类型以及FileFooter 的长度信息等。在读取文件时，会 seek 到文件尾部读 PostScript，从里面解析到File Footer 长度，再读 FileFooter，从里面解析到各个 Stripe 信息，再读各个 Stripe，即从后往前读。

### Parquet格式

Parquet 文件是以二进制方式存储的，所以是不可以直接读取的，文件中包括该文件的数据和元数据，<font color=red>因此 Parquet 格式文件是自解析的</font>。

1. 行组(Row Group)：每一个行组包含一定的行数，在一个 HDFS 文件中至少存储一个行组，类似于 orc 的 stripe 的概念。
2. 列块(Column Chunk)：在一个行组中每一列保存在一个列块中，行组中的所有列连续的存储在这个行组文件中。一个列块中的值都是相同类型的，不同的列块可能使用不同的算法进行压缩。
3. 页(Page)：每一个列块划分为多个页，一个页是最小的编码的单位，在同一个列块的不同页可能使用不同的编码方式。

通常情况下，在存储 Parquet 数据的时候会按照 Block 大小设置行组的大小，由于一般情况下每一个 Mapper 任务处理数据的最小单位是一个 Block，这样可以把每一个行组由一个 Mapper 任务处理，增大任务执行并行度。Parquet 文件的格式如图 6-12 所示。

![image-20211024231433214](https://raw.githubusercontent.com/Jasong321/PicBed/master/202110242314598.png)

在实际的项目开发当中，hive 表的数据存储格式一般选择：orc 或 parquet。压缩方式一般选择 snappy，lzo。



## 表的优化

### 小表、大表Join

将 key 相对分散，并且数据量小的表放在 join 的左边，这样可以有效减少内存溢出错误发生的几率；再进一步，可以使用 map join 让小的维度表（1000 条以下的记录条数）先进内存。在 map 端完成 reduce。
实际测试发现：新版的 hive 已经对小表 JOIN 大表和大表 JOIN 小表进行了优化。小表放在左边和右边已经没有明显区别。

### 大表Join大表

#### 空KEY过滤

有时 join 超时是因为某些 key 对应的数据太多，而相同 key 对应的数据都会发送到相同的 reducer 上，从而导致内存不够。此时我们应该仔细分析这些异常的 key，很多情况下，这些 key 对应的数据是异常数据，我们需要在 SQL 语句中进行过滤。例如 key 对应的字段为空，操作如下：

```sql
insert overwrite table jointable select n.* from (select * from
nullidtable where id is not null ) n left join ori o on n.id = o.id;

```

#### 空KEY转换

有时虽然某个 key 为空对应的数据很多，但是相应的数据不是异常数据，必须要包含在join 的结果中，此时我们可以表 a 中 key 为空的字段赋一个随机的值，使得数据随机均匀地分不到不同的 reducer 上。例如：

```sql
insert overwrite table jointable
select n.* from nullidtable n full join ori o on 
case when n.id is null then concat('hive', rand()) else n.id end = o.id;
```

#### MapJoin（小表Join大表）

如果不指定 MapJoin 或者不符合 MapJoin 的条件，那么 Hive 解析器会将 Join 操作转换成 Common Join，即：在 Reduce 阶段完成 join。容易发生数据倾斜。可以用 MapJoin 把小表全部加载到内存在 map 端进行 join，避免 reducer 处理。

##### 开启MapJoin参数设置

1、设置自动选择MapJoin

```sql
set hive.auto.convert.join = true; 默认为 true
```

2、大表小表的阈值设置（默认25M以下认为是小表）

```sql
set hive.mapjoin.smalltable.filesize=25000000;
```

![image-20211025231944924](https://raw.githubusercontent.com/Jasong321/PicBed/master/202110252320905.png)

### Group By

数据量小的时候无所谓，数据量大的情况下，由于 COUNT DISTINCT 的全聚合操作，即使设定了 reduce task 个数，set mapred.reduce.tasks=100；hive 也只会启动一个 reducer。，这就造成一个 Reduce 处理的数据量太大，导致整个 Job 很难完成，一般 COUNT DISTINCT使用先 GROUP BY 再 COUNT 的方式替换。

### 笛卡尔积

尽量避免笛卡尔积，join 的时候不加 on 条件，或者无效的 on 条件，Hive 只能使用 1 个reducer 来完成笛卡尔积。

### 行列过滤

列处理：在 SELECT 中，只拿需要的列，如果有，尽量使用分区过滤，少用 SELECT *。

行处理：在分区剪裁中，当使用外关联时，如果将副表的过滤条件写在 Where 后面，那么就会先全表关联，之后再过滤。

### 动态分区调整

关系型数据库中，对分区表 Insert 数据时候，数据库自动会根据分区字段的值，将数据插入到相应的分区中，Hive 中也提供了类似的机制，即动态分区(Dynamic Partition)，只不过，使用 Hive 的动态分区，需要进行相应的配置。

#### 开启动态分区参数设置

1、开启动态分区功能

```sql
hive.exec.dynamic.partition=true
```

2、设置为非严格模式（动态分区的模式，默认 strict，表示必须指定至少一个分区为静态分区，nonstrict 模式表示允许所有的分区字段都可以使用动态分区。）

```sql
hive.exec.dynamic.partition.mode=nonstrict
```

3、在所有执行 MR 的节点上，最大一共可以创建多少个动态分区。默认 1000

```sql
hive.exec.max.dynamic.partitions=1000
```

4、在每个执行 MR 的节点上，最大可以创建多少个动态分区。该参数需要根据实际的数据来设定。比如：源数据中包含了一年的数据，即 day 字段有 365 个值，那么该参数就需要设置成大于 365，如果使用默认值 100，则会报错。

```sql
hive.exec.max.dynamic.partitions.pernode=365
```

5、整个 MR Job 中，最大可以创建多少个 HDFS 文件。默认 100000

```sql
hive.exec.max.created.files=100000
```

6、当有空分区生成时，是否抛出异常。一般不需要设置。默认 false

```sql
hive.error.on.empty.partition=false
```



## 数据倾斜

 ### 合理设置Map数

1、通常情况下，作业会通过 input 的目录产生一个或者多个 map 任务。

主要的决定因素有：input 的文件总个数，input 的文件大小，集群设置的文件块大小。

2、是不是 map 数越多越好？

答案是否定的。如果一个任务有很多小文件（远远小于块大小 128m），则每个小文件也会被当做一个块，用一个 map 任务来完成，而一个 map 任务启动和初始化的时间远远大于逻辑处理的时间，就会造成很大的资源浪费。而且，同时可执行的 map 数是受限的。

3、是不是保证每个 map 处理接近 128m 的文件块，就高枕无忧了？

答案也是不一定。比如有一个 127m 的文件，正常会用一个 map 去完成，但这个文件只有一个或者两个小字段，却有几千万的记录，如果 map 处理的逻辑比较复杂，用一个map 任务去做，肯定也比较耗时。

针对上面的问题 2 和 3，我们需要采取两种方式来解决：即减少 map 数和增加 map数。



### 小文件进行合并

在 map 执行前合并小文件，减少 map 数：CombineHiveInputFormat 具有对小文件进行合并的功能（系统默认的格式）。HiveInputFormat 没有对小文件合并功能。

```sql
set hive.input.format= org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
```



### 复杂文件增加Map数

当 input 的文件都很大，任务逻辑复杂，map 执行非常慢的时候，可以考虑增加 Map数，来使得每个 map 处理的数据量减少，从而提高任务的执行效率。

增加 map 的方法为：根据
computeSliteSize(Math.max(minSize,Math.min(maxSize,blocksize)))=blocksize=128M 公式，调整 maxSize 最大值。让 maxSize 最大值低于 blocksize 就可以增加 map 的个数。

```sql
set mapreduce.input.fileinputformat.split.maxsize=100;
```

### 合理设置Reduce数

1、每个 Reduce 处理的数据量默认是 256MB

```sql
hive.exec.reducers.bytes.per.reducer=256000000
```

2、每个任务最大的 reduce 数，默认为 1009

```sql
hive.exec.reducers.max=1009
```

3、在hadoop 的 mapred-default.xml 文件中修改

```sql
set mapreduce.job.reduces = 15;
```



reduce 个数并不是越多越好。

1、过多的启动和初始化 reduce 也会消耗时间和资源；

2、另外，有多少个 reduce，就会有多少个输出文件，如果生成了很多个小文件，那么如果这些小文件作为下一个任务的输入，则也会出现小文件过多的问题；



## 并行执行

Hive 会将一个查询转化成一个或者多个阶段。这样的阶段可以是 MapReduce 阶段、抽样阶段、合并阶段、limit 阶段。或者 Hive 执行过程中可能需要的其他阶段。默认情况下，Hive 一次只会执行一个阶段。不过，某个特定的 job 可能包含众多的阶段，而这些阶段可能并非完全互相依赖的，也就是说有些阶段是可以并行执行的，这样可能使得整个 job 的执行时间缩短。不过，如果有更多的阶段可以并行执行，那么 job 可能就越快完成。

通过设置参数 hive.exec.parallel 值为 true，就可以开启并发执行。不过，在共享集群中，需要注意下，如果 job 中并行阶段增多，那么集群利用率就会增加。

```sql
set hive.exec.parallel=true;
set hive.exec.parallel.thread.number=16; 
```



## 严格模式

Hive 提供了一个严格模式，可以防止用户执行那些可能意想不到的不好的影响的查询。

通过设置属性 hive.mapred.mode 值为默认是非严格模式 nonstrict 。开启严格模式需要、修改 hive.mapred.mode 值为 strict，开启严格模式可以禁止 3 种类型的查询。

```xml
<property>
 
<name>hive.mapred.mode</name>
 
<value>strict</value>
 
<description>
 
The mode in which the Hive operations are being performed. 
 
In strict mode, some risky queries are not allowed to run. They include:
 
Cartesian Product.
 
No partition being picked up for a query.
 
Comparing bigints and strings.
 
Comparing bigints and doubles.
 
Orderby without limit.
</description>
</property>

```

1、对于分区表，除非 where 语句中含有分区字段过滤条件来限制范围，否则不允许执行。换句话说，就是用户不允许扫描所有分区。进行这个限制的原因是，通常分区表都拥有非常大的数据集，而且数据增加迅速。没有进行分区限制的查询可能会消耗令人不可接受的巨大资源来处理这个表。

2、对于使用了 order by 语句的查询，要求必须使用 limit 语句。因为 order by 为了执行排序过程会将所有的结果数据分发到同一个 Reducer 中进行处理，强制要求用户增加这个LIMIT 语句可以防止 Reducer 额外执行很长一段时间。

3、限制笛卡尔积的查询。对关系型数据库非常了解的用户可能期望在执行 JOIN 查询的时候不使用 ON 语句而是使用 where 语句，这样关系数据库的执行优化器就可以高效地将WHERE 语句转化成那个 ON 语句。不幸的是，Hive 并不会执行这种优化，因此，如果表足够大，那么这个查询就会出现不可控的情况。

## JVM重用

JVM 重用是 Hadoop 调优参数的内容，其对 Hive 的性能具有非常大的影响，特别是对于很难避免小文件的场景或 task 特别多的场景，这类场景大多数执行时间都很短。Hadoop 的默认配置通常是使用派生 JVM 来执行 map 和 Reduce 任务的。这时 JVM 的
启动过程可能会造成相当大的开销，尤其是执行的 job 包含有成百上千 task 任务的情况。
JVM重用可以使得JVM实例在同一个 job 中重新使用N次。N的值可以在Hadoop的 mapred-site.xml 文件中进行配置。通常在 10-20 之间，具体多少需要根据具体业务场景测试得出。

```xml
<property>
 
<name>mapreduce.job.jvm.numtasks</name>
 
<value>10</value>
 
<description>How many tasks to run per jvm. If set to -1, there is
no limit. 
 
</description>
</property>

```

这个功能的缺点是，开启 JVM 重用将一直占用使用到的 task 插槽，以便进行重用，直到任务完成后才能释放。如果某个“不平衡的”job 中有某几个 reduce task 执行的时间要比其他 Reduce task 消耗的时间多的多的话，那么保留的插槽就会一直空闲着却无法被其他的 job使用，直到所有的 task 都结束了才会释放。



## 推测执行

在分布式集群环境下，因为程序 Bug（包括 Hadoop 本身的 bug），负载不均衡或者资源分布不均等原因，会造成同一个作业的多个任务之间运行速度不一致，有些任务的运行速度可能明显慢于其他任务（比如一个作业的某个任务进度只有 50%，而其他所有任务已经运行完毕），则这些任务会拖慢作业的整体执行进度。为了避免这种情况发生，Hadoop 采用了推测执行（Speculative Execution）机制，它根据一定的法则推测出“拖后腿”的任务，并为这样的任务启动一个备份任务，让该任务与原始任务同时处理同一份数据，并最终选用最先成功运行完成任务的计算结果作为最终结果。

设置开启推测执行参数：Hadoop 的 mapred-site.xml 文件中进行配置，默认是 true

```xml
<property>
 
<name>mapreduce.map.speculative</name>
 
<value>true</value>
 
<description>If true, then multiple instances of some map tasks 
 
may be executed in parallel.</description>
</property>
<property>
 
<name>mapreduce.reduce.speculative</name>
 
<value>true</value>
 
<description>If true, then multiple instances of some reduce tasks 
 
may be executed in parallel.</description>
</property>
```

不过 hive 本身也提供了配置项来控制 reduce-side 的推测执行：默认是 true

```xml
<property> 
<name>hive.mapred.reduce.tasks.speculative.execution</name>
    <value>true</value>
 
<description>Whether speculative execution for reducers should be turned on. 
</description>
</property>

```

