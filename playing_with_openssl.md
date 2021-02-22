

# 公钥加密学

**公钥加密**（public key cryptography）体系中，通信实体拥有**公钥**（public key）和**私钥**（private key），其中私钥需要绝密保存，公钥可以公开。公钥加密体系主要有两种应用场景：

1. **通信加密** ，发送方使用接受方的公钥加密的通信内容，只有接受方的私钥可以解密
   ![](imgs\ssl\public_key_enc.png)
2. **数字签名**，发布者发布信息时，首先对信息内容进行哈希摘要，然后使用自己的私钥对摘要进行签名得到签名档，最后将原始信息和签名档打包发布。信息被消费者拿到后，首先使用发布者的公钥将签名档解密得到信息的摘要，然后使用相同算法对原始信息进行摘要，如果两个摘要一致，证明信息确实是由发布者发布的且没有被篡改过。
   ![](imgs\ssl\public_key_sign.png)



# 公钥基础设施（PKI）

上节提到公钥加密的两种应用场景（通信加密和数字签名）都用到了公钥（public key），但是**如何确保公钥的真实性呢？** 这就是PKI要解决的问题！

PKI主要提供公钥（public key）和实体身份之间的绑定，PKI的具体定义如下：

> **PKI**（**P**ublic **K**ey **I**nfrastructure）公钥基础设施，是创建、管理、分发、使用、存储和撤销**数字证书**以及管理**公钥加密**所需角色、策略、硬件、软件和过程的集合。PKI主要目的是为电子商务、网上银行及机密电子邮件等一系列网络活动提供安全的电子信息传输通道。
>
> `wikipedia`



PKI主要包含如下三种角色，即：

>1. **RA**（**R**egistration **A**uthority）主要负责申请实体的身份确认、核实、审查
>
>2. **CA**（**C**ertificate **A**uthority）主要负责签署、颁发、存储数字证书
>
>3. **VA**（**V**alidation **A**uthority）主要负责数字证书实体的信息查询

![](imgs\ssl\ca_comp.png)

申请者将自己的**公钥+基本信息**发送给PKI的RA，RA对申请者的身份进行核实，核实通过后CA使用自己的私钥对申请者的**公钥+基本信息**进行签名生成数字证书。用户访问服务时，服务将其数字证书发送给用户代理（如浏览器、操作系统等），用户代理验证数字证书的合法性（用户代理中已经预置了CA的公钥，所以可以验证服务数字证书的合法性），如果证书合法，那么就代表用户代理信任此服务的公钥，随后用户代理使用服务的公钥对通信内容进行加密后传送给服务，从而建立起安全的信道。

![](imgs\ssl\ca_arch.png)



# 数字证书

数字证书使用CA的私钥进行签名，数字证书包含申请主体信息、签发机构信息、证书编号及有效期。证书编号用于快速检索数字证书，在一个CA机构中证书编号必须唯一。CA机构颁发的数字证书主要有如下几类：

>1. **Wildcard（通配符）证书**
>     跟站点URL相关联的数字证书
>2. **SAN（主体别名）证书**
>     **S**ubject **A**lternative **N**ame，主体别名证书，可以关联多个URL或者IP
>3. **Code Signing证书**
>     用于给应用程序代码签名
>4. **Root CA证书**
>     根CA证书是使用根CA的私钥对根CA的公钥自签名（self-signed）形成的



颁发数字证书的CA分为两类：**Private CA**（组织自建的，颁发的证书只在组织范围内被信任）、**Public CA**（公开的权威CA，其颁发的证书在全球范围内被信任，如Symantec、Sectigo、GoDaddy）



**证书分层结构（Certificate Hierarchy）**：Root CA证书是采用自签名形式生成的，Root CA签发数字证书时，通过设定相应的权限可以使该数字证书可以用来签发其他数字证书，从而形成层次结构

![](imgs\ssl\ca_hiera.png)

# 数字证书格式

目前最广泛采用的数字证书格式为**X.509**，X.509包含三个版本，即，X.509v1、X.509v2、X.509v3，目前最常用的版本是X.509v3，因为X.509v3**支持扩展**。X.509作为一种证书格式规范，有多种实现，包括：

1. **DER/CER**
   **D**istinguished **E**ncoding **R**ules（唯一编码规则）和**C**anonical **E**ncoding **R**ules（规范编码规则）是数字证书的二进制编码格式，扩展名分别是`.der`和`.cer`

