# Hive

Created by: Guo YuJie
Created time: October 23, 2023 2:38 PM
Tags:  Document

## 特点

1. Hive最大的特点是通过类SQL来分析大数据，而避免了写MapReduce程序来分析数据，这样使得分析数据更容易。
2. 数据是存储在HDFS上的，Hive本身并不提供数据的存储功能，它可以使已经存储的数据结构化。
3. Hive是将数据映射成数据库和一张张的表，库和表的元数据信息一般存在[关系型数据库](https://cloud.tencent.com/product/cdb-overview?from_column=20065&from=20065)上（比如[MySQL](https://cloud.tencent.com/product/cdb?from_column=20065&from=20065)）。
4. 数据存储方面：它能够存储很大的数据集，可以直接访问存储在Apache HDFS或其他数据存储系统（如Apache [HBase](https://cloud.tencent.com/product/hbase?from_column=20065&from=20065)）中的文件。
5. 数据处理方面：因为Hive语句最终会生成MapReduce任务去计算，所以不适用于实时计算的场景，它适用于离线分析。
6. Hive除了支持MapReduce计算引擎，还支持Spark和Tez这两种分布式计算引擎；
7. 数据的存储格式有多种，比如数据源是二进制格式，普通文本格式等等；

---


---

## 面试题

### **1. hive 内部表和外部表的区别**

未被external修饰的是内部表（managed table）

被external修饰的为外部表（external table）

区别：

- 内部表数据由Hive自身管理，**外部表数据由HDFS管理**；
- 内部表数据存储的位置是hive.metastore.warehouse.dir（默认：/user/hive/warehouse），外部表数据的存储位置由自己制定（如果没有LOCATION，Hive将在HDFS上的/user/hive/warehouse文件夹下以外部表的表名创建一个文件夹，并将属于这个表的数据存放在这里）；
- 删除内部表会直接**删除元数据（metadata）及存储数据**；删除外部表仅仅会删除元数据，HDFS上的文件并不会被删除；

### **2. hive 有索引吗**

Hive支持索引，但是Hive的索引与关系型数据库中的索引并不相同，比如，Hive不支持主键或者外键。

Hive索引可以建立在表中的某些列上，以提升一些操作的效率，例如减少MapReduce任务中需要读取的数据块的数量。

在可以预见到分区数据非常庞大的情况下，索引常常是优于分区的。

虽然Hive并不像事物数据库那样针对个别的行来执行查询、更新、删除等操作。它更多的用在多任务节点的场景下，快速地全表扫描大规模数据。但是在某些场景下，建立索引还是可以提高Hive表指定列的查询速度。（虽然效果差强人意）

- 索引适用的场景

适用于不更新的静态字段。以免总是重建索引数据。每次建立、更新数据后，都要重建索引以构建索引表。

- Hive索引的机制如下：

hive在指定列上建立索引，会产生一张索引表（Hive的一张物理表），里面的字段包括，索引列的值、该值对应的HDFS文件路径、该值在文件中的偏移量;

v0.8后引入bitmap索引处理器，这个处理器适用于排重后，值较少的列（例如，某字段的取值只可能是几个枚举值）

因为索引是用空间换时间，索引列的取值过多会导致建立bitmap索引表过大。

但是，很少遇到hive用索引的。说明还是有缺陷or不合适的地方的。

### 3.**sort by 和 order by 的区别**

order by 会对输入做全局排序，因此只有一个reducer（多个reducer无法保证全局有序）只有一个reducer，会导致当输入规模较大时，需要较长的计算时间。

sort by不是全局排序，其在数据进入reducer前完成排序.

因此，如果用sort by进行排序，并且设置mapred.reduce.tasks>1， 则sort by只保证每个reducer的输出有序，不保证全局有序。

### 4.**hive优化有哪些**

1. **数据存储及压缩**：
    1. 针对hive中表的存储格式通常有orc和parquet，压缩格式一般使用snappy。相比与textfile格式表，orc占有更少的存储。因为hive底层使用MR计算架构，数据流是hdfs到磁盘再到hdfs，而且会有很多次，所以使用orc数据格式和snappy压缩策略可以降低IO读写，还能降低网络传输量，这样在一定程度上可以节省存储，还能提升hql任务执行效率；
2. **通过调参优化**：
    1. 并行执行，调节parallel参数；调节jvm参数，重用jvm；设置map、reduce的参数；开启strict mode模式；关闭推测执行设置。
3. 有效地减小数据集将**大表拆分成子表**；结合使用外部表和分区表。
4. SQL优化大表对大表
    1. 尽量减少数据集，可以通过分区表，避免扫描全表或者全字段；大表对小表：设置自动识别小表，将小表放入内存中去执行。