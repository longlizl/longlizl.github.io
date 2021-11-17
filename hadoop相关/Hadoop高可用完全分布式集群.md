# Hadoop集群

## 服务及端口说明

1. HDFS端口

| 参数                      | 描述                          | 默认  | 配置文件       | 说明               |
| ------------------------- | ----------------------------- | ----- | -------------- | ------------------ |
| fs.default.name namenode  | namenode RPC交互端口          | 9000  | core-site.xml  | hdfs://master:9000 |
| dfs.http.address          | NameNode web管理端口          | 50070 | hdfs- site.xml | 0.0.0.0:50070      |
| dfs.datanode.address      | datanode　控制端口            | 50010 | hdfs -site.xml | 0.0.0.0:50010      |
| dfs.datanode.ipc.address  | datanode的RPC服务器地址和端口 | 50020 | hdfs-site.xml  | 0.0.0.0:50020      |
| dfs.datanode.http.address | datanode的HTTP服务器和端口    | 50075 | hdfs-site.xml  | 0.0.0.0:50075      |

2. MR端口

| 参数                             | 描述                   | 默认  | 配置文件        | 说明                |
| -------------------------------- | ---------------------- | ----- | --------------- | ------------------- |
| mapred.job.tracker               | job-tracker交互端口    | 8021  | mapred-site.xml | hdfs://master:8021/ |
| job                              | tracker的web管理端口   | 50030 | mapred-site.xml | 0.0.0.0:50030       |
| mapred.task.tracker.http.address | task-tracker的HTTP端口 | 50060 | mapred-site.xml | 0.0.0.0:50060       |

3. 其它端口

| 参数            | 描述            | 默认 | 配置文件      | 说明                                                         |
| --------------- | --------------- | ---- | ------------- | ------------------------------------------------------------ |
| yarn            | web             | 8088 | yarn-site.xml | http://hostname:8088高可用有一台会被重定向到另一台，需要将主机名与ip添加到本地hosts解析 |
| ResourceManager | ResourceManager | 8042 | yarn-site.xml |                                                              |
|                 |                 |      |               |                                                              |

