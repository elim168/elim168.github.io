
### 1.创建group
```shell
shell> groupadd mysql
```

### 2.创建user
```shell
shell> useradd -r -g mysql -s /bin/false mysql
```

### 3.解压缩mysql压缩包到你想要安装的目录，解压缩时可以使用-C指定解压到的目标目录。
```shell
shell> tar -zxvf mysql-5.5.58.tar.gz
```

解压后可使用mv进行重命名，重命名为mysql-5.5.58
```shell
shell> mv mysql-5.5.58-linux-glibc2.12-x86_64 mysql-5.5.58
```

### 4.把解压缩后的mysql文件的所属用户和组都改为mysql。
```shell
shell> chown -R mysql mysql-5.5.58
shell> chgrp -R mysql mysql-5.5.58
```

### 5.初始化mysql数据库的核心数据，在mysql-5.5.58目录下执行如下指令。
```shell
shell> scripts/mysql_install_db --user=mysql
```

### 6.启动mysql服务
```shell
shell> bin/mysqld_safe --user=mysql
```

### 7.验证mysql服务是否成功启动，可以使用如下命令。
```shell
shell> bin/mysqladmin version
```
如果能够正确的输出版本信息则表示安装成功，如：
```
bin/mysqladmin  Ver 8.42 Distrib 5.5.58, for linux-glibc2.12 on x86_64
Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Server version		5.5.58
Protocol version	10
Connection		Localhost via UNIX socket
UNIX socket		/tmp/mysql.sock
Uptime:			3 min 50 sec

Threads: 1  Questions: 2  Slow queries: 0  Opens: 33  Flush tables: 1  Open tables: 26  Queries per second avg: 0.008

```

### 8.登录mysql
初始化的mysql的root用户是没有密码的，所以我们可以使用`mysql -u root`进行登录。
```shell
shell> bin/mysql -u root
```

### 9.为root用户指定密码
基本语法如下：
```shell
mysql> SET PASSWORD FOR 'root'@'host' = PASSWORD('new_password');
```


查看mysql.user表中我们将会看到root用户和localhost的组合，所以我们可以通过如下方式指定初始密码。
```shell
mysql> SET PASSWORD FOR 'root'@'localhost' = PASSWORD('elim');
Query OK, 0 rows affected (0.00 sec)
```


也可以通过直接更新mysql.user表中的密码。
```shell
shell> mysql -u root
mysql> UPDATE mysql.user SET Password = PASSWORD('new_password')
    ->     WHERE User = 'root';
mysql> FLUSH PRIVILEGES;
```


还可以通过mysqladmin来设置root用户的密码
```shell
shell> mysqladmin -u root password "new_password"
shell> mysqladmin -u root -h host_name password "new_password"
```


指定了密码后，我们再通过不指定root用户密码的方式登录将不能再登录了。
```shell
shell> bin/mysql -u root
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: NO)
```


接着就可以使用我们刚刚指定的密码elim来进行登录了。
```shell
shell> bin/mysql -uroot -pelim
```

### 10.安装mysql为linux服务
复制support-files目录下的mysql.server到/etc/init.d目录下，并命名为mysql。
```shell
shell> cp support-files/mysql.server /etc/init.d/mysql
```


此外，需要确保/etc/init.d/mysql拥有执行权限（通常会自动带上）。
```shell
shell> chmod +x /etc/init.d/mysql
```


复制support-files目录下的一个my-xxx.cnf文件到/etc目录下，并命名为my.cnf
```shell
shell> cp support-files/my-medium.cnf /etc/my.cnf
```


如果mysql的基本路径不是/usr/local/mysql，则需要在/etc/my.cnf的mysqld块下指定basedir和datadir，如：
```
[mysqld]
basedir=/usr/local/mysql-5.5.58
datadir=/usr/local/mysql-5.5.58/data
socket=/var/tmp/mysql.sock
port=3306
user=mysql
```


然后使用`systemctl daemon-reload`重新加载服务信息后就可以通过`service mysql start`来启动mysql服务了。


### 参考文档
* https://dev.mysql.com/doc/refman/5.5/en/binary-installation.html



















