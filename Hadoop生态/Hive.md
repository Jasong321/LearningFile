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

