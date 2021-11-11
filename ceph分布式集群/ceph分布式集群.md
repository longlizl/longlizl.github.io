# ceph分布式文件系统

## 一：系统环境

| 基础环境   |                 |             |          |                 |             |
| ---------- | --------------- | ----------- | -------- | --------------- | ----------- |
| 系统环境   | IP              | 主机名      | 启动用户 | chronyd时钟服务 | ssh互为免密 |
| centos 7.4 | 192.168.205.190 | ceph-admin  | ceph     | server          | Y           |
| centos 7.4 | 192.168.205.191 | ceph-node-1 | ceph     | client          | Y           |
| centos 7.4 | 192.168.205.192 | ceph-node-2 | ceph     | client          | Y           |

## 1. 配置hosts主机（3节点一样）

vim /etc/hosts

```shell
192.168.205.190 ceph-admin

192.168.205.191 ceph-node-1

192.168.205.192 ceph-node-2
```

创建 Ceph 部署用户（3节点一样）

```
useradd ceph

passwd ceph
```

添加sudo权限

vim /etc/sudoers

```shell
ceph    ALL=(ALL)       NOPASSWD: ALL
```

![img](https://longlizl.github.io/ceph分布式集群/images/1.png)

## 2. ssh 免密登录（主节点到2节点）

修改你管理节点上的 ~/.ssh/config ，以使未指定用户名时默认用你刚刚创建的用户名

```shell
Host ceph-node-1

   Hostname ceph-node-1

   User ceph

Host ceph-node-2

   Hostname ceph-node-2

   User ceph
```

![img](https://longlizl.github.io/ceph分布式集群/images/2.png)

```shell
chmod 600 config
```



## 3. 配置chronyd时间同步

默认centos已安装chronyd服务，主节点默认时间是同步centos官方服务器时间。只需将其他2台客户端指向chrony的主节点

主机名：ceph-admin

vim /etc/chrony.conf

允许本地同步网络段

![img](https://longlizl.github.io/ceph分布式集群/images/3.png)

```shell
systemctl restart chronyd
```

确认主节点时间同步状态

![img](https://longlizl.github.io/ceph分布式集群/images/4.png)

主机名：ceph-node-1，ceph-node-2

客户端配置指向主节点ip（2台做同样设置）

vim /etc/chrony.conf

![img](https://longlizl.github.io/ceph分布式集群/images/5.png)

同步主节点时间

```shell
chronyc sources -v
```

![img](https://longlizl.github.io/ceph分布式集群/images/6.png)

![img](https://longlizl.github.io/ceph分布式集群/images/7.png)

**二 ：ceph集群部署**

## 1. 主节点（ceph-admin）上安装部署工具 ceph-deploy

\# yum 配置其他依赖包 

```shell
$ sudo yum install -y yum-utils && sudo yum-config-manager --add-repo https://dl.fedoraproject.org/pub/epel/7/x86_64/ && sudo yum install --nogpgcheck -y epel-release && sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7

sudo rm /etc/yum.repos.d/dl.fedoraproject.org*

yum -y update

```

```shell
# 加入ceph.repo源
sudo vim /etc/yum.repos.d/ceph.repo
[Ceph-noarch]

name=Ceph noarch packages

baseurl=http://download.ceph.com/rpm-jewel/el7/noarch

enabled=1

gpgcheck=1

type=rpm-md

gpgkey=https://download.ceph.com/keys/release.asc

priority=1
```

```shell
# 安装ceph-deploy

sudo yum -y install ceph-deploy

# 创建执行目录(ceph-admin上执行)

mkdir ~/ceph-cluster && cd ~/ceph-cluster

# 创建集群(ceph-admin上执行)

# 指定ceph-admin为初始监视器

ceph-deploy new ceph-admin

# 以下是创建集群信息以及新生成的文件

ceph.conf 		# 为 ceph 配置文件

ceph-deploy-ceph.log 	# 为 ceph-deploy 日志文件

ceph.mon.keyring  # 为ceph monitor 的密钥环
```

![img](https://longlizl.github.io/ceph分布式集群/images/8.png)

修改配置如下：

vim ceph.conf

![img](https://longlizl.github.io/ceph分布式集群/images/9.png)

**2. 通过 ceph-deploy部署工具 在各个节点安装ceph**

![img](https://longlizl.github.io/ceph分布式集群/images/10.png)

![img](https://longlizl.github.io/ceph分布式集群/images/11.png)

以上2报错信息通过单独安装epel-release 并 sudo yum remove ceph-release 再执行

ceph-deploy install ceph-admin ceph-node-1 ceph-node-2  可以解决

![img](https://longlizl.github.io/ceph分布式集群/images/12.png)

**解决办法**

此问题安装Deltarpm包（增量 RPM 套件）即可解决，当然您也可以先使用一下命令，查看是哪个包提供applydeltarpm

```shell
yum provides '*/applydeltarpm'   yum install deltarpm -y
```

![img](https://longlizl.github.io/ceph分布式集群/images/13.png)

解决办法都到对应节点下载相应包并更新epel-release为最新

**3. 初始化 monitor 节点并收集所有密钥**

ceph-deploy mon create-initial

![img](https://longlizl.github.io/ceph分布式集群/images/14.png)

**4. 添加2个OSD**

1. 登录到 ceph-node-1,ceph-node-2 节点、并给 OSD 守护进程创建一个目录，并添加权限。

```shell
ceph-node-1:

sudo mkdir /opt/osd1

sudo chown -R ceph:ceph /opt/osd1/
```

![img](https://longlizl.github.io/ceph分布式集群/images/15.png)

```shell
ceph-node-2:

sudo mkdir /opt/osd2

sudo chown -R ceph:ceph /opt/osd2/
```

![img](https://longlizl.github.io/ceph分布式集群/images/16.png)

1.   从管理节点ceph-admin执行  prepare OSD 操作

```shell
ceph-deploy osd prepare ceph-node-1:/opt/osd1 ceph-node-2:/opt/osd2
```

![img](https://longlizl.github.io/ceph分布式集群/images/17.png)

1. 激活 activate OSD

```shell
ceph-deploy osd activate ceph-node-1:/opt/osd1 ceph-node-2:/opt/osd2
```

![img](https://longlizl.github.io/ceph分布式集群/images/18.png)

1. 通过 ceph-deploy admin 将配置文件和 admin 密钥同步到各个节点

```shell
ceph-deploy admin ceph-admin ceph-node-1 ceph-node-2
```

![img](https://longlizl.github.io/ceph分布式集群/images/19.png)

1.  **确保你对 ceph.client.admin.keyring 有正确的操作权限**

sudo chmod +r /etc/ceph/ceph.client.admin.keyring

![img](https://longlizl.github.io/ceph分布式集群/images/20.png)

1. **查看集群状态**

```
ceph health
```

![img](https://longlizl.github.io/ceph分布式集群/images/21.png)

```shell
ceph -s
```

![img](https://longlizl.github.io/ceph分布式集群/images/22.png)

**查看集群 OSD 信息**

```shell
ceph osd tree
```

![img](https://longlizl.github.io/ceph分布式集群/images/23.png)

**查看集群 OSD 使用情况**

```shell
ceph osd df
```

![img](https://longlizl.github.io/ceph分布式集群/images/24.png)

**三: 扩展集群（扩容）**

**添加MONITORS**

在 cepth-node-1 和 ceph-node-2 添加监控节点。

1. 修改 mon_initial_members、mon_host 和 public network 配置：

![img](https://longlizl.github.io/ceph分布式集群/images/25.png)

1. 推送至其他节点：(注：因为之前将主节点也设置成了监控节点所以主节点也推送了配置)

```shell
ceph-deploy --overwrite-conf config push ceph-admin ceph-node-1 ceph-node-2
```

![img](https://longlizl.github.io/ceph分布式集群/images/26.png)

1. 添加监控节点:

```shell
ceph-deploy mon add ceph-node-1
ceph-deploy mon add ceph-node-2
```



1. 查看集群状态和监控节点：

集群状态

```shell
ceph -s
```

监控状态

```shell
ceph mon stat
```

![img](https://longlizl.github.io/ceph分布式集群/images/27.png)

**五: ceph-dash集群监控工具部署**

在监控节点安装ceph-dash

```shell
mkdir ceph-dash

cd ceph-dash/
```

![img](https://longlizl.github.io/ceph分布式集群/images/28.png)

安装git工具

```shell
yum -y install git

git clone https://github.com/Crapworks/ceph-dash.git

cd ceph-dash

nohup python ceph-dash.py &
```

![img](https://longlizl.github.io/ceph分布式集群/images/29.png)

访问：http://192.168.205.190:5000/

![img](https://longlizl.github.io/ceph分布式集群/images/30.png)

