# 概述

> **Apache Ranger的愿景是为Hadoop生态圈提供全方位的安全保障。Apache Ranger主要负责Hadoop生态圈各组件的访问授权（Authorization），Hadoop生态圈各组件身份认证（Authentication）由其他框架负责，如Kerberos**
>
> Apache Ranger提供了一个集中式的安全框架来管理对Hadoop相关组件（如Hive、HBase、HDFS等）的细粒度访问控制。使用Apache Ranger管理控制台，管理员可以轻松配置特定用户/用户组访问资源(文件、文件夹、数据库、表、列等)的策略，并在Hadoop中强制执行这些策略。它们还可以支持审计跟踪和策略分析，从而对集群资源进行更深入的控制。Apache Ranger还提供了将某些数据的管理委托给其他人的能力，目的是分散数据所有权

# Ranger架构

Apache Ranger的核心是一个集中式的web应用程序，它由策略管理、审计和报告模块组成。安全管理员使用web工具或者REST API配置安全策略，安装在各个组件（如HDFS、Hive）中的Ranger插件定时拉取安全策略，并在适当的时候强制应用安全策略

![](imgs\ranger_arch.png)

截止到目前（2021-01-29），Apache Ranger支持Hadoop如下组件的细粒度授权和审计

1. Apache Hadoop

   >Ranger需要在Hadoop的NameNode上安装插件，插件拦截读写路径，确认当前操作是否可以授权，收集并保存审计日志。针对Hadoop，Ranger支持为特定的用户/组在特定资源集上（文件、目录等）设定特定的权限集合（读、写、可执行）。Ranger设定的访问权限独立于HDFS自身的文件权限

   ![](imgs\ranger_hdfs_workflow.png)

2. Apache Hive

   > Ranger需要在Hive的Hiveserver2上安装插件，拦截发送到Hiveserver2的语句，并确定当前操作是否可以授权。Ranager支持Hive的细粒度权限控制，可以支持到列级别

3. Apache HBase

   > Apache Ranger提供了一个coprocessor，它被添加到HBase中，包含了执行授权检查和收集审计数据的逻辑。

4. Apache Knox

   > Apache Knox目前为用户/组提供服务级别授权，这些acl存储在一个本地文件中。Apache Ranger为Knox构建了一个插件，允许通过中央UI/REST api管理这些策略，并支持对用户访问行为进行审计

5. Apache Solr

   > 通过Ranger，管理员可以设定某个用户可以读取solr的哪些collection

6. Apache Kafka

   > Apache Ranger可以管理Kafka每个topic的acl访问列表，管理员可以使用Ranger来控制谁可以往某个topic中发布消息，谁可以订阅某个topic。除了按用户和组设定策略外，Apache Ranger还支持基于IP地址的发布或订阅权限

7. Apache Yarn

   > YARN在Hadoop生态系统中作为资源监控和任务调度层被广泛使用。管理员可以使用YARN设定队列的容量，应用程序可以被授予在某个队列中执行任务的权限。使用Apache Ranger，管理员可以设定哪些人可以在哪些队列中执行任务的策略

8. Apache Storm

9. Apache Atlas

10. Elasticsearch

11. Apache Kylin

12. Apache Ozone

13. Presto

14. Apache Sqoop



访问授权控制包括三个要素：`主体`、`客体`和`控制策略`。主体是提出资源访问请求的实体；客体是指被访问的资源实体；控制策略是主体对客体的相关访问规则集合。在Ranger架构中，客体指的是Hadoop组件，Ranger在Hadoop组件中安装相关插件， 并以服务的方式将插件注册到Ranger中，主体指的是用户或者标签，Ranger提供的`usersync`插件支持同步操作系统或者ldap中的用户，`tagsync`插件支持同步atlas中的标签（tag）



此外，ranger还对hadoop kms进行了扩展，原始的hadoop kms将密钥存储在keytool生成的keystore文件中。`ranger-kms`支持将密钥存储在数据库中。

# 编译构建ranger

1. ranger没有提供预编译的软件包，需要自己进行编译。首先下载ranger-release-ranger-2.1.0.zip并解压

   ```
   1) wget https://codeload.github.com/apache/ranger/zip/release-ranger-2.1.0
   
   2) unzip  ranger-release-ranger-2.1.0.zip
   ```

2. ranger是使用java语言开发的，采用maven作为构建工具，首先配置maven环境

   ```
   1) 下载maven二进制软件包并解压
      wget https://mirror.bit.edu.cn/apache/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz
      tar xzvf apache-maven-3.6.3-bin.tar.gz
   
   2) 配置环境变量，在`/etc/profile`文件中新增如下配置
      export MAVEN_HOME=/software/apache-maven-3.6.3
      export PATH=$PATH:$MAVEN_HOME/bin
   
   3) 使环境变量生效
      source /etc/profile
   
   4) 根据网络环境，可能需要将maven的中央仓库替换为阿里云的镜像仓库，具体操作不赘述
   ```

