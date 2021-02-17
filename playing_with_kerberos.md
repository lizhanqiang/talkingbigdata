## 1. Kerberos概述

> Kerberos是希腊神话中守护冥王哈迪斯的三头犬，它是麻省理工学院设计的一种计算机网络认证（Authentication）协议，基于票据（ticket）实现了网络节点在不安全的通信环境下以安全的方式互相证明其身份，认证过程基于对称加密，可防止窃听和重放攻击。Kerberos默认使用UDP的88端口
>
> `wikipedia`

![](imgs\kerberos\Cerberus and Hades.jpg)



## 2. Kerberos认证原理

Kerberos认证依赖于一个可信第三方实体，这个可信第三方在Kerberos中被称为**KDC**（**K**ey **D**istribution **C**enter）。客户端用户和服务进程都需要在KDC中进行注册，它们被统称为Principal，客户端用户在Kerberos中被称为**UPN**（**U**ser **P**rincipal **N**ame），服务进程在Kerberos中被称为**SPN**（**S**ervice **P**rincipal **N**ame）。SPN和UPN的principal和password只有它自身和KDC知道。一组UPN和SPN可以组成认证域（Realm）

>UPN的principal一般表示为`user@REALM`，如`hive@BIGDATA.COM`
>
>SPN的principal一般表示为 `serviceclass/fqdn@REALM`，如`imap/mail.bigdata.com/BIGDATA.COM`



Kerberos KDC包含两个组件`认证服务器` **AS**（**A**uthentication **S**erver）和`票据授权服务器`**TGS**（**T**icket **G**ranting **S**erver）。其中AS负责用户身份的验证，TGS负责给客户端颁发服务票据**ST**（**S**ervice **T**icket），客户端凭服务票据即可访问相应的服务



Kerberos的认证的详细过程如下图（`from wikipedia`）所示

![](imgs\kerberos\kerberos_detail.png)

1. 如上图①所示，客户端将用户标识（principal）发送给KDC的认证服务器AS

2. 如上图②所示，AS服务器确认数据库中此用户是否存在。如果存在，AS从数据库中拿到用户密码，然后将用户密码采用约定的方式进行哈希或者加密后生成用户密钥K~C~。最后AS给客户端回复两条消息

   >消息A：采用用户密钥K~C~加密的Client/TGS Session Key（K~C-TGS~）
   >
   >消息B：采用TGS的密钥K~TGS~加密的一揽子信息，包括：客户端信息、本票据的有效期、K~C-TGS~，该条消息一般被称为票据授权票据TGT（**T**icket **G**ranting **T**icket）

3. 客户端收到消息A和B之后，输入用户密码，客户端将用户密码采用约定方式进行哈希或者加密后生成用户秘钥K~C~，然后使用K~C~解密消息A得到K~C-TGS~，客户端原样保存消息B，即TGT。经过上述三个步骤，就完成了用户自身身份的验证，在kerberos中使用`kinit`命令完成上述三个步骤

4. 如上图③所示，如果客户端想要访问某个服务，客户端向TGS发送如下两条消息

   >消息C：包含消息B（TGT）和想要访问的服务的SPN
   >
   >消息D：采用K~C-TGS~密钥加密的客户端信息、时间戳

5. 如上图④所示，TGS收到消息C和D之后，首先从消息C中得到消息B（TGT），然后使用K~TGS~解密TGT得到客户端信息info1、本票据的有效期、K~C-TGS~，最后使用K~C-TGS~解密消息D得到客户端信息info2、时间戳并比较info1和info2，如果两者一致TGS向客户端返回如下两条消息

   >消息E：采用服务的密钥K~S~加密的客户端信息、本票据的有效期、Client/Server Session Key（K~C-S~）
   >
   >消息F：采用K~C_TGS~加密的K~C-S~

6. 如上图⑤所示，当客户端从TGS收到消息E和F之后，客户端给服务端发送如下两条消息

   >消息E
   >
   >消息G：采用K~C-S~加密的客户端信息、时间戳

