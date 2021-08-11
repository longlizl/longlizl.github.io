**一：环境准备：**

| 应用                   | 主机            |
| ---------------------- | --------------- |
| mysql-master           | 192.168.205.184 |
| mysql-slave            | 192.168.205.185 |
| mycat-01,keeplived,jdk | 192.168.205.182 |
| mycat-02,keeplived,jdk | 192.168.205.183 |

mysql主从环境（略）

**二: 主机（192.168.205.183，192.168.205.182）上安装jdk，mycat，keeplived**

以192.168.205.183主机为例，另外一台主机配置与183主机一致：

**1.安装jdk**

上传jdk安装包解压安装到/usr/local/jdk

```shell
vim /etc/profile

export JAVA_HOME=/usr/local/jdk

export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar

export PATH=$JAVA_HOME/bin:$PATH
```

. /etc/profile

**2.官网下载安装mycat**

```shell
wget http://dl.mycat.org.cn/1.6.7.4/Mycat-server-1.6.7.4-release/Mycat-server-1.6.7.4-release-20200105164103-linux.tar.gz

tar -zxvf Mycat-server-1.6.7.4-release-20200105164103-linux.tar.gz

mv mycat/ /opt/

3.修改配置文件

cd /opt/mycat/conf

vim server.xml
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/1.png)

vim schema.xml

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/2.png)

**4.启动mycat服务**

```shell
cd /opt/mycat/bin
./mycat start
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/3.png)

```shell
mycat启动后会有2个端口8066（数据连接端口），9066（管理端口）
ss -ntupl
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/4.png)

**5.通过mysql客户端连接8066（我们在mysql-master节点登录mycat试下）**

```shell
mysql -ulilong -p111111 -P8066 -h 192.168.205.183
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/5.png)

**6.查看mycat里数据库信息**

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/6.png)

**7.进入TSDB创建表city**

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/7.png)

```shell
crate table city(id int,name varchar(8),area float(5,2),people_num int);
查看数据库中是否创建成功
show tables;
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/8.png)

在2台数据库节点看是否已存在此表（由于做了读写分离在创建表时走了写节点也就是主库，从库也同步过来了）

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/9.png)

**8.通过mysql客户端连接9066可以查看到相关dataNode信息**

```shell
mysql -ulilong -p111111 -P9066 -h 192.168.205.183
```

show @@dataNode;

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/10.png)

**在2台mycat节点上安装keepalived并单独配置日志文件（默认在/var/log/messages日志查看不方便）**

```shell
192.168.205.182:
yum -y install keepalived
vim /etc/sysconfig/keepalived
KEEPALIVED_OPTIONS="-D -d -S 0"
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/11.png)

```shell
vim /etc/rsyslog.conf
local0.*                        /var/log/keepalived.log
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/12.png)

```shell
systemctl restart rsyslog 
vim /etc/keepalived/keepalived.conf
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/13.png)

```shell
192.168.205.183:
yum -y install keepalived
vim /etc/sysconfig/keepalived
KEEPALIVED_OPTIONS="-D -d -S 0"
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/14.png)

```shell
vim /etc/rsyslog.conf
local0.*                        /var/log/keepalived.log
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/15.png)

```shell
systemctl restart rsyslog 
vim /etc/keepalived/keepalived.conf
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/16.png)

**分别启动2节点keepalived**

```shell
192.168.205.182:
systemctl start keepalived
可以看到vip接口在182节点上
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/17.png)

```shell
192.168.205.183:
systemctl start keepalived
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/18.png)

在mysql节点上我们通过vip接口登录试下

```mysql
mysql -ulilong -p111111 -P8066 -h 192.168.205.250
```

下图我们看到已经登录成功

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/19.png)

我们停用192.168.205.182的keepalived试下看是否转移到192.168.205.183上

可以看到182节点vip接口不在了 

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/20.png)

在看下183节点vip接口已飘逸到此节点上

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/21.png)

我们再次启动主节点上keepalived发现vip接口已经飘逸回来，默认情况下keepalived是抢占模式所以主节点恢复后会直接抢占回来

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/22.png)

**到此为止基本完成配置，可仔细想想如果是所在keepalived的master节点mycat服务有问题那它还会切换到备用节点吗。答案肯定是否定的。这就需要在keepalived 配置脚本来检测mycat服务状态如果所在节点master的mycat服务挂掉了， 那就主动结束所在master节点的keepalived进程切换至备用节点继续提供服务**

2节点分别创建存放脚本的文件

```shell
mkdir -p /etc/keepalived/scripts
vim  /etc/keepalived/scripts/chk_mycat.sh
#!/bin/bash
MYCAT_PORT=`ss -ntupl | egrep '8066|9066' | wc -l`
if [ $MYCAT_PORT -ne 2 ];then
        pkill keepalived
