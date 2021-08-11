**MHA架构图**

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\ed9a8fddfbeb46b3bb3521f42f86cd9c\clipboard.png)

一：资源规划：

| 主机名       | 系统     | IP              | 服务器角色 | 备注                  | ssh互通     |
| ------------ | -------- | --------------- | ---------- | --------------------- | ----------- |
| manage-mysql | centos 7 | 192.168.205.188 | MHA管理端  | 监控各节点            | ssh免密互通 |
| master-mysql | centos 7 | 192.168.205.189 | 主数据库   | 开启bin-log,relay-log | ssh免密互通 |
| slave1-mysql | centos 7 | 192.168.205.190 | 从数据库   | 开启bin-log,relay-log | ssh免密互通 |
| slave2-mysql | centos 7 | 192.168.205.191 | 从数据库   | 开启bin-log,relay-log | ssh免密互通 |

二：搭建一主二从架构：（数据库主从复制略）

如果使用gtid复制需在配置文件中开启，在从服务器上执行

```mysql
change master to master_host='192.168.205.189',master_user='repl',master_password='111111',master_auto_position=1;
```

需要注意以下几点

1.uuid每个节点需要唯一，不然主从复制会失败

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\71302353cbad4826babb826f99df0ab7\clipboard.png)

cat /var/lib/mysql/auto.cnf

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\c11857605de34b96be8da2c3e176f43d\clipboard.png)

2. 开启gtid复制（更高效,本例中主从复制仍然采用binglog日志同步）

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\2774d2f3491045d3aa31ea7667485f0e\clipboard.png)

3. 给每节点授权一个管理账号（主节点授权即可，其他节点会自动同步）

   grant all on *.* to 'mhaadmin'@'%' identified by '111111';

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\9b7f837d19d1439badfcefc7f48bfbf2\clipboard.png)

4. 在4个节点上安装以下软件包

```mysql
yum install epel-release -y

yum -y install mha4mysql-node-0.54-0.el6.noarch.rpm
```

​	在管理节点上安装mha4mysql-manage

```shell
yum -y install mha4mysql-manager-0.54-0.el6.noarch.rpm
```

5. 定义 MHA 管理配置文件

```shell
mkdir /etc/mha_master vim /etc/mha_master/mha.cnf

[server default]
user=mhaadmin
password=111111
manager_workdir=/etc/mha_master/workspace
manager_log=/etc/mha_master/manager.log
remote_workdir=/mydata/mha_master/workspace
ssh_user=root
repl_user=repl
repl_password=111111
ping_interval=1
[server1]
hostname=192.168.205.189
ssh_port=22
candidate_master=1
[server2]
hostname=192.168.205.190
ssh_port=22
candidate_master=1
[server3]
hostname=192.168.205.191
ssh_port=22
candidate_master=1
```

6. 检测各节点间 ssh 互信通信配置是否 ok

   在 Manager 机器上输入下述命令来检测：

   masterha_check_ssh -conf=/etc/mha_master/mha.cnf

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\5cbdad83c26347208675560a570d1525\clipboard.png)

7. 检查管理的MySQL复制集群的连接配置参数是否OK（注：复制账号repl需要在mysql各个节点上授权不然检测会报错）

   ```mysql
   grant replication slave on *.* to 'repl'@'%'  identified by '111111';
   masterha_check_repl -conf=/etc/mha_master/mha.cnf
   ```

   ![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\db7dbed46a474a1f8e9ba4b6c5832ae8\clipboard.png)