7. 如上图⑥所示，服务端接受到消息E和F之后，首先使用K~S~解密消息E得到客户端信息info1、本票据的有效期、K~C-S~，然后使用K~C-S~解密消息G得到客户端信息info2和时间戳timestamp1，最后比较info1和info2是否一致，如果一致服务端同意给客户端提供服务并给客户端发送如下消息

   > 消息H：采用K~C-S~加密的上述时间戳timestamp1

8. 客户端采用K~C-S~解密消息H得到时间戳，并验证时间戳是否正确，如果正确客户端即可向服务端发送请求

9. 客户端和服务端正常通信



> **特别注意：**Kerberos的票据有失效时间，所以要求Kerberos客户端和服务器的时间要保持一致，最好不要超过5分钟的误差。最好配备ntp网络时间同步服务



## 3. Kerberos实战

### 3.1. 实战环境

>1. hadoop01.bigdata.com
>   部署kerberos kdc和kerberos客户端，ip为192.168.56.101
>2. hadoop02.bigdata.com
>   部署kerberos客户端，ip为192.168.56.102
>3. hadoop03.bigdata.com
>   部署kerberos客户端，ip为192.168.56.103

### 3.2. 实战内容

本次实战内容如下图所示，具体来说包含如下几个步骤：

>1. 在hadoop01.bigdata.com节点安装kerberos server和kerberos客户端
>2. 在hadoop02.bigdata.com和hadoop03.bigdata.com节点安装kerberos客户端
>3. 配置ssh以支撑kerberos认证协议
>4. 在KDC中添加用户talkingbigdata（UPN）
>5. 在KDC中添加host服务hadoop01、hadoop02、hadoop03（SPN）
>6. 在hadoop02.bigdata.com节点添加操作系统用户talkingbigdata
>7. 在hadoop03.bigdata.com节点上生成UPN为talkingbigdata用户的票据
>8. 在hadoop03.bigdata.com节点上以talkingbigdata用户的身份登录hadoop02.bigdata.com。如果不用输入密码就能登录成功的话，则证明实验成功

![](imgs\kerberos\kerberos_ex.png)

#### 3.2.1. 在hadoop01.bigdata.com节点上安装配置kerberos server和kerberos客户端

1. 执行如下命令安装kerberos server和kerberos client，其中krb5-server是kerberos server安装包、krb5-workstation是kerberos客户端安装包、pam_krb5是sshd的kerberos认证插件实现包

   ```
   yum install krb5-server krb5-workstation pam_krb5
   ```

2. 修改`/var/kerberos/krb5kdc/kdc.conf`文件（kdc配置文件），内容如下所示

   ```
   [kdcdefaults]
    kdc_ports = 88
    kdc_tcp_ports = 88
   
   [realms]
    BIGDATA.COM = { 
     master_key_type = aes256-cts
     acl_file = /var/kerberos/krb5kdc/kadm5.acl
     dict_file = /usr/share/dict/words
     admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
     supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
   ```

3. 修改`/var/kerberos/krb5kdc/kadm5.acl`，设定在哪些节点上可以调用kadmin、可以执行哪些操作

   ```
   */admin@BIGDATA.COM *
   ```

4. 修改`/etc/krb5.conf`（kerberos客户端配置文件），内容如下

   ```
   # Configuration snippets may be placed in this directory as well
   includedir /etc/krb5.conf.d/
   
   [logging]
    default = FILE:/var/log/krb5libs.log
    kdc = FILE:/var/log/krb5kdc.log
    admin_server = FILE:/var/log/kadmind.log
   
   [libdefaults]
    dns_lookup_realm = false
    ticket_lifetime = 24h 
    renew_lifetime = 7d
    forwardable = true
    rdns = false
    pkinit_anchors = /etc/pki/tls/certs/ca-bundle.crt
    default_realm = BIGDATA.COM
   
   [realms]
    BIGDATA.COM = { 
     kdc = hadoop01.bigdata.com
     admin_server = hadoop01.bigdata.com
    }
   
   [domain_realm]
    .bigdata.com = BIGDATA.COM
    bigdata.com = BIGDATA.COM
   ```

