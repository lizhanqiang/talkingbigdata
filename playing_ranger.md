> `Apache Ranger的愿景是为Hadoop生态圈提供全方位的安全保障`
>
> Apache Ranger提供了一个集中式的安全框架来管理对Hadoop相关组件（如Hive、HBase、HDFS等）的细粒度访问控制。使用Apache Ranger管理控制台，管理员可以轻松配置特定用户/用户组访问资源(文件、文件夹、数据库、表、列等)的策略，并在Hadoop中强制执行这些策略。它们还可以支持审计跟踪和策略分析，从而对集群资源进行更深入的控制。Apache Ranger还提供了将某些数据的管理委托给其他人的能力，目的是分散数据所有权

# Ranger架构

Apache Ranger的核心是一个集中式的web应用程序，它由策略管理、审计和报告模块组成。安全管理员使用web工具或者REST API配置安全策略，安装在各个组件（如HDFS、Hive）中的Ranger插件定时拉取安全策略，并在适当的时候应用安全策略

![](imgs\ranger_arch.png)

截止到目前（2021-01-29），Apache Ranger支持Hadoop如下组件的细粒度授权和审计

1. Apache Hadoop

   >Ranger需要在Hadoop的NameNode上安装插件，插件拦截读写路径，确实当前操作是否可以授权，收集并保存审计日志。针对Hadoop，Ranger支持为特定的用户/组在特定资源集上（文件、目录等）设定特定的权限集合（读、写、可执行）。Ranger设定的访问权限独立于HDFS自身的文件权限

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



Apache Ranger has the following goals:

- Centralized security administration to manage all security related tasks in a central UI or using REST APIs.
- Fine grained authorization to do a specific action and/or operation with Hadoop component/tool and managed through a central administration tool
- Standardize authorization method across all Hadoop components.
- Enhanced support for different authorization methods - Role based access control, attribute based access control etc.
- Centralize auditing of user access and administrative actions (security related) within all the components of Hadoop.