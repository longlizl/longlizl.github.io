遇到这种情况，如现有项目的数据库已经建好，数据表也已经创建完成。

问题来的，数据库不能插入中文，调试时候发现中文数据从发送请求到最后请求处理完成这些步骤，中文还没有发生乱码。

只有在存储到数据库后查询数据并显示才会乱码。

那么到此可以确定是mysql出现问题了。

将数据库的编码修改为utf8编码格式，因为安装mysql默认使用的字符编码latin1

what?这个编码是什么鬼，见都没见过。

查了下，Latin1是[ISO-8859-1](https://baike.baidu.com/item/ISO-8859-1)的别名。因为ISO-8859-1编码范围使用了单[字节](https://baike.baidu.com/item/字节)内的所有空间，在支持ISO-8859-1的系统中传输和存储其他任何编码的[字节流](https://baike.baidu.com/item/字节流)都不会被抛弃

这个latin1编码是单字节编码，而一般汉字是需要两个字节存储，所以这个编码格式不支持汉字。

 

接下来就看怎么改吧

数据库字符优先级有:系统级、数据库级、表级、字段。这5个优先级中字段优先级最高。举个列子。我们要向表中存储中文数据。如果表的字符编码是utf8，而字段的字符编码是latin1。那么如果我们存储中文还是会出现乱码，因为使用的编码是字段的字符编码latin1

在数据库创建时如果不设置数据库的默认字符编码，即缺省时会使用系统的字符编码latin1。创建表缺省时使用数据库的字符编码，字段同理。

从上面可以得出一个重要结论，创建数据库时一定要指定默认字符编码！

1、

首先我们查看mysql数据库服务器，客户端，数据库连接，文件系统等的字符编码

show variables like '%char%';

+--------------------------+-------------------------------------+------

| Variable_name | Value |......

+--------------------------+-------------------------------------+------

| character_set_client | utf8 |...... -- 客户端字符集

| character_set_connection | utf8 |......

| character_set_database | latin1|...... -- 数据库字符集

| character_set_filesystem | binary |......

| character_set_results | utf8 |......

| character_set_server | latin1|...... -- 服务器字符集

| character_set_system | utf8 |......

| character_sets_dir | D:\MySQL Server 5.0\share\charsets\ |......

+--------------------------+-------------------------------------+------

默认安装完mysql数据库时，服务器和数据库使用的编码是latinn1编码格式。

这个上面没有问题，可以先不用考虑

网上看到很多人要修改这个系统级的字符编码，我也不知道什么原因，改完之后中文乱码问题就得到解决了

2、

查看数据库的创建语句。以默认的数据库mysql为例

show create database mysql

可以发现其的default character set默认字符编码格式是latin1。由此可以看出，缺省时会自动使用latin1

所以我们需要在创建数据库时指定默认的编码格式

create database 数据库名 CHARACTER SET utf8 COLLATE utf8_general_ci;

修改已创建的数据库的编码

alter database 数据库名 CHARACTER SET utf8 COLLATE utf8_general_ci;

3、

查看数据库中匹配到的表的编码格式。

show table status from 数据库名 like 'pattern/匹配模式';

查看数据库中所有表的编码格式

show table status from mysql like '%%';

4、

修改表的默认编码格式。有两种方法

- alter table 表名 character set utf8 COLLATE utf8_general_ci;
- alter table 表名 convert to character set utf8;（表的字符编码，字段的字符编码全改变。使用这个方法修改）

第一种是仅仅修改表的字符编码，而字段的字符编码还是latin1编码格式，这种改变没有意义

　　　　可以查看数据表中所有列的字符编码，就可以发现字段的字符编码是否发生改变。

　　　　show full columns from 表名;

第二种会将表和字段的编码都更改为utf8编码格式。

5、

中文乱码问题得到解决的，但是数据库里那么多表，我们总不可能一个表一个表的进行修改吧，所以我们要使用存储过程，

加动态sql语句的方式使用循环自动修改数据库内的所有数据表的编码格式。

 

注意：如果修改完后中文存储还是乱码。将连接数据库的连接后面指定编码格式为utf8

如果还是不行，那么将mysql默认的字符编码进行修改，即下面的characterEncoding

jdbcUrl = jdbc:mysql://主机域名:3306/数据库名?characterEncoding=utf8&useSSL=false&useUnicode=true

useSSL:与服务器进行通信时使用SSL，默认值为“假

参考主要参数，表数据来源[Mysql JDBC Url参数说明](http://elf8848.iteye.com/blog/1684414)

| 参数名称              | 参数说明                                                     | 缺省值 | 最低版本要求 |
| --------------------- | ------------------------------------------------------------ | ------ | ------------ |
| user                  | 数据库用户名（用于连接数据库）                               |        | 所有版本     |
| password              | 用户密码（用于连接数据库）                                   |        | 所有版本     |
| useUnicode            | 是否使用Unicode字符集，如果参数characterEncoding设置为gb2312或gbk，本参数值必须设置为true | false  | 1.1g         |
| characterEncoding     | 当useUnicode设置为true时，指定字符编码。比如可设置为gb2312或gbk | false  | 1.1g         |
| autoReconnect         | 当数据库连接异常中断时，是否自动重新连接？                   | false  | 1.1          |
| autoReconnectForPools | 是否使用针对数据库连接池的重连策略                           | false  | 3.1.3        |
| failOverReadOnly      | 自动重连成功后，连接是否设置为只读？                         | true   | 3.0.12       |
| maxReconnects         | autoReconnect设置为true时，重试连接的次数                    | 3      | 1.1          |
| initialTimeout        | autoReconnect设置为true时，两次重连之间的时间间隔，单位：秒  | 2      | 1.1          |
| connectTimeout        | 和数据库服务器建立socket连接时的超时，单位：毫秒。 0表示永不超时，适用于JDK 1.4及更高版本 | 0      | 3.0.1        |
| socketTimeout         | socket操作（读写）超时，单位：毫秒。 0表示永不超时           |        |              |