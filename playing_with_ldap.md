LDAP（Lightweight Directory Access Protocol） protocol used to access and modify centrally stored information over a network



实现

>OpenLDAP
>
>Microsoft Active Directory
>
>389 Directory Server
>
>IBM Security Directory Server
>
>NetIQ eDirectory



LDAP **Entry**（Object）由DN（Distinguished Name）唯一确定，entity包含**Attribute**，attribute由type、value组成

attribute如CN（Common Name）



**objectClass**特殊的attribute，ldap中所有的entry都包含objectClass属性，定义了此类entry需要包含哪些属性，objectClass本身的定义被存储在schema文件中



**Schema**：决定目录结构和内容的一组规则，包含attribute type definitions、objectClass definition、and so on，比如某个object的attribute是否是必填的

```
core.schema：定义LDAPv3基础的attribute和object
inetorgperson.schema：存储inetOrgPerson object的定义及其attribute，这个object常用于存储用户联系方式
LDIF：Ldap Data Interchange Format，plain-text file for LDAP entries，用于从ldap中导入导出时使用这种格式。ldap server之间交换数据也可以使用这种格式
```



OpenLDAP服务端的进程

>slapd：LDAP服务进程，接受客户端连接并响应请求
>
>slurpd：ldap复制进程，用于将一个slapd的改变传递到另一个slapd，只有在用到多个ldap server才会有用



DN是entry‘s fully qualified name



CN（Common Name）

DC（Domain Component）

OU（Organizational Unit Name）

O（OrganizationName）

ST（State or Province Name）

UID（UserID）

STREET（StreetAddress）

C（CountryName）





yum install openldap openldap-servers openldap-clients



systemctl enable slapd

systemctl start slapd



不建议直接修改ldap的配置，先写到一个文件中然后使用ldapadd ldapmodify



创建ldap的管理用户：

```
[root@10-255-20-154 ~]# slappasswd 
New password: 
Re-enter new password: 
{SSHA}EJ0/S4bOuqdLVvl2mPA7NtgmDziPnkf9
```





vim ldaprootpasswd.ldif

```
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}EJ0/S4bOuqdLVvl2mPA7NtgmDziPnkf9
```



```
ldapadd -Y EXTERNAL -H ldapi:/// -f ldaprootpasswd.ldif
```







```
cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
systemctl restart slapd
```



```
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```



vim ldapdomain.ldif

```
dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth"
  read by dn.base="cn=Manager,dc=talkingbigdata,dc=com" read by * none

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=talkingbigdata,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=Manager,dc=talkingbigdata,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}EJ0/S4bOuqdLVvl2mPA7NtgmDziPnkf9

dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to attrs=userPassword,shadowLastChange by
  dn="cn=Manager,dc=talkingbigdata,dc=com" write by anonymous auth by self write by * none
olcAccess: {1}to dn.base="" by * read
olcAccess: {2}to * by dn="cn=Manager,dc=talkingbigdata,dc=com" write by * read
```



```
ldapmodify -Y EXTERNAL -H ldapi:/// -f ldapdomain.ldif
```





vim baseldapdomain.ldif

```
dn: dc=talkingbigdata,dc=com
objectClass: top
objectClass: dcObject
objectclass: organization
o: talkingbigdata com
dc: talkingbigdata

dn: cn=Manager,dc=talkingbigdata,dc=com
objectClass: organizationalRole
cn: Manager
description: Directory Manager

dn: ou=People,dc=talkingbigdata,dc=com
objectClass: organizationalUnit
ou: People

dn: ou=Group,dc=talkingbigdata,dc=com
objectClass: organizationalUnit
ou: Group

```







