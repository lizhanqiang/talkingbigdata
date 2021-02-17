

root用户改密码

sudo passwd root



网络

![](imgs\ubuntu\netplan.png)

```
systemctl stop NetworkManager
systemctl disable NetworkManager
systemctl mask NetworkManager

systemctl stop networking
systemctl disable networking
systemctl mask networking
```

```
systemctl status systemd-networkd
systemctl enable systemd-networkd
systemctl restart systemd-networkd
```

rm -rf  /etc/netplan/*

vim /etc/netplan/systemd-networkd.yaml

```
network:
    version: 2
    renderer: networkd
    ethernets:
        ens33:
            dhcp4: no
            dhcp6: no
            addresses: [192.168.56.201/24,]
            gateway4: 192.168.56.2
            nameservers:
                addresses: [114.114.114.114,223.5.5.5]
```

![](imgs\ubuntu\netplan-config.png)



netplan apply

apt install net-tools



ssh允许root登录

vim /etc/ssh/sshd_config

```
PermitRootLogin yes
```





配置阿里云源

```
cp sources.list sources.list.bak
```

vim /etc/apt/sources.list

```
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
```

apt update

apt upgrade





防火墙

**UFW（Uncomplicated Firewall）**

```
ufw enable
ufw status
ufw disable
```

