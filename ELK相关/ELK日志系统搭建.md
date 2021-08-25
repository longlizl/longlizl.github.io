

# ELK日志系统

## 架构图

![image-20210825102151832](https://longlizl.github.io/ELK相关/images/1.png)

## 一. 系统环境需求

| 主机名      | ip              | 服务名                         |
| ----------- | --------------- | ------------------------------ |
| elk-01      | 192.168.205.132 | jdk-11.0.12,Elasticsearch 7.14 |
| elk-02      | 192.168.205.133 | jdk-11.0.12,Elasticsearch 7.14 |
| elk-03      | 192.168.205.134 | jdk-11.0.12,Elasticsearch 7.14 |
| kinbana     | 192.168.205.135 | jdk-11.0.12,kinbana 7.14       |
| filebeat-01 | 192.168.205.136 | jdk-11.0.12,filebeat 7.14      |
| redis       | 192.168.205.137 | redis 5.0                      |
| logstash    | 192.168.205.138 | jdk-11.0.12,logstash 7.14      |

1. 关闭防火墙和selinux

2. 基本系统参数调整

```shell
cat >> /etc/sysctl.conf << "EOF"
vm.max_map_count=655360
EOF
sysctl -p 
cp /etc/security/limits.conf /etc/security/limits.conf.bak
cp /etc/security/limits.d/20-nproc.conf /etc/security/limits.d/20-nproc.conf.bak
echo -e "* soft nofile 102400\n* hard nofile 102401" >> /etc/security/limits.conf
echo -e "* soft nproc 65536\n* hard nproc 65536\nroot soft nproc 70000\nroot hard nproc unlimited" >> /etc/security/limits.d/20-nproc.conf
```

 添加hosts解析

```shell
cat >> /etc/hosts << "EOF"
192.168.205.132    elk-01
192.168.205.133    elk-02
192.168.205.134    elk-03
EOF
```

4. java环境安装

   上传jdk8安装包解压安装

```shell
tar -zxvf jdk-11.0.12_linux-x64_bin.tar.gz && mv jdk-11.0.12 /usr/local/jdk
cat >> /etc/profile << "EOF"
export JAVA_HOME=/usr/local/jdk
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$JAVA_HOME/bin:$PATH
EOF
source /etc/profile
```
## 二. elasticsearch集群安装

### 1. 安装配置

elk-01:

```yaml
# 下载安装
curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.14.0-x86_64.rpm
rpm -ivh elasticsearch-7.14.0-x86_64.rpm
# 设置开机启动
systemctl daemon-reload
systemctl enable elasticsearch
# 配置elasticsearch
vim /etc/elasticsearch/elasticsearch.yml
cluster.name: mycluster
node.name: node-1
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 192.168.205.132
http.port: 9200
# 集群主机列表
discovery.seed_hosts: ["elk-01","elk-02","elk-03"]
# 可被选举为主节点的列表，注：列表中为各节点的node.name
cluster.initial_master_nodes: ["node-1","node-2","node-03"]
```

elk-02:

```yaml
# 下载安装
curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.14.0-x86_64.rpm
rpm -ivh elasticsearch-7.14.0-x86_64.rpm
# 设置开机启动
systemctl daemon-reload
systemctl enable elasticsearch
# 配置elasticsearch
vim /etc/elasticsearch/elasticsearch.yml
cluster.name: mycluster
node.name: node-2
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 192.168.205.133
http.port: 9200
# 集群主机列表
discovery.seed_hosts: ["elk-01","elk-02","elk-03"]
# 可被选举为主节点的列表，注：列表中为各节点的node.name
cluster.initial_master_nodes: ["node-1","node-2","node-03"]
```

elk-03:

```yaml
# 下载安装
curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.14.0-x86_64.rpm
rpm -ivh elasticsearch-7.14.0-x86_64.rpm
# 设置开机启动
systemctl daemon-reload
systemctl enable elasticsearch
# 配置elasticsearch
vim /etc/elasticsearch/elasticsearch.yml
cluster.name: mycluster
node.name: node-3
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 192.168.205.134
http.port: 9200
# 集群主机列表
discovery.seed_hosts: ["elk-01","elk-02","elk-03"]
# 可被选举为主节点的列表，注：列表中为各节点的node.name
cluster.initial_master_nodes: ["node-1","node-2","node-03"]
```

各个节点启动elasticsearch

```
systemctl start elasticsearch
```

### 2. 状态查询

```shell
集群健康状态：
curl 192.168.205.132:9200/_cat/health?v
集群的节点列表：
curl 192.168.205.132:9200/_cat/nodes?v
查看全部索引：
curl 192.168.205.132:9200/_cat/indices?v
```

![image-20210824164142781](https://longlizl.github.io/ELK相关/images/2.png)

### 3. 安装es-head插件

1. github上下在google插件

   https://github.com/liufengji/es-head

2. 安装插件

   下载后解压将elasticsearch-head.crx文件重命名为elasticsearch-head.rar再解压压一次

   打开google的扩展程序-调成开发者模式-点击加载已解压的扩展程序-选择解压后的目录

   ![image-20210824175152616](https://longlizl.github.io/ELK相关/images/3.png)

   再打开扩展程序将插件固定到菜单栏上

   ![image-20210824175456318](https://longlizl.github.io/ELK相关/images/4.png)

   ### 3. 浏览器访问集群

   点击右边菜单栏的搜索小图标，输入其中一节点小图标连接

   ![image-20210824175656307](https://longlizl.github.io/ELK相关/images/5.png)

   其中索引分片默认为5分片1副本，其中绿色方块边框未加粗的为副本分片
## 三. kinbana安装

kinbana:
	pass
............
............
............	

