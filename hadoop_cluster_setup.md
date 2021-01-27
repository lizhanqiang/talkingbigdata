## 1. 环境准备

本次使用3台Centos7.7虚拟机安装Hadoop集群，每台虚拟机分配4G内存、40G磁盘。三个节点的信息如下表格所示

| ip地址         | 主机名               | 备注            |
| :------------- | :------------------- | :-------------- |
| 192.168.56.101 | hadoop01.bigdata.com | NN、SNN、DN、NM |
| 192.168.56.102 | hadoop02.bigdata.com | RM、DN、NN      |
| 192.168.56.103 | hadoop03.bigdata.com | DN、NN、JHS     |

### 1.1. 基础配置

#### 1.1.1. 配置hosts

在三个节点`/etc/hosts`文件末尾追加如下内容

```
192.168.56.101 hadoop01.bigdata.com
192.168.56.102 hadoop02.bigdata.com
192.168.56.103 hadoop03.bigdata.com
```

#### 1.1.2. 关闭防火墙

Hadoop集群节点之间，广泛采用RPC通信机制，需要定制大量的防火墙规则。简单起见，我们直接关闭防火墙

> systemctl stop firewalld
>
> systemctl disable firewalld

#### 1.1.3. 禁用selinux

使用`vim`修改`/etc/selinux/config`文件，将`SELINUX=enforcing`改为`SELINUX=disabled`

#### 1.1.4. 配置yum源

为了方便软件包（如openJDK）的安装，我们将操作系统iso镜像配置成yum源，步骤如下：

- 首先在hadoop03.bigdata.com节点上挂载操作系统iso镜像

  ```
  mount -o loop CentOS-7-x86_64-DVD-1908.iso /mnt/cdrom
  ```

- 启动httpd，并将httpd添加到开机自启动列表

  ```
  systemctl restart httpd
  systemctl enable httpd
  ```

- 将操作系统镜像拷贝到httpd服务目录

  ```
  cp -r /mnt/cdrom/* /var/www/html/os/
  ```

- 在`/etc/yum.repos.d/`目录下新建`os.repo`文件，内容如下：

  ```
  [os_iso]
  name=os_iso
  baseurl=http://hadoop03.bigdata.com/os
  gpgcheck=0
  enabled=1
  ```

- 将os.repo文件拷贝到hadoop01.bigdata.com和hadoop03.bigdata.com

  ```
  scp os.repo hadoop01.bigdata.com:/etc/yum.repos.d/
  scp os.repo hadoop02.bigdata.com:/etc/yum.repos.d/
  ```

#### 1.1.5. 新建hdfs和yarn用户

为了后续精细化管理，我们新建hdfs和yarn用户，后续使用hdfs用户启动hdfs服务，使用yarn用户启动yarn服务。hdfs和yarn都属于hadoop组

```
groupadd hdfs
groupadd hadoop
useradd hdfs -g hadoop -G hdfs
useradd yarn -g hadoop
passwd hdfs
passwd yarn
```

#### 1.1.6. 为hdfs和yarn用户配置ssh免密登录

hdfs作为分布式存储框架，yarn作为分布式资源管理和任务调度框架，它们在启动时需要在多个集群节点上手动启动相应的守护进程。为了简化hdfs和yarn的启动流程，hadoop提供了`start-dfs.sh`和`start-yarn.sh`分别用来启动hdfs和yarn，这两个脚本内部使用了ssh远程启动守护进程。接下来我们针对hdfs和yarn用户分别配置ssh免密登录

首先在三个节点上分别执行如下命令，生成公私钥对

```
ssh-keygen -t rsa
```

然后在三个节点上分布执行如下命令，将本节点的公钥添加到目标主机的`authorized_keys`文件中

```
for host in hadoop0{1..3}.bigdata.com;do ssh-copy-id $host;done
```

### 1.2. 安装JDK

Hadoop的核心代码是使用Java编写而成，安装Hadoop首先需要安装JDK，Hadoop和Java的兼容性矩阵如下所示：

- Hadoop 3.3及以上，支持Java8和Java11
- Hadoop 3.0.x至Hadoop 3.2.x，仅支持Java8
- Hadoop 2.7.x至Hadoop 2.10.x，支持Java7和Java8



