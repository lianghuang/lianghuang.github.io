---
layout: post
title: "问题定位-mysql在虚拟机重启后，通过service mysqld 启动失败"
date: 2016-09-02
tags:
    - mysql
    - 问题定位
    - Work
author: "huangliangliang"
---

本文主要记录了mysql启动时候遇到的错误及解决方案。

一些mysql的说明：

远程执行mysql脚本
mysql -utest -p -Dcloudmaster < 123335.sql
-D参数指明数据库

mysql 默认配置文件路径：/etc/my.cnf

样例：

```
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
user=mysql

skip-name-resolve
#skip-grant-tables
lower_case_table_names=1
skip-external-locking
skip-host-cache

# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0

max_allowed_packet = 10M
max_connections=1000


[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
```

# 问题：mysql在虚拟机重启后，通过service mysqld 启动失败

1. 通过mysqld_safe命令启动mysql，显示启动失败
2. 查看日志文件 /var/log/mysqld.log

```
160413 21:36:27 mysqld_safe Starting mysqld daemon with databases from /var/lib/mysql
160413 21:36:27  InnoDB: Initializing buffer pool, size = 8.0M
160413 21:36:27  InnoDB: Completed initialization of buffer pool
160413 21:36:27  InnoDB: Started; log sequence number 0 1287245
160413 21:36:27 [ERROR] /usr/libexec/mysqld: Can't create/write to file '/var/run/mysqld/mysqld.pid' (Errcode: 2)
160413 21:36:27 [ERROR] Can't start server: can't create PID file: No such file or directory
160413 21:36:27 mysqld_safe mysqld from pid file /var/run/mysqld/mysqld.pid ended
```

显示无法创建文件mysqld.pid权限

3. 看日志是说，在/var/run/目录下未找到mysqld文件夹，因此首先创建它。同时赋文件夹权限给mysql

```
chown mysql:mysql /var/run/mysqld
```

4. 之前还遇到过异常启动mysql后，步骤一描述的mysql.sock被锁定，无法被重启后的mysql进行获取的情况。
所以如果报sock问题，可以把mysql.sock删除掉。让新进程重新创建。


附：
stackoverflow推荐的解决mysql问题流程：

```
Check the permission of mysql data dir using below command
# ls -ld /var/lib/mysql/
Check the permission of databases inside mysql data dir using below command
# ls -lh /var/lib/mysql/
Check the listening network tcp ports using below command
# netstat -ntlp
-t (tcp)仅显示tcp相关选项
-n 拒绝显示别名，能显示数字的全部转化成数字。就是以IP地址显示出来
-l 仅列出有在 Listen (监听) 的服务状态
-p 显示建立相关链接的程序名  就是pid
Check the mysql log files for any error using below command.
# cat /var/log/mysql/mysqld.log
Try to start mysql using below command
# mysqld_safe --defaults-file=/etc/my.cf
```
