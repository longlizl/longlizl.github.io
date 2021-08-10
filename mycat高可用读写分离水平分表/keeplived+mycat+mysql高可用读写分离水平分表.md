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

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\47748751b0b249eda78b0e76a31a3a25\clipboard.png)

vim schema.xml

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\4a6e2272a3c24ae1bf06aafb47d5b8f2\clipboard.png)

**4.启动mycat服务**

cd /opt/mycat/bin

./mycat start

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\aa916809fe144397b79d3cdbcaa5e88a\clipboard.png)

mycat启动后会有2个端口8066（数据连接端口），9066（管理端口）

ss -ntupl

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\5441a772b678428f866773daaaf7da56\clipboard.png)

**5.通过mysql客户端连接8066（我们在mysql-master节点登录mycat试下）**

mysql -ulilong -p111111 -P8066 -h 192.168.205.183

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\cd3090d4b4fc4f19a6aebfe665fabbb3\clipboard.png)

**6.查看mycat里数据库信息**

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\9b25441680984d7095f722393006a873\clipboard.png)

**7.进入TSDB创建表city**

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\2ac83a331f824e65990ad4971f5d1784\clipboard.png)

crate table city(id int,name varchar(8),area float(5,2),people_num int);

查看数据库中是否创建成功

show tables;

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\b256a1a4155241ab99a9cfba7a9452b1\clipboard.png)

在2台数据库节点看是否已存在此表（由于做了读写分离在创建表时走了写节点也就是主库，从库也同步过来了）

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\0f08c69148424080a4a5e81f2476f1b6\clipboard.png)

**8.通过mysql客户端连接9066可以查看到相关dataNode信息**

mysql -ulilong -p111111 -P9066 -h 192.168.205.183

show @@dataNode;

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\1c7b59bf333e4777967dc55c09601866\clipboard.png)

**在2台mycat节点上安装keepalived并单独配置日志文件（默认在/var/log/messages日志查看不方便）**

192.168.205.182:

yum -y install keepalived

vim /etc/sysconfig/keepalived

KEEPALIVED_OPTIONS="-D -d -S 0"

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\ac8b00cd089345249f46836229b78f4b\clipboard.png)

vim /etc/rsyslog.conf

local0.*                        /var/log/keepalived.log

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\1568e04045df472a8301c92a574c082a\clipboard.png)

systemctl restart rsyslog 

vim /etc/keepalived/keepalived.conf

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\1ba43f69fb5c430a9f63b69e75d1fba8\clipboard.png)

192.168.205.183:

yum -y install keepalived

vim /etc/sysconfig/keepalived

KEEPALIVED_OPTIONS="-D -d -S 0"

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\5a84e3baba9d4d6b86b3d74d9de3175a\clipboard.png)

vim /etc/rsyslog.conf

local0.*                        /var/log/keepalived.log

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\f955351ce5404c4c8c29de02057ff44c\clipboard.png)

systemctl restart rsyslog 

vim /etc/keepalived/keepalived.conf

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\ba7180bd53df44a5a43ccc3a7f0fe325\clipboard.png)

**分别启动2节点keepalived**

192.168.205.182:

systemctl start keepalived

可以看到vip接口在182节点上

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\7fb196428c8649099560818572d586e1\clipboard.png)

192.168.205.183:

systemctl start keepalived

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\e5c65a6a2d1b4a5e9c0392389de867b8\clipboard.png)

在mysql节点上我们通过vip接口登录试下

mysql -ulilong -p111111 -P8066 -h 192.168.205.250

下图我们看到已经登录成功

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\13aba28e0a2d40d4a34facd04cbf2fe9\clipboard.png)

我们停用192.168.205.182的keepalived试下看是否转移到192.168.205.183上

可以看到182节点vip接口不在了 

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\ca12d79b9282472c8767e90e1fc8bb49\clipboard.png)

在看下183节点vip接口已飘逸到此节点上

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\726906f5333c46ea9c7d1dcc3e5a9a79\clipboard.png)

我们再次启动主节点上keepalived发现vip接口已经飘逸回来，默认情况下keepalived是抢占模式所以主节点恢复后会直接抢占回来

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\da6194db418d46fc81497231ac5f59d3\clipboard.png)

**到此为止基本完成配置，可仔细想想如果是所在keepalived的master节点mycat服务有问题那它还会切换到备用节点吗。答案肯定是否定的。这就需要在keepalived 配置脚本来检测mycat服务状态如果所在节点master的mycat服务挂掉了， 那就主动结束所在master节点的keepalived进程切换至备用节点继续提供服务**

2节点分别创建存放脚本的文件

mkdir -p /etc/keepalived/scripts

vim  /etc/keepalived/scripts/chk_mycat.sh

\#!/bin/bash

MYCAT_PORT=`ss -ntupl | egrep '8066|9066' | wc -l`

