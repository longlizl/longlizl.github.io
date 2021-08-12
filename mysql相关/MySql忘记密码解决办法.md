MySql忘记root密码：

* 一、更改my.cnf配置文件

1. 用命令编辑/etc/my.cnf配置文件，即：vim /etc/my.cnf

![img](https://longlizl.github.io/mysql相关/images/mysql1.png)

2. 在[mysqld]下添加skip-grant-tables，然后保存并退出

![img](https://longlizl.github.io/mysql相关/images/mysql2.png)

3. 重启mysql服务：

   ```
   systemctl restart mariadb
   ```

* 二、更改root用户名

1. 重启以后，执行mysql命令进入mysql命令行

![img](https://longlizl.github.io/mysql相关/images/mysql3.png)

2. 修改root用户密码

```mysql
MySQL> UPDATE mysql.user SET Password=PASSWORD('新密码') where USER='root';

MySQL> flush privileges;

MySQL> exit
```

3. 把/etc/my.cnf中的skip-grant-tables注释掉，然后重启mysql

```mysql
# 注MySQL5.7及后续版本,改密码无password字段,password字段改成了authentication_string

update  mysql.user  set authentication_string=password('新密码') where user='root' ;
```

