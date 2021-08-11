**行转列**

例如：把图1转换成图2结果展示

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\ecd315cbef5447b5ac1bff33f0390a1c\30-456550329.png)

图1 

 

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\28fe938b8a694340b512b6b1798c5efa\7-1085872632.png)

行转列SQL：

```mysql
SELECT user_name ,

  MAX(CASE course WHEN '数学' THEN score ELSE 0 END ) 数学,

  MAX(CASE course WHEN '语文' THEN score ELSE 0 END ) 语文,

  MAX(CASE course WHEN '英语' THEN score ELSE 0 END ) 英语

FROM test_tb

GROUP BY USER_NAME;
```