Hadoop自身使用**OpenJDK**进行构建、测试、发布。简单起见，我们干脆也使用OpenJDK。使用如下命令安装openJDK

```
yum install java-1.8.0-openjdk
yum install java-1.8.0-openjdk-devel
```

## 2. 配置Hadoop

在hadoop01.bigdata.com节点上下载Hadoop并解压

```
wget https://downloads.apache.org/hadoop/common/hadoop-3.2.2/hadoop-3.2.2.tar.gz
tar zxvf hadoop-3.2.2.tar.gz
```

### 2.1. 配置Hadoop守护进程运行环境

在`/etc/profile`文件末尾新增

```
export HADOOP_HOME=/software/hadoop-3.2.2
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```

在`etc/hadoop/hadoop-env.sh`文件中修改如下配置项

```
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.222.b03-1.el7.x86_64
```

### 2.2. 配置Hadoop守护进程运行参数

修改`etc/hadoop/core-site.xml`，添加如下参数

```
<configuration>
  <property>
      <name>fs.defaultFS</name>
      <value>hdfs://hadoop01.bigdata.com:8020</value>
  </property>
  <property>
      <name>io.file.buffer.size</name>
      <value>131072</value>
  </property>
</configuration>
```

修改`etc/hadoop/hdfs-site.xml`，添加如下参数

```
<configuration>
  <!--hdfs文件系统元数据存储目录，支持逗号分隔多个目录（每个目录存储一份元数据） -->
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>/hadoop/name</value>
  </property>
  <property>
    <name>dfs.replication</name>
    <value>3</value>
  </property>
  <property>
    <name>dfs.blocksize</name>
    <value>268435456</value>
  </property>
  <property>
    <name>dfs.namenode.handler.count</name>
    <value>100</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>/hadoop/data</value>
  </property>
  <property>
    <name>dfs.permissions.superusergroup</name>
    <value>hdfs</value>
  </property>
</configuration>
```

修改`etc/hadoop/yarn-site.xml`，添加如下参数

```
<configuration>
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>hadoop02.bigdata.com</value>
  </property>
  <property>
    <name>yarn.log-aggregation-enable</name>
    <value>true</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.nodemanager.local-dirs</name>
    <value>/hadoop/yarn-local-dir</value>
  </property>
  <property>
    <name>yarn.nodemanager.log-dirs</name>
    <value>/hadoop/yarn-logs</value>
  </property>
</configuration>
```

修改`etc/hadoop/mapred-site.xml`，添加如下参数

```
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
  <property>
    <name>yarn.app.mapreduce.am.env</name>
    <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
  </property>
  <property>
    <name>mapreduce.map.env</name>
    <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
  </property>
  <property>
    <name>mapreduce.reduce.env</name>
    <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
  </property>
</configuration>
```

修改`etc/hadoop/workers`，配置worker节点（DataNode和NodeManager）

```
hadoop01.bigdata.com
hadoop02.bigdata.com
hadoop03.bigdata.com
```

### 2.3. 新建相应目录并配置权限

```
mkdir -p /hadoop/name
mkdir -p /hadoop/data
chown -R hdfs:hadoop /hadoop/
chmod 775 /hadoop
```

## 3. 启动hadoop

调用`su - hdfs`命令，切换到hdfs用户并格式化hdfs

```
bin/hadoop namenode -format
```

使用hdfs用户启动hdfs服务

```
start-dfs.sh
```

调用`su - yarn`命令，切换到yarn用户，启动yarn和jobhistoryserver服务

```
start-yarn.sh
mapred --daemon start historyserver
```

## 4. 验证安装是否成功

使用hdfs用户，执行如下命令验证hdfs是否正常

```
hdfs dfsadmin -report
```

使用yarn用户，执行wordcount样例程序验证yarn是否正常

```
yarn jar hadoop-mapreduce-examples-3.2.2.jar wordcount  /data/test /data/output
```

验证HDFS NameNode WebUI和Yarn ResourceManager WebUI是否能正常打开

http://192.168.56.101:9870

http://192.168.56.102:8088

