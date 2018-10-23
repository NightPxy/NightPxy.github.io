---
layout: post
title:  "hadoop-clondrera-集群安装"
date:   2018-07-01 13:31:01 +0800
categories: hadoop
tag: hadoop
---

* content
{:toc}


## Parquet 

目标版本 CDH-5.14.2



## 环境准备

###  节点规划 

|ip|主机名|用途|
|---|---|---|
|192.168.198.181|hadoopcm001|中心机|
|192.168.198.190|hadoopcm002|节点机|
|192.168.198.191|hadoopcm003|节点机|

### 操作系统  CentorOS-7.5-Mini

### 外网&主机名  

**网卡设置**  

```shell
cd /etc/sysconfig/network-scripts/
vi ifcfg-ens33

TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
#UUID=373fb627-0e87-4fda-9731-3eb3a2a1456b
DEVICE=ens33
ONBOOT=yes
IPADDR=192.168.198.181
NETMASK=255.255.255.0
HWADDR=00:0c:29:79:60:23
DNS1=192.168.198.2
GATEWAY=192.168.198.2
```

**主机名**  

```shell
hostnamectl set-hostname "hadoopcm001"

vi /etc/sysconfig/network
NETWORKING=yes
NETWORKING_IPV6=yes
HOSTNAME=hadoopcm001
```


### 关闭 防火墙&SELINUX

```shell
systemctl stop firewalld
systemctl disable firewalld

# 验证
systemctl status firewalld 

vi /etc/selinux/config 

SELINUX=disabled
```

### 配置Hosts

```shell
vi /etc/hosts

192.168.198.181 hadoopcm001
192.168.198.190 hadoopcm002
192.168.198.191 hadoopcm003
```

### 常用工具

```shell
yum install epel-release -y 
yum install yum-axelget -y
yum update  
reboot
yum install -y lrzsz net-tools unzip zip iptables-services telnet  git vim ntpdate wget
yum install -y openssl openssl-devel svn ncurses-devel zlib-devel libtool
yum install -y snappy snappy-devel bzip2 bzip2-devel lzo lzo-devel lzop autoconf automake
yum install -y gcc gcc-c++ make cmake
```

### jdk

```shell
# jdk标准路径
mkdir /usr/java/
# 解压
tar -xzvf /usr/setup/jdk-8u162-linux-x64.tar.gz -C /usr/java/

# 加入环境变量
vi /etc/profile

export JAVA_HOME=/usr/java/jdk1.8.0_162
export PATH=.:$JAVA_HOME/bin:$PATH

source /etc/profile
```

### Python  

```shell
# 检查安装Python版本 
python --version

# 建议是centOS-7原生的 2.7.5
```

### 时钟同步

```shell
yum install ntp

# 主节点配置 Begin
vi /etc/ntp.conf 
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst
server 0.cn.pool.ntp.org
server 2.asia.pool.ntp.org
server 1.asia.pool.ntp.org
# 开放ip段的ntp访问
restrict 192.168.198.0 mask 255.255.255.0 nomodify notrap
SYNC_HWCLOCK=yes


chkconfig ntpd on

service ntpd start
# 主节点配置 End

# 所有节点配置 Begin
crontab -e
*/30 * * * * /usr/sbin/ntpdate 192.168.198.181
# 所有节点配置 End

# 检查时区
timedatectl | grep "Time zone"
# 检查NTP
/usr/sbin/ntpdate 192.168.198.181

```

### 关闭大页面

```shell
echo never > /sys/kernel/mm/transparent_hugepage/defrag
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo 'echo never > /sys/kernel/mm/transparent_hugepage/defrag'>>  /etc/rc.local

echo 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'>>  /etc/rc.local
```

### 设置Swap

```shell
echo 'vm.swappiness = 10' >> /etc/sysctl.conf

# 检查
sysctl -p
```

### http

```shell
yum install -y httpd


systemctl start  httpd 
systemctl enable  httpd

#检查
systemctl status  httpd
systemctl is-enabled httpd
```

## MySQL

主节点安装  mysql-5.6.23-linux-glibc2.5-x86_64.tar.gz

