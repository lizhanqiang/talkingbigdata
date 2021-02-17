# LDAP概述

>**LDAP**（Lightweight Directory Access Protocol）轻量级目录访问协议，是一个开放的、供应商中立的、行业标准的应用协议，用于通过网络访问和维护分布式目录信息服务，是X.500标准的精简版。LDAP是C/S架构，包含client和server，server负责目录条、接受并响应客户端访问请求
>
>`wikipedia`



LDAP主要用于存储个类信息，如可为组织提供“黄页”服务，例如，用户或员工的联系信息—电话号码、地址、部门等，可以看成一种进行了读优化的特殊分布式数据库。LDAP中主要的概念描述如下：

> 1. ldap由目录条目（**entry**）组成，entry具有唯一的标识**DN**（Distinguished Name）
> 2. 目录条目（**entry**）由一组属性（**attribute**）来描述
> 3. attribute包含**名字**和若干**取值**，每个entry都包含一个特殊的属性objectClass，objectClass限定了entry需要包含哪些属性，objectClass本身的定义被存储在**schema**文件中
> 4. **schema**决定了entry的结构和内容，比如它可以限定某个entry包含的属性以及属性是否是必填的



LDAP是一种协议，有多种实现，常见的实现产品有：

>**OpenLDAP**（下文主要介绍它）
>
>Microsoft Active Directory
>
>RedHat 389 Directory Server
>
>Apache Directory Server
>
>IBM Security Directory Server
>
>NetIQ eDirectory



LDAP的entry被组织成树状结构，LDAP最常见的树状结构的最上层是**DC**（**D**omain **C**omponent），DC下面可以包含**O**（**O**rganizationName），O下面可以包含**OU**（**O**rganizational **U**nit），OU下面是具体的entry实体，entry一般包含**CN**（**C**ommon **N**ame）、UID（UserID）属性





![](imgs\ldap\ldap_tree.png)



# OpenLDAP安装配置



1. OpenLDAP是Unix-like操作系统上广泛使用的LDAP实现产品，运行如下命令安装OpenLDAP

   ```
   yum -y install openldap openldap-clients openldap-servers
   ```

   OpenLDAP是C/S架构，服务端主要维护目录结构、接受客户端请求并返回响应。OpenLDAP服务端支持分布式部署，不同的server负责不同目录空间。OpenLDAP服务端主要包含如下两个进程:

   > **slapd**：LDAP服务进程，接受客户端连接并响应请求
   >
   > **slurpd**：ldap复制进程，用于将一个slapd的改变传递到另一个slapd，只有在用到多个ldap server才会有用

2. 启动OpenLDAP服务端进程并设置为开机启动

   ```
   systemctl start slapd
   systemctl enable slapd
   ```

3. 配置OpenLDAP，OpenLdap的配置信息以前存储在`/etc/openldap/slapd.d/slapd.conf`文件中，现在存储在cn=config数据库中（位于`/etc/openldap/slapd.d/cn=config`），推荐使用ldapmodify命令修改该数据库


   >OpenLDAP采用LDIF文件格式修改配置文件、新增修改entry
   >
   >**LDIF**（LDAP Data Interchange Format）LDAP数据交换格式，是一种标准的纯文本数据交换格式，用于表示LDAP目录内容和更新请求，用于从ldap中导入导出时使用这种格式。ldap server之间交换数据也可以使用这种格式

4. 后续我们需要修改ldap server的admin用户的密码，执行如下命令生成密码（bigdata）

   ```
   [root@hadoop03 ~]# slappasswd 
   New password: 
   Re-enter new password: 
   {SSHA}ZE7uz+eHdUb11zicZZE8hNhmSPU/NCkO
   ```

