[MySQL性能指标及计算方法](https://www.cnblogs.com/yuyue2014/p/3679628.html)

绝大多数MySQL性能指标可以通过以下两种方式获取：

（1）mysqladmin

使用mysqladmin extended-status命令获得的MySQL的性能指标，默认为累计值。**如果想了解当前状态，需要进行差值计算**；加上参数 --relative(-r)，就可以看到各个指标的差值，配合参数--sleep(-i)就可以指定刷新的频率。

mysqladmin -uroot extended-status  --relative --sleep=10 -p 

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\0dab7f3b5d1a4e5e99e66562d109e2de\clipboard.png)

（2）Show global status

可以列出MySQL服务器运行各种状态值，**累计值**。

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\571d89fc570441a5b2aa67077e72265c\001199511762.png)

mysqladmin extended-status命令及show global status得到的指标项特别多。实际应用中，重点关注以下性能指标：

**1. tps/qps**

tps: Transactions Per Second，每秒事务数；

qps: Queries Per Second每秒查询数；

通常有两种方法计算tps/qps：

方法1：基于  com_commit、com_rollback 计算tps，基于 questions  计算qps。

TPS = Com_commit/s + Com_rollback/s

其中，

Com_commit /s= mysqladmin -uroot -p extended-status --relative --sleep=1|grep -w Com_commit

Com_rollback/s = mysqladmin -uroot -p extended-status --relative --sleep=1|grep -w Com_rollback

QPS 是指MySQL Server 每秒执行的Query总量，通过Questions (客户的查询数目)状态值每秒内的变化量来近似表示，所以有：

QPS = mysqladmin -uroot -h 172.16.1.8 -P -p extended-status --relative --sleep=1|grep -w Questions

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\e8a24edf8b2c4b35a3d64574f64b3885\clipboard.png)

 

仿照上面的方法还可以得到，mysql每秒select、insert、update、delete的次数等，如：

Com_select/s = mysqladmin -uroot -p extended-status --relative --sleep=1|grep -w Com_select

Com_select/s：平均每秒select语句执行次数

Com_insert/s：平均每秒insert语句执行次数

Com_update/s：平均每秒update语句执行次数

Com_delete/s：平均每秒delete语句执行次数

 

方法2: 基于com_%计算tps ,qps

tps= Com_insert/s + Com_update/s + Com_delete/s

qps=Com_select/s + Com_insert/s + Com_update/s + Com_delete/s

**2. 线程状态**

threads_running：当前正处于激活状态的线程个数

threads_connected：当前连接的线程的个数

**3. 流量状态**

Bytes_received/s：平均每秒从所有客户端接收到的字节数，单位KB

Bytes_sent/s：平均每秒发送给所有客户端的字节数，单位KB

**4. innodb文件读写次数**

innodb_data_reads：innodb平均每秒从文件中读取的次数

innodb_data_writes：innodb平均每秒从文件中写入的次数

innodb_data_fsyncs：innodb平均每秒进行fsync()操作的次数

**5. innodb读写量**

innodb_data_read：innodb平均每秒钟读取的数据量，单位为KB

innodb_data_written：innodb平均每秒钟写入的数据量，单位为KB

**6. innodb缓冲池状态**

innodb_buffer_pool_reads: 平均每秒从物理磁盘读取页的次数 

innodb_buffer_pool_read_requests: 平均每秒从innodb缓冲池的读次数（逻辑读请求数）

innodb_buffer_pool_write_requests: 平均每秒向innodb缓冲池的写次数

innodb_buffer_pool_pages_dirty: 平均每秒innodb缓存池中脏页的数目

innodb_buffer_pool_pages_flushed: 平均每秒innodb缓存池中刷新页请求的数目

innodb缓冲池的读命中率

innodb_buffer_read_hit_ratio = ( 1 - Innodb_buffer_pool_reads/Innodb_buffer_pool_read_requests) * 100

Innodb缓冲池的利用率

Innodb_buffer_usage = ( 1 - Innodb_buffer_pool_pages_free / Innodb_buffer_pool_pages_total) * 100

**7. innodb日志**

innodb_os_log_fsyncs: 平均每秒向日志文件完成的fsync()写数量

innodb_os_log_written: 平均每秒写入日志文件的字节数

innodb_log_writes: 平均每秒向日志文件的物理写次数

innodb_log_write_requests: 平均每秒日志写请求数

**8. innodb行**

innodb_rows_deleted: 平均每秒从innodb表删除的行数

innodb_rows_inserted: 平均每秒从innodb表插入的行数

innodb_rows_read: 平均每秒从innodb表读取的行数

innodb_rows_updated: 平均每秒从innodb表更新的行数

innodb_row_lock_waits:  一行锁定必须等待的时间数

innodb_row_lock_time: 行锁定花费的总时间，单位毫秒

innodb_row_lock_time_avg: 行锁定的平均时间，单位毫秒

**9. MyISAM读写次数**

key_read_requests: MyISAM平均每秒钟从缓冲池中的读取次数

Key_write_requests: MyISAM平均每秒钟从缓冲池中的写入次数

key_reads : MyISAM平均每秒钟从硬盘上读取的次数

key_writes : MyISAM平均每秒钟从硬盘上写入的次数

**10. MyISAM缓冲池**

MyISAM平均每秒key buffer利用率

Key_usage_ratio =Key_blocks_used/(Key_blocks_used+Key_blocks_unused)*100

MyISAM平均每秒key buffer读命中率

Key_read_hit_ratio=(1-Key_reads/Key_read_requests)*100

MyISAM平均每秒key buffer写命中率

Key_write_hit_ratio =(1-Key_writes/Key_write_requests)*100

**11. 临时表**

Created_tmp_disk_tables: 服务器执行语句时在硬盘上自动创建的临时表的数量

Created_tmp_tables: 服务器执行语句时自动创建的内存中的临时表的数量

Created_tmp_disk_tables/Created_tmp_tables比值最好不要超过10%，如果Created_tmp_tables值比较大，可能是排序句子过多或者连接句子不够优化

**12. 其他**

slow_queries: 执行时间超过long_query_time秒的查询的个数（重要）

sort_rows: 已经排序的行数

open_files: 打开的文件的数目

open_tables: 当前打开的表的数量

select_scan: 对第一个表进行完全扫描的联接的数量

 

此外，还有一些性能指标不能通过mysqladmin extended-status或show global status直接得到，但是十分重要。

**13. response time: 响应时间**

Percona提供了[tcprstat工具](http://www.mysqlperformanceblog.com/2010/08/31/introducing-tcprstat-a-tcp-response-time-tool/)统计响应时间，此功能默认是关闭的，可以通过设置参数query_response_time_stats=1打开这个功能。

有两种方法查看响应时间：

（1）通过命令SHOW QUERY_RESPONSE_TIME查看响应时间统计；

（2）通过INFORMATION_SCHEMA里面的表QUERY_RESPONSE_TIME来查看。