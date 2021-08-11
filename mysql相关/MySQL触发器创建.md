[**`[**一、MySQL触发器创建：**]()`**]()

**``````** 

**`**１、MySQL触发器的创建语法：**`**

**`**CREATE**　[DEFINER = { 'user' | CURRENT_USER }]`**　

**`**TRIGGER** trigger_name`**

**`trigger_time trigger_event`**

**`**ON** table_name`**

**`**FOR** EACH ROW`**

**`[trigger_order]`**

**`trigger_body`** 

**`**2、MySQL创建语法中的关键词解释：**`**

| **`字段`**          | **`含义`**                                                   | **`可能的值`**                             |
| ------------------- | ------------------------------------------------------------ | ------------------------------------------ |
| **`DEFINER=`**      | **`可选参数，指定创建者，默认为当前登录用户（CURRENT_USER）；该触发器将以此参数指定的用户执行，所以需要考虑权限问题；`** | **`DEFINER='root@%'DEFINER=CURRENT_USER`** |
| **`trigger_name`**  | **`触发器名称，最好由表名+触发事件关键词+触发时间关键词组成；`** | **``````**                                 |
| **`trigger_time`**  | **`触发时间，在某个事件之前还是之后；`**                     | **`BEFORE、AFTER`**                        |
| **`trigger_event`** | **`触发事件，如插入时触发、删除时触发；　　INSERT：插入操作触发器，INSERT、LOAD DATA、REPLACE时触发；　　UPDATE：更新操作触发器，UPDATE操作时触发；　　DELETE：删除操作触发器，DELETE、REPLACE操作时触发；`** | **`INSERT、UPDATE、DELETE`**               |
| **`table_name`**    | **`触发操作时间的表名；`**                                   | **``````**                                 |
| **`trigger_order`** | **`可选参数，如果定义了多个具有相同触发事件和触法时间的触发器时（如：BEFORE UPDATE），默认触发顺序与触发器的创建顺序一致，可以使用此参数来改变它们触发顺序。mysql 5.7.2起开始支持此参数。　　FOLLOWS：当前创建触发器在现有触发器之后激活；　　PRECEDES：当前创建触发器在现有触发器之前激活；`** | **`FOLLOWS、PRECEDES`**                    |
| **`trigger_body`**  | **`触发执行的SQL语句内容，一般以begin开头，end结尾`**        | **`begin .. end`**                         |

**``````**　

**``````** 

**`**３、触发执行语句内容（trigger_body）中的OLD，NEW：**`**

　　**`在trigger_body中，我们可以使用NEW表示将要插入的新行（相当于MS SQL的INSERTED），OLD表示将要删除的旧行（相当于MS SQL的DELETED）。通过OLD，NEW中获取它们的字段内容，方便在触发操作中使用，下面是对应事件是否支持OLD、NEW的对应关系：`**

| **`事件`**   | **`OLD`** | **`NEW`** |
| ------------ | --------- | --------- |
| **`INSERT`** | **`×`**   | **`√`**   |
| **`DELETE`** | **`√`**   | **`×`**   |
| **`UPDATE`** | **`√`**   | **`√`**   |

　　**`由于UPDATE相当于删除旧行（OLD），然后插入新行（NEW），所以UPDATE同时支持OLD、NEW；`**

**``````** 

**`**４、MySQL分隔符（DELIMITER）：**`**

　　**`MySQL默认使用“;”作为分隔符，SQL语句遇到“;”就会提交。而我们的触发器中可能会有多个“;”符，为了防止触发器创建语句过早的提交，我们需要临时修改MySQL分隔符，创建完后，再将分隔符改回来。使用DELIMITER可以修改分隔符，如下：`**

**`DELIMITER $`**

**`... --触发器创建语句；`**

**`$ --提交创建语句；`**

**`DELIMITER ;`** 

**`**二、MySQL触发器创建进阶：**`**

**``````** 

**`**1、MySQL触发器中使用变量：**`**

　　**`MySQL触发器中变量变量前面加'@'，无需定义，可以直接使用：`**

**`-- 变量直接赋值`**

**`**set** @num=999;`**

 **`-- 使用select语句查询出来的数据方式赋值，需要加括号：`**

**`**set** @**name** =(**select** **name** **from** **table**);`** 

**`**2、MySQL触发器中使用if语做条件判断：**`**

**`-- 简单的if语句：`**

**`**set** sex = if (new.sex=1, '男', '女');`**

**``````** 

**`-- 多条件if语句：`**

**`if old.type=1 **then**`**

 **`**update** **table** ...;`**

**`elseif old.type=2 **then**`**

 **`**update** **table** ...;`**

**`**end** if;`**

**`**三、ＭｙSQL查看触发器：**`**

　　**`可以使用“show triggers;”查看触发器。由于MySQL创建的触发器保存在“information_schema库中的triggers表中，所以还可以通过查询此表查看触发器：`**

**`-- 通过information_schema.triggers表查看触发器：`**