if [ $MYCAT_PORT -ne 2 ];then

​        pkill keepalived

fi

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\69a7b749070a438e835e4c2792a2c794\clipboard.png)

在 /etc/keepalived/keepalived.conf配置文件中将以下内容加入到相应位置

vrrp_script chk_mycat {

​    script "/etc/keepalived/scripts/chk_mycat.sh"

​    interval 2

​    weight -50

​    fall 3

​    rise 3

​    timeout 3

}

   track_script {

​        chk_mycat

   }

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\fd0ffb45155b4533a341767af4481a11\clipboard.png)

重新启动keepalived服务

systemctl restart keepalived

验证检测脚本是否生效：

将master节点上mycat服务手动停掉

cd /opt/mycat/bin && ./mycat stop

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\a685245b70f64777b013760e286c78cc\clipboard.png)

**此时master节点keepalived服务已经停止了**

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\18e7a7456ad341b39f8899a096d520da\clipboard.png)

在看下backup节点，发现vip已经飘移到此节点上

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\24c99290c6b04dd5a9d83f182582bbe8\clipboard.png)

**从mysql客户端访问正常**

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\b7d92ad0327b4dfa97b10b00fa3d740c\clipboard.png)

**配置MASTER<-->slave 相互切换时邮件通知**

**2个keepalived节点安装postfix邮件服务（默认已安装）和 发送邮件插件mailx** 

yum -y install mailx

vim /etc/mail.rc

set  from=test@hzs.com.cn

set  smtp=smtp.exmail.qq.com

set  smtp-auth-user=test@hzs.com.cn

set  smtp-auth-password=*******

smtp-auth=login

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\c9dfe2dcb7d645cdbbd18e84f9b6c273\clipboard.png)

将以下配置加入下面相应位置（2台做同样操作）

MASTER，BACKUP，FAULT（大小写均可）

vim /etc/keepalived/keepalived.conf

notify_master "/etc/keepalived/scripts/notify.sh MASTER"

notify_backup "/etc/keepalived/scripts/notify.sh BACKUP"

notify_fault "/etc/keepalived/scripts/notify.sh FAULT"

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\40b85c10723247a79e6c3965909eb8a7\clipboard.png)

**创建相应脚本文件(2台做同样操作)**

cd  /etc/keepalived/scripts

vim notify.sh

\#!/bin/bash

SEND_to_MAIL='1550789579@qq.com'

notify() {

​    MAIL_SUBJECT="$(hostname) to be $1, vip 转移"

​    MAIL_TEXT="$(date +'%F %T'): vrrp transition, $(hostname) changed to be $1"

​    echo "$MAIL_TEXT" | mail -s "$MAIL_SUBJECT" $SEND_to_MAIL

}

case $1 in

MASTER)

​    notify MASTER

​    ;;

BACKUP)

​    notify BACKUP

​    ;;

FAULT)

​    notify FAULT

​    ;;

*)

​    echo "Usage: $(basename $0) {MASTER|BACKUP|FAULT}"

​    exit 1

​    ;;

esac

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\93a343d1002048d48dc2ee45c69f2745\clipboard.png)

更改脚本执行权限

chmod +x notify

重新启动keepalived服务

systemctl restart  keepalived

已可以看到vip已转移到备用节点上

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\b63ee270dbef456abb91eaf99cff4ad3\clipboard.png)

\-------------------------------------------------------------------------------------------------------------------------------------

**mycat分表**

1.我们以2个dataNode节点为例子在mysql-master库上新建cs1_db，mysql-slave会同步cs1_db库，之前已建过cs_db库，在2个库里分别新建表'wuhan'

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\8825fea49bd6481b960158c0e031047c\clipboard.png)

2.mycat节点（2台分表做以下配置）

vim schema.xml

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\fb3cfffd86a24d79b69dd3edc577113a\clipboard.png)

vim rule.xml

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\03f1f10315ef4cfc843f75393b896e55\clipboard.png)

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\1dca401ecc5444ab929ad759f674f92c\clipboard.png)

3.重启mycat服务

cd /opt/mycat/bin && ./mycat restart

4.通过VIP接口连接mycat服务，插入数据

mysql -ulilong -p111111 -P8066 -h 192.168.205.250

insert into wuhan(id,address) values(7,'xx'),(8,'xg'),(9,'lx'),(10,'ob'),(11,'kx'),(12,'hx');

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\95da44ae7b0447678b0b713ddae2ce22\clipboard.png)

5.查看2个数据库wuhan表中数据，发现数据已经分表插入到了2个数据库中同一表里

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\eaa13cdb283a472285fb1aa00e1f883a\clipboard.png)

6.可以看下mysql-slave库也同步了

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\72c5810de42d400296317f1e1f57c4f2\clipboard.png)

到此所有配置已结束