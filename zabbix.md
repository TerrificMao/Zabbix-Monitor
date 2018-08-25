## 安装Zabbix

Zabbix 是一个企业级的分布式开源监控方案。

Zabbix是一款能够监控各种网络参数以及服务器健康性和完整性的软件。Zabbix使用灵活的通知机制，允许用户为几乎任何事件配置基于邮件的告警。这样可以快速反馈服务器的问题。基于已存储的数据，Zabbix提供了出色的报告和数据可视化功能。这些功能使得Zabbix成为容量规划的理想方案。

安装要求: Zabbix同时需要物理内存和磁盘空间。刚开始使用Zabbix，建议128MB物理内存和256MB可用磁盘空间。如果计划对监控的参数进行长期保存，应考虑至少在数据库中预留几个GB的空间，以用来保留历史数据。 每个Zabbix的守护进程需要与数据库服务器建立多个连接。分配给连接的内存数量，取决于数据库引擎的配置。

![Image one](image/zabbix-dashboard.png)

https://yunjin-keji.com/install-zabbix3-on-amazon-linux 

## 一、安装Zabbix以及核心组件

监控主机数量达到上千台，数据库建议是使用PostgreSQL 8.3以上的版本**

```bash
$ sudo yum update -y
$ cat /etc/system-release
Amazon Linux AMI release 2018.03
```

安装http24

```bash
[ec2-user@ip-10-200-1-44 ~]$ sudo -s
[root@ip-10-200-1-44 ec2-user]# yum install httpd24 -y
[root@ip-10-200-1-44 ec2-user]# chkconfig httpd on
[root@ip-10-200-1-44 ec2-user]# chkconfig httpd on  //设置开机自动启动
[root@ip-10-200-1-44 ec2-user]# service httpd start //启动httpd
Starting httpd: 

```

安装PHP5.6

```bash
[root@ip-10-200-1-44 ec2-user]# yum install php56 php56-devel php56-mbstring php56-mcrypt php56-pgsql php56-bcmath php56-gd php56-ldap -y
```

修改PHP时间戳

```bash
[root@ip-10-200-1-44 ec2-user]# sed -i -e "s/;date.timezone =/date.timezone = Asia\/Shanghai/g" /etc/php.ini
```

安装Zabbix，yum安装Zabbix及相关安装包

```bash
[root@ip-10-200-1-44 ec2-user]# rpm -ivh http://repo.zabbix.com/zabbix/3.0/rhel/6/x86_64/zabbix-release-3.0-1.el6.noarch.rpm
```

最新的版本请查看
https://www.zabbix.com/documentation/3.4/zh/manual/installation/install_from_packages
安装Zabbix agent以及相关的组件

```bash
[root@ip-10-200-1-44 ec2-user]# yum install -y \
> zabbix-agent \
> zabbix-web \
> zabbix-web-pgsql \
> zabbix-server-pgsql \
> zabbix-sender \
> zabbix-java-gateway \
> zabbix-get
```

```
也可以安装mysql引擎，但是生产环境机器数据很多的话官网建议使用pgsql
yum install zabbix-server-mysql zabbix-web-mysql
```

拷贝Zabbix使用的httpd配置文件

```bash
[root@ip-10-200-1-44 doc]# cp -p /usr/share/doc/zabbix-web-3.0.20/httpd24-example.conf /etc/httpd/conf.d/zabbix.conf
[root@ip-10-200-1-44 doc]# service httpd restart
Stopping httpd:                                            [  OK  ]
Starting httpd:                                            [  OK  ]
```

安装postgresql

```bash
[root@ip-10-200-1-44 doc]# yum install postgresql postgresql-server postgresql-devel postgresql-contrib -y
确认安装的版本
[root@ip-10-200-1-44 doc]# psql --version
psql (PostgreSQL) 9.2.24
```

初始化及启动postgresql

```bash
[root@ip-10-200-1-44 doc]# service postgresql initdb
Initializing database:                                     [  OK  ]
[root@ip-10-200-1-44 doc]# service postgresql start
Starting postgresql service:                               [  OK  ]
```

默认用户postgres设定密码

