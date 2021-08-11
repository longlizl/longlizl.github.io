**一、备份单个数据库**

**1、备份命令：mysqldump**

```mysql
　　MySQL数据库自带的一个很好用的备份命令。是逻辑备份，导出 的是SQL语句。也就是把数据从MySQL库中以逻辑的SQL语句的形式直接输出或生成备份的文件的过程。

单实例语法（Syntax）: 
mysqldump -u <username> -p <dbname> > /path/to/*.sql 
多实例的备份语法（Syntax）： 
mysqldump -u <username> -p <dbname>  -S <sockPath> > /path/to/*.sql

eg: mysqldump -u root -p wordpress > /opt/wordpress_$(date +%F).sql
```

**2、参数解析**

```mysql
 1 -A --all-databases：导出全部数据库 

2 -Y --all-tablespaces：导出全部表空间 

3 -y --no-tablespaces：不导出任何表空间信息 

4 --add-drop-database每个数据库创建之前添加drop数据库语句。 

5 --add-drop-table每个数据表创建之前添加drop数据表语句。(默认为打开状态，使用--skip-add-drop-table取消选项) 

6 --add-locks在每个表导出之前增加LOCK TABLES并且之后UNLOCK TABLE。(默认为打开状态，使用--skip-add-locks取消选项) 

7 --comments附加注释信息。默认为打开，可以用--skip-comments取消 

8 --compact导出更少的输出信息(用于调试)。去掉注释和头尾等结构。可以使用选项：--skip-add-drop-table --skip-add-locks --skip-comments --skip-disable-keys 

9 -c --complete-insert：使用完整的insert语句(包含列名称)。这么做能提高插入效率，但是可能会受到max_allowed_packet参数的影响而导致插入失败。 

10 -C --compress：在客户端和服务器之间启用压缩传递所有信息 

11 -B--databases：导出几个数据库。参数后面所有名字参量都被看作数据库名。 

12 --debug输出debug信息，用于调试。默认值为：d:t:o,/tmp/ 

13 --debug-info输出调试信息并退出 

14 --default-character-set设置默认字符集，默认值为utf8 

15 --delayed-insert采用延时插入方式（INSERT DELAYED）导出数据 

16 -E--events：导出事件。 

17 --master-data：在备份文件中写入备份时的binlog文件，在恢复进，增量数据从这个文件之后的日志开始恢复。值为1时，binlog文件名和位置没有注释，为2时，则在备份文件中将binlog的文件名和位置进行注释 

18 --flush-logs开始导出之前刷新日志。请注意：假如一次导出多个数据库(使用选项--databases或者--all-databases)，将会逐个数据库刷新日志。除使用--lock-all-tables或者--master-data外。在这种情况下，日志将会被刷新一次，相应的所以表同时被锁定。因此，如果打算同时导出和刷新日志应该使用--lock-all-tables 或者--master-data 和--flush-logs。 

19 --flush-privileges在导出mysql数据库之后，发出一条FLUSH PRIVILEGES 语句。为了正确恢复，该选项应该用于导出mysql数据库和依赖mysql数据库数据的任何时候。 

20 --force在导出过程中忽略出现的SQL错误。 

21 -h --host：需要导出的主机信息 

22 --ignore-table不导出指定表。指定忽略多个表时，需要重复多次，每次一个表。每个表必须同时指定数据库和表名。例如：--ignore-table=database.table1 --ignore-table=database.table2 …… 

23 -x --lock-all-tables：提交请求锁定所有数据库中的所有表，以保证数据的一致性。这是一个全局读锁，并且自动关闭--single-transaction 和--lock-tables 选项。 

24 -l --lock-tables：开始导出前，锁定所有表。用READ LOCAL锁定表以允许MyISAM表并行插入。对于支持事务的表例如InnoDB和BDB，--single-transaction是一个更好的选择，因为它根本不需要锁定表。请注意当导出多个数据库时，--lock-tables分别为每个数据库锁定表。因此，该选项不能保证导出文件中的表在数据库之间的逻辑一致性。不同数据库表的导出状态可以完全不同。 

25 --single-transaction：适合innodb事务数据库的备份。保证备份的一致性，原理是设定本次会话的隔离级别为Repeatable read，来保证本次会话（也就是dump）时，不会看到其它会话已经提交了的数据。 

26 -F：刷新binlog，如果binlog打开了，-F参数会在备份时自动刷新binlog进行切换。 

27 -n --no-create-db：只导出数据，而不添加CREATE DATABASE 语句。 

28 -t --no-create-info：只导出数据，而不添加CREATE TABLE 语句。 

29 -d --no-data：不导出任何数据，只导出数据库表结构。 

30 -p --password：连接数据库密码 

31 -P --port：连接数据库端口号 

32 -u --user：指定连接的用户名。

33 --set-gtid-purged=OFF  关闭gtid信息显示
```

