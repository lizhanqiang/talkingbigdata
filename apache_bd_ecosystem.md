### Apache Drill

https://www.slideshare.net/MapRTechnologies/introduction-to-apache-drill-10-48312637

https://www.slideshare.net/tdunning/apache-drill-16513485

各个节点运行Drillbit，mpp架构，不遵循mr范式，可以连接各种数据源（hbase、hdfs、mongo、alluxio、local file等）

interactive analysis of large-scale datasets



支持SQL 2003



Google Dremel的开源实现





### Apache Druid

预计算 kylin

2012年MetaMarkets公司开源

查询以聚合为主，需支持离线和实时数据源，使用列式存储，自动聚合

is a open source fast distributed column-oriented data store.

designed for low latency ingestion and very fast ad-hoc aggregation based analytics

RealTime node + Historical node + Broker（路由+合并）+Coordinator

lambda architecture



data stored into segments, data is stored by timestamp,dimensions,metrics

segment stored in deep storage



Historical Node：加载全部或部分segments，相应读操作，不提供写入



![](imgs\hadoop_ecosystem\druid_arc.png)



![](imgs\hadoop_ecosystem\druid_deployment.png)







![](imgs\hadoop_ecosystem\druid_arc2.png)

### Apache Impala

no mr jobs



impalad



灵感来自Google的论文Dremel



列存

### Apache Kudu

![](imgs\hadoop_ecosystem\kudu_logo.png)

Hadoop Storage for Fast Analytics on Fast Data

a simultaneous combination of sequential and random reads and writes



HDFS的使用场景：

![](imgs\hadoop_ecosystem\kudu_goal.png)

Cloudera



Kudu跟HBase在设计上有相同之处，两者都是实时的支持key索引的记录检索和更新，Kudu跟HBase不同之处在于(Kudu的数据模型更像传统关系模型，有限列，kudu的每个列都有类型，而HBase是无模式的。Kudu在磁盘上的数据文件是纯列存格式)，多个列可以组合成primary key



NoSQL style API



kudu+impala/spark(sql)



![](imgs\hadoop_ecosystem\kudu-comps.png)

* Master Server
  元数据管理、集群管理等
* Tablet Server
  负责存储table的tablet片段，并提供数据读写服务，一个tablet由tablet leader和tablet follower组成



table由tablet组成，每个tablet包括MetaData元信息及若干个RowSet，RowSet包含一个MemRowSet及若干个DiskRowSet，DiskRowSet包含BloomFile、Ad_hoc_Index、BaseData、DeltaMem及若干RedoFile和UndoFile



### ClickHouse

update是异步的，更新结果不能立即可见，2018年之后开始支持

原来 alter table update，代价高，使用批量不常用的更新，异步更新



insert代替update

* ReplacingMergeTree Engine
  replace data by pri key，newer version replace older version
  replacement is performed during backgroup merge operation，not immediately，no guarantee it happens at all
* Aggregate functions
* AggregatingMergeTree



查询时新老数据都有，可以加上final关键字，查询时替换

### Presto

![](imgs\hadoop_ecosystem\presto_arc.png)

### TiDB

TiKV：row-base storage engine，for OLTP，sorted key-value pair（RocksDB），range of key form a Region，Region has multiple repicas，forming a Raft group



transactional processing and analytical queries are isolated



TiFlash：columnar storage engine，for OLAP



PD（Placement Driver）用来管理Region，存储region可key的对应关系和存储位置，region的平衡，PD可以包含多个节点，没有持久化状态，启动时构建



TiDB内部扩展了Raft协议实现了强一致性和资源隔离性

TiKV是Leader和Follower，TiFlash是Learner

客户端从TiFlash读数据时，强制同步，完成强一致性保障

上述是storage

计算层也有两种：SQL engine和TiSpark

![](imgs\hadoop_ecosystem\tidb_arc1.png)





![](imgs\hadoop_ecosystem\tidb_arc2.png)



![](imgs\hadoop_ecosystem\tidb_arc3.png)





### GaussDB

2019年5月15日，正式推出

GaussDB并不是一个产品，而是系列产品的统称，包含面向OLTP的数据库、面向OLAP的数据库、面向HTAP的数据库

基于PostgreSQL 9.2基础上的魔改，目前GaussDB 100单机版已开源



### Postgres-XL（PGXL）

基于Postgres-XC（PGXC），XC主要面向分布式OLTP，XL还能解决MPP部分问题



### PolarDB

阿里RDS（Relational Database Service） 阿里云关系型数据库

阿里DRDS（Distributed Relational Database Service） 阿里分布式关系型数据库服务



存储和计算分离

![](imgs\hadoop_ecosystem\polardb_arc.png)



