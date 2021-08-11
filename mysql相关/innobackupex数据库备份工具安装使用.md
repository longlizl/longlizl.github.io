mysql 备份工具 innobackupex 安装

```shell
yum -y install https://repo.percona.com/yum/percona-release-latest.noarch.rpm
```

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\dadfb889ca8c47d98d43ded995efdeae\clipboard.png)

```shell
yum install -y  percona-xtrabackup-24
```

压缩工具：

```shell
yum install -y  qpress
```

备份方案

**1.全备，还原**

新建备份目录

```shell
mkdir -p /root/backup/{full_backup,increment_data}

innobackupex  --defaults-file=/etc/my.cnf --user=root --password=111111  /root/backup/full_backup/

cd /root/backup/full_backup
```

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\23d2f45ec82643d58a51de36fe2fe559\clipboard.png)

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\be7df17d70684886bd25fb2e0a600432\clipboard.png)

```shell
# 利用--apply-log的作用是通过回滚未提交的事务及同步已经提交的事务至数据文件使数据文件处于一致性状态利用
# --apply-log的作用是通过回滚未提交的事务及同步已经提交的事务至数据文件使数据文件处于一致性状态

innobackupex --apply-log /root/backup/full_backup/2021-03-29_15-42-06/

还原数据(mysql服务已停止，被还原数据库数据目录为空)

datadir=/var/lib/mysql #就是my.cnf这个配置文件指定的目录没任何文件

innobackupex --defaults-file=/etc/my.cnf --copy-back /root/backup/full_backup/2021-03-29_15-42-06/

将数据目录用户及组更改为mysql用户

chown -R mysql:mysql /var/lib/mysql 
```

重启mysql服务检查数据

**2.增备，还原**

```mysql
第一次增备: /root/backup/full_backup/2021-03-29_15-42-06
第一次增备文件：/root/backup/increment_data/2021-03-31_15-23-06
第二次增备文件：/root/backup/increment_data/2021-03-31_20-45-53

全备文件：
innobackupex --defaults-file=/etc/my.cnf --user=root --password=111111  --incremental-basedir=/root/backup/full_backup/2021-03-29_15-42-06/ --incremental /root/backup/increment_data

第二次增备（再第一次增备基础上）:
innobackupex --defaults-file=/etc/my.cnf --user=root --password=111111  --incremental-basedir=/root/backup/2021-03-29_15-42-06/ --incremental /root/backup/increment_data

还原：
1. 先replay全量备份
innobackupex --apply-log --redo-only /root/backup/full_backup/2021-03-29_15-42-06/ 
2. 再replay第一次增量备份
innobackupex --apply-log --redo-only /root/backup/full_backup/2021-03-29_15-42-06/ --incremental /root/backup/increment_data/2021-03-31_15-23-06
3. 再replay第二次增量备份
innobackupex --apply-log --redo-only /root/backup/full_backup/2021-03-29_15-42-06/ --incremental /root/backup/increment_data/2021-03-31_20-45-53
4. 再把所有整合的replay一次
innobackupex --apply-log  /root/backup/full_backup/2021-03-29_15-42-06/ 


```


