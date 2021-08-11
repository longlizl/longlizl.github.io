编辑配置文件在【mysald】段加入如下: character-set-server=utf8

vim /etc/my.cnf

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\921e98d154de486384d0e069c17e8512\clipboard.png)

重启服务进入数据库查询字符集：show variables like '%char%';

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\bba6be13c83d4f93ab50dfc4f568a857\clipboard.png)

客户端显示编码可以在上面加入[client]段 default-character-set=utf8

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\778930e9e47140fa859273f212a39f41\clipboard.png)