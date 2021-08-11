【**MySQL修改root密码的多种方法**】

```MYSQL
方法1： 用SET PASSWORD命令

　　mysql -u root

　　mysql> SET PASSWORD FOR 'root'@'localhost' = PASSWORD('newpass');

方法3：用ALTER

		ALTER USER 'dog'@'localhost' IDENTIFIED BY '123456';

方法4：用mysqladmin

　　mysqladmin -u root password "newpass"

　　如果root已经设置过密码，采用如下方法

　　mysqladmin -u root -p'oldpass' password 'newpass'

方法4： 用UPDATE直接编辑user表

　　mysql -u root

　　mysql> use mysql;

　　mysql> UPDATE user SET Password = PASSWORD('newpass') WHERE user = 'root';

　　mysql> FLUSH PRIVILEGES;

 在丢失root密码的时候，可以这样

　　mysqld_safe --skip-grant-tables&

　　mysql -u root mysql

　　mysql> UPDATE user SET password=PASSWORD("new password") WHERE user='root';

　　mysql> FLUSH PRIVILEGES;
```

\-------------------------------------------------------------------------------------------------------

```mysql
mysql 5.7 更新密码

mysql> use mysql;

mysql> update user set authentication_string=password('newpass') where user='root' and Host='localhost';

mysql> FLUSH PRIVILEGES;

更改密码：

alter user 'root'@'localhost' identified by 'passwd';
```

