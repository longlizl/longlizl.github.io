

# ELK日志系统

## 架构图

![image-20210825102151832](https://longlizl.github.io/ELK相关/images/1.png)

## 一. 系统环境需求

| 主机名      | ip              | 服务名                         | 操作系统   |
| ----------- | --------------- | ------------------------------ | ---------- |
| elk-01      | 192.168.205.132 | jdk-11.0.12,Elasticsearch 7.14 | centos 7.9 |
| elk-02      | 192.168.205.133 | jdk-11.0.12,Elasticsearch 7.14 | centos 7.9 |
| elk-03      | 192.168.205.134 | jdk-11.0.12,Elasticsearch 7.14 |        centos 7.9 |
| kibana      | 192.168.205.135 | kibana 7.14                    |       centos 7.9 |
| filebeat-01 | 192.168.205.136 | filebeat 7.14                  |       centos 7.9 |
| redis       | 192.168.205.137 | redis 5.0                      |       centos 7.9 |
| logstash    | 192.168.205.138 | jdk-11.0.12,logstash 7.14      |       centos 7.9 |

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

   上传jdk11安装包解压安装

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
cluster.initial_master_nodes: ["node-1","node-2","node-3"]
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
cluster.initial_master_nodes: ["node-1","node-2","node-3"]
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
cluster.initial_master_nodes: ["node-1","node-2","node-3"]
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

### 4. 浏览器访问集群

   点击右边菜单栏的搜索小图标，输入其中一节点小图标连接

   ![image-20210824175656307](https://longlizl.github.io/ELK相关/images/5.png)

   其中索引分片默认为5分片1副本，其中绿色方块边框未加粗的为副本分片
## 三. kibana安装

1. 下载安装kibana

   > kibana:
   >
   > ```shell
   > # 下载安装
   > curl -L -O https://artifacts.elastic.co/downloads/kibana/kibana-7.14.0-x86_64.rpm
   > rpm -ivh kibana-7.14.0-x86_64.rpm
   > # 修改配置
   > vim /etc/kibana/kibana.yml
   > server.port: 5601
   > server.host: "192.168.205.135"
   > # elasticsearch集群中所有可能成为master的节点列表
   > elasticsearch.hosts: ["http://192.168.205.132:9200","http://192.168.205.133:9200","http://192.168.205.134:9200"]
   > ```

2. 启动并访问kibana

   >  启动kibana:
   >
   >  ```shell
   >  systemctl enable kibana
   >  systemctl start kibana
   >  ```
   >
   >  访问kibana:

  ![image-20210826110536218](https://longlizl.github.io/ELK相关/images/6.png)

## 四. redis安装

1. 下载安装redis

   > 这里缓存队列只安装单机作为演示，有条件可以安装集群
   >
   > redis:
   >
   > ```shell
   > # 系统内核参数设置
   > cat >>/etc/sysctl.conf << "EOF"
   > net.core.somaxconn=1024
   > vm.overcommit_memory=1
   > EOF
   > sysctl -p
   > # 此设置需要重启服务器生效
   > cat >> /etc/rc.local << "EOF"
   > echo "never" > /sys/kernel/mm/transparent_hugepage/enabled
   > EOF
   > chmod a+x /etc/rc.d/rc.local
   > # 下载redis
   > yum -y install gcc
   > mkdir -p /usr/local/redis/{log,data,conf}
   > curl -L -O https://download.redis.io/releases/redis-5.0.13.tar.gz
   > tar -zxvf redis-5.0.13.tar.gz
   > # 编译安装
   > cd redis-5.0.13 && make -j2
   > cd src && make install PREFIX=/usr/local/redis
   > cd ../ && cp redis.conf /usr/local/redis/conf
   > ```


2. 配置redis

   > ```shell
   > # 修改配置如下
   > vim /usr/local/redis/redis.conf
   > bind 192.168.205.137
   > port 6379
   > daemonize yes
   > logfile "/usr/local/redis/log/redis.log"
   > dir "/usr/local/redis/data"
   > appendonly yes
   > requirepass 111111
   > ```

3. 启动redis

   > ```shell
   > # 启动redis
   > cd /usr/local/redis/
   > bin/redis-server /usr/local/redis/conf/redis.conf
   > ```

## 五. filebeat安装

​	安装filebeat前已经提前在需要收集日志的服务器安装好了nginx和tomcat

1. 下载filebeat

   > filebeat:
   >
   > ```shell
   > # 下载安装
   > curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.14.0-x86_64.rpm
   > rpm -ivh filebeat-7.14.0-x86_64.rpm
   > ```

2. 配置filebeat

   > 1. 配置filebeat采集nginx和tomcat日志
   >
   >    ```yaml
   >    cp /etc/filebeat/filebeat.yml /etc/filebeat/filebeat.yml.bak
   >    vim /etc/filebeat/filebeat.yml
   >    # ============================== Filebeat inputs ===============================
   >    
   >    filebeat.inputs:
   >    - type: log
   >      enabled: true
   >      paths:
   >        - /var/log/nginx/access.log
   >      fields:
   >        web: nginx-access
   >      fields_under_root: true
   >    
   >    - type: log
   >      enabled: true
   >      paths:
   >        - /var/log/nginx/error.log
   >      fields:
   >        web: nginx-error
   >      fields_under_root: true
   >    
   >    - type: log
   >      enabled: true
   >      paths:
   >        - /opt/apache-tomcat-10.0.10/logs/catalina.out
   >      fields:
   >        web: tomcat-log
   >      fields_under_root: true
   >      # Multiline options
   >      multiline.type: pattern
   >      multiline.pattern: ^\[
   >      multiline.negate: true
   >      multiline.match: after
   >    
   >    # ------------------------------ 日志配置 -----------------------------------
   >    
   >    logging.level: info
   >    logging.to_files: true
   >    logging.files:
   >      path: /var/log/filebeat
   >      name: filebeat
   >      keepfiles: 7
   >      permissions: 0644
   >    
   >    # ------------------------------ Redis Output -------------------------------
   >    
   >    output.redis:
   >    hosts: ["192.168.205.137:6379"]
   >    password: "111111"
   >    key: "filebeat"
   >    db: 0
   >    ```


3. 启动filebeat

   > ```shell
   > systemctl start filebeat
   > systemctl enable filebeat
   > ```

3. 查看日志数据是否输出到redis

   > ```shell
   > # 查看key的value列表1-5范围的值
   > [root@redis redis]# bin/redis-cli -h 192.168.205.137 -a 111111
   > Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
   > 192.168.205.137:6379> KEYS *
   > 1) "filebeat"
   > 192.168.205.137:6379> LLEN filebeat
   > (integer) 10
   > 192.168.205.137:6379> LRANGE filebeat 1 5
   > 1) "{\"@timestamp\":\"2021-08-26T08:17:32.965Z\",\"@metadata\":{\"beat\":\"filebeat\",\"type\":\"_doc\",\"version\":\"7.14.0\"},\"agent\":{\"name\":\"filebeat-01\",\"type\":\"filebeat\",\"version\":\"7.14.0\",\"hostname\":\"filebeat-01\",\"ephemeral_id\":\"60f9f9be-f50c-455a-a254-d689e818b156\",\"id\":\"dca8adca-31b9-4df8-8c0d-3f5314dfd0a6\"},\"log\":{\"file\":{\"path\":\"/var/log/nginx/error.log\"},\"offset\":71},\"message\":\"2021/08/26 15:09:42 [notice] 1654#1654: nginx/1.20.1\",\"input\":{\"type\":\"log\"},\"web\":\"nginx-error\",\"ecs\":{\"version\":\"1.10.0\"},\"host\":{\"name\":\"filebeat-01\"}}"
   > 2) "{\"@timestamp\":\"2021-08-26T08:17:32.965Z\",\"@metadata\":{\"beat\":\"filebeat\",\"type\":\"_doc\",\"version\":\"7.14.0\"},\"log\":{\"offset\":124,\"file\":{\"path\":\"/var/log/nginx/error.log\"}},\"message\":\"2021/08/26 15:09:42 [notice] 1654#1654: built by gcc 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC) \",\"input\":{\"type\":\"log\"},\"web\":\"nginx-error\",\"ecs\":{\"version\":\"1.10.0\"},\"host\":{\"name\":\"filebeat-01\"},\"agent\":{\"id\":\"dca8adca-31b9-4df8-8c0d-3f5314dfd0a6\",\"name\":\"filebeat-01\",\"type\":\"filebeat\",\"version\":\"7.14.0\",\"hostname\":\"filebeat-01\",\"ephemeral_id\":\"60f9f9be-f50c-455a-a254-d689e818b156\"}}"
   > 3) "{\"@timestamp\":\"2021-08-26T08:17:32.965Z\",\"@metadata\":{\"beat\":\"filebeat\",\"type\":\"_doc\",\"version\":\"7.14.0\"},\"ecs\":{\"version\":\"1.10.0\"},\"log\":{\"offset\":218,\"file\":{\"path\":\"/var/log/nginx/error.log\"}},\"message\":\"2021/08/26 15:09:42 [notice] 1654#1654: OS: Linux 3.10.0-1160.36.2.el7.x86_64\",\"input\":{\"type\":\"log\"},\"web\":\"nginx-error\",\"host\":{\"name\":\"filebeat-01\"},\"agent\":{\"id\":\"dca8adca-31b9-4df8-8c0d-3f5314dfd0a6\",\"name\":\"filebeat-01\",\"type\":\"filebeat\",\"version\":\"7.14.0\",\"hostname\":\"filebeat-01\",\"ephemeral_id\":\"60f9f9be-f50c-455a-a254-d689e818b156\"}}"
   > 4) "{\"@timestamp\":\"2021-08-26T08:17:32.965Z\",\"@metadata\":{\"beat\":\"filebeat\",\"type\":\"_doc\",\"version\":\"7.14.0\"},\"log\":{\"file\":{\"path\":\"/var/log/nginx/error.log\"},\"offset\":296},\"message\":\"2021/08/26 15:09:42 [notice] 1654#1654: getrlimit(RLIMIT_NOFILE): 1024:4096\",\"input\":{\"type\":\"log\"},\"web\":\"nginx-error\",\"ecs\":{\"version\":\"1.10.0\"},\"host\":{\"name\":\"filebeat-01\"},\"agent\":{\"name\":\"filebeat-01\",\"type\":\"filebeat\",\"version\":\"7.14.0\",\"hostname\":\"filebeat-01\",\"ephemeral_id\":\"60f9f9be-f50c-455a-a254-d689e818b156\",\"id\":\"dca8adca-31b9-4df8-8c0d-3f5314dfd0a6\"}}"
   > 5) "{\"@timestamp\":\"2021-08-26T08:17:32.965Z\",\"@metadata\":{\"beat\":\"filebeat\",\"type\":\"_doc\",\"version\":\"7.14.0\"},\"input\":{\"type\":\"log\"},\"web\":\"nginx-error\",\"ecs\":{\"version\":\"1.10.0\"},\"host\":{\"name\":\"filebeat-01\"},\"agent\":{\"version\":\"7.14.0\",\"hostname\":\"filebeat-01\",\"ephemeral_id\":\"60f9f9be-f50c-455a-a254-d689e818b156\",\"id\":\"dca8adca-31b9-4df8-8c0d-3f5314dfd0a6\",\"name\":\"filebeat-01\",\"type\":\"filebeat\"},\"log\":{\"offset\":372,\"file\":{\"path\":\"/var/log/nginx/error.log\"}},\"message\":\"2021/08/26 15:09:42 [notice] 1655#1655: start worker processes\"}"
   > ```

## 六. logstash安装   

1. 安装java环境

   > ```shell
   > # 上次jdk1安装包
   > tar -zxvf jdk-11.0.12_linux-x64_bin.tar.gz && mv jdk-11.0.12 /usr/local/jdk
   > cat >> /etc/profile << "EOF"
   > export JAVA_HOME=/usr/local/jdk
   > export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
   > export PATH=$JAVA_HOME/bin:$PATH
   > EOF
   > source /etc/profile
   > ```

2. 下载安装logstash

> ```shell
> wget -c https://artifacts.elastic.co/downloads/logstash/logstash-7.14.0-x86_64.rpm 
> rpm-ivh logstash-7.14.0-x86_64.rpm
> ```

3. 配置logstash

   > ```
   > 
   > ```
   >
   > 

   

4.  	