![img](https://longlizl.github.io/hadoop相关/images/1.png)

![img](https://longlizl.github.io/hadoop相关/images/2.png)

- NameNode

> ```
> 1. 存储文件的metadata，运行时所有数据都保存到内存，整个HDFS可存储的文件数受限于NameNode的内存大小
> 2. 一个Block在NameNode中对应一条记录（一般一个block占用150字节），如果是大量的小文件，会消耗大量内存。同时map task的数量是由splits来决定的，所以用MapReduce处理大量的小文件时，就会产生过多的map task，线程管理开销将会增加作业时间。处理大量小文件的速度远远小于处理同等大小的大文件的速度。因此Hadoop建议存储大文件
> 3. 数据会定时保存到本地磁盘，但不保存block的位置信息，而是由DataNode注册时上报和运行时维护（NameNode中与DataNode相关的信息并不保存到NameNode的文件系统中，而是NameNode每次重启后，动态重建）
> 4. NameNode失效则整个HDFS都失效了，所以要保证NameNode的可用性
> ```

- Secondary NameNode

  > ```
  > 定时与NameNode进行同步（定期合并文件系统镜像和编辑日日志;然后把合并后的传给NameNode，替换其镜像，并清空编辑日志，类似于CheckPoint机制），但NameNode失效后仍需要手工将其设置成主机
  > ```

- DataNode

  > ```
  > 1. 保存具体的block数据
  > 2. 负责数据的读写操作和复制操作
  > 3. DataNode启动时会向NameNode报告当前存储的数据块信息，后续也会定时报告修改信息
  > 4. DataNode之间会进行通信，复制数据块，保证数据的冗余性
  > ```
- ResourceManager

> ResourceManager主要完成一下几个功能:
>
> ```
> 1. 与客户端交互，处理来自客户端的请求。
> 2. 启动和管理ApplicatinMaster,并在它运行失败时重新启动它。
> 3. 管理NodeManager，接受来自NodeManager的资源管理汇报信息，并向NodeManager下达管理命令等。
> 4. 资源管理与调度，接收来自ApplicationMaster的资源申请请求，并为之分配资源（核心）。
> ```

- NodeManager

  > ```
  > NodeManager是YARN中节点的代理， 它需要与应用程序的ApplicationMaster和集群管理者ResourceManager交互;它从ApplicationMaster上接收有关Container的命令并执行(比如启动、停止Contaner);向ResourceManager汇报各个Container运行状态和节点健康状况，并领取有关Container的命令（比如清理Container）。
  > ```

## 服务器功能规划：

| hadoopmaster | hadoop-node-1   | hadoop-node-2   | 备注                                                         |
| ------------ | --------------- | --------------- | ------------------------------------------------------------ |
| NameNode     | NameNode        |                 |                                                              |
|              | resourcemanager | resourcemanager | yarn-site.xml，这里配哪一台，哪一台启动ResourceManager       |
|              | NodeManager     | NodeManager     | DataNode、NodeManager决定于：slaves文件。（默认localhost，删掉即可）谁运行dataNode，slaves文件写谁。当namenode跑的时候，会通过配置文件开始扫描slaves文件，slaves文件有谁，谁启动dataNode.当启动yarn时，会通过扫描配置文件开始扫描slaves文件，slaves文件有谁，谁启动NodeManager |
| ZKFC         | ZKFC            |                 | ZKFC对Namenode状态监控并故障转移                             |
|              | DataNode        | DataNode        |                                                              |
| journalnode  | journalnode     | journalnode     | jounalnode集群（在hadoop高可用中两个namenode节点为了数据同步会通过Journalnode相互通信。） |
| zookeeper    | zookeeper       | zookeeper       | zookeeper集群                                                |

# 一.环境准备

```shell
Hadoop Master：192.168.205.175（hadoop-master） hostname:hadoop-master
Hadoop Slave：192.168.205.180（hadoop-slave-1） hostname:hadoop-node-1
Hadoop Slave：192.168.205.185（hadoop-slave-2） hostname:hadoop-node-2
JDK1.8版本，系统：centos8.2.2004
```



- 各个节点都设置对应hostname及hosts主机与ip的映射关系

```
现以hadoop-master这台主节点为例，其他节点按照主节点设置即可：
hostnamectl set-hostname hadoop-master（设置完后断开会话重新进入即可）
```

- 设置ip与主机名的映射关系：

vim /etc/hosts 

```shell
192.168.205.175 hadoop-master 
192.168.205.180 hadoop-node-1 
192.168.205.185 hadoop-node-2
```

## 1. JDK安装，环境变量配置

官网下载并上传到linux

```shell
tar -zxvf jdk-8u191-linux-x64.tar.gz
mv jdk-8u191-linux-x64 /usr/local/jdk
```

设置java环境及hadoop相关环境变量在/etc/profile文件末尾加上如下配置并加载环境变量

```shell
export JAVA_HOME=/usr/local/jdk
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export HADOOP_HOME=/opt/hadoop
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export HADOOP_CLASSPATH=`${HADOOP_HOME}/bin/hadoop classpath`
export PATH=$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
. /etc/profile
```

## 设置各节点免密互访（各节点都执行如下操作）

```shell
ssh-keygen
ssh-copy-id root@hadoop-master
ssh-copy-id root@hadoop-node-1
ssh-copy-id root@hadoop-node-2
```

# 二.zookeeper集群部署

## 1.下载安装zookeeper

```shell
curl -O https://mirror.bit.edu.cn/apache/zookeeper/zookeeper-3.6.2/apache-zookeeper-3.6.2-bin.tar.gz
tar -zxvf apache-zookeeper-3.6.2-bin.tar.gz 
mv apache-zookeeper-3.6.2-bin /opt/zookeeper
cd /opt/zookeeper/conf &&  cp zoo_sample.cfg zoo.cfg
```

##  2. 修改配置文件

```shell
mkdir -p /opt/zookeeper/{data,datalogs}
vim zoo.cfg
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/opt/zookeeper/data
dataLogDir=/opt/zookeeper/datalogs
clientPort=2181
server.1=hadoop-master:2888:3888
server.2=hadoop-node-1:2888:3888
server.3=hadoop-node-2:2888:3888
```

​	将配置文件zoo.cfg拷贝其他2节点

## 3.新建myid文件

```shell
# 在各节点写入对应的server.id
cd /opt/zookeeper/data
# 节点：hadoop-master
echo 1 >  myid
# 节点：hadoop-node-1
echo 2 >  myid
# 节点：hadoop-node-2
echo  3 > myid
```

# 三.启动zookeeper集群(一台台启动)

```shell
cd /opt/zookeeper/bin
./zkServer.sh start
```

​	查看各节点状态

![img](https://longlizl.github.io/hadoop相关/images/3.png)

![img](https://longlizl.github.io/hadoop相关/images/4.png)

![img](https://longlizl.github.io/hadoop相关/images/5.png)

# 四.搭建hadoop集群

1. 官方下载并安装（以主节点为例配置完一台将配置文件拷贝到其他节点）

```shell
wget -c https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/hadoop-2.10.1/hadoop-2.10.1.tar.gz
scp hadoop-2.10.1.tar.gz root@hadoop-node-1:/root/
scp hadoop-2.10.1.tar.gz root@hadoop-node-2:/root/
tar -zxvf apache-zookeeper-3.6.2-bin.tar.gz
tar -zxvf hadoop-2.10.1.tar.gz 
mv hadoop-2.10.1 /opt/hadoop
```

2. 修改如下配置文件

  > 1. hadoop-env.sh
  > 2. core-site.xml
  > 3. hdfs-site.xml
  > 4. mapred-site.xml
  > 5. yarn-site.xml
  > 6. slaves
- 1. hadoop-env.sh
```shell
cd /opt/hadoop/etc/hadoop
vim hadoop-env.sh
```

![img](https://longlizl.github.io/hadoop相关/images/6.png)
-  2. core-site.xml:

​	     vim core-site.xml

```xml
在<configuration></configuration>中间加上如下配置：
<!-- 指定hdfs的nameservice为cluster1 -->
<property>
    <name>fs.defaultFS</name>
    <value>hdfs://cluster1</value>
</property>
<!--指定hadoop数据临时存放目录-->
<property>
    <name>hadoop.tmp.dir</name>
    <value>/opt/hadoop/tmp</value>
</property>
<!--指定hdfs操作数据的缓冲区大小 可以不配-->
<property>
    <name>io.file.buffer.size</name>
    <value>4096</value>
</property>
<!--指定zookeeper地址-->
<property>
    <name>ha.zookeeper.quorum</name>
    <value>hadoop-master:2181,hadoop-node-1:2181,hadoop-node-2:2181</value>
</property>
```

- 3. hdfs-site.xml

     vim hdfs-site.xml

```xml
在<configuration></configuration>中间加上如下配置：
<!--指定hdfs的nameservice为cluster1，需要和core-site.xml中的保持一致 -->
<property>
    <name>dfs.nameservices</name>
    <value>cluster1</value>
</property>
<!-- cluster1下面有两个NameNode，分别是nn1,nn2 这里为逻辑名随便命名 -->
<property>
    <name>dfs.ha.namenodes.cluster1</name>
    <value>nn1,nn2</value>
</property>
<!-- nn1的RPC通信地址 -->
<property>
    <name>dfs.namenode.rpc-address.cluster1.nn1</name>
    <value>hadoop-master:9000</value>
</property>
<!-- nn1的http通信地址 -->
<property>
    <name>dfs.namenode.http-address.cluster1.nn1</name>
    <value>hadoop-master:50070</value>
</property>
<!-- nn2的RPC通信地址 -->
<property>
    <name>dfs.namenode.rpc-address.cluster1.nn2</name>
    <value>hadoop-node-1:9000</value>
</property>
<!-- nn2的http通信地址 -->
<property>
    <name>dfs.namenode.http-address.cluster1.nn2</name>
    <value>hadoop-node-1:50070</value>
</property>
<!-- 指定NameNode的元数据在JournalNode上的存放位置 -->
<property>
    <name>dfs.namenode.shared.edits.dir</name>
    <value>qjournal://hadoop-master:8485;hadoop-node-1:8485;hadoop-node-2:8485/cluster1</value>
</property>
<!-- 指定JournalNode在本地磁盘存放数据的位置 -->
<property>
    <name>dfs.journalnode.edits.dir</name>
    <value>/opt/hadoop/journaldata</value>
</property>
<!-- 开启NameNode故障时自动切换 -->
<property>
    <name>dfs.ha.automatic-failover.enabled</name>
    <value>true</value>
</property>
<!-- 配置失败自动切换实现方式 -->
<property>
    <name>dfs.client.failover.proxy.provider.cluster1</name>
    <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
</property>
<!-- 配置隔离机制 -->
<property>
    <name>dfs.ha.fencing.methods</name>
    <value>sshfence</value>
</property>
<!-- 使用隔离机制时需要ssh免登陆 -->
<property>
    <name>dfs.ha.fencing.ssh.private-key-files</name>
    <value>/root/.ssh/id_rsa</value>
</property>
<!-- namenode存储位置 -->
<property>
    <name>dfs.namenode.name.dir</name>
    <value>/opt/hadoop/namenodedata</value>
</property>
<!-- dataode存储位置 -->
<property>
    <name>dfs.datanode.data.dir</name>
    <value>/opt/hadoop/datanodedate</value>
</property>
<!-- 副本数量根据自己的需求配置，这里配置2个 -->
<property>
    <name>dfs.replication</name>
    <value>2</value>
</property>
<!-- 在NN和DN上开启WebHDFS (REST API)功能,不是必须 -->
<property>
    <name>dfs.webhdfs.enabled</name>
    <value>true</value>
</property>
```

- 4. mapred-site.xml

     vim mapred-site.xml

```xml
在<configuration></configuration>中间加上如下配置：
<property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
</property>
```

- 5. yarn-site.xml

  ​       vim yarn-site.xml

```xml
在<configuration></configuration>中间加上如下配置：
<!-- reducer获取数据的方式 -->
<property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
</property>
<!--启用resourcemanager ha-->
<property>
    <name>yarn.resourcemanager.ha.enabled</name>
    <value>true</value>
</property>
<!--声明两台resourcemanager的地址-->
<property>
    <name>yarn.resourcemanager.cluster-id</name>
    <value>rmCluster</value>
</property>
<property>
    <name>yarn.resourcemanager.ha.rm-ids</name>
    <value>rm1,rm2</value>
</property>
<property>
    <name>yarn.resourcemanager.hostname.rm1</name>
    <value>hadoop-node-1</value>
</property>
<property>
    <name>yarn.resourcemanager.hostname.rm2</name>
    <value>hadoop-node-2</value>
</property>
<!--指定zookeeper集群的地址-->
<property>
    <name>yarn.resourcemanager.zk-address</name>
    <value>hadoop-master:2181,hadoop-node-1:2181,hadoop-node-2:2181</value>
</property>
<!--启用自动恢复-->
<property>
    <name>yarn.resourcemanager.recovery.enabled</name>
    <value>true</value>
</property>
<!--指定resourcemanager的状态信息存储在zookeeper集群-->
<property>
    <name>yarn.resourcemanager.store.class</name>
    <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
</property>
```

- 6. slaves

  ​      写入datanode节点主机名：
  
  ​      vim slaves

```shell
hadoop-node-1
hadoop-node-2
```

# 五. 启动hadoop

> 分别启动各节点资源

```shell
启动journalnode集群（第一次每个节点都需要执行）
cd /opt/hadoop/sbin
./hadoop-daemons.sh start journalnode
```

1. **格式化zkfc**

```shell
# 第一次启动要格式化
cd /opt/hadoop/bin
./hdfs zkfc -formatZK
```

2. **格式化hdfs**

```shell
# 第一次启动要格式化
cd /opt/hadoop/bin
./hdfs namenode -format
```

3. **启动NameNode**

```shell
在hadoop-master上：
cd /opt/hadoop/sbin
./hadoop-daemon.sh start namenode
在hadoop-node-1上：
cd /opt/hadoop/bin
./hdfs namenode -bootstrapStandby #把NameNode的数据同步到hadoop-node-1上
cd /opt/hadoop/sbin  
./hadoop-daemon.sh start namenode #启动备用的namenode 
```

4. **启动DataNode(在任意节点执行会同时启动2数据节点)**

```shell
cd /opt/hadoop/sbin
./hadoop-daemons.sh start datanode
```

5. **启动ResourceManager，yarn**

```shell
# 先启动2节点的ResourceManager（再任意节点启动可以启动2节点resourcemanager）
cd /opt/hadoop/sbin
./yarn-daemon.sh start resourcemanager
```

​	**然后主节点启动：**

```shell
cd /opt/hadoop/sbin
./start-yarn.sh
```

6. **启动ZKFC**

     在hadoop-master

```shell
cd /opt/hadoop/sbin
./hadoop-daemon.sh start zkfc
```

​	   在hadoop-node-1

```shell
cd /opt/hadoop/sbin
./hadoop-daemon.sh start zkfc
```

---

>  快速启动hadoop服务：

1. **格式化zkfc**

```shell
# 第一次启动要格式化
cd /opt/hadoop/bin
./hdfs zkfc -formatZK
```

2. **格式化hdfs**

```shell
# 第一次启动要格式化
cd /opt/hadoop/bin
./hdfs namenode -format
```

3. **启动NameNode**

   在hadoop-master上：

```shell
cd /opt/hadoop/bin
./hdfs --daemon start namenode
```

​	   在hadoop-node-1上：

```shell
cd /opt/hadoop/bin
./hdfs namenode -bootstrapStandby #把NameNode的数据同步到hadoop-node-1上
cd /opt/hadoop/sbin
./hdfs --daemon start namenode #启动备用的namenode
```

​	  停止2台NameNode(主要先同步主节点数据然后重启)

```shell
cd /opt/hadoop/bin
./hdfs --daemon stop namenode
```

4. **启动2节点resourcemanager服务**

```shell
# 在任意节点启动会启动所有resourcemanager节点
cd /opt/hadoop/sbin
./yarn-daemon.sh start resourcemanager
```

5. **在主namenode节点启动所有资源(快速启动)**

```shell
cd /opt/hadoop/sbin
./start-all.sh
```

![img](https://longlizl.github.io/hadoop相关/images/7.png)

# 各节点启动进程如图

hadoop-master:

![img](https://longlizl.github.io/hadoop相关/images/8.png)

hadoop-node-1:

![img](https://longlizl.github.io/hadoop相关/images/9.png)

hadoop-node-2:

![img](https://longlizl.github.io/hadoop相关/images/10.png)

---

# 新版本启动报错解决办法

![img](https://longlizl.github.io/hadoop相关/images/11.png)

 输入如下命令，在环境变量中添加下面的配置（其中用户根据所在服务启动的用户修改）

```shell
vim /etc/profile
export HDFS_NAMENODE_USER=root export HDFS_DATANODE_USER=root
export HDFS_JOURNALNODE_USER=root
export HDFS_ZKFC_USER=root 
export HDFS_SECONDARYNAMENODE_USER=root 
export YARN_RESOURCEMANAGER_USER=root 
export YARN_NODEMANAGER_USER=root 
```

 