2. **PEM**
   **P**rivacy enhanced **E**lectronic **M**ail（隐私增强电子邮件）是一种ASCII文件格式，扩展名一般为`.pem`、`.crt`、`.cer`、`.key`。这种证书格式非常普遍，一般以`-----BEGIN CERTIFICATE-----`作为开头，以`-----END CERTIFICATE-----`作为结尾

3. **PFX/P12/PKCS#12**
   **P**ublic **K**ey **C**ryptography **S**tandards 12源自微软的PFX格式，是一种二进制文件格式，Java的密钥库keystore就是采用的这种格式。可作为容器格式，打包私钥和数字证书，扩展名一般为`.pfx`、`.p12`

4. **P7B**

   **P**ublic **K**ey **C**ryptography **S**tandards 7是一种ASCII文件格式，一般以`-----BEGIN PKCS7-----`作为开头，以`-----END PKCS7-----`作为结尾，扩展名一般为`.p7b`、`.p7c`

# 证书吊销

证书吊销列表（CRL：Certificate Revocation List）是CA颁发的存储已吊销证书的结构化文件。CRL列表中的证书被吊销的原因一般是因为私钥丢失等原因需要在证书到期之前被吊销。证书吊销列表分发点（CDP：CRL Distribution Point）是CA提供的下载CRL的站点。

CRL不是实时更新，对于需要实时检验数字证书有效性的场景无能为力。好在有些CA提供了证书状态在线查询协议（OCSP：Online Certificate Status Protocol）可以实时查询证书的吊销状态

# 实战

本节的实战内容主要包括：创建Root CA证书、创建中间CA证书、形成证书链；创建终端用户证书；创建Apache网站服务证书；将Root CA证书添加到浏览器`受信任的根证书颁发机构`；通过浏览器访问网站验证加密效果。本次的实验环境如下：

>操作系统：Ubuntu 18.04.5 LTS
>
>IP地址：192.168.56.201
>
>主机名：learn01.bigdata.com



1. 配置apt使用阿里源，并使用`apt`命令安装openssl

   ```
   apt install openssl
   ```

2. 本次实验基于`OpenSSL 1.1.1`，使用如下命令查看openssl版本号

   ```
   openssl version
   ```

3. 公钥加密体系中，私钥需要绝密保存。建议对私钥文件设置密码进行保护，之后每次访问私钥时都需要输入此密码

   ```
   # 建议将私钥的密码保存到文件中，这里将私钥密码talkingbigdata保存到文件priv.pass
   root@learn01:/root/work# echo talkingbigdata > priv.pass
   
   # 私钥的密码文件建议以pbkdf2（Password-Based Key Derivation Function）方式进行加密保存
   root@learn01:/root/work# openssl enc -aes256 -pbkdf2 -salt -in priv.pass -out priv.pass.enc
   enter aes-256-cbc encryption password:[bigdata]
   Verifying - enter aes-256-cbc encryption password:[bigdata]
   
   # 使用如下命令可以解密私钥密码文件
   root@learn01:/root/work# openssl enc -aes256 -pbkdf2 -salt -d -in priv.pass.enc
   enter aes-256-cbc decryption password:
   talkingbigdata
   ```

   ![](imgs\ssl\pbkdf.png)

4. 为Root CA创建必要的目录结构和文件

   ```
   # Root CA的根目录
   mkdir -p /etc/pki/tls
   cd /etc/pki/tls
   
   # certs存储Root CA签发的证书,private存储Root CA的私钥
   mkdir certs private
   
   # serial存储Root CA上次颁发的数字证书的序号
   echo 01 > serial
   
   # index.db存储Root CA颁发的数字证书的信息
   touch index.db
   echo "unique_subject = yes" > index.db.attr
   
   cp /root/work/priv.pass.enc /etc/pki/tls
   ```

5. 配置Root CA，将`/etc/ssl/openssl.cnf`文件拷贝到`/etc/pki/tls/openssl.cnf`，搜索并修改或新增如下配置项

   ```
   [ CA_default ]
   dir = /etc/pki/tls
   database = $dir/index.db
   new_certs_dir = $dir/certs
   certificate = $dir/certs/cacert.pem
   policy = policy_match
   
   
   [ v3_intermediate_ca ]
   subjectKeyIdentifier = hash
   authorityKeyIdentifier = keyid:always,issuer
   basicConstraints = critical, CA:true, pathlen:0
   keyUsage = critical, digitalSignature, cRLSign, keyCertSign
   
   ```