举例使用：

```mysql
# a、导出整个数据库(包括数据库中的数据） 
   mysqldump -u username -p dbname > dbname.sql  

# b、导出数据库结构（不含数据）
   mysqldump -u username -p -d dbname > dbname.sql 

# c、导出数据库中的某张数据表（包含数据） 
   mysqldump -u username -p dbname tablename > tablename.sql 

# d、导出数据库中的某张数据表的表结构（不含数据） 
   mysqldump -u username -p -d dbname tablename > tablename.sql
```

**3、恢复操作**

```mysql
语法（Syntax）： 
mysql -u<username> -p<password> <dbname> < /opt/mytest_bak.sql   #库必须保留，空库也可 说明：指定dbname，相当于use <dbname>
```

**4、示例**

```mysql
 1. 无参数备份数据库mytest和恢复
	备份操作: 
	a、备份 mysqldump -uroot -p‘123456’ mytest > /mnt/mytest_bak_$(date +%F).sql
	
	恢复操作: 
	a、删除student表（库必须要保留，空库都行） mysql -uroot -p'123456' -e "use mytest;drop table student;" 
	b、恢复数据 mysql -uroot -p'123456' mytest < /mnt/mytest_bak.sql  
	c、查看数据 mysql -uroot -p'123456' -e "select * from mytest.student;"
	
2. -B参数备份和恢复（建议使用）
   备份操作:
   a、备份 mysqldump -uroot -p'123456' -B mytest > /mnt/mytest_bak_B.sql
```

**b.  备份并忽略其中某个表**

```mysql
mysqldump -uusername -ppassword -h192.168.0.1 -P3306 dbname --ignore-table=dbname.dbtanles > dump.sql  
```

说明：加了-B参数后，备份文件中多的Create database和use mytest的命令 加-B参数的好处： 加上-B参数后，导出的数据文件中已存在创建库和使用库的语句，不需要手动在原库是创建库的操作，在恢复过程中不需要手动建库，可以直接还原恢复。

```mysql
1 恢复操作 a、删除mytest库 mysql -uroot -p'123456' -e "drop database mytest;" b、恢复数据 

2 使用不带参数的导出文件导入（导入时不指定要恢复的数据库），报错 mysql -uroot - p'123456' < /mnt/mytest_bak.sql    ERROR 1046 (3D000) at line 22: No database selected 

3 使用带-B参数的导出文件导入（导入时也不指定要恢复的数据库），成功 mysql -uroot -p'123456' < /mnt/mytest_bak_B.sql 

4 通过二进制文件恢复数据
```

```mysql
mysqlbinlog mysql-bin.000002 | mysql -uroot -p111111
```

通过二进制文件重开始点787到结束点1668的数据进行恢复

```mysql
mysqlbinlog mysql-bin.000002 --start-position=787   --stop-position=1668|mysql -uroot -p111111
# 查看数据 
mysql -uroot -p'123456' -e "select * from mytest.student;"
```

**（1）--compact参数优化备份文小大小，减少输出注释（一般用于Debug调试）**

```mysql
（1）备份 mysqldump -uroot -p'123456' --compact -B mytest > /mnt/mytest_bak_Compact.sql 
# 说明： 使用--compact参数，可以优化输出内容的大小，让容量更少，适合调试。便会忽略--skip-add-drop-table，--no-set-names，--skip-disable-keys，--skip-add-locks等几个参数的功能。
```

**（2）指定压缩命令来压缩备份文件**

