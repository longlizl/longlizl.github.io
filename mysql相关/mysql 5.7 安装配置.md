**一. 安装mysql5.7**

1.yum源安装mysql

```shell
vim /etc/yum.repos.d/mysql-community.repo

# Enable to use MySQL 5.7

[mysql57-community]

name=MySQL 5.7 Community Server

baseurl=http://repo.mysql.com/yum/mysql-5.7-community/el/7/$basearch/

enabled=1

gpgcheck=1

gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql
```

2.导入gpg-key(可以将gpgcheck=0不用验证gpgkey则省略此步骤)

```shell
cat << EOF > /etc/pki/rpm-gpg/RPM-GPG-KEY-mysql

-----BEGIN PGP PUBLIC KEY BLOCK-----

Version: PGP Universal 2.9.1 (Build 347)

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

=Eto8

-----END PGP PUBLIC KEY BLOCK-----

EOF
```

**yum -y install mysql-community-server**

**3.修改my.cnf配置参数（此配置需要安装后启动初始后再配置不然会报错）**

```mysql
character-set-server=utf8
default-time_zone='+8:00'
validate_password=off #密码使用策略关闭，默认开启。
log_timestamps=system   #默认此值为UTC比中国时区慢8小时，mysqld.log日志时间相差8小时，设置system即可。
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION  #sql模式
```

**4.启动mysql服务**

```shell
systemctl start mysqld

systemctl enable mysqld
```

**5.mysql5.7默认启动后会生成一个随机密码，需要修改密码后进行数据库操作**

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\d744cf938e4d42228c1bff8300e5d787\clipboard.png)

```mysql
mysqladmin -u root -p'3!atNVzjr1la' password 'Password@1'
```

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\310422c49a304df19ec188d23d11a79d\clipboard.png)

**6.查看密码策略**

```mysql
show global VARIABLES LIKE 'validate_password%';
```

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\7148e96548974c71a3aa7f6c14a57f88\clipboard.png)

**7.关闭密码复杂度（动态修改）**

```mysql
set global validate_password_policy=0;
```

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\0e36267b2d9a4e0690ae25fadc3b320b\clipboard.png)

**8.修改密码**

```mysql
alter user 'root'@'localhost' identified by 'password';
```

**二.授权:**

```mysql
命令:GRANT ALL privileges ON databasename.tablename TO 'username'@'host' WITH GRANT OPTION

PS: privileges - 用户的操作权限,如SELECT , INSERT , UPDATE 等(详细列表见该文最后面).如果要授予所的权限则使用ALL.;databasename - 数据库名,tablename-表名,如果要授予该用户对所有数据库和表的相应操作权限则可用*表示, 如*.*

例子: GRANT SELECT, INSERT ON mq.* TO 'dog'@'localhost';
```

**三.创建用户同时授权**

```mysql
mysql> grant all privileges on mq.* to test@localhost identified by '1234';

Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> flush privileges;

Query OK, 0 rows affected (0.01 sec)

PS:必须执行**flush privileges;** 
```

