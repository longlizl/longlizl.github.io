创建存储过程时

出错信息：

ERROR 1418 (HY000): This function has none of DETERMINISTIC, NO SQL, or READS SQL DATA in its declaration and binary logging is enabled (you *might* want to use the less safe log_bin_trust_function_creators variable)

 

原因：

这是我们开启了bin-log, 我们就必须指定我们的函数是否是

1 DETERMINISTIC 不确定的

2 NO SQL 没有SQl语句，当然也不会修改数据

3 READS SQL DATA 只是读取数据，当然也不会修改数据

4 MODIFIES SQL DATA 要修改数据

5 CONTAINS SQL 包含了SQL语句

其中在function里面，只有 DETERMINISTIC, NO SQL 和 READS SQL DATA 被支持。如果我们开启了 bin-log, 我们就必须为我们的function指定一个参数。

解决方法：

SQL code

mysql> show variables like 'log_bin_trust_function_creators';

+---------------------------------+-------+

| Variable_name          | Value |

+---------------------------------+-------+

| log_bin_trust_function_creators | OFF  |

+---------------------------------+-------+

mysql> set global log_bin_trust_function_creators=**1**;

mysql> show variables like 'log_bin_trust_function_creators';

+---------------------------------+-------+

| Variable_name          | Value |

+---------------------------------+-------+

| log_bin_trust_function_creators | ON  |

+---------------------------------+-------+

这样添加了参数以后，如果mysqld重启，那个参数又会消失，因此记得在my.cnf配置文件中添加：

log_bin_trust_function_creators=1