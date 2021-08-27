**连接mysql并创建插入数据**

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


