# MySQL 安装

## 版本

mysql 社区版 5.5.9

## server

### 下载

```bash
wget --no-check-certificate https://downloads.mysql.com/archives/get/file/MySQL-server-5.5.9-1.rhel5.x86_64.rpm
```

### 安装

```bash
rpm -ivh MySQL-server-5.5.9-1.rhel5.x86_64.rpm
Preparing...                ########################################### [100%]
   1:MySQL-server           ########################################### [100%]

PLEASE REMEMBER TO SET A PASSWORD FOR THE MySQL root USER !
To do so, start the server, then issue the following commands:

/usr/bin/mysqladmin -u root password 'new-password'
/usr/bin/mysqladmin -u root -h vr105 password 'new-password'

Alternatively you can run:
/usr/bin/mysql_secure_installation

which will also give you the option of removing the test
databases and anonymous user created by default.  This is
strongly recommended for production servers.

See the manual for more instructions.

Please report any problems with the /usr/bin/mysqlbug script!
```

## client

### 下载

```bash
wget --no-check-certificate https://downloads.mysql.com/archives/get/file/MySQL-client-5.5.9-1.rhel5.x86_64.rpm
```

### 安装

```bash
rpm -ivh MySQL-client-5.5.9-1.rhel5.x86_64.rpm
Preparing...                ########################################### [100%]
   1:MySQL-client           ########################################### [100%]
```

## 启动

```bash
service mysql start
Starting MySQL..                                           [  OK  ]
```

## 设置密码

```bash
/usr/bin/mysqladmin -u root password 'meizu.com'
/usr/bin/mysqladmin -u root -h vr105 password 'meizu.com'
```

## 允许远程访问

```bash
mysql
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 20
Server version: 5.5.9 MySQL Community Server (GPL)

Copyright (c) 2000, 2010, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

输入如下 sql

```sql
grant all privileges on *.* to 'root'@'%' identified by 'meizu.com' with grant option;
```

授权成功

```text
Query OK, 0 rows affected (0.00 sec)
```

## 配置

### 配置文件

mysql 配置文件是 /etc/my.cnf，但是 rpm 方式安装的话，这个文件并不存在，这时只需要将 /usr/share/mysql/my-medium.cnf 复制到 /etc 下并更名为 my.cnf 即可

### 默认字符集

编辑 /etc/my.cnf，将默认字符集修改为utf8mb4

* 在 \[client\] 小节，增加 default-character-set=utf8mb4
* 在 \[mysqld\] 小节，增加 character-set\_server=utf8mb4，注意不要用 default-character-set=utf8mb4，否则 mysql 无法启动

