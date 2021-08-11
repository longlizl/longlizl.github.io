mysql主从复制原理

从库生成两个线程，一个I/O线程，一个SQL线程；

 

i/o线程去请求主库 的binlog，并将得到的binlog日志写到relay log（中继日志） 文件中；

主库会生成一个 log dump 线程，用来给从库 i/o线程传binlog；

 

SQL 线程，会读取relay log文件中的日志，并解析成具体操作，来实现主从的操作一致，而最终数据一致；

\------------------------------------------------------------------------------------------------------ 

master:172.29.20.74

slave:172.29.20.98

一.master数据库配置

vim  /etc/my.cnf

[mysqld]

datadir=/var/lib/mysql

socket=/var/lib/mysql/mysql.sock

\# Disabling symbolic-links is recommended to prevent assorted security risks

symbolic-links=0

\# Settings user and group are ignored when systemd is used.

\# If you need to run mysqld under a different user or group,

\# customize your systemd unit file for mariadb according to the

\# instructions in http://fedoraproject.org/wiki/Systemd

server-id=1

log-bin=mysql-bin

[mysqld_safe]

log-error=/var/log/mariadb/mariadb.log

pid-file=/var/run/mariadb/mariadb.pid

\#

\# include all files from the config directory

\#

!includedir /etc/my.cnf.d

1.重启mysql，查看master数据库状态

MariaDB [(none)]> show master status;

+------------------+----------+--------------+------------------+

| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |

+------------------+----------+--------------+------------------+

| mysql-bin.000001 |     2935 |              |                  |

+------------------+----------+--------------+------------------+

1 row in set (0.00 sec)

2.创建用作同步数据的用户

grant replication slave on **.** to 'repl'@'172.29.20.%'  identified by '111111';

3.备份数据库

flush tables with read lock;#需要先锁定表

mysqldump -uroot -p111111 -A  -B > all.sql

show master status;

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\a91a67f196ac455fab3cdf75ac67a347\clipboard.png)

二.slave数据库配置

vim /etc/my.cnf

server-id=2

relay-log=relay-bin

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\e03a8aac60ae492681ed158fe24ebe81\clipboard.png)

systemctl restart mariadb

将master中备份的数据库文件all.sql拷贝到slave主机中并导入数据库：

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\ca37015e3d0d42f4bdb3c5e749759615\clipboard.png)

stop slave

三.设置主从同步(从库执行)：

change master to

master_host='172.29.20.74',master_user='repl',master_password='111111',master_log_file='mysql-bin.000002',master_log_pos=245;

1.start slave

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\9a92c3f6912e4aecbafb1bf0f1af098d\clipboard.png)

2.在master库执行解锁操作

UNLOCK TABLES;

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\e00141fde89145e6a4cd680f19986f9d\clipboard.png)

通过二进制文件恢复数据

mysqlbinlog mysql-bin.000002 --start-position=787   --stop-position=1668|mysql -uroot -p111111