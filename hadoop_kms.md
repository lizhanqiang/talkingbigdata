# 数据库加密概述

近年来由于数据库**被拖库**导致的数据泄露事故频发，严重侵犯了用户的安全隐私，如：

> 1. 2018年8月28，华住集团旗下的所有酒店（包括汉庭酒店、美爵、禧玥、漫心、诺富特、美居、桔子、全季、星程、宜必思等）的约1.3亿的开房记录泄露，在暗网内明码标价8个比特比进行拍卖。
> 2. 2011年11月21，CSDN网站的600多万的用户名密码遭到泄露



防止数据泄露的有效手段之一就是数据入库是进行加密，这样就算数据被心怀叵测的黑客或者粗心大意的员工拖库，只要加密密钥不泄露他们就无法完成解密。数据库加密产品有多种实现方式，主要包括

> 1. **应用层加密**：写入时，应用层代码执行加密操作，并将加密后的数据存到数据库中；读取时，应用层代码从数据库读取密文，并将密文进行解密
> 2. **调用数据库的加解密函数**：有些数据库，如PostgreSQL内置了加解密函数，应用层代码直接调用即可
> 3. **透明加密**：Transparent Data Encryption（TDE）透明加密产品，一般通过拦截数据库数据文件的读写操作完成加解密，即，数据文件落盘时自动加密，数据文件被读取时自动解密。这种加密方式无须修改应用层代码，且通过技术手段可以实现**密文索引**，应用较广



本文介绍的Hadoop KMS就属于透明加密产品，它可以对写入HDFS的文件进行透明加密，客户端读取文件时进行透明解密

# Hadoop KMS原理

Hadoop KMS（Key Management Server）是基于Hadoop KeyProvider API实现的秘钥管理服务，它包括客户端和服务器两部分。KMS服务器负责加密密钥的管理，KMS客户端通过Rest API从KMS服务器得到加密密钥。常见的KMS实现有Apache Hadoop KMS、Cloudera Key Trustee KMS、IBM Watson KMS、Ranger KMS，本文我们重点介绍Apache Hadoop KMS

KMS加密时，首先需要创建加密区（Encrypted Zone），加密区必须是HDFS上的一个空目录，数据写入加密区时被自动加密，从加密区读取数据时自动解密。每个加密区都关联一个唯一的加密区密钥**EZK（Encrypted Zone Key）**，加密区中的每个文件都使用不同的数据加密密钥**DEK（Data Encryption Key）**进行加密，DEK使用EZK进行加密生成**EDEK（Encrypted Data Encryption Key）**，EDEK被存储到文件的元数据中。KMS中用到的三种Key的作用总结如下：

> - Encrypted Zone Key（EZK），加密区密钥
>   KMS支持将某个空的HDFS目录设置为加密区，每个加密区都有唯一的EZK关联
> - Data Encryption Key（DEK/EK），数据加密密钥
>   原始的、未经加密的密钥，负责HDFS文件的加解密，每个HDFS文件都有一个DEK
> - Encrypted Data Encryption Key（EDEK/EEK），经过加密处理的数据加密密钥
>   当客户端往HDFS加密区写文件时，NameNode负责向KMS申请DEK，KMS使用EZK将DEK加密成EDEK，NameNode将EDEK写入到文件的元数据中。当客户端从HDFS加密区读文件时，NameNode将文件的EDEK返回给客户端，客户端请求KMS解密EDEK，KMS使用EZK将EDEK解密成DEK，客户端使用DEK解密文件

<img src="https://github.com/lizhanqiang/talkingbigdata/blob/master/diagram/kms.png?raw=true"/>

下图以客户端读取加密文件为例，说明了整个解密流程:

> 1. 客户端向NameNode发起文件读取请求
> 2. NameNode将文件元数据中保存的EDEK返回给客户端
> 3. 客户端将EDEK提交给KMS
> 4. KMS使用EZK将EDEK解密为DEK并返回给客户端
> 5. 客户端从DataName读取数据并使用DEK进行解密操作

<img src="https://github.com/lizhanqiang/talkingbigdata/blob/master/diagram/kms_read_path.png?raw=true"/>

# Hadoop KMS安装配置

