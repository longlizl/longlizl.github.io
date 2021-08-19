

# 一. nc命令端口检测的用法

## 1. 端口探测

```shell
nc -vz -w 10 IP  PORT
```

​	参数说明：

```
-v 显示指令执行过程。

-w <超时秒数>  设置等待连线的时间。

-u 表示使用UDP协议

-z 使用0输入/输出模式，只在扫描通信端口时使用
```

​	例1：扫描指定的8080端口

```shell
nc -v -w 10 -z 192.168.0.100 8080 

Connection to 192.168.0.100 8080 port [tcp/http] succeeded!
```

​	例2：扫描20到25的端口范围，并详细输出。（范围扫描推荐使用nmap）

```shell
nc -v -w 2 -z 192.168.0.100 20-25 
```

```shell
nc: connect to 192.168.0.100 port 20 (tcp) failed: Connection refused

nc: connect to 192.168.0.100 port 21 (tcp) failed: Connection refused

Connection to 192.168.0.100 22 port [tcp/ssh] succeeded!

nc: connect to 192.168.0.100 port 23 (tcp) failed: Connection refused

nc: connect to 192.168.0.100 port 24 (tcp) failed: Connection refused

nc: connect to 192.168.0.100 port 25 (tcp) failed: Connection refused
```

​	例3：扫描1到65535的端口范围，只输出打开的端口（去掉-v参数即可）

```shell
nc -w 1 -z 192.168.0.100 1-65535 
```

```shell
Connection to 192.168.0.100 22 port [tcp/ssh] succeeded!

Connection to 192.168.0.100 80 port [tcp/http] succeeded!

Connection to 192.168.0.100 2121 port [tcp/scientia-ssdb] succeeded!

Connection to 192.168.0.100 4004 port [tcp/pxc-roid] succeeded!

Connection to 192.168.0.100 8081 port [tcp/tproxy] succeeded!

Connection to 192.168.0.100 11211 port [tcp/*] succeeded!
```

# 二、批量检测服务器指定端口开放情况

## 1、假如我们要监控一堆指定的IP和端口，可新建一个文件。

```shell
vim /scripts/ip-ports.txt

192.168.0.100 80  
192.168.0.100 8081  
192.168.0.101 8082  
192.168.1.100 21 
```

## 2、我们可以写这样一个脚本来批量检测端口是否开放：

```shell
vim /scripts/ncports.sh

#!/bin/bash  
#检测服务器端口是否开放，成功会返回0值显示ok，失败会返回1值显示fail  
cat /scripts/ip-ports.txt | while read line  
do  
 	nc -w 10 -z $line > /dev/null 2>&1  
 	if [ $? -eq 0 ];then  
  		echo $line:ok  
 	else  
  		echo $line:fail  
 	fi  
done 
```

## 3、执行脚本查看运行结果如下：

```shell
sh ncports.sh

192.168.0.100 80：ok
192.168.0.100 8081：ok
192.168.0.101 8082：ok
192.168.1.100 21：fail
```

# 三. 文件传输，硬盘克隆

## 1. 传输文件/目录

```shell
# 文件传输
HOSTA: nc -lp 1234 > 1.mp4
HOSTB: nc -nv 192.168.205.106 1234 < 1.mp4 -q 1
或者：
HOSTA: nc -q 1 -lp 1234 < a.mp4
HOSTB: nc -nv 192.168.205.106 1234 > 2.mp4
```

```shell
# 传输目录
HOSTA: tar -cvf - [目录路径] | nc -lp 1234 -q 1
HOSTB: nc -nv 192.168.205.106 1234 | tar -xvf -
 -c   	# 创建一个新归档
 -x   	# 从归档中解出文件
 -q 1 	# 传输完成后延迟1秒退出
```

## 2. 加密传文件:

```shell
HOSTA: nc -lp 1234 | mcrypt --flush -Fbqd -a rijndael-256 -m ecb > 1.mp4
HOSTB: mcrypt --flush -Fbq -a rijndael-256 -m ecb < a.mp4 | nc -nv 192.168.205.106 1234 -q 1
```

## 3. 流媒体服务器:

```shell
HOSTA: cat a.mp4 | nc -lp 1234
HOSTB: nc -nv 192.168.205.106 1234 | mplayer -vo x11 -cache 3000 -
```

## 4. 端口扫描:

```shell
# 扫描tcp
nc -nvz 192.168.205.106 1-65535 
#扫描udp
nc -nvzu 192.168.205.106 1-1024 
```

## 5. 进程审计:

```shell
# 被审计端
HOSTA: ps aux | nc -nv 192.168.133 1234 -q 1 # [-q 1]的意思是连接成功时候，1s后连接会自动断掉。 以便于我们判断连接是否连接成功。
# 审计端
HOSTB: nc -lp 1234 > ps.txt  # 将产生进程的文件导入ps.txt
```

## 6. 远程克隆硬盘: #电子取证

```shell
HOSTA: nc -lp 1234 | dd of=/dev/sdb
HOSTB: dd if=/dev/sda | nc -nv 192.168.205.106 1234 -q 1
```

