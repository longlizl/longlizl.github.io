### 1.  整理某个数据库中表碎片空间大于8M的表

```shell
# 清理某个数据库中表碎片空间大于8M的表
#!/bin/bash
mysql_datables=shelf_tilt
mysql_user=root
mysql_pd=123123
mysql_host=11.207.11.11
mysql_port=3306
sql='SELECT table_schema,TABLE_NAME , concat(data_free/1024/1024,"M") FROM `information_schema`.tables WHERE data_free >8*1024*1024 AND ENGINE ="innodb" and table_schema="shelf_tilt" ORDER BY data_free DESC;'
tables_info=/root/shelf_tilt_tables.txt
table=/root/tables.txt
mysql -h $mysql_host -P$mysql_port -u$mysql_user -p$mysql_pd -e "$sql" > $tables_info
cat $tables_info | awk '{print $2}' | sed -n '2,$'p > /root/tables.txt
cat $table | while read line 
do
        mysql -h $mysql_host -P$mysql_port -u$mysql_user -p$mysql_pd $mysql_datables -e "ALTER TABLE $line ENGINE = Innodb;" > /dev/null 2>&1
done
```

### 2.  整理所有库表空间碎片

```shell
#!/bin/bash
mysql_user=root
mysql_pd=123@123t
mysql_host=11.11.11.11
mysql_port=3306
databases=/root/show_databases.txt
table=/root/tables.txt
mysql -h $mysql_host -P$mysql_port -u$mysql_user -p$mysql_pd -e 'show databases' | egrep -v 'Database|information_schema|performance_schema|mysql' > $databases
cat $databases | while read db
do
        mysql -h $mysql_host -P$mysql_port -u$mysql_user -p$mysql_pd $db -e "show tables" | grep -v 'Tables' > $table
        cat $table | while read tb
        do
                mysql -h $mysql_host -P$mysql_port -u$mysql_user -p$mysql_pd $db -e "ALTER TABLE $tb ENGINE = Innodb;" > /dev/null 2>&1
        done
done
```