8. MHA数据库切换脚本

   ```shell
   vim /usr/bin/master_ip_failover
   #!/usr/bin/env perl
   use strict;
   use warnings FATAL => 'all';
   use Getopt::Long;**
   my (
       $command,   $ssh_user,  $orig_master_host,
       $orig_master_ip,$orig_master_port, $new_master_host, $new_master_ip,$new_master_port
   );
   #定义VIP变量
   my $vip = '192.168.205.200/24';
   my $key = '1';
   my $ssh_start_vip = "/sbin/ifconfig ens33:$key $vip";
   my $ssh_stop_vip = "/sbin/ifconfig ens33:$key down";
   GetOptions(
       'command=s'     => \$command,
       'ssh_user=s'        => \$ssh_user,
       'orig_master_host=s'    => \$orig_master_host,
       'orig_master_ip=s'  => \$orig_master_ip,
       'orig_master_port=i'    => \$orig_master_port,
       'new_master_host=s' => \$new_master_host,
       'new_master_ip=s'   => \$new_master_ip,
       'new_master_port=i' => \$new_master_port,
   );
   exit &main();
   sub main {
       print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n";
       if ( $command eq "stop" || $command eq "stopssh" ) {
           my $exit_code = 1;
           eval {
               print "Disabling the VIP on old master: $orig_master_host \n";
               &stop_vip();
               $exit_code = 0;
           };
           if ($@) {
               warn "Got Error: $@\n";
               exit $exit_code;
           }
           exit $exit_code;
       }   
       elsif ( $command eq "start" ) {
       my $exit_code = 10; 
       eval {
           print "Enabling the VIP - $vip on the new master - $new_master_host \n";
           &start_vip();
           $exit_code = 0;
       };  
       if ($@) {
           warn $@;
           exit $exit_code;
           }
       exit $exit_code;
       }
       elsif ( $command eq "status" ) {
           print "Checking the Status of the script.. OK \n";
           exit 0;
       }   
       else {
           &usage();
           exit 1;
       }  
   }   
   sub start_vip() {
      `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
   }
   sub stop_vip() {
       return 0 unless ($ssh_user);
       `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
   }
   sub usage {
   	print
   	"Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
   }
   ```

   9.授予脚本执行权限

   ​	chmod a+x /usr/bin/master_ip_failover

   ​	在配置文件中加入脚本配置文件路径

   ​	vim /etc/mha_master/mha.cnf

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\6b93cd695a324b70ae4747f6ed5c7faa\clipboard.png)

10. 启动manager

```shell
nohup masterha_manager -conf=/etc/mha_master/mha.cnf 2>&1 > /etc/mha_master/manager.log &
```

查看后台运行任务

```shell
jobs -l
```

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\98ac5ab4a3904750a7c9abc8478efb8e\clipboard.png)

11. 查看MHA状态（目前192.168.205.190这个ip为主库）

```shell
masterha_check_status  --conf=/etc/mha_master/mha.cnf
```

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\7aae92032d5b4ac0acfab9671f62bae5\clipboard.png)

​	第一次配置VIP时，需要手动添加主库的虚拟IP,现在主库是192.168.205.190这个主机，所以在这个主机上加第一个VIP地址

```shell
ifconfig ens33:1 192.168.205.200/24
```

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\dd29e817ca8244eebfa77c9204c2e567\clipboard.png)

三：现在我们人为重现主库故障，停止192.168.205.190这个主机mysql服务

```shell
systemctl stop mysqld
```

我们发现vip接口地址已经不在此节点上

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\252bb6f12d52490f96ae24c2a85023e2\clipboard.png)

现在我们看下manager日志信息，master已切换到189这个主机上

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\708360da08664b26a41cd6261068876b\clipboard.png)

还有一个很重要的信息，原有主机down掉后面恢复后变为从，需要手动去同步。里面有条语句很重要需要在原有主库恢复后执行

```
All other slaves should start replication from here. Statement should be: CHANGE MASTER TO MASTER_HOST='192.168.205.189', MASTER_PORT=3306, MASTER_LOG_FILE='mysql-bin.000005'
, MASTER_LOG_POS=234, MASTER_USER='repl', MASTER_PASSWORD='xxx';
```

**备注：MASTER_PASSWORD='xxx';  是复制账号密码MASTER_PASSWORD='111111'**

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\dd7744eccf4c40e1a951d974e1830312\clipboard.png)

看下ip地址vip已将切换到189这台主机上

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\b74344fecd8445aa9975c09037cd0af9\clipboard.png)

*到191这台主机节点查看mysql相关复制信息发现master已经切换成功*

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\566e16a92f0f4bd28b8f48a6a078f0c5\clipboard.png)

登录190这台主机，启动mysql服务并连接mysql执行以下语句

```mysql
CHANGE MASTER TO MASTER_HOST='192.168.205.189', MASTER_PORT=3306, MASTER_LOG_FILE='mysql-bin.000005'
, MASTER_LOG_POS=234, MASTER_USER='repl', MASTER_PASSWORD='111111';
start slave;
```

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\426587eef2d34a4e934636abc564c6b6\clipboard.png)

查看同步情况

```mysql
show slave status\G
```

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\51c2277d97a44af589bdceaa7fc09407\clipboard.png)

到此所有功能已恢复，最后我们需要重新启动manager脚本监控即可。

```shell
nohup masterha_manager -conf=/etc/mha_master/mha.cnf 2>&1 > /etc/mha_master/manager.log &
```