**`**select** * **from** information_schema.triggers;`**

**``````** 

**`-- mysql 查看当前数据库的触发器`**

**`show triggers;`**

**``````** 

**`-- mysql 查看指定数据库"aiezu"的触发器`**

**`show triggers **from** aiezu;`** 

**`**四、MySQL删除触发器：**`**

**``````** 

**`**1、可以使用drop trigger删除触发器：**`**

| **`1`** | **`**drop** **trigger** trigger_name;`** |
| ------- | ---------------------------------------- |
|         |                                          |

**``````** 

**`**2、删除前先判断触发器是否存在：**`**

| **`1`** | **`**drop** **trigger** if exists trigger_name`** |
| ------- | ------------------------------------------------- |
|         |                                                   |

**``````** 

**`**五、Msql触发器用法举例：**`**

**``````** 

**`**1、MySQL触发器Insert触发更新同一张表：**`**

　　**`下面我们有一个表“tmp1”，tmp1表有两个整型字段：n1、n2。我们要通过触发器实现，在tmp插入记录时，自动将n2字段的值设置为n1字段的5倍。`**

　**`创建测试表和触发器：`**

**`-- 创建测试表`**

**`**drop** **table** if exists tmp1;`**

**`**create** **table** tmp1 (n1 **int**, n2 **int**);`**

**``````** 

**`-- 创建触发器`**

**`DELIMITER $`**

**`**drop** **trigger** if exists tmp1_insert$`**

**`**create** **trigger** tmp1_insert`**

**`before **insert** **on** tmp1`**

**`**for** each row`**

**`**begin**`**

 **`**set** new.n2 = new.n1*5;`**

**`**end**$`**

**`DELIMITER ;`**

**`测试触发更新效果：`**

**`mysql> **insert** tmp1(n1) **values**(18);`**

**`Query OK, 1 row affected (0.01 sec)`**

**``````** 

**`mysql> **insert** tmp1(n1) **values**(99);`**

**`Query OK, 1 row affected (0.00 sec)`**

**``````** 

**`mysql> **select** * **from** tmp1;`**

**`+------+------+`**

**`| n1 | n2 |`**

**`+------+------+`**

**`| 18 | 90 |`**

**`| 99 | 495 |`**

**`+------+------+`**

**`2 **rows** in **set** (0.00 sec)`** 

**`**２、MySQL触发器Update触发更新另一张表：**`**

　　**`下面有有两个表tmp1、tmp2，两个表都有一个相同的字段name。使用触发器实现更新一个表的name时，将另外一个表的name也更新。`**

　**`创建测试表和触发器：`**

**`-- 创建测试表和插入测试数据`**

**`**drop** **table** if exists tmp1;`**

**`**drop** **table** if exists tmp2;`**

**`**create** **table** tmp1 (id **int**, **name** **varchar**(128)) **default** charset='utf8';`**

**`**create** **table** tmp2 (fid **int**, **name** **varchar**(128)) **default** charset='utf8';`**

**`**insert** **into** tmp1 **values**(1, '爱E族');`**

**`**insert** **into** tmp2 **values**(1, '爱E族');`**

**``````** 

**`-- 创建触发器`**

**`DELIMITER $`**

**`**drop** **trigger** if exists tmp1_update$`**

**`**create** **trigger** tmp1_update`**

**`**after** **update** **on** tmp1`**

**`**for** each row`**

**`**begin**`**

 **`**update** tmp2 **set** **name**=new.**name** **where** fid=new.id;`**

**`**end**$`**

**`DELIMITER ;`**

**`测试触发更新效果：`**

**`mysql> **select** * **from** tmp1;`**

**`+------+---------+`**

**`| id | **name** |`**

**`+------+---------+`**

**`| 1 | 爱E族 |`**

**`+------+---------+`**

**`1 row in **set** (0.00 sec)`**

**``````** 

**`mysql> **select** * **from** tmp2;`**

**`+------+---------+`**

**`| fid | **name** |`**

**`+------+---------+`**

**`| 1 | 爱E族 |`**

**`+------+---------+`**

**`1 row in **set** (0.00 sec)`**

**``````** 

**`mysql> **update** tmp1 **set** **name**='aiezu.com' **where** id=1;`**

**`Query OK, 1 row affected (0.00 sec)`**

**`**Rows** matched: 1 Changed: 1 Warnings: 0`**

**``````** 

**`mysql> **select** * **from** tmp1;`**

**`+------+-----------+`**

**`| id | **name**  |`**

**`+------+-----------+`**

**`| 1 | aiezu.com |`**

**`+------+-----------+`**

**`1 row in **set** (0.00 sec)`**

**``````** 

**`mysql> **select** * **from** tmp2;`**

**`+------+-----------+`**

**`| fid | **name**  |`**

**`+------+-----------+`**

**`| 1 | aiezu.com |`**

**`+------+-----------+`**

**`1 row in **set** (0.00 sec)`**