```mysql
（1）备份 

mysqldump -uroot -p'123456'  -B mytest | gzip > /mnt/mytest_bak_.sql.gz

（2）还原 

gunzip < /mnt/mytest_bak_.sql.gz | mysqldump -uroot -p'123456' 

# 如果备份未加-B参数 还原时则必须指定数据库名

gunzip < /mnt/mytest_bak_.sql.gz | mysqldump -uroot -p'123456' databasename 

# 说明： mysqldump导出的文件是文本文件，压缩效率很高
```

**（3）备份多个数据库**

```mysql
（1）说明 通过-B参数指定相关数据库，每个数据库名之前用空格分格。当使用-B参数后，将所有数据库全部列全，则此时等同于-A参数。 （2）备份 mysqldump -uroot -p'123456' -B mytest wiki | gzip > /mnt/mytestAndWiki_bak.sql.gz
```

**（4）分库备份**

　　分库备份实际上就是执行一个备份语句就备份一个库，有多个库时，就执行多条相同的备份语句，只是备份的库名和备份文件名不同而已。可能通过shell脚本自动生成并执行相应的操作，也可以把所有单个备份语句写在一个shell脚本中，通过cron定时任务来备份。

分库备份的意义是在所有库都备份成一个备份文件时，恢复其中一个库的数据是比较麻烦的，所以分库备份，利于恢复。分库备份脚本如下：

```mysql
for dbname in ` mysql -uroot -p'123456' -e "show databases;" | grep -Evi "database|infor|perfor"` do    mysqldump -uroot -p"123456" --events -B $dbname | gzip > /mnt/${dbname}_bak.sql.gz done
```

说明：${dbname}_bak，由于要求备份文件名以$dbname_bak.sql.gz格式命令，但系统无法辨别变量是$dbname还是$dbname_bak，所以此时就需要用大括号“{}”将变量括起来，就是${dbname}_bak.sql.gz了。

**（5）-d参数，只备份数据库中表结构**

```mysql
mysqldump -uroot -p'123456' -d mytest > /mnt/mytestDesc_bak.sql
```

**（6）-A参数备份全库，并且-F刷新和切换binlog**

```mysql
mysqldump -uroot -p'123456' -A -B -F > /mnt/All_bak.sql
```

**（7）--master-data参数在备份文件中写入当前binlog文件号**

```mysql
mysqldump -uroot -p'123456' --master-data=1 --compact mytest > /mnt/All_bak.sql mysqldump -uroot -p'123456' --master-data=2 --compact mytest > /mnt/All_bak.sql
```



**二、备份单个表**

```mysql
语法（Syntax）：
不能加-B参数 mysqldump -u<username> -p<password> dbname tablename1 tablename2... > /path/to/*.sql
```

示例：

```mysql
示例1：备份mytest库中的student表 mysqldump -uroot -p'123456' mytest student > /mnt/table_bak/student_bak.sql 

示例2：备份mytest库中所有表，就是备份mytest库 mysqldump -uroot -p'123456' mytest  > /mnt/table_bak/all_bak.sql 

示例3：备份mytest库中的student和test表 mysqldump -uroot -p'123456' mytest student test > /mnt/table_bak/two_bak.sql 

示例4：-d参数，只备份表结构 mysqldump -uroot -p'123456' -d mytest stusent > /mnt/studentDesc_bak.sql 

示例5：-t参数，只备份数据 mysqldump -uroot -p'123456' --compact -t mytest stusent > /mnt/studentData_bak.sql INSERT INTO `student` VALUES (1,'Tom',20,'S11'),(2,'Jary',21,'S12'),(3,'King',25,'S10'),(4,'Smith',19,'S11'),(5,'??',20,'S11'),(6,'张三',20,'S11');
```



**三、企业生产场景不同引擎备份命令参数**

**1**、mysqldump的关键参数

```mysql
-B：指定多个库，在备份文件中增加建库语句和use语句 --compact：去掉备份文件中的注释，适合调试，生产场景不用 
-A：备份所有库 
-F：刷新binlog日志 --master-data：在备份文件中增加binlog日志文件名及对应的位置点 
-x  --lock-all-tables：锁表 
-l：只读锁表 
-d：只备份表结构 
-t：只备份数据 --single-transaction：适合innodb事务数据库的备份  InnoDB表在备份时，通常启用选项--single-transaction来保证备份的一致性，原理是设定本次会话的隔离级别为Repeatable read，来保证本次会话（也就是dump）时，不会看到其它会话已经提交了的数据。
```

