### Apache Drill

https://www.slideshare.net/MapRTechnologies/introduction-to-apache-drill-10-48312637

https://www.slideshare.net/tdunning/apache-drill-16513485

各个节点运行Drillbit，mpp架构，不遵循mr范式，可以连接各种数据源（hbase、hdfs、mongo、alluxio、local file等）

interactive analysis of large-scale datasets



支持SQL 2003



Google Dremel的开源实现





### Apache Druid



lambda architecture



### Apache Impala

no mr jobs



impalad



灵感来自Google的论文Dremel



列存

### Apache Kudu

![](imgs\hadoop_ecosystem\kudu_logo.png)

Hadoop Storage for Fast Analytics on Fast Data

![](imgs\hadoop_ecosystem\kudu_goal.png)

Cloudera



Kudu跟HBase在设计上有相同之处，两者都是实时的支持key索引的记录检索和更新，Kudu跟HBase不同之处在于(Kudu的数据模型更像传统关系模型，kudu的每个列都有类型，而HBase是无模式的。Kudu在磁盘上的数据文件是纯列存格式)



kudu+impala/spark(sql)



![](imgs\hadoop_ecosystem\kudu-comps.png)

* Master Server
  元数据管理、集群管理等
* Tablet Server
  负责存储table的tablet片段，并提供数据读写服务，一个tablet由tablet leader和tablet follower组成



table由tablet组成，每个tablet包括MetaData元信息及若干个RowSet，RowSet包含一个MemRowSet及若干个DiskRowSet，DiskRowSet包含BloomFile、Ad_hoc_Index、BaseData、DeltaMem及若干RedoFile和UndoFile