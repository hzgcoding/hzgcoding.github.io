---
title: linux 安装指定版本MySql
date: 2020-01-23 15:12:39
tags: linux
categories: Mysql
---

由于工作环境、生产环境，我们使用的操作系统为为CentOS6.9，所需mysql版本为5.7，目前CentOS6.x系统默认mysql版本为5.1，这个版本是实在是太旧了。

### 彻底卸载系统已经安装的旧版本
- 检查系统已经安装的mysql
` rpm -qa|grep -i mysql`

- 删除包
`rpm -ev mysql_lib_xxxx`

- 删除老版本安装残留文件
`find / -iname mysql*` 删除对应目录已经文件

- 删除my.cnf配置文件

### 使用yum安装MySql5.7

- 下载mysql5.7源
`wget dev.mysql.com/get/mysql-community-release-el6-5.noarch.rpm`

- 安装源
`yum localinstall mysql-community-release-el6-5.noarch.rpm`

- 查看可用源中包含哪些版本并开启指定版本
```
yum repolist all | grep mysql
yum-config-manager --disable mysql56-community
yum-config-manager --disable mysql55-community
yum-config-manager --enable mysql57-community
```

- yum安装mysql
`yum install mysql-community-server`

- 启动mysql
`service mysqld start`

- 设置开机自动启动

```
chkconfig --list | grep mysqld
chkconfig mysqld on
```

- 安装设置命令
`mysql_secure_installation`

### 修改root密码
因为刚才启动的时候是系统默认配置的临时密码
使用如下命令可以查看，并且修改：
```
sudo grep 'temporary password' /var/log/mysqld.log
mysql -u root -p 
ALTER USER 'root'@'localhost' IDENTIFIED BY 'newPassword';
```

### 设置允许连接数据库
命令如下：
```
mysql -u root -p 
grant all privileges on *.* to root@"%" identified by 'passwordith grant option;  
flush privileges;
```

### 遇到的问题
- 比较奇怪，域名解析错误
 在yum install 的时候，发现大量的[Errno 14] PYCURL ERROR 6 - "Couldn't resolve host 'mirrors.aliyun.com'"错误，开始以为自己源设置错误，后来才知道，机器卡死过一次，导致系统莫名其妙的错误，了解网络的很快就知道需要设置系统的DNS
 在`/etc/resolv.conf`文件中加入如下内容：
 ```
 nameserver 8.8.8.8
  nameserver 114.114.114.114
 ```
 之前是啥也没有的，都是些没用的注释解释信息