6. 生成Root CA的私钥文件`cakey.pem`（公钥可以从私钥中提取），使用des3对称加密算法对私钥文件进行加密，加密密码为第3步设定的`talkingbigdata`

   ```
   root@learn01:/etc/pki/tls# openssl genrsa -des3 -passout file:priv.pass.enc -out private/cakey.pem 4096
   ```

   查看Root CA的私钥`cakey.pem`的内容

   ```
   root@learn01:/etc/pki/tls# openssl rsa -noout -text -in private/cakey.pem -passin file:priv.pass.enc
   ```

7. 生成Root CA的自签名数字证书`cacert.pem`，即使用Root CA的私钥对Root CA的`公钥+基本信息`进行签名

   ```
   root@learn01:/etc/pki/tls# openssl req -new -x509 -days 3650 -passin file:priv.pass.enc -config openssl.cnf -extensions v3_ca -key private/cakey.pem -out certs/cacert.pem
   You are about to be asked to enter information that will be incorporated
   into your certificate request.
   What you are about to enter is what is called a Distinguished Name or a DN.
   There are quite a few fields but you can leave some blank
   For some fields there will be a default value,
   If you enter '.', the field will be left blank.
   -----
   Country Name (2 letter code) [AU]:CN
   State or Province Name (full name) [Some-State]:Shandong                                        
   Locality Name (eg, city) []:Jinan      
   Organization Name (eg, company) [Internet Widgits Pty Ltd]:TalkingBigData
   Organizational Unit Name (eg, section) []:Group co.,ltd 
   Common Name (e.g. server FQDN or YOUR name) []:Offical Account
   Email Address []:admin@talkingbigdata.com
   ```

   查看Root CA的数字证书`cacert.pem`的内容

   ```
   root@learn01:/etc/pki/tls# openssl x509 -noout -text -in certs/cacert.pem
   ```

8. Root CA一般不签发面向终端服务或者客户的数字证书，因为签发过程需要访问Root CA的私钥，而私钥一旦泄露危害无穷。Root CA作为顶层证书机构向二级证书机构颁发数字证书，二级证书机构向更低层级的证书机构、终端服务、终端客户颁发数字证书，最终形成证书链（Certificate Chain）。Root CA的私钥最好离线保存到专用硬件中
   ![](imgs\ssl\cert_chain.jpg)

   下面创建中间证书机构（Intermediate CA）必要的目录结构和文件

   ```
   mkdir /etc/pki/tls/intermediate
   cd /etc/pki/tls/intermediate
   mkdir certs csr private
   touch index.db
   echo 01 > serial
   echo 01 > crlnumber
   ```

9. 配置中间证书机构Intermediate CA，将`/etc/ssl/openssl.cnf`文件拷贝到`/etc/pki/tls/intermediate`目录，修改如下配置项

   ```
   dir = /etc/pki/tls/intermediate
   database = $dir/index.db
   certificate = $dir/certs/intermediate.cacert.pem
   private_key = $dir/private/intermediate.cakey.pem
   policy = policy_anything
   ```

10. 生成中间证书机构Intermediate CA的私钥文件`intermediate.cakey.pem`

    ```
    root@learn01:/etc/pki/tls/intermediate# openssl genrsa -des3 -passout file:priv.pass.enc -out private/intermediate.cakey.pem 4096
    ```

11. 生成中间证书机构Intermediate CA的证书签名请求文件CSR（Certificate Signing Request）`intermediate.csr.pem`，输入必要的国别、身份、Common Name等信息

    ```
    root@learn01:/etc/pki/tls/intermediate# openssl req -new -sha256 -config openssl.cnf -passin file:priv.pass.enc -key private/intermediate.cakey.pem  -out csr/intermediate.csr.pem
    You are about to be asked to enter information that will be incorporated
    into your certificate request.
    What you are about to enter is what is called a Distinguished Name or a DN.
    There are quite a few fields but you can leave some blank
    For some fields there will be a default value,
    If you enter '.', the field will be left blank.
    -----
    Country Name (2 letter code) [AU]:CN
    State or Province Name (full name) [Some-State]:Shandong
    Locality Name (eg, city) []:Jinan
    Organization Name (eg, company) [Internet Widgits Pty Ltd]:TalkingBigData
    Organizational Unit Name (eg, section) []:WebGUI Unit
    Common Name (e.g. server FQDN or YOUR name) []:TalkingBigData WebGUI Unit
    Email Address []:webgui@talkingbigdata.com
    
    Please enter the following 'extra' attributes
    to be sent with your certificate request
    A challenge password []:
    An optional company name []:
    ```

