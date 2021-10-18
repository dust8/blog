---
title: ubuntu20安装mysql5.7
date: 2021-10-18 05:15:31
tags:
  - ubuntu
  - mysql
---


ubuntu20 默认使用 `mysql-server` 安装的是mysql8,所以需要额外操作.


## 安装
```
wget https://dev.mysql.com/get/mysql-apt-config_0.8.12-1_all.deb
# 选择 Bionic, mysql5.7
dpkg -i mysql-apt-config_0.8.12-1_all.deb

apt-get update

# 查看各个版本,看有没有5.7的
apt-cache policy mysql-server

# 安装
apt install -f mysql-client=5.7* mysql-community-server=5.7* mysql-server=5.7*

# 配置
mysql_secure_installation

# 连接测试
mysql -u root -p 
SELECT VERSION()
```

## 配置(可选)
### 远程访问
```
vim /etc/mysql/mysql.conf.d/mysqld.cnf

bind-address   = 0.0.0.0
```

### 创建新用户
```
CREATE USER 'user'@'%' IDENTIFIED BY 'MyStrongPass.';
GRANT ALL PRIVILEGES ON * . * TO 'user'@'%'; 
FLUSH PRIVILEGES;
```

### 修改密码
```
格式：mysqladmin -u用户名 -p旧密码 password 新密码 
例子：mysqladmin -uroot -p123456 password 123 
```

## 一些坑
### 坑1: 先安装高版本导致低版本启动不了
因为先安装了mysql8, 卸载后安装mysql5.8 一直报错,启动不了, 报错信息很少, 最后是在日志文件 `/var/log/mysqld.log` 中发现 `[ERROR] [FATAL] InnoDB: Table flags are 0 in the data dictionary but the flags in file ./ibdata1 are 0x4800!` 这样的报错信息,搜索发现高版本的mysql文件未卸载干净,删掉 `/var/lib/mysql` 整个目录 重新安装就可以了.

### 坑2: 程序连不到mysql
程序里面用的是`127.0.0.1`, 报错说连不到`root@localhost`
```
select user, host, plugin, authentication_string from mysql.user;
```
发现 root 用户不是密码登录, 可以修改为密码登录的
```
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'xxx';
```

## 参考链接
- [How To Install MySQL 5.7 on Ubuntu 20.04](https://computingforgeeks.com/how-to-install-mysql-on-ubuntu-focal/)
- [InnoDB: Table flags are 0 in the data dictionary but the flags in file ./ibdata1 are 0x4800!](https://www.jianshu.com/p/7af95b08b954)
- [从输入任何密码都可以直接登录 MySQL 的 root 用户谈 auth_socket 验证插件](https://blog.csdn.net/weixin_43424368/article/details/109600313)