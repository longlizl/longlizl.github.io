mysql分区类型

   **RANGE 分区：**

​     基于属于一个给定连续区间的列值，把多行分配给分区。

   **基于RANGE COLUMNS的分区方案**

RANGE COLUMNS可以直接基于列，而无需像上述RANGE那种，分区的对象只能为整数。

   **LIST 分区：**

​     类似于按RANGE分区，区别在于LIST分区是基于列值匹配一个离散值集合中的某个值来进行选择。

   **HASH分区：**

​     基于用户定义的表达式的返回值来进行选择的分区，该表达式使用将要插入到表中的这些行的列值进行计算。这个函数可以包含MySQL中有效的、产生非负整数值的任何表达式。

   **KEY分区：**

​    类似于按HASH分区，区别在于KEY分区只支持计算一列或多列，且MySQL服务器提供其自身的哈希函数。必须有一列或多列包含整数值。

   **复合分区：**

​     基于RANGE/LIST 类型的分区表中每个分区的再次分割。子分区可以是 HASH/KEY 等类型。

**show plugins;**

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\d8b0644017c5485fafc1721a9498053b\clipboard.png)

**-- mysql hash 表分区**

```shell
ALTER TABLE t_realstatusbyhour_test

PARTITION BY HASH(id)

PARTITIONS 4;
```

**-- 查看表分区详细情况**

```mysql
SELECT PARTITION_NAME,PARTITION_METHOD,PARTITION_EXPRESSION,PARTITION_DESCRIPTION,TABLE_ROWS,SUBPARTITION_NAME,SUBPARTITION_METHOD,SUBPARTITION_EXPRESSION

FROM information_schema.PARTITIONS 

WHERE TABLE_SCHEMA=SCHEMA() AND TABLE_NAME='t_realstatusbyhour_test';
```

 **-- RANGE 表分区：**

```mysql
ALTER TABLE t_battery_statu_hour

PARTITION by RANGE(id)

(PARTITION p0 VALUES less than (24949956) ENGINE = InnoDB,

PARTITION p1 VALUES less than (27449956) ENGINE = InnoDB,

PARTITION p2 VALUES less than (29949956) ENGINE = InnoDB,

PARTITION p3 VALUES less than (32449956) ENGINE = InnoDB,

PARTITION p4 VALUES less than (34949956) ENGINE = InnoDB,

PARTITION p5 VALUES less than (37449956) ENGINE = InnoDB,

PARTITION p6 VALUES less than (39949956) ENGINE = InnoDB,

PARTITION p7 VALUES less than (42449956) ENGINE = InnoDB,

PARTITION p8 VALUES less than (44949956) ENGINE = InnoDB,

PARTITION p9 VALUES less than (47449956) ENGINE = InnoDB,

PARTITION p11 VALUES less than (49949956) ENGINE = InnoDB,

PARTITION p12 VALUES less than (52449956) ENGINE = InnoDB,

PARTITION p13 VALUES less than (54949956) ENGINE = InnoDB,

PARTITION p14 VALUES less than (57449956) ENGINE = InnoDB,

PARTITION p15 VALUES less than (59949956) ENGINE = InnoDB,

PARTITION p16 VALUES less than (62449956) ENGINE = InnoDB,

PARTITION p17 VALUES less than (64949956) ENGINE = InnoDB,

PARTITION p18 VALUES less than (67449956) ENGINE = InnoDB,

PARTITION p19 VALUES less than (69949956) ENGINE = InnoDB,

PARTITION p20 VALUES less than (72449956) ENGINE = InnoDB,

PARTITION p21 VALUES less than (74949956) ENGINE = InnoDB,

PARTITION p22 VALUES less than (77449956) ENGINE = InnoDB,

PARTITION p23 VALUES less than (79949956) ENGINE = InnoDB,

PARTITION p24 VALUES less than (82449956) ENGINE = InnoDB,

PARTITION p25 VALUES LESS THAN MAXVALUE ENGINE = InnoDB);
```

--删除表分区命令

ALTER TABLE t_realstatusbyhour_test PARTITION p0;

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\459d0c0704914bbc84069eb18e6cdba7\clipboard.png)