12. 使用Root CA的私钥为中间证书机构Intermediate CA签署数字证书`intermediate.cacert.pem`

    ```
    root@learn01:/etc/pki/tls# openssl ca -config openssl.cnf -extensions v3_intermediate_ca -days 2600 -notext -batch -passin file:priv.pass.enc -in intermediate/csr/intermediate.csr.pem  -out intermediate/certs/intermediate.cacert.pem
    ```

13. 生成Root CA和Intermediate CA的数字证书链`ca-chain-bundle.cert.pem`

    ```
    root@learn01:/etc/pki/tls# cat intermediate/certs/intermediate.cacert.pem certs/cacert.pem > intermediate/certs/ca-chain-bundle.cert.pem
    ```

14. 验证Root CA和Intermediate CA的数字证书链

    ```
    root@learn01:/etc/pki/tls# openssl verify -CAfile certs/cacert.pem intermediate/certs/ca-chain-bundle.cert.pem 
    ```

    openssl支持使用如下命令查看任意网站的证书链，此处以`百度`为例

    ```
    root@learn01:/etc/pki/tls# openssl s_client -quiet -connect baidu.com:443
    depth=2 C = US, O = DigiCert Inc, OU = www.digicert.com, CN = DigiCert Global Root CA
    verify return:1
    depth=1 C = US, O = DigiCert Inc, CN = DigiCert SHA2 Secure Server CA
    verify return:1
    depth=0 C = CN, ST = Beijing, O = "BeiJing Baidu Netcom Science Technology Co., Ltd", OU = service operation department, CN = www.baidu.cn
    verify return:1
    ```

15. 生成client的私钥文件`client.key.pem`

    ```
    root@learn01:/root/work# openssl genrsa -out client.key.pem 4096
    ```

16. 创建client的证书签名请求文件`client.csr`

    ```
    root@learn01:/root/work# openssl req -new -key client.key.pem -out client.csr
    You are about to be asked to enter information that will be incorporated
    into your certificate request.
    What you are about to enter is what is called a Distinguished Name or a DN.
    There are quite a few fields but you can leave some blank
    For some fields there will be a default value,
    If you enter '.', the field will be left blank.
    -----
    Country Name (2 letter code) [AU]:CN
    State or Province Name (full name) [Some-State]:Shandong
    Locality Name (eg, city) []:Jinan
    Organization Name (eg, company) [Internet Widgits Pty Ltd]:TalkingBigData
    Organizational Unit Name (eg, section) []:Market
    Common Name (e.g. server FQDN or YOUR name) []:employee001
    Email Address []:
    
    Please enter the following 'extra' attributes
    to be sent with your certificate request
    A challenge password []:
    An optional company name []:
    
    ```

17. x509v3支持扩展属性，为client添加如下扩展属性，并保存到`client_cert_ext.cnf`文件中

    ```
    basicConstraints = CA:FALSE
    nsCertType = client, email
    nsComment = "OpenSSL Generated Client Certificate"
    subjectKeyIdentifier = hash
    authorityKeyIdentifier = keyid,issuer
    keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
    extendedKeyUsage = clientAuth, emailProtection
    ```

    上面各个扩展属性的含义可以使用如下命令查看

    ```
    man x509v3_config
    ```

18. 使用中间证书机构Intermediate CA的私钥和证书为client签发数字证书`client.cert.pem`

    ```
    root@learn01:/root/work# openssl x509 -req -in client.csr -passin file:priv.pass.enc -CA /etc/pki/tls/intermediate/certs/ca-chain-bundle.cert.pem -CAkey /etc/pki/tls/intermediate//private/intermediate.cakey.pem -out client.cert.pem -CAcreateserial -days 365 -sha256 -extfile client_cert_ext.cnf 
    ```

19. 生成网站服务的私钥文件`server.key.pem`，后续apache web容器会使用此私钥

    ```
    root@learn01:/root/work# openssl genrsa -out server.key.pem 4096
    ```

