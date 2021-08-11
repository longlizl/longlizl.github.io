**mysql 数据库 修改默认时区**

1.mysql数据库创建后。默认的时区比东八区少了八个小时。如果sql语句中使用到mysql的时间的话就会比正常时间少了八个小时。所以需要修改mysql的系统时区。

select now(); 查看mysql系统时间。和当前时间做对比

SHOW GLOBAL VARIABLES LIKE '%time_zone%'

set global time_zone = '+8:00';设置时区更改为东八区

flush privileges; 刷新权限

然后退出后重新登录就可以了，显示当前时间和我现在的时间一致了。

2.配置文件中添加下面一行

在mysqld下边的配置中添加一行：

default-time_zone = '+8:00'