[hadoop分布式集群搭建](https://github.com/lizhanqiang/talkingbigdata/blob/master/hadoop_cluster_setup.md)完成后，我们需要配置kms。kms需要将EZK保存到密钥库里面，首先我们使用如下命令来创建密钥库（keystore）

> keytool -genkey -alias kms_keys

执行上述命令后，会在家目录中生成`.keystore`文件，这个文件就是我们的密钥库。执行命令时需要设置密钥库密码并提供必要信息



然后在hadoop03.bigdata.com节点上修改`${HADOOP_HOME}/etc/hadoop/kms-site.xml`文件，新增如下配置项。其中hadoop.kms.key.provider.uri参数用于指定密钥库的位置，hadoop.security.keystore.java-keystore-provider.password-file参数用于指定保存密钥库密码的文件的路径，dfs.encryption.key.provider.uri参数用于指定kms的服务地址

```
<property>
  <name>hadoop.kms.key.provider.uri</name>
  <value>jceks://file@${user.home}/.keystore</value>
</property>
<property>
  <name>hadoop.security.keystore.java-keystore-provider.password-file</name>
  <value>kms.passwd</value>
</property>
<property>
  <name>dfs.encryption.key.provider.uri</name>
  <value>kms://http@hadoop03.bigdata.com:9600/kms</value>
</property>
```



修改`${HADOOP_HOME}/etc/hadoop/core-site.xml`文件，增加如下配置项

```
<property>
  <name>hadoop.security.key.provider.path</name>
  <value>kms://http@hadoop03.bigdata.com:9600/kms</value>
</property>
```



最后将hadoop03.bigdata.com节点上的`core-site.xml`和`kms-site.xml`文件拷贝到hadoop01.bigdata.com和hadoop02.bigdata.com的对应目录。



启停kms：

> hadoop --daemon start/stop kms

# 实验验证

1. 创建加密密钥（EZK）并命名为test_kms

   ```
   hadoop key create test_kms
   ```

2. 查询密钥库（keystore）中的所有加密密钥，验证第1步的密钥是否创建成功

   ```
   hadoop key list
   ```

3. 在hdfs上创建空目录/test_kms

   ```
   hadoop fs -mkdir /test_kms
   ```

4. 将第3步创建的目录设置为加密区，将加密区的加密密钥设置为第1步创建的密钥test_kms

   ```
   hdfs crypto -createZone -keyName test_kms -path /test_kms
   ```

5. 列出所有加密区，验证第4步的加密区是否创建成功

   ```
   hdfs crypto -listZones
   ```

6. 新建文本文件digit.dat，使用命令`hadoop fs -put /tmp/digit.dat /test_kms/`将之上传到hdfs的/test_kms路径下，文件内容如下：

   ```
   1,one
   2,two
   3,three
   4,four
   5,five
   6,six
   7,seven
   8,eight
   9,nine
   10,ten
   ```

7. 执行完第6步，文件已经被加密存储到hdfs上了，通过如下命令找到其对应的本地操作系统文件

   ```
   hdfs fsck /test_kms/digit.dat -files -blocks -locations
   
   # 命令的部分输出如下
   /test_kms/digit.dat 70 bytes, replicated: replication=3, 1 block(s):  OK
   0. BP-350012288-192.168.56.101-1611711571588:blk_1073741871_1047 len=70 Live_repl=3  [DatanodeInfoWithStorage[192.168.56.102:9866,DS-d1bc9d8a-ec8b-42c1-b33c-8d5810257227,DISK], DatanodeInfoWithStorage[192.168.56.101:9866,DS-f7afa564-3cbd-47d2-9b14-8510dc007038,DISK], DatanodeInfoWithStorage[192.168.56.103:9866,DS-7281c7a7-11c1-4a0f-a8b2-355e40f5ef1f,DISK]]
   ```

8. 找到blk_1073741871块，并使用vim查看文件块内容，验证确实已经加密了

   ```
   H<87><8e>ý24s|Ä!÷<9a>^[<98>Í¬<8f><9f>Ða·=ðybO×ò@¯ é^Sü·×½@õ<80>4³·<9e>r"<8d><8e><90>ÅTýÁ`ü0v¿M<85>ÂçS<91><99>^D);ÓÌ
   ```

9. 通过客户端读取加密文件，验证发现可以透明的解密

   ```
   hadoop fs -text /test_kms/digit.dat
   ```

10. 实验非常成功

















