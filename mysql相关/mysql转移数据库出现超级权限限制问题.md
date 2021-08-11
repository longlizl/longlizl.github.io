将导出的数据库文件导入到另一台服务器数据库权限问题解决方法

```mysql
[root@iZbp165w3l5pvp5iuqibtdZ ~]# mysql -h gz-cdb-39jqhzex.sql.tcentcdb.com -P 61860 -u root -p < demo.sql 

Enter password: 

ERROR 1227 (42000) at line 5898: Access denied; you need (at least one of) the SUPER privilege(s) for this operation
```

```mysql
以上错误是由于备份的文件中有DEFINER=`root`@`localhost `导致的由于是要导入的数据库平台是云平台对本机root权限有限制。把备份文件中有这个语句的行全部替换为DEFINER=`root`@`%`保存即可

5987 CREATE DEFINER=`root`@`localhost` PROCEDURE `pro_battery_charge_discharge`(in in_start_time varchar(64),in in_end_time varchar(64))

使用vim编辑器打开备份文件输入：%s/DEFINER=`root`@`localhost `/DEFINER=`root`@`%`/g
```

