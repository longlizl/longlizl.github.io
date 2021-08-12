### 1.连接创建数据库

```shell
#!/bin/bash
mysql="mysql -uroot -p111111"
#sql="show tables from mysql"
sql="create table test.user(
        id int unsigned auto_increment primary key,
        username varchar(20),
        password varchar(30)
)"
$mysql -e "$sql"    
```

### 2.使用带参数脚本

```shell
#!/bin/bash
mysql="mysql -uroot -p111111"
#sql="create database test1"
#sql="create table test1.user(
#       id int not null primary key,
#       name char(30),
#       password char(50)
#)"
#sql="insert into test1.user(id,name,password) values(1,'user1',password("123")),(2,'user2',password("123")),(3,'user3',password("123"))"
case $1 in
        select|*)
                sql="select * from test1.user order by id"
                ;;
        delete)
                sql="delete from test1.user where id=$2"
                ;;
        insert)
                sql="insert into test1.user(id,name,password) values('$2','$3','$4')"
                ;;
        update)
                sql="select * from test1.user order by id"
                ;;
esac
$mysql -e "$sql"
```

### 3.将nginx访问数据按照ip访问量统计并写入数据库

创建好存放数据的表

```mysql
mysql -uroot -p111111 -e "create table test1.accesstable(
       id int unsigned  not null  auto_increment  primary key,
       access_time char(50),
       access_ip char(50),
       access_count int(20)
 )"
```

```shell
#!/bin/bash
#accessdabase
datetime=`date +%Y-%m-%d`
tempfile=temp.txt
# 前一天的日志
logfile="/var/log/nginx/access.log-`date -d '-1 day' +'%Y%m%d'`.gz"
zcat $logfile | awk '{print $1}'| sort | uniq -c | awk '{print $2":"$1}' > $tempfile
mysql="mysql -uroot -p111111"
for i in `cat $tempfile`
do
        ip=`echo $i|awk -F: '{print $1}'`
        count=`echo $i | awk -F: '{print $2}'`
        sql="insert into test1.accesstable(access_time,access_ip,access_count) values('$datetime','$ip','$count')"
        $mysql -e "$sql"
done
sql="select * from test1.accesstable"
$mysql -e "$sql"
```