```bash
[root@ip-10-200-1-44 doc]# su - postgres
-bash-4.2$ psql
psql (9.2.24)
Type "help" for help.

postgres=# alter role postgres with password 'postgres';
ALTER ROLE
postgres=# \q
```

修改postgresql认证方法

```bash
$ vi /var/lib/pgsql9/data/pg_hba.conf
# "local" is for Unix domain socket connections only
local   all             all                                     md5
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
-bash-4.2$ exit
logout

[root@ip-10-200-1-44 doc]# service postgresql restart
Stopping postgresql service:                               [  OK  ]
Starting postgresql service:                               [  OK  ]
```

创建用户zabbix，并设定密码。

```bash
[root@ip-10-200-1-44 doc]# su - postgres
上一次登录：五 8月 24 16:42:01 UTC 2018pts/0 上
-bash-4.2$ createuser -P zabbix
Enter password for new role: 输入密码
Enter it again: 
Password: 再次输入密码
```

创建数据库。

```bash
-bash-4.2$ createdb -O zabbix -E UTF8 zabbix
-bash-4.2$ psql -l
Password:
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 zabbix    | zabbix   | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
(4 rows)
```

导入zabbix数据

```bash
zcat /usr/share/doc/zabbix-server-pgsql-3.0.20/create.sql.gz | psql -U zabbix zabbix -W
INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1
COMMIT
```

修改zabbix配置文件，配置数据库连接信息，主要是DBHost=localhost、DBName=zabbix、DBUser=zabbix、DBPassword=zabbix

```bash
-bash-4.2$ exit
logout
# vi /etc/zabbix/zabbix_server.conf
---
# DBPassword=
↓
DBPassword=<Password>
```

启动zabbix-server并配置为开机自动启动

```bash
[root@ip-10-200-1-44 doc]# vi /etc/zabbix/zabbix_server.conf
---
# DBPassword=
↓
DBPassword=<Password>
[root@ip-10-200-1-44 doc]# service zabbix-server start //启动Zabbix Server进程 
Starting Zabbix server:                                    [  OK  ]
[root@ip-10-200-1-44 doc]# chkconfig zabbix-server on  //设置开机自启动
```

**看情况编辑Zabbix前端的PHP配置**

Zabbix前端的Apache配置文件位于 /etc/apache2/conf.d/zabbix 或者 /etc/apache2/conf-enabled/zabbix.conf 。一些PHP设置已经完成了配置。

```bash
php_value max_execution_time 300
php_value memory_limit 128M
php_value post_max_size 16M
php_value upload_max_filesize 2M
php_value max_input_time 300
php_value always_populate_raw_post_data -1
# php_value date.timezone Europe/Riga
```

依据所在时区，你可以取消 “date.timezone” 设置的注释，并正确配置它。在配置文件更改后，需要重启Apache Web服务器。

```bash
service apache2 restart
```

## 二、Web界面安装Zabbix

![Image one](image/01.png)

公网IP: 52.80.221.137，Zabbix UI界面安装地址: http://52.80.221.137/zabbix/

![Image one](image/02.png)

Zabbix监控平台安装成功。默认的用户名／密码为 Admin/zabbix。

![Image one](image/03.png)

## 三、Zabbix升级，大致骤分为7个。

https://www.zabbix.com/documentation/3.4/zh/manual/installation/upgrade

## 四、Zabbix用户管理

在*管理（Administration） → 用户（Users）*下 查看用户信息

 Zabbix在安装后只定义了两个用户。 

-  'Admin' 用户是Zabbix的一个超级管理员，拥有所有权限。 
-  'Guest' 用户是一个特殊的默认用户。如果你没有登陆，你访问Zabbix的时候使用的其实是“guest”权限。默认情况下，“guest”用户对Zabbix中的对象没有任何权限。

例如，我创建了一个Damon Mao的用户，并加入到Zabbix administrator组。

![Image one](image/04.png)

## 五、添加监控主机

Zabbix中，可以通过*配置（Configuration）→ 主机（Hosts）*菜单，查看已配置的主机信息。默认已有一个名为'Zabbix server'的预先定义好的主机。

![Image one](image/05.png)

