**2、不同引擎备份命令参数用法**

```mysql
（1）Myisam引擎： mysqldump -uroot -p123456 -A -B --master-data=1 -x| gzip > /data/all_$(date +%F).sql.gz 

（2）InnoDB引擎： mysqldump -uroot -p123456 -A -B  --master-data=1 --single-transaction > /data/bak.sql 

（3）生产环境DBA给出的命令 a、for MyISAM mysqldump --user=root --all-databases --flush-privileges --lock-all-tables \ --master-data=1 --flush-logs --triggers --routines --events \ --hex-blob > $BACKUP_DIR/full_dump_$BACKUP_TIMESTAMP.sql b、for InnoDB mysqldump --user=root --all-databases --flush-privileges --single-transaction \ --master-data=1 --flush-logs --triggers --routines --events \ --hex-blob > $BACKUP_DIR/full_dump_$BACKUP_TIMESTAMP.sql
```

**将本地导出的数据库导入到远程主机：**

```mysql
mysql -h gz-cdb-1728jq2fex.sql.tencentcdb.com -P 62360 -u root -p < demo.sql
```

**将远程服务器的数据库拷到本地/复制他人数据库**

```mysql
mysqldump -h '114.212.111.123' -uTHATUSER -pTHATPWD --opt --compress THATDB --skip-lock-tables | mysql -h localhost -uMYUSER -pMYPWD MYDB
```

解释： 

```txt
   114.212.111.123 远程服务器名称 
   THATUSER 远程数据库登录名 
   THATPWD 远程数据库登录密码 
   THATDB远程数据库名（即：复制的源） 
   localhost 本地数据库名称 
   MYUSER 本地数据库登录名 
   MYUSER 本地数据库登录密码 
   MYDB 本地（即：复制的目的） 
   sql解释： 
   mysqldump 是mysql的一个专门用于拷贝操作的命令 
   –opt 操作的意思 
   –compress 压缩要传输的数据 
   –skip-lock 忽略锁住的表（加上这句能防止当表有外键时的报错） 
   -tables 某数据库所有表 
   -h 服务器名称 
   -u 用户名（*后面无空格，直接加用户名） 
   -p 密码（*后面无空格，直接加密码,这是必须的） 
   注意： 
   -u、-p的后面没有空格，直接加用户名和密码！！！
```

———————————————-库操作———————————————-

1.①导出一个库结构

```mysql
mysqldump -d dbname -u root -p > xxx.sql
```

②导出多个库结构

```mysql
mysqldump -d -B dbname1 dbname2 -u root -p > xxx.sql
```

 2.①导出一个库数据

```mysql
mysqldump -t dbname -u root -p > xxx.sql
```

②导出多个库数据

```mysql
mysqldump -t -B dbname1 dbname2 -u root -p > xxx.sql
```

 3.①导出一个库结构以及数据

```mysql
mysqldump dbname1 -u root -p > xxx.sql
```

②导出多个库结构以及数据

```mysql
mysqldump -B dbname1 dbname2 -u root -p > xxx.sql
```

———————————————-表操作———————————————-

4.①导出一个表结构

```mysql
mysqldump -d dbname1 tablename1 -u root -p > xxx.sql
```

②导出多个表结构

```mysql
mysqldump -d  dbname1 --tables tablename1 tablename2 -u root -p > xxx.sql
```

 5.①导出一个表数据

```mysql
mysqldump -t dbname1 tablename1 -u root -p > xxx.sql
```

②导出多个表数据

```mysql
mysqldump -t  dbname1 --tables tablename1 tablename2 -u root -p > xxx.sql
```

 6.①导出一个表结构以及数据

```mysql
mysqldump dbname1 tablename1 -u root -p > xxx.sql
```

②导出多个表结构以及数据

```mysql
mysqldump  dbname1 --tables tablename1 tablename2 -u root -p > xxx.sql
```

————————————–存储过程&函数操作————————————-

7.只导出存储过程和函数(不导出结构和数据，要同时导出结构的话，需要同时使用-d)

```mysql
mysqldump -R -ndt dbname1 -u root -p > xxx.sql
```

———————————————-事件操作———————————————-

8.只导出事件

```mysql
mysqldump -E -ndt dbname1 -u root -p > xxx.sql
```

—————————————–触发器操作——————————————–

9.不导出触发器（触发器是默认导出的–triggers，使用–skip-triggers屏蔽导出触发器）

```mysql
mysqldump --skip-triggers dbname1 -u root -p > xxx.sql
```

————————————————————————————————

10.导入

source xxx.sql

————————————————————————————————

总结一下：

```mysql
-d 结构(--no-data:不导出任何数据，只导出数据库表结构)

-t 数据(--no-create-info:只导出数据，而不添加CREATE TABLE 语句)

-n (--no-create-db:只导出数据，而不添加CREATE DATABASE 语句）

-R (--routines:导出存储过程以及自定义函数)

-E (--events:导出事件)

--triggers (默认导出触发器，使用--skip-triggers屏蔽导出)

-B (--databases:导出数据库列表，单个库时可省略）

--tables 表列表（单个表时可省略）

①同时导出结构以及数据时可同时省略-d和-t

②同时 不 导出结构和数据可使用-ntd

③只导出存储过程和函数可使用-R -ntd
```

**④导出所有(结构&数据&存储过程&函数&事件&触发器)使用-R -E(相当于①，省略了-d -t;触发器默认导出)**

```mysql
mysqldump -u root -p  -R -E -B dbname1  > xxx.sql
```

**⑤只导出结构&函数&事件&触发器使用 -R -E -d**

8.查看数据库使用的引擎，保证导出数据一致性。（针对InnoDB引擎才可以使用 **--single-transaction参数**）

```mysql
mysqldump -u root -p --set-gtid-purged=OFF --single-transaction  -R -E -B dbname1  > xxx.sql
```

**查看数据库引擎**

```mysql
show variables like '%storage_engine%';
```

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\493bff9114ef434e975340016743d807\e357f4c92bd062384bc4aa5605fd887.png)

----------------------------------------------------------------------------------------------------------------------------

**通过二进制文件恢复数据**

```mysql
mysqlbinlog mysql-bin.000002 --start-position=787   --stop-position=1668|mysql -uroot -p111111
```

**查看经过加密过的数据**

```mysql
mysqlbinlog  --base64-output=DECODE-ROWS -v binlog_mysqlbin.000240    --start-datetime="2019-9-11 09:32:00" --stop-datetime="2019-9-11 09:38:00"
```

常用参数选项解释：

```mysql
--start-position=875 #起始pos点

--stop-position=954 #结束pos点

--start-datetime="2016-9-25 22:01:08" #起始时间点

--stop-datetime="2019-9-25 22:09:46" #结束时间点

--database=zyyshop #指定只恢复zyyshop数据库(一台主机上往往有多个数据库，只限本地log日志)
```

**远程主机完整备份：**

```mysql
mysqldump -uroot -p -P 1638 -h 21.22.71.23 --set-gtid-purged=OFF --single-transaction  -R -E -B xg_foms > xg_foms.sql
```

\---------------------------------------------------------------------------------------------------------

**mysqldump全量备份+mysqlbinlog二进制日志增量备份**

```mysql
mysqldump -uroot -p -P 1000 -h 192.168.1.1 --set-gtid-purged=OFF --single-transaction  --flush-logs --master-data=2 -R -E -B fomsv4 > fomsv4.sql
```

导入全量和增量数据

```mysql
mysqldump -uroot -p < fomsv4.sql
```

在生成下新的binglog日志文件（此时新文件为mysql-bin.002322）

```mysql
flush logs
```

我们恢复binglog日志为（mysql-bin.002321）

下图记录了新增binglog日志文件，恢复binglog日志到数据库即可

```mysql
mysqlbinlog mysql-bin.002321 | mysql -uroot -p111111
```

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\f0b95217c1984693bb2f0e4b2e1a4ffa\clipboard.png)

\-------------------------------------------------------------------------------------------------------------------------------------

将binglog日志转成sql文件

```mysql
mysqlbinlog   mysql-bin.000011   > fomsv4.sql
```