20. 创建网站服务的证书签名请求文件`server.csr`

    ```
    root@learn01:/root/work# openssl req -new -key server.key.pem -out server.csr
    You are about to be asked to enter information that will be incorporated
    into your certificate request.
    What you are about to enter is what is called a Distinguished Name or a DN.
    There are quite a few fields but you can leave some blank
    For some fields there will be a default value,
    If you enter '.', the field will be left blank.
    -----
    Country Name (2 letter code) [AU]:CN
    State or Province Name (full name) [Some-State]:Shandong
    Locality Name (eg, city) []:Jinan
    Organization Name (eg, company) [Internet Widgits Pty Ltd]:TalkingBigData
    Organizational Unit Name (eg, section) []:Software Department
    Common Name (e.g. server FQDN or YOUR name) []:learn01.bigdata.com
    Email Address []:soft@talkingbigdata.com 
    
    Please enter the following 'extra' attributes
    to be sent with your certificate request
    A challenge password []:
    An optional company name []:
    ```

21. 配置网站服务数字证书的x509扩展属性，普通证书只能关联一个域名或者IP，通过`subjectAltName`属性可以创建**SAN**（**S**ubject **A**lternative **N**ame）证书，SAN证书可以关联多个IP或者域名。新建文件`server_cert_ext.cnf`并在其中填写如下扩展属性

    ```
    basicConstraints = CA:FALSE
    nsCertType = server
    nsComment = "OpenSSL Generated Server Certificate"
    subjectKeyIdentifier = hash
    authorityKeyIdentifier = keyid,issuer:always
    keyUsage = critical, digitalSignature, keyEncipherment
    extendedKeyUsage = serverAuth
    subjectAltName=@alt_names
    
    [alt_names]
    DNS.1=learn01.bigdata.com
    IP.1=192.168.56.201
    ```

22. 使用中间证书机构Intermediate CA的私钥和证书为网站服务签署数字证书`server.cert.pem`，后续apache web服务会使用此证书

    ```
    root@learn01:/root/work# openssl x509 -req -in server.csr -passin file:priv.pass.enc -CA /etc/pki/tls/intermediate/certs/ca-chain-bundle.cert.pem -CAkey /etc/pki/tls/intermediate/private/intermediate.cakey.pem -out server.cert.pem -CAcreateserial -days 365 -sha256 -extfile server_cert_ext.cnf 
    ```
    
23. 安装并配置apache服务器

    ```
    # 安装apache服务器
    apt install apache2
    
    # 创建存放网站私钥和证书的目录
    mkdir /etc/apache2/ssl
    chmod -R 600 /etc/apache2/ssl/
    
    # 将私钥和证书拷贝到上述目录
    cp /root/work/server.key.pem  /etc/apache2/ssl/
    cp /root/work/server.cert.pem /etc/apache2/ssl/
    cp /etc/pki/tls/intermediate/certs/ca-chain-bundle.cert.pem  /etc/apache2/ssl/
    
    # 启用apache的ssl模块
    a2enmod ssl
    
    # 编辑如下配置文件
    vim /etc/apache2/site-available/default-ssl.conf
    # 新增如下配置项
    SSLEngine on
    SSLCertificateFile  /etc/apache2/ssl/server.cert.pem
    SSLCertificateKeyFile /etc/apache2/ssl/server.key.pem
    SSLCertificateChainFile /etc/apache2/ssl/ca-chain-bundle.cert.pem
    
    # 启用站点配置
    a2ensite default-ssl.conf
    
    # 重启apache服务器
    systemctl restart apache2
    ```

24. 安装curl命令，并使用curl命令访问apache服务验证证书是否生效

    ```
    # 安装curl软件包
    apt install curl
    ```

    ```
    curl https://learn01.bigdata.com:443 -v
    
    curl: (60) SSL certificate problem: self signed certificate in certificate chain
    More details here: https://curl.haxx.se/docs/sslcerts.html
    
    curl failed to verify the legitimacy of the server and therefore could not
    establish a secure connection to it. To learn more about this situation and
    how to fix it, please visit the web page mentioned above.
    ```

    使用如下命令指定客户端私钥、客户端证书、证书链，可以正常访问apache服务

    ```
    curl --key client.key.pem --cert client.cert.pem --cacert /etc/pki/tls/intermediate/certs/ca-chain-bundle.cert.pem https://learn01.bigdata.com:443 -v
    ```

25. 使用Chrome浏览器访问网站，提示网站不安全
    ![](imgs\ssl\site_unsecure.png)

26. 将Root CA的证书导入到浏览器的`受信任的根证书颁发机构`，再次访问网站，提示`连接是安全的`

    ![](imgs\ssl\site_unsecure2.png)

    