5. 使用如下命令初始化KDC数据库

   ```
   kdb5_util create -s -r BIGDATA.COM
   ```

6. 启动kerberos kdc和kadmin（管理控制台，可以添加、删除、列出票据、生成keytab文件等）

   ```
   systemctl start krb5kdc
   systemctl start kadmin
   ```

### 3.2.2. 在hadoop02和hadoop03上安装kerberos客户端
1. 分别在两个节点上执行如下命令安装kerberos客户端

   ```
   yum install krb5-workstation pam_krb5
   ```

2. 在hadoop01.bigdata.com上执行如下命令，将kerberos客户端的配置文件同步到hadoop2和hadoop3上

   ```
   scp /etc/krb5.conf hadoop02.bigdata.com:/etc/
   scp /etc/krb5.conf hadoop03.bigdata.com:/etc/
   ```

#### 3.2.3. 配置ssh以支撑kerberos认证协议
1. 分别在三个节点修改`/etc/ssh/sshd_config`文件，确定包含如下配置项

   ```
   GSSAPIAuthentication yes
   ```

2. 分别在三个节点修改`/etc/ssh/ssh_config`文件，确定包含如下配置项

   ```
   GSSAPIAuthentication yes 
   GSSAPIDelegateCredentials yes
   ```

3. 分别在三个节点执行如下命令并重启sshd服务

   ```
   authconfig --enablekrb5  --update
   systemctl restart sshd
   ```

#### 3.2.4. 在KDC中添加用户talkingbigdata（UPN）

在hadoop01.bigdata.com节点上，执行`kadmin.local`进入kerberos交互式管理控制台执行如下命令

>#添加管理员用户
>
>addprinc root/admin
>
>#添加talkingbigdata用户
>
>addprinc talkingbigdata
>
>#列出所有的principal
>
> listprincs

#### 3.2.5. 在KDC中添加host服务hadoop01、hadoop02、hadoop03（SPN）
分别在hadoop01、hadoop02、hadoop03上执行`kadmin`进入管理控制台，然后执行如下命令

```
addprinc -randkey host/hadoop01.bigdata.com
# 导出keytab文件
ktadd host/hadoop01.bigdata.com
```

#### 3.2.6. 在hadoop02节点添加操作系统用户talkingbigdata

```
useradd talkingbigdata
passwd talkingbigdata
```

#### 3.2.7. 在hadoop03节点上生成talkingbigdata票据
1. 在hadoop03节点上执行`klist`命令查看当前登录的kerberos用户票据（正常情况下应该没有任何票据）

   ```
   [root@hadoop03 ~]# klist
   klist: No credentials cache found (filename: /tmp/krb5cc_0)
   ```

2. 在hadoop03上执行`kinit talkingbigdata`命令，初始化talkingbigdata用户的票据

   ```
   [root@hadoop03 ~]# kinit talkingbigdata
   Password for talkingbigdata@BIGDATA.COM:
   ```

3. 在hadoop03节点上执行`klist`命令查看当前登录的kerberos用户票据

   ```
   [root@hadoop03 ~]# klist
   Ticket cache: FILE:/tmp/krb5cc_0
   Default principal: talkingbigdata@BIGDATA.COM
   
   Valid starting       Expires              Service principal
   02/07/2021 22:48:46  02/08/2021 22:48:46  krbtgt/BIGDATA.COM@BIGDATA.COM
   ```

#### 3.2.8. 在hadoop03上节点上以talkingbigdata的身份登录hadoop02验证效果
由于hadoop03节点上已经初始化了UPN为talkingbigdata用户的票据，正常情况下下述操作不用再次输入密码

```
[root@hadoop03 ~]# ssh talkingbigdata@hadoop02.bigdata.com
Last login: Sun Feb  7 21:37:45 2021 from hadoop03.bigdata.com
[talkingbigdata@hadoop02 ~]$ 
```

#### 3.2.9. 实验成功

> Hadoop自身提供的身份认证机制是基于用户宣称的操作系统用户，非常不安全。为了强化Hadoop的身份认证机制，大量Hadoop组件采用Kerberos身份认证机制。