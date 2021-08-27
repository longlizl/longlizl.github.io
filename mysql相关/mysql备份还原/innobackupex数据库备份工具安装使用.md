# innobackupex 备份工具安装使用

## 1. innobackupex 安装

```shell
yum -y install https://repo.percona.com/yum/percona-release-latest.noarch.rpm
```

![img](https://longlizl.github.io/mysql相关/mysql备份还原/images/1.png)

```shell
yum install -y  percona-xtrabackup-24
```

## 2. qpress压缩工具安装：

```shell
yum install -y  qpress
```

# 备份方案

## 1. 全备，还原

```shell
# 新建备份目录
mkdir -p /root/backup/{full_backup,increment_data}
innobackupex  --defaults-file=/etc/my.cnf --user=root --password=111111  /root/backup/full_backup/
cd /root/backup/full_backup
```

​	查看备份后文件（可以看到）

![img](https://longlizl.github.io/mysql相关/mysql备份还原/images/2.png)

![img](https://longlizl.github.io/mysql相关/mysql备份还原/images/3.png)

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
第一次全备文件目录: /root/backup/full_backup/2021-03-29_15-42-06
第一次增备文件目录：/root/backup/increment_data/2021-03-31_15-23-06
第二次增备文件目录：/root/backup/increment_data/2021-03-31_20-45-53

第一次增备（再全备基础上）：
innobackupex --defaults-file=/etc/my.cnf --user=root --password=111111  --incremental-basedir=/root/backup/full_backup/2021-03-29_15-42-06 --incremental /root/backup/increment_data

第二次增备（再第一次增备基础上）:
innobackupex --defaults-file=/etc/my.cnf --user=root --password=111111  --incremental-basedir=/root/backup/increment_data/2021-03-31_15-23-06 --incremental /root/backup/increment_data

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