## CDH

### CDH-Parcels

http://archive.cloudera.com/cdh5/parcels/5.14.2/CDH-5.14.2-1.cdh5.14.2.p0.3-el7.parcel
http://archive.cloudera.com/cdh5/parcels/5.14.2/CDH-5.14.2-1.cdh5.14.2.p0.3-el7.parcel.sha1
http://archive.cloudera.com/cdh5/parcels/5.14.2/manifest.json

```shell
cd /var/www/html
mkdir parcels
cd parcels

# 上传CDH-parcels三个文件
# 注意上传后需要将sha1更名为sha,否则安装时会以为未下载完全还安装失败

# 验证文件未损坏
sha1sum CDH-5.14.2-1.cdh5.14.2.p0.3-el7.parcel
cat CDH-5.14.2-1.cdh5.14.2.p0.3-el7.parcel.sha
```

### CDH-Manager

http://archive.cloudera.com/cm5/repo-as-tarball/5.14.2/cm5.14.2-centos7.tar.gz

```shell
tar -xvvf ./cm5.14.2-centos7.tar.gz -C /var/www/html

# 必须要解压到  /var/www/html
# 与 parcels 保持目录平级
# /var/www/html/cm
# /var/www/html/parcels

# 布置成于官网下载的结构
mkdir -p /var/www/html/cm5/redhat/6/x86_64/
mv /var/www/html/cm /var/www/html/cm5/redhat/6/x86_64/


# 配置本地源(每个节点)
vi /etc/yum.repos.d/cloudera-manager.repo

[cloudera-manager]
name = Cloudera Manager, Version 5.14.2
baseurl = http://192.168.198.181/cm5/redhat/6/x86_64/cm/5/
gpgcheck = 0

# 检查数据源 浏览器是否能打开
# http://192.168.198.181/parcels/     
# http://192.168.198.181/cm5/redhat/6/x86_64/cm/5/
```

### 安装

```shell
# server 
cd /var/www/html/cm5/redhat/6/x86_64/cm/5/RPMS/x86_64

# 所有节点
yum install -y cloudera-manager-daemons-5.14.2-1.cm5142.p0.8.el7.x86_64.rpm

yum install -y cloudera-manager-agent-5.14.2-1.cm5142.p0.8.el7.x86_64.rpm


# 主节点+
yum install -y cloudera-manager-server-5.14.2-1.cm5142.p0.8.el7.x86_64.rpm 
```

### mysql

```shell
# mysql驱动 上传
mkdir -p /usr/share/java
cd /usr/share/java

# 上传 mysql-connector-java-5.1.27.jar
mv ./mysql-connector-java-5.1.27.jar mysql-connector-java.jar
```

```sql
# 创建数据库
create database cmf DEFAULT CHARACTER SET utf8;
grant all on cmf.* TO 'cmf'@'localhost' IDENTIFIED BY 'cmf_password';
grant all on cmf.* TO 'cmf'@'%' IDENTIFIED BY 'cmf_password';
grant all on cmf.* TO 'cmf'@'192.168.198.181' IDENTIFIED BY 'cmf_password';
flush privileges;
```

```shell
# 配置数据库
 cd /etc/cloudera-scm-server/
 vi db.properties

# Currently 'mysql', 'postgresql' and 'oracle' are valid databases.
com.cloudera.cmf.db.type=mysql

# The database host
# If a non standard port is needed, use 'hostname:port'
com.cloudera.cmf.db.host=192.168.198.181

# The database name
com.cloudera.cmf.db.name=cmf

# The database user
com.cloudera.cmf.db.user=cmf

# The database user's password
com.cloudera.cmf.db.password=cmf_password

# The db setup type
# By default, it is set to INIT
# If scm-server uses Embedded DB then it is set to EMBEDDED
# If scm-server uses External DB then it is set to EXTERNAL
com.cloudera.cmf.db.setupType==EXTERNAL
```

### 启动

```shell

service cloudera-scm-server start

# 日志错误检查
cd /var/log/cloudera-scm-server/
cat cloudera-scm-server.log

http://192.168.198.181:7180
```
