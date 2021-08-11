**查看源实例row_format 为Fixed 的表名称，请使用如下语句。**

```mysql
SELECT table_schema,table_name,row_format from information_schema.tables where Row_format like '%fix%' AND table_schema NOT IN ('mysql','performance_schema','information_schema');
```

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\3019f5eb1845416e9fe052849b41dbc9\clipboard.png)

修改row_format参数语句：

```mysql
alter table [table_name] row_format=Dynamic;
```