5. 新建ldif文件`my_config.ldif`，该文件的目的是修改`olcSuffix`和`olcRootDN`属性，文件内容如下

   ```
   dn: olcDatabase={2}hdb,cn=config
   changetype: modify
   replace: olcSuffix
   olcSuffix: dc=talkingbigdata,dc=com
   
   dn: olcDatabase={2}hdb,cn=config
   changetype: modify
   replace: olcRootDN
   olcRootDN: cn=admin,dc=talkingbigdata,dc=com
   ```


   执行如下命令使`my_config.ldif`文件的声明生效

   ```
   ldapmodify -Y EXTERNAL -H ldapi:/// -f my_config.ldif 
   ```

6. 新建ldif文件`change_root_pw.ldif`，该文件的目的是修改`olcRootPW`属性，文件内容如下

   ```
   dn: olcDatabase={2}hdb,cn=config
   changetype: modify
   add: olcRootPW
   olcRootPW: {SSHA}ZE7uz+eHdUb11zicZZE8hNhmSPU/NCkO
   ```


   执行如下命令使`change_root_pw.ldif`文件的声明生效

   ```
   ldapmodify  -Y EXTERNAL -H ldapi:/// -f change_root_pw.ldif
   ```

7. 新建ldif文件`change_access.ldif`，该文件的目的是修改`olcAccess`属性，文件内容如下

   ```
   dn: olcDatabase={1}monitor,cn=config
   changetype: modify
   replace: olcAccess
   olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" read by dn.base="cn=admin,dc=talkingbigdata,dc=com" read by * none
   ```


   执行如下命令使`change_root_pw.ldif`文件的声明生效

   ```
   ldapmodify -Y EXTERNAL -H ldapi:/// -f change_access.ldif
   ```

8. 执行如下命令验证上述所有属性是否修改成功

   ```
   ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config olcDatabase=\*
   ```

9. 验证文件格式是否正确

   ```
   slaptest -u
   ```

# OpenLDAP使用

1. 新建ldif文件`add_org.ldif`，该文件的目的是新增组织(**O**)，文件内容如下：

   ```
   dn: dc=talkingbigdata,dc=com
   objectClass: dcObject
   objectClass: organization
   dc: talkingbigdata
   o: talkingbigdata
   ```


   执行如下命令使`add_org.ldif`文件的声明生效

   ```
   ldapadd -f add_org.ldif -D cn=admin,dc=talkingbigdata,dc=com -w bigdata
   ```


   验证组织是否创建成功

   ```
   ldapsearch -x -b dc=talkingbigdata,dc=com
   ```

2. 新建ldif文件`add_ou.ldif`，该文件的目的是新增组织单元（**OU**），文件内容如下：

   ```
   dn: ou=users,dc=talkingbigdata,dc=com
   objectClass: organizationalUnit
   ou: users
   ```


   执行如下命令使`add_org.ldif`文件的声明生效

   ```
   ldapadd -f add_ou.ldif -D cn=admin,dc=talkingbigdata,dc=com -w bigdata
   ```


   执行如下命令验证组织单元是否创建成功

   ```
   ldapsearch -x -b dc=talkingbigdata,dc=com
   ```

3. 新建ldif文件`add_user.ldif`，该文件的目的是新增用户，文件内容如下：

   ```
   dn: cn=LeBron James,ou=users,dc=talkingbigdata,dc=com
   cn: LeBron
   sn: James
   objectClass: inetOrgPerson
   userPassword: bigdata
   uid: lebron
   ```


   执行如下命令加载必要的schema文件

   ```
   ldapadd -Y EXTERNAL -H ldapi:// -f /etc/openldap/schema/cosine.ldif
   ldapadd -Y EXTERNAL -H ldapi:// -f /etc/openldap/schema/nis.ldif
   ldapadd -Y EXTERNAL -H ldapi:// -f /etc/openldap/schema/inetorgperson.ldif
   ```


   执行如下命令使`add_user.ldif`文件的声明生效

   ```
   ldapadd -f add_user.ldif -x -D cn=admin,dc=talkingbigdata,dc=com -w bigdata
   ```

4. 新建ldif文件`add_grp.ldif`，该文件的目的是新增用户组，文件的内容如下：

   ```
   dn: cn=lakers,ou=users,dc=talkingbigdata,dc=com
   cn: lakers
   objectClass: groupOfNames
   member: cn=LeBron James,ou=users,dc=talkingbigdata,dc=com
   ```


   执行如下命令使`add_grp.ldif`文件的声明生效

   ```
   ldapadd -f add_grp.ldif -x -D cn=admin,dc=talkingbigdata,dc=com -w bigdata
   ```

5. (可选) 执行如下命令可删除OpenLDAP中的entry

   ```
   ldapdelete "cn=lakers,ou=users,dc=talkingbigdata,dc=com" -D cn=admin,dc=talkingbigdata,dc=com -w bigdata
   ```

# 使用OpenLDAP中的用户登录Linux

> **SSSD**（**S**ystem **S**ecurity **S**ervices **D**aemon），系统安全服务守护进程(SSSD)最初是为Linux操作系统(OS)开发的软件，它提供了一组守护进程来管理对远程目录服务的访问和身份验证机制
>
> `wikipedia`



1. 运行如下命令安装必要软件

   ```
   yum install -y authconfig authconfig-gtk openldap-clients sssd oddjob-mkhomedir.x86_64 pam_ldap
   ```

2. 执行如下命令，配置linux操作系统支持OpenLDAP用户登录

   ```
   authconfig --enableldap --enableldapauth --ldapserver=192.168.56.103 --ldapbasedn="dc=talkingbigdata,dc=com" --disableldaptls --update
   ```

3. 在文件`/etc/pam.d/system-auth`和`/etc/pam.d/sshd`追加如下内容

   ```
   session     required      pam_oddjob_mkhomedir.so
   ```

4. 重启相关服务

   ```
   systemctl restart oddjobd
   systemctl enable oddjobd
   systemctl restart sshd
   ```

5. 新建ldif文件`add_linux_ldap_user.ldif`，该文件的目的是新增Linux用户，文件内容如下

   ```
   dn: uid=ldaptest,ou=users,dc=talkingbigdata,dc=com
   uid: ldaptest
   cn: ldaptest
   objectClass: shadowAccount
   objectClass: top 
   objectClass: person
   objectClass: inetOrgPerson
   objectClass: posixAccount
   userPassword: {SSHA}ZE7uz+eHdUb11zicZZE8hNhmSPU/NCkO
   shadowLastChange: 17016
   shadowMin: 0
   shadowMax: 99999
   shadowWarning: 7
   loginShell: /bin/bash
   uidNumber: 2001
   gidNumber: 2001
   homeDirectory: /home/ldaptest
   sn: ldaptest
   mail: ldaptest@talkingbigdata.com
   ```


   执行如下命令使`add_linux_ldap_user.ldif`文件的声明生效

   ```
   ldapadd -f add_linux_ldap_user.ldif -x -D cn=admin,dc=talkingbigdata,dc=com -w bigdata
   ```

6. 新建ldif文件`add_linux_ldap_grp.ldif`，该文件目的是新增Linux用户组，文件内容如下

   ```
   dn: cn=ldaptest,ou=users,dc=talkingbigdata,dc=com
   objectClass: posixGroup
   objectClass: top 
   cn: ldaptest
   userPassword: {crypt}x
   gidNumber: 2001
   ```


   执行如下命令使`add_linux_ldap_grp.ldif`文件的声明生效

   ```
   ldapadd -f add_linux_ldap_grp.ldif -x -D cn=admin,dc=talkingbigdata,dc=com -w bigdata
   ```

7. 在任意节点测试能不能通过ssh正常登录hadoop03.bigdata.com

   ```
   ssh ldaptest@192.168.56.103
   ```

8. 发现可以从其他节点以OpenLDAP中的用户ldaptest登录hadoop03，实验成功

