当Innodb 表数据频繁 update ,更新的数据会重新放置，旧数据会形成空洞，随着时间的推移，空洞会越来越大。

可以通过 information_schema.table 表查看数据空洞过大的表

SQL如下：

SELECT table_schema,TABLE_NAME , concat(data_free/**1024**/**1024**,"M") FROM `information_schema`.tables WHERE data_free >**8*****1024*****1024** AND ENGINE ='innodb'  ORDER BY data_free DESC;

数据空洞过大，会影响SQL的执行速度， 要彻底解决空洞问题需要从 update 语句入手，确定更新是否有意义， 此外通过 

 **回收表空间**

 ALTER TABLE table_name ENGINE = Innodb;

**查看表碎片信息**

SHOW TABLE STATUS LIKE 't_operationcpl'

**-- 查询数据库表碎片空间大于8M的表**

SELECT table_schema,TABLE_NAME , concat(data_free/1024/1024,"M") FROM `information_schema`.tables WHERE data_free >8*1024*1024 AND ENGINE ='innodb'  ORDER BY data_free DESC;

SELECT * from `information_schema`.tables