fi
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/23.png)

在 /etc/keepalived/keepalived.conf配置文件中将以下内容加入到相应位置

```shell
vrrp_script chk_mycat {
    script "/etc/keepalived/scripts/chk_mycat.sh"
    interval 2
    weight -50
    fall 3
    rise 3
    timeout 3
}
   track_script {
        chk_mycat
   }
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/24.png)

重新启动keepalived服务

```shell
systemctl restart keepalived
```

验证检测脚本是否生效：

将master节点上mycat服务手动停掉

```shell
cd /opt/mycat/bin && ./mycat stop
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/25.png)

**此时master节点keepalived服务已经停止了**

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/26.png)

在看下backup节点，发现vip已经飘移到此节点上

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/27.png)

**从mysql客户端访问正常**

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/28.png)

**配置MASTER<-->slave 相互切换时邮件通知**

**2个keepalived节点安装postfix邮件服务（默认已安装）和 发送邮件插件mailx** 

```shell
yum -y install mailx
vim /etc/mail.rc
set  from=test@hzs.com.cn
set  smtp=smtp.exmail.qq.com
set  smtp-auth-user=test@hzs.com.cn
set  smtp-auth-password=*******
smtp-auth=login
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/29.png)

将以下配置加入下面相应位置（2台做同样操作）

```shell
MASTER，BACKUP，FAULT（大小写均可）
vim /etc/keepalived/keepalived.conf
notify_master "/etc/keepalived/scripts/notify.sh MASTER"
notify_backup "/etc/keepalived/scripts/notify.sh BACKUP"
notify_fault "/etc/keepalived/scripts/notify.sh FAULT"
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/30.png)

**创建相应脚本文件(2台做同样操作)**

```shell
cd  /etc/keepalived/scripts
vim notify.sh
#!/bin/bash
SEND_to_MAIL='1550789579@qq.com'
notify() {
    MAIL_SUBJECT="$(hostname) to be $1, vip 转移"
    MAIL_TEXT="$(date +'%F %T'): vrrp transition, $(hostname) changed to be $1"
    echo "$MAIL_TEXT" | mail -s "$MAIL_SUBJECT" $SEND_to_MAIL
}
case $1 in
MASTER)
    notify MASTER
    ;;
BACKUP)
    notify BACKUP
    ;;
FAULT)
    notify FAULT
    ;;
*)
    echo "Usage: $(basename $0) {MASTER|BACKUP|FAULT}"
    exit 1
    ;;
esac
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/31.png)

更改脚本执行权限

```shell
chmod +x notify
```

重新启动keepalived服务

```shell
systemctl restart  keepalived
```

已可以看到vip已转移到备用节点上

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/32.png)

---

**mycat分表**

1.我们以2个dataNode节点为例子在mysql-master库上新建cs1_db，mysql-slave会同步cs1_db库，之前已建过cs_db库，在2个库里分别新建表'wuhan'

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/33.png)

2.mycat节点（2台分表做以下配置）

```
vim schema.xml
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/34.png)

```
vim rule.xml
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/35.png)

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/36.png)

3.重启mycat服务

```shell
cd /opt/mycat/bin && ./mycat restart
```

4.通过VIP接口连接mycat服务，插入数据

```mysql
mysql -ulilong -p111111 -P8066 -h 192.168.205.250
insert into wuhan(id,address) values(7,'xx'),(8,'xg'),(9,'lx'),(10,'ob'),(11,'kx'),(12,'hx');
```

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/37.png)

5.查看2个数据库wuhan表中数据，发现数据已经分表插入到了2个数据库中同一表里

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/38.png)

6.可以看下mysql-slave库也同步了

![img](https://longlizl.github.io/mycat高可用读写分离水平分表/images/39.png)

到此所有配置已结束