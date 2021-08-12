```mysql
MySQL5.7 mysql.user表没有password字段改 authentication_string；
```

1. **创建用户:**

```mysql
命令: CREATE USER 'username'@'host' IDENTIFIED BY 'password';

例子: CREATE USER 'test'@'localhost' IDENTIFIED BY '123456';

      CREATE USER 'test1'@'localhost' IDENTIFIED BY '';
```

```
exp：username - 你将创建的用户名,
host - 指定该用户在哪个主机上可以登陆，此处的"localhost"，是指该用户只能在本地登录，不能在另外一台机器上远程登录，如果想远程登录的话，将"localhost"改为"%"，表示在任何一台电脑上都可以登录;也可以指定某台机器可以远程登录;

password - 该用户的登陆密码,密码可以为空,如果为空则该用户可以不需要密码登陆服务器。
```

2. **授权:**

```mysql
命令:GRANT ALL privileges ON databasename.tablename TO 'username'@'host' WITH GRANT OPTION
exp: privileges - 用户的操作权限,如SELECT , INSERT , UPDATE 等(详细列表见该文最后面).如果要授予所的权限则使用ALL.;databasename - 数据库名,tablename-表名,如果要授予该用户对所有数据库和表的相应操作权限则可用*表示, 如*.*
例子: GRANT SELECT, INSERT ON mq.* TO 'dog'@'localhost';
```

3. **创建用户同时授权**

```mysql
mysql> grant all privileges on mq.* to test@localhost identified by '1234';

Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> flush privileges;

Query OK, 0 rows affected (0.01 sec)

exp:必须执行 flush privileges;
否则登录时提示：ERROR 1045 (28000): Access denied for user 'user'@'localhost' (using password: YES ) 
```

4. **设置与更改用户密码**

```mysql
1.命令:SET PASSWORD FOR 'username'@'host' = PASSWORD('newpassword');

例子: SET PASSWORD FOR 'dog2'@'localhost' = PASSWORD("dog");

2.alter user 'root'@'localhost' identified by 'Password@1'

```

5. **撤销用户权限**

```mysql
命令: REVOKE all ON databasename.tablename FROM 'username'@'host';
说明: privilege, databasename, tablename - 同授权部分.
例子: REVOKE SELECT ON mq.* FROM 'dog2'@'localhost';

exp: 假如你在给用户'dog'@'localhost'
GRANT SELECT ON test.user TO 'dog'@'localhost', 则在使用REVOKE SELECT ON *.* FROM 'dog'@'localhost';命令并不能撤销该用户对test数据库中user表的SELECT 操作.
相反,如果授权使用的是GRANT SELECT ON *.* TO 'dog'@'localhost';则REVOKE SELECT ON test.user FROM 'dog'@'localhost';命令也不能撤销该用户对test数据库中user表的Select 权限.
具体信息可以用命令SHOW GRANTS FOR 'dog'@'localhost'; 查看.
```

6. **删除用户**

```mysql
DROP USER 'username'@'host';
```

7. **查看用户的授权**

```mysql
mysql> show grants for dog@localhost;

+---------------------------------------------+

| Grants for dog@localhost |

+---------------------------------------------+

| GRANT USAGE ON *.* TO 'dog'@'localhost' |

| GRANT INSERT ON `mq`.* TO 'dog'@'localhost' |

+---------------------------------------------+

2 rows in set (0.00 sec)

PS:GRANT USAGE:mysql usage权限就是空权限，默认create user的权限，只能连库，啥也不能干
```

8. **创建库并设置默认编码：**

```mysql
CREATE DATABASE `mydb` CHARACTER SET utf8 COLLATE utf8_general_ci;
```

9. **某用户csuser无法查询到存储过程数据需要给mysql库授权**

```mysql
grant select on mysql.* TO 'csuser'@'%';
```

10. **删除数据库中所有表：**

```mysql
SELECT CONCAT('drop table ',table_name,';') FROM information_schema.`TABLES` WHERE table_schema='jumpserver';
```

