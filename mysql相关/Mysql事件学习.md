Mysql事件学习

在系统管理或者数据库管理中，经常要周期性的执行某一个命令或者SQL语句。对于linux系统熟悉的人都知道linux的cron计划任务，能很方便地实现定期运行指定命令的功能。Mysql在5.1以后推出了事件调度器(Event Scheduler)，和linux的cron功能一样，能方便地实现 mysql数据库的计划任务，而且能精确到秒。使用起来非常简单和方便。

由于最近需要用到事件这个功能，因此学习了一下，感觉非常棒，总结一下，方便以后使用，也希望能对其他的初学者有帮助。

一、  如果开启事件

在使用事件这个功能，首先要保证你的mysql的版本是5.1以上，然后还要查看你的mysql服务器上的事件是否开启。

查看事件是否开启，使用如下命令查看：

SHOW VARIABLES LIKE 'event_scheduler';

SELECT @@event_scheduler;

SHOW PROCESSLIST;

如果看到event_scheduler为on或者PROCESSLIST中显示有event_scheduler的信息说明就已经开启了事件。如果显示为off或者在PROCESSLIST中查看不到event_scheduler的信息，那么就说明事件没有开启，我们需要开启它。

开启mysql的事件，通过如下三种方式开启：

Ø 通过动态参数修改

SET GLOBAL event_scheduler = ON;

更改完这个参数就立刻生效了

注意：还是要在my.cnf中添加event_scheduler=ON。因为如果没有添加的话，mysql重启事件又会回到原来的状态了。

Ø 更改配置文件然后重启

在my.cnf中的[mysqld]部分添加如下内容，然后重启mysql。

event_scheduler=ON

Ø 通过制定事件参数启动

mysqld ... --event_scheduler=ON

 

二、  Mysql事件的语法简介

1. 创建事件的语法

CREATE

  [DEFINER = { user | CURRENT_USER }]

  EVENT

  [IF NOT EXISTS]

  event_name

  ON SCHEDULE schedule

  [ON COMPLETION [NOT] PRESERVE]

  [ENABLE | DISABLE | DISABLE ON SLAVE]

  [COMMENT 'comment']

  DO event_body;

 

schedule:

  AT timestamp [+ INTERVAL interval] ...

   | EVERY interval

  [STARTS timestamp [+ INTERVAL interval] ...]

  [ENDS timestamp [+ INTERVAL interval] ...]

interval:

 quantity {YEAR | QUARTER | MONTH | DAY | HOUR | MINUTE |

​       WEEK | SECOND | YEAR_MONTH | DAY_HOUR |

DAY_MINUTE |DAY_SECOND | HOUR_MINUTE |

HOUR_SECOND | MINUTE_SECOND}

参数详细说明：

DEFINER: 定义事件执行的时候检查权限的用户。

ON SCHEDULE schedule: 定义执行的时间和时间间隔。

ON COMPLETION [NOT] PRESERVE: 定义事件是一次执行还是永久执行，默认为一次执行，即NOT PRESERVE。

ENABLE | DISABLE | DISABLE ON SLAVE: 定义事件创建以后是开启还是关闭，以及在从上关闭。如果是从服务器自动同步主上的创建事件的语句的话，会自动加上DISABLE ON SLAVE。

COMMENT 'comment': 定义事件的注释。

 

\2.    更改事件的语法

ALTER

  [DEFINER = { user | CURRENT_USER }]

  EVENT event_name

  [ON SCHEDULE schedule]

  [ON COMPLETION [NOT] PRESERVE]

  [RENAME TO new_event_name]

  [ENABLE | DISABLE | DISABLE ON SLAVE]

  [COMMENT 'comment']

  [DO event_body]

\3.    删除事件的语法

DROP EVENT [IF EXISTS] event_name

三、  Mysql事件实战

\1.     测试环境

创建一个用于测试的test表：

CREATE TABLE `test` (

 `id` int(11) NOT NULL AUTO_INCREMENT,

 `t1` datetime DEFAULT NULL,

 `id2` int(11) NOT NULL DEFAULT '0',

 PRIMARY KEY (`id`)

) ENGINE=InnoDB AUTO_INCREMENT=106 DEFAULT CHARSET=utf8

\2.     实战1

Ø 创建一个每隔3秒往test表中插入一条数据的事件，代码如下：

CREATE EVENT IF NOT EXISTS test ON SCHEDULE EVERY 3 SECOND

ON COMPLETION PRESERVE

DO INSERT INTO test(id,t1) VALUES('',NOW());

Ø 创建一个10分钟后清空test表数据的事件

CREATE EVENT IF NOT EXISTS test

ON SCHEDULE

AT CURRENT_TIMESTAMP + INTERVAL 1 MINUTE

DO TRUNCATE TABLE test.aaa;

Ø 创建一个在2012-08-23 00:00:00时刻清空test表数据的事件，代码如下：

CREATE EVENT IF NOT EXISTS test

ON SCHEDULE

AT TIMESTAMP '2012-08-23 00:00:00'

DO TRUNCATE TABLE test;

Ø 创建一个从2012年8月22日21点45分开始到10分钟后结束，运行每隔3秒往test表中插入一条数据的事件，代码如下：

CREATE EVENT IF NOT EXISTS test ON SCHEDULE EVERY 3 SECOND

STARTS '2012-08-22 21:49:00' 

ENDS '2012-08-22 21:49:00'+ INTERVAL 10 MINUTE

ON COMPLETION PRESERVE

DO INSERT INTO test(id,t1) VALUES('',NOW());

 

\3.     实战2

通常的应用场景是通过事件来定期的调用存储过程，下面是一个简单的示例：

创建一个让test表的id2字段每行加基数2的存储过程，存储过程代码如下：

DROP PROCEDURE IF EXISTS test_add;

DELIMITER //

CREATE PROCEDURE test_add()

BEGIN

DECLARE 1_id INT DEFAULT 1;

DECLARE 1_id2 INT DEFAULT 0;

DECLARE error_status INT DEFAULT 0;

DECLARE datas CURSOR FOR SELECT id FROM test;

DECLARE CONTINUE HANDLER FOR NOT FOUND SET error_status=1;

OPEN datas;

FETCH datas INTO 1_id;

REPEAT

SET 1_id2=1_id2+2;

UPDATE test SET id2=1_id2 WHERE id=1_id;

FETCH datas INTO 1_id;

UNTIL error_status

END REPEAT;

CLOSE datas;

END

//

事件设置2012-08-22 00:00:00时刻开始运行，每隔1调用一次存储过程，40天后结束，代码如下：

CREATE EVENT test ON SCHEDULE EVERY 1 DAY

STARTS '2012-08-22 00:00:00'

ENDS '2012-08-22 00:00:00'+INTERVAL 40 DAY

ON COMPLETION PRESERVE DO

CALL test_add();