3. ranger构建时，需要构建环境提供一些软件包，采用如下命令安装

   ```
   yum install bzip2 gcc fontconfig
   ```

4. ranger构建时依赖于python，建议安装python的科学计算发行版Anaconda，采用如下命令下载安装Anaconda

   ```
   1) wget https://repo.anaconda.com/archive/Anaconda3-2020.11-Linux-x86_64.sh
   
   2) sh Anaconda3-2020.11-Linux-x86_64.sh
   ```

5. 完成上述准备，就可以开始编译ranger了，进入`ranger-release-ranger-2.1.0`目录，执行如下构建命令

   ```
   mvn clean compile package install -DskipTests -DskipJSTests
   ```

6. 不出意外的话，构建过程会失败，报错信息为软件仓库中找不到`calcite-linq4j-1.16.0-kylin-r2`软件包

   ![](imgs\ranger_build_failed.png)

   

   经过确认软件仓库中确实没有这个包，而且这个问题2020-09-12号就被发现了，但是社区一直没解决这个问题，如下：
   ![](imgs\ranger_kylin_error.png)

   上述错误是构建ranger的kylin插件时发生的，为了绕过这个问题，我们可以修改pom.xml文件，注释掉kylin插件的构建部分即可
   ![](imgs\disable_kylin.png)

7. 重新执行第5步的构建命令，稍等片刻即可构建成功

   ![](imgs\ranger_builder_sucess.png)

   

8. 进入`ranger-release-ranger-2.1.0/target`目录，即可看到构建成功的ranger插件列表，如下

   ![](imgs\ranger_tar_gz.png)



# 安装配置ranger-admin

1. ranger-admin提供了一套web界面，安全管理员可通过此界面添加服务插件、配置安全策略、查询审计日志、执行用户同步、新建kms加密密钥等。由于ranger的安全策略和`ranger-kms`的加密密钥需要存储到数据库中，下面我们就来安装配置mariadb

   ```
   1）安装mariadb
      yum install mariadb-server
   2）启动mariadb
      systemctl restart mariadb
   3）配置mariadb，此步骤需要用户多次做出选择，请仔细根据提示操作
      mysql_secure_installation
   ```

2. ranger的审计日志支持存储到solr或者elasticsearch中，下面我们来配置elasticsearch

   ```
   1) 下载elasticsearch
      wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.10.2-linux-x86_64.tar.gz
   2) 解压elasticsearchtar
      zxvf elasticsearch-7.10.2-linux-x86_64.tar.gz
   3) 进入`config`目录，修改`elasticsearch.yml`配置文件
      cluster.name: ranger_audit
      node.name: node-1
      path.data: /data/elasticsearch
      path.logs: /log/elasticsearch
      network.host: 192.168.56.102
      http.port: 9200
      cluster.initial_master_nodes: ["node-1"]
   4) 运行elasticsearch-7.10.2最低需要jdk11，具体操作不再赘述
   5) 新建elasticsearch用户，并使用该用户运行elasticsearch
      bin/elasticsearch -d
   6) 验证elasticsearch是否安装成功
      curl http://192.168.56.102:9200/
   ```

3. 解压ranger-admin

   ```
   tar zxvf ranger-2.1.0-admin.tar.gz
   ```

4. 修改`ranger-2.1.0-admin/install.properties`配置文件

   ```
   #使用mysql数据库
   
   DB_FLAVOR=MYSQL
   
   #mysql jdbc驱动包的路径
   
   SQL_CONNECTOR_JAR=/usr/share/java/mysql-connector-java-5.1.47.jar
   
   #mysql的root用户名
   
   db_root_user=root
   
   #mysql的root密码
   
   db_root_password=root
   
   #mysql地址和端口号
   
   db_host=localhost:3306
   
   #admin用户的密码
   
   rangerAdmin_password=KeyAdmin123!
   #rangertagsync用户的密码
   
   rangerTagsync_password=KeyAdmin123!
   
   #rangerusersync用户的密码
   
   rangerUsersync_password=KeyAdmin123!
   
   #keyadmin用户的密码
   
   keyadmin_password=KeyAdmin123!
   
   #审计日志配置
   audit_store=elasticsearch
   
   audit_elasticsearch_urls=192.168.56.102
   
   audit_elasticsearch_port=9200
   
   policymgr_external_url=http://hadoop02.bigdata.com:6080
   
   hadoop_conf=/software/hadoop-3.2.2/etc/hadoop/
   ```

5. 进入`ranger-2.1.0-admin`目录，执行如下命令安装ranger-admin

   ```
   ./setup.sh
   ```

6. 启动ranger-admin

   ```
   ranger-admin start
   ```

7. 以`admin/KeyAdmin123!`的身份登录以验证是否安装成功

   ![](imgs\ranger_homepage.png)

# 安装配置ranger-usersync

使用ranger配置安全策略之前，首先需要同步操作系统或者ldap中的用户到ranger-admin中。ranger提供的用户同步插件`usersync`就是干这件事儿的。该插件的安装配置过程如下

1. 解压`ranger-2.1.0-usersync.tar.gz`

   ```
   tar zxvf `ranger-2.1.0-usersync.tar.gz`
   ```

2. 进入`ranger-2.1.0-usersync`目录，编辑`install.properties`配置文件

   ```
   POLICY_MGR_URL =http://hadoop02.bigdata.com:6080
   
   SYNC_SOURCE = unix
   
   MIN_UNIX_USER_ID_TO_SYNC = 0
   
   MIN_UNIX_GROUP_ID_TO_SYNC = 0
   
   SYNC_INTERVAL =1
   
   rangerUsersync_password=KeyAdmin123!
   
   hadoop_conf=/software/hadoop-3.2.2/etc/hadoop
   ```

3. 使用如下命令启动/停止`usersync`用户同步服务

   ```
   ranger-usersync start/stop
   ```

4. 使用`rangerusersync/KeyAdmin123!`用户登录ranger-admin，验证同步效果
   ![](imgs\ranger-usersync.png)

# 安装配置ranger-hdfs-plugin

1. 解压`ranger-2.1.0-hdfs-plugin.tar.gz`

   ```
   tar zxvf ranger-2.1.0-hdfs-plugin.tar.gz
   ```

2. 进入`ranger-2.1.0-hdfs-plugin`目录，编辑`install.properties`配置文件

   ```
   POLICY_MGR_URL=http://hadoop02.bigdata.com:6080
   
   #后续在ranger-admin添加hdfs服务时，需要跟这里命名一致
   
   REPOSITORY_NAME=hadoop_hdfs
   
   COMPONENT_INSTALL_DIR_NAME=/software/hadoop-3.2.2
   
   CUSTOM_USER=hdfs
   
   CUSTOM_GROUP=hadoop
   ```

3. 进入`ranger-2.1.0-hdfs-plugin`目录，启动`ranger-hdfs`插件

   ```
   ./enable-hdfs-plugin.sh
   ```

4. 重启hdfs

   ```
   start-dfs.sh
   
   stop-dfs.sh
   ```

5. 在`ranger-admin`界面添加hdfs服务
   ![](imgs\ranger-add-hdfs.png)

6. 在`ranger-admin`点击`hadoop-hdfs`服务名，进入hdfs权限配置界面
   ![](imgs\ranger-hdfs-manager.png)

7. ranger可以对不同的用户对不同的文件路径设置不同的访问权限，如下
   ![](imgs\ranger-hdfs-config.png)

# 安装配置ranger-kms

1. 解压`ranger-2.1.0-kms.tar.gz`

   ```
   tar zxvf ranger-2.1.0-kms.tar.gz
   ```

2. 进入`ranger-2.1.0-kms`目录，编辑`install.properties`配置文件

   ```
   DB_FLAVOR=MYSQL
   
   SQL_CONNECTOR_JAR=/usr/share/java/mysql-connector-java-5.1.47.jar
   
   #存储加密密钥的数据库连接信息
   
   db_root_user=root
   db_root_password=root
   db_host=localhost:3306
   
   db_name=rangerkms
   db_user=root
   db_password=root
   
   hadoop_conf=/software/hadoop-3.2.2/etc/hadoop
   
   POLICY_MGR_URL=http://hadoop02.bigdata.com:6080
   
   REPOSITORY_NAME=ranger_kms
   ```

3. 进入`ranger-2.1.0-kms`目录，执行安装

   ```
   ./setup.sh
   ```

4. 修改`hdfs-site.xml`文件，新增如下配置

   ```
   <property>
       <name>dfs.encryption.key.provider.uri</name>
       <value>kms://http@hadoop02.bigdata.com:9292/kms</value>
   </property>
   ```

5. 修改`core-site.xml`文件，新增如下配置

   ```
   <property>
       <name>hadoop.security.key.provider.path</name>
       <value>kms://http@hadoop02.bigdata.com:9292/kms</value>
   </property>
   ```

6. 将hdfs-site.xml和core-site.xml文件分发到其他hadoop节点

7. 使用如下命令启动/停止`ranger-kms`

   ```
   ranger-kms start/stop
   ```

8. 使用`keyadmin/KeyAdmin123!`登录`ranger-admin`添加ranger_kms服务

   ![](imgs\ranger-kms-config.png)

9. 通过`Encryption->Key Manager`菜单新增加密密钥。也可以通过`hadoop key create <key_name>`命令创建
   ![](imgs\ranger-kms-addkey.png)

10. 验证`ranger-kms`加密效果，请参考另一篇文章：[Hadoop KMS加密原理+实战](https://github.com/lizhanqiang/talkingbigdata/blob/master/hadoop_kms.md )

    

