# ClickHouse 高级常见问题排查

### 1 分布式 DDL 某数据节点的副本不执行

```
（1）问题：
	使用分布式 ddl 执行命令 create table on cluster xxxx 某个节点上没有创建
	表，但是 client 返回正常，查看日志有如下报错。
	<Error> xxx.xxx: Retrying createReplica(), because some other replicas
	were created at the same time
（2）解决办法：
	重启该不执行的节点。
```

### 2 数据副本表和数据不一致

```
（1）问题：
	由于某个数据节点副本异常，导致两数据副本表不一致，某个数据副本缺
	少表，需要将两个数据副本调整一致。
（2）解决办法：
	在缺少表的数据副本节点上创建缺少的表，创建为本地表，表结构可以在其他数据副本
	通过 show crete table xxxx 获取。
	表结构创建后，clickhouse 会自动从其他副本同步该表数据，验证数据量是否一致即可。
```

### 3 副本节点全量恢复

```
（1）问题：
	某个数据副本异常无法启动，需要重新搭建副本。
（2）解决办法：
	清空异常副本节点的 metadata 和 data 目录。
	从另一个正常副本将 metadata 目录拷贝过来（这一步之后可以启动数据库，但是只有
	表结构没有数据）。
	执行 sudo -u clickhouse touch /data/clickhouse/flags/force_restore_data
	启动数据库。
	数据副本启
```

### 4 动缺少 zk 表

```
（1）问题：某个数据副本表在 zk 上丢失数据，或者不存在，但是 metadata 元数据里
	存在，导致启动异常，报错：
	Can’t get data for node /clickhouse/tables/01-
	02/xxxxx/xxxxxxx/replicas/xxx/metadata: node doesn’t exist (No node):
	Cannot attach table xxxxxxx
（2）解决办法：
	metadata 中移除该表的结构文件，如果多个表报错都移除
	mv metadata/xxxxxx/xxxxxxxx.sql /tmp/
	启动数据库
	手工创建缺少的表，表结构从其他节点 show create table 获取。
	创建后会自动同步数据，验证数据是否一致。
```



### 5 ZK table replicas 数据未删除，导致重建表报错

```
（1）问题：
	重建表过程中，先使用 drop table xxx on cluster xxx ,各节点在 clickhouse 上
	table 已物理删除，但是 zk 里面针对某个 clickhouse 节点的 table meta 信息未被删除（低概
	率事件），因 zk 里仍存在该表的 meta 信息，导致再次创建该表 create table xxx on cluster, 该
	节点无法创建表(其他节点创建表成功)，报错：
	Replica /clickhouse/tables/01-03/xxxxxx/xxx/replicas/xxx already exists..
（2）解决办法：
	从其他数据副本 cp 该 table 的 metadata sql 过来.
	重启节点。
```

### 6 Clickhouse 节点意外关闭

```
（1）问题：
	模拟其中一个节点意外宕机，在大量 insert 数据的情况下，关闭某个节点。
（2）现象：
	数据写入不受影响、数据查询不受影响、建表 DDL 执行到异常节点会卡住，
（3）报错：
	Code: 159. DB::Exception: Received from localhost:9000. DB::Exception:
	Watching task /clickhouse/task_queue/ddl/query-0000565925 is executing
	longer than distributed_ddl_task_timeout (=180) seconds. There are 1
	unfinished hosts (0 of them are currently active), they are going to
	execute the query in background.
（4）解决办法：启动异常节点，期间其他副本写入数据会自动同步过来，其他副本的
	建表 DDL 也会同步。
```

### 7 其他问题参考

```
# 阿里云相关问题参考
https://help.aliyun.com/document_detail/162815.html?spm=a2c4g.11186623.6.652.312e79bd17U8IO
```

