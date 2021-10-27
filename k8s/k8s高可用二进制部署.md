## k8s高可用集群

组件介绍

> Master组件
>
> Node组件

**Master组件**

- kube-apiserver

```
Kubernetes API，集群的统一入口，各组件协调者，以RESTful API提供接口服务，所有对象资源的增删改查和监听操作都交给APIServer处理后再提交给Etcd存储。
```

- kube-controller-manager

```
集群内部的管理控制中心，其主要目的是实现Kubernetes集群的故障检测和恢复的自动化工作，比如根据RC的定义完成Pod的复制或移除，以确保Pod实例数符合RC副本的定义；根据Service与Pod的管理关系，完成服务的Endpoints对象的创建和更新；其他诸如Node的发现、管理和状态监控、死亡容器所占磁盘空间及本地缓存的镜像文件的清理等工作也是由Controller Manager完成的。
```

- kube-scheduler

```
根据调度算法为新创建的Pod选择一个Node节点，可以任意部署,可以部署在同一个节点上,也可以部署在不同的节点上。
```

- etcd

```
用于保存集群所有的网络配置和对象的状态信息
```

**Node组件**

- kubelet

```
负责本Node节点上的Pod的创建、修改、监控、删除等全生命周期管理，同时Kubelet定时“上报”本Node的状态信息到API Server里。
```

- kube-proxy

```
在Node节点上实现Pod网络代理，维护网络规则和四层负载均衡工作。
```

- docker

```
容器引擎
```



### 一. 环境准备

 先安装一个单master节点集群，后面再扩展多master 

1. 主机规划

   | 主机名        | IP                                      | 组件                                                         |
   | ------------- | --------------------------------------- | ------------------------------------------------------------ |
   | k8s-master-01 | 192.168.205.150                         | kube-apiserver，kube-controller-manager，kube-scheduler，etcd |
   | k8s-master-02 | 192.168.205.151                         | kube-apiserver，kube-controller-manager，kube-scheduler，etcd |
   | k8s-node-03   | 192.168.205.152                         | kubelet，kube-proxy，docker etcd                             |
   | k8s-node-04   | 192.168.205.154                         | kubelet，kube-proxy，docker                                  |
   | 负载均衡      | 192.168.205.155 ，192.168.205.157 (VIP) | Nginx,keeplived                                              |
   | 负载均衡      | 192.168.205.156                         | Nginx,keeplived                                              |

2. 系统环境配置

   2.1 添加hosts解析

   ```shell
   cat >> /etc/hosts << 'EOF'
   192.168.205.150  K8S-master-01
   192.168.205.151  K8S-master-02
   192.168.205.152  K8S-node-03
   192.168.205.154  K8S-node-04
   EOF
   ```

   2.2 关闭firewalld,selinux,swap

   ```shell
   # 禁用防火墙
   systemctl stop firewalld && systemctl disable firewalld
   # 禁用selinux
   setenforce 0
   sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
   # 禁用swap
   swapoff -a
   ```

   2.3 添加K8S内核参数文件

   ```shell
   # 添加内核模块
   modprobe br_netfilter
   # 添加参数
   cat >> /etc/sysctl.conf << 'EOF'
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   net.ipv4.ip_forward = 1
   EOF
   # 使参数生效
   sysctl -p
   ```

### 二. 生成集群证书

1. 安装cfssl证书生成工具

   选择一台机器制作证书（这里在K8S-master-01节点操作）

   ```shell
   # 下载cfssl证书制作工具
   wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
   wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
   wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
   # 安装
   chmod +x cfssl* 
   mv cfssl_linux-amd64 /usr/local/bin/cfssl
   mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
   mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
   ```

2. 生成etcd证书

   2.1 创建证书目录（这里有2套证书一套etcd用得，和一套k8s各个组件用的）

   ​      mkdir -p /opt/ssl/{etcd,k8s}

   ​      cd /opt/ssl/etcd

   2.2 创建ca证书配置文件及证书签名请求文件

   ​      可以使用命令生成模板然后修改配置 

   ```
   cfssl print-defaults config > ca-config.json
   cfssl print-defaults csr > ca-csr.json
   ```

   ​      修改对应配置文件

   ​      vim ca-config.json

   ```json
   {
       "signing": {
           "default": {
               "expiry": "876000h"
           },
           "profiles": {
               "www": {
                   "expiry": "87600h",
                   "usages": [
                       "signing",
                       "key encipherment",
                       "server auth",
                       "client auth"
                   ]
               }
           }
       }
   }
   
   ```

   ​    vim ca-csr.json 

   ```json
   {
       "CN": "ETCD",
       "key": {
           "algo": "rsa",
           "size": 2048
       },
       "names": [
           {
               "C": "CN",
               "L": "WH",
               "ST": "WH"
           }
       ]
   }
   ```

   ​      生成证书

   ```shell
   cfssl gencert -initca ca-csr.json | cfssljson -bare ca
   ```

   ​      使用CA签发Etcd https证书

   ```json
   cat > server-csr.json << EOF
   {
       "CN": "etcd",
       "hosts": [
       "192.168.205.150",
       "192.168.205.151",
       "192.168.205.152",
       "192.168.205.153",
       "192.168.205.154",
       "192.168.205.155",
       "192.168.205.156",
       "192.168.205.157"
       ],
       "key": {
           "algo": "rsa",
           "size": 2048
       },
       "names": [
           {
               "C": "CN",
               "L": "WH",
               "ST": "WH"
           }
       ]
   }
   EOF
   # 其中hosts字段中IP是etcd节点的集群内部通信IP，为方便后期扩容可以多预留几个IP。
   ```

   ​      生成证书

   ```shell
   cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server
   ```

   2.3 生成kube-apiserver证书

   ```shell
   cd /opt/ssl/k8s
   
   1. 签发ca证书
   cat > ca-config.json << EOF
   {
     "signing": {
       "default": {
         "expiry": "87600h"
       },
       "profiles": {
         "kubernetes": {
            "expiry": "87600h",
            "usages": [
               "signing",
               "key encipherment",
               "server auth",
               "client auth"
           ]
         }
       }
     }
   }
   EOF
   
   cat > ca-csr.json << EOF
   {
       "CN": "kubernetes",
       "key": {
           "algo": "rsa",
           "size": 2048
       },
       "names": [
           {
               "C": "CN",
               "L": "WH",
               "ST": "WH",
               "O": "k8s",
               "OU": "System"
           }
       ]
   }
   EOF
   
   2. 生成证书：
   cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
   3. 签发kube-apiserver HTTPS证书
   cat > server-csr.json << EOF
   {
       "CN": "kubernetes",
       "hosts": [
         "10.0.0.1",
         "127.0.0.1",
         "192.168.205.150",
         "192.168.205.151",
         "192.168.205.152",
         "192.168.205.153",
         "192.168.205.154",
         "192.168.205.155",
         "192.168.205.156",
         "192.168.205.157",
         "192.168.205.158",
         "192.168.205.159",
         "192.168.205.160",
         "kubernetes",
         "kubernetes.default",
         "kubernetes.default.svc",
         "kubernetes.default.svc.cluster",
         "kubernetes.default.svc.cluster.local"
       ],
       "key": {
           "algo": "rsa",
           "size": 2048
       },
       "names": [
           {
               "C": "CN",
               "L": "BeiJing",
               "ST": "BeiJing",
               "O": "k8s",
               "OU": "System"
           }
       ]
   }
   EOF
   * 这里hosts包括所有节点ip还有vip，多预留几个
   
   # 生成证书
   cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server
   
   ```

   2.4 生成kube-proxy证书

   ```shell
   # 创建证书请求文件
   cat > kube-proxy-csr.json << EOF
   {
     "CN": "system:kube-proxy",
     "hosts": [],
     "key": {
       "algo": "rsa",
       "size": 2048
     },
     "names": [
       {
         "C": "CN",
         "L": "WH",
         "ST": "WH",
         "O": "k8s",
         "OU": "System"
       }
     ]
   }
   EOF
   # 生成证书
   cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
   ```


### 三. 部署集群

1. 所有节点安装docker

   > 添加阿里云镜像站点

   ```shell
   yum install -y yum-utils
   yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
   yum -y install docker-ce
   systemctl enable docker && systemctl start docker
   ```

   > 添加阿里云docker仓库加速器 

   ```shell
   tee /etc/docker/daemon.json <<-'EOF'
   {
     "registry-mirrors": ["https://fl791z1h.mirror.aliyuncs.com"]
   }
   EOF
   systemctl daemon-reload
   systemctl restart docker
   ```

2. ETCD集群部署

   2.1 在K8S-master-01,K8S-master-02,K8S-node-03部署安装ETCD集群

   ```shell
   # 下载安装包
   wget -c https://github.com/etcd-io/etcd/releases/download/v3.4.18/etcd-v3.4.18-linux-amd64.tar.gz
   
   # 新建etcd配置文件，证书存储目录
   mkdir /opt/etcd/{bin,conf,ssl} -p
   mv etcd-v3.4.18-linux-amd64/{etcd,etcdctl} /opt/etcd/bin/
   
   # 创建etcd配置文件
   cat > /opt/etcd/conf/etcd.conf << 'EOF'
   #[Member]
   ETCD_NAME="etcd-1"
   ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
   ETCD_LISTEN_PEER_URLS="https://192.168.205.150:2380"
   ETCD_LISTEN_CLIENT_URLS="https://192.168.205.150:2379"
   
   #[Clustering]
   ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.205.150:2380"
   ETCD_ADVERTISE_CLIENT_URLS="https://192.168.205.150:2379"
   ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.205.150:2380,etcd-2=https://192.168.205.151:2380,etcd-3=https://192.168.205.152:2380"
   ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
   ETCD_INITIAL_CLUSTER_STATE="new"
   EOF
   
   # 配置文件说明
   ETCD_NAME：节点名称，集群中唯一
   ETCD_DATA_DIR：数据存储目录
   ETCD_LISTEN_PEER_URLS：集群通信监听地址
   ETCD_LISTEN_CLIENT_URLS：客户端访问监听地址
   ETCD_INITIAL_ADVERTISE_PEER_URLS：集群通告地址
   ETCD_ADVERTISE_CLIENT_URLS：客户端通告地址
   ETCD_INITIAL_CLUSTER：集群节点地址
   ETCD_INITIAL_CLUSTER_TOKEN：集群Token
   ETCD_INITIAL_CLUSTER_STATE：加入集群的当前状态，new是新集群，existing表示加入已有集群
   
   # 配置systemd管理（其他2节点做相同操作）
   cat > /usr/lib/systemd/system/etcd.service << 'EOF'
   [Unit]
   Description=Etcd Server
   After=network.target
   After=network-online.target
   Wants=network-online.target
   
   [Service]
   Type=notify
   EnvironmentFile=/opt/etcd/conf/etcd.conf
   ExecStart=/opt/etcd/bin/etcd \
   --cert-file=/opt/etcd/ssl/server.pem \
   --key-file=/opt/etcd/ssl/server-key.pem \
   --peer-cert-file=/opt/etcd/ssl/server.pem \
   --peer-key-file=/opt/etcd/ssl/server-key.pem \
   --trusted-ca-file=/opt/etcd/ssl/ca.pem \
   --peer-trusted-ca-file=/opt/etcd/ssl/ca.pem \
   --logger=zap
   Restart=on-failure
   LimitNOFILE=65536
   
   [Install]
   WantedBy=multi-user.target
   EOF
   # 拷贝证书到/opt/etcd/ssl目录下
   cp /opt/ssl/etcd/ca*pem /opt/ssl/etcd/server*pem /opt/etcd/ssl/
   # 将/opt/etcd目录拷贝到其他2个ETCD节点并修改相应配置
   192.168.205.151节点：
   scp -r /opt/etcd/ 192.168.205.151:/opt/
   sed -i '1,2s/etcd-1/etcd-2/g;4,9s/192.168.205.150/192.168.205.151/g' /opt/etcd/conf/etcd.conf
   192.168.205.152节点：
   scp -r /opt/etcd/ 192.168.205.152:/opt/
   sed -i '1,2s/etcd-1/etcd-3/g;4,9s/192.168.205.150/192.168.205.152/g' /opt/etcd/conf/etcd.conf
   ```

   2.2 启动所有etcd节点并设置开机启动

   ```shell
   systemctl daemon-reload
   systemctl enable etcd
   systemctl start etcd
   ```

   2.3 查看ETC集群健康状态

   ```shell
   [root@k8s-master-01 etcd]# /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://192.168.205.150:2379,https://192.168.205.151:2379,https://192.168.205.152:2379" endpoint health
   https://192.168.205.152:2379 is healthy: successfully committed proposal: took = 15.433801ms
   https://192.168.205.150:2379 is healthy: successfully committed proposal: took = 58.876777ms
   https://192.168.205.151:2379 is healthy: successfully committed proposal: took = 53.78972ms
   ```

3. 部署Master,Node

   3.1 下载软件包

   ```shell
   # 下载包已包含master和node节点组件
   wget https://storage.googleapis.com/kubernetes-release/release/v1.18.3/kubernetes-server-linux-amd64.tar.gz
   # 新建k8s安装目录
   mkdir -p /opt/kubernetes/{bin,conf,ssl,logs}
   cp kube-apiserver kube-scheduler kube-controller-manager /opt/kubernetes/bin
   cp kubectl /usr/bin
   ```

   3.2 部署api-server

   ​     拷贝/opt/ssl/k8s目录下证书文件

   ```shell
   cp /opt/ssl/k8s/ca*pem /opt/ssl/k8s/server*pem /opt/kubernetes/ssl
   ```

> 启用TLS Bootstrapping机制 
   ​TLS Bootstraping：Master apiserver启用TLS认证后，Node节点kubelet和kube-proxy要与kube-apiserver进行通信，必须使用 CA签发的有效证书才可以，当Node节点很多时，这种客户端证书颁发需要大量工作，同样也会增加集群扩展复杂度。

   ```shell
   # 生成token
   head -c 16 /dev/urandom | od -An -t x | tr -d ' '  
   d63332b37383e5484dec574ed402b660
   # 创建token文件(格式：token，用户名，UID，用户组)
   cat > /opt/kubernetes/conf/token.csv << EOF
   d63332b37383e5484dec574ed402b660,kubelet-bootstrap,10001,"system:node-bootstrapper"
   EOF
   ```

​		3.2.1 创建配置文件

```shell
cat > /opt/kubernetes/conf/kube-apiserver.conf << 'EOF'
KUBE_APISERVER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--etcd-servers=https://192.168.205.150:2379,https://192.168.205.151:2379,https://192.168.205.152:2379 \
--bind-address=192.168.205.150 \
--secure-port=6443 \
--advertise-address=192.168.205.150 \
--allow-privileged=true \
--service-cluster-ip-range=10.0.0.0/24 \
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \
--authorization-mode=RBAC,Node \
--enable-bootstrap-token-auth=true \
--token-auth-file=/opt/kubernetes/conf/token.csv \
--service-node-port-range=30000-32767 \
--kubelet-client-certificate=/opt/kubernetes/ssl/server.pem \
--kubelet-client-key=/opt/kubernetes/ssl/server-key.pem \
--tls-cert-file=/opt/kubernetes/ssl/server.pem  \
--tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \
--client-ca-file=/opt/kubernetes/ssl/ca.pem \
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \
--etcd-cafile=/opt/etcd/ssl/ca.pem \
--etcd-certfile=/opt/etcd/ssl/server.pem \
--etcd-keyfile=/opt/etcd/ssl/server-key.pem \
--audit-log-maxage=30 \
--audit-log-maxbackup=3 \
--audit-log-maxsize=100 \
--audit-log-path=/opt/kubernetes/logs/k8s-audit.log"
EOF

# 参数说明
–logtostderr：设置为 false 表示将日志写入文件，不写入 stderr
—v：日志等级
–log-dir：日志目录
–etcd-servers：etcd集群地址
–bind-address：监听地址
–secure-port：https安全端口
–advertise-address：集群通告地址
–allow-privileged：启用授权
–service-cluster-ip-range：Service虚拟IP地址段
–enable-admission-plugins：准入控制模块
–authorization-mode：认证授权，启用RBAC授权和节点自管理
–enable-bootstrap-token-auth：启用TLS bootstrap机制
–token-auth-file：bootstrap token文件
–service-node-port-range：Service nodeport类型默认分配端口范围
–kubelet-client-xxx：apiserver访问kubelet客户端证书
–tls-xxx-file：apiserver https证书
–etcd-xxxfile：连接Etcd集群证书
–audit-log-*：审计日志
```

​		3.2.2 配置systemd管理apiserver

```shell
cat > /usr/lib/systemd/system/kube-apiserver.service << 'EOF'
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/conf/kube-apiserver.conf
ExecStart=/opt/kubernetes/bin/kube-apiserver $KUBE_APISERVER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

​		3.2.3 启动服务

```shell
systemctl daemon-reload
systemctl start kube-apiserver
systemctl enable kube-apiserver
```

​		3.2.4 授权kubelet-bootstrap用户允许请求证书

```shell
kubectl create clusterrolebinding kubelet-bootstrap \
--clusterrole=system:node-bootstrapper \
--user=kubelet-bootstrap
```

​	3.3 部署kube-controller-manager

​		3.3.1 创建配置文件

```shell
cat > /opt/kubernetes/conf/kube-controller-manager.conf << 'EOF'
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--leader-elect=true \
--master=127.0.0.1:8080 \
--bind-address=127.0.0.1 \
--allocate-node-cidrs=true \
--cluster-cidr=10.244.0.0/16 \
--service-cluster-ip-range=10.0.0.0/24 \
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem  \
--root-ca-file=/opt/kubernetes/ssl/ca.pem \
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \
--experimental-cluster-signing-duration=87600h0m0s"
EOF
```

​		3.3.2 配置systemd管理controller-manager

```shell
cat > /usr/lib/systemd/system/kube-controller-manager.service << 'EOF'
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/conf/kube-controller-manager.conf
ExecStart=/opt/kubernetes/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 启动服务
systemctl daemon-reload
systemctl start kube-controller-manager
systemctl enable kube-controller-manager
```

​	3.4 部署kube-scheduler

​		3.4.1 创建配置文件

```shell
cat > /opt/kubernetes/conf/kube-scheduler.conf << 'EOF'
KUBE_SCHEDULER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--leader-elect \
--master=127.0.0.1:8080 \
--bind-address=127.0.0.1"
EOF

# 配置systemd管理scheduler
cat > /usr/lib/systemd/system/kube-scheduler.service << 'EOF'
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/conf/kube-scheduler.conf
ExecStart=/opt/kubernetes/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 启动服务
systemctl daemon-reload
systemctl start kube-scheduler
systemctl enable kube-scheduler
```

​	3.5 查看集群组件状态

```shell
[root@k8s-master-01 logs]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-1               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}
```

4. 部署Node节点

   4.1 在所有Node节点创建目录

   ```shell
   mkdir -p /opt/kubernetes/{bin,conf,ssl,logs} 
   ```

   4.2 将master主机上Node节点组件拷贝到各个工作节点

   ```shell
   cd kubernetes/server/bin
   for host in {192.168.205.152,192.168.205.154}; do scp kubelet kube-proxy $host:/opt/kubernetes/bin;done
   ```

   4.3 部署kubelet

   ​	先在192.168.205.152节点上操作

   ​	4.3.1 创建配置文件

   ```shell
   cat > /opt/kubernetes/conf/kubelet.conf << 'EOF'
   KUBELET_OPTS="--logtostderr=false \
   --v=2 \
   --log-dir=/opt/kubernetes/logs \
   --hostname-override=k8s-node-03 \
   --network-plugin=cni \
   --kubeconfig=/opt/kubernetes/conf/kubelet.kubeconfig \
   --bootstrap-kubeconfig=/opt/kubernetes/conf/bootstrap.kubeconfig \
   --config=/opt/kubernetes/conf/kubelet-config.yml \
   --cert-dir=/opt/kubernetes/ssl \
   --pod-infra-container-image=mirrorgooglecontainers/pause-amd64:3.0"
   EOF
   ```

   ​	4.3.2 创建配置参数

   ```shell
   cat > /opt/kubernetes/conf/kubelet-config.yml << 'EOF'
   kind: KubeletConfiguration
   apiVersion: kubelet.config.k8s.io/v1beta1
   address: 0.0.0.0
   port: 10250
   readOnlyPort: 10255
   cgroupDriver: cgroupfs
   clusterDNS:
   - 10.0.0.2
   clusterDomain: cluster.local 
   failSwapOn: false
   authentication:
     anonymous:
       enabled: false
     webhook:
       cacheTTL: 2m0s
       enabled: true
     x509:
       clientCAFile: /opt/kubernetes/ssl/ca.pem 
   authorization:
     mode: Webhook
     webhook:
       cacheAuthorizedTTL: 5m0s
       cacheUnauthorizedTTL: 30s
   evictionHard:
     imagefs.available: 15%
     memory.available: 100Mi
     nodefs.available: 10%
     nodefs.inodesFree: 5%
   maxOpenFiles: 1000000
   maxPods: 110
   EOF
   ```

   ​	4.3.3 在master节点生成bootstrap.kubeconfig文件

   ```shell
   # 设置变量
   # cat /opt/kubernetes/conf/token.csv | awk -F',' '{print  $1}'
   # d63332b37383e5484dec574ed402b660
   KUBE_APISERVER="https://192.168.205.150:6443"  # apiserver地址
   TOKEN="d63332b37383e5484dec574ed402b660"  # 与token.csv里保持一致
   
   # 生成 kubelet bootstrap kubeconfig 配置文件
   kubectl config set-cluster kubernetes \
     --certificate-authority=/opt/kubernetes/ssl/ca.pem \
     --embed-certs=true \
     --server=${KUBE_APISERVER} \
     --kubeconfig=bootstrap.kubeconfig
   kubectl config set-credentials "kubelet-bootstrap" \
     --token=${TOKEN} \
     --kubeconfig=bootstrap.kubeconfig
   kubectl config set-context default \
     --cluster=kubernetes \
     --user="kubelet-bootstrap" \
     --kubeconfig=bootstrap.kubeconfig
   kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
   
   # 设置systemd管理kubelet
   cat > /usr/lib/systemd/system/kubelet.service << 'EOF'
   [Unit]
   Description=Kubernetes Kubelet
   After=docker.service
   
   [Service]
   EnvironmentFile=/opt/kubernetes/conf/kubelet.conf
   ExecStart=/opt/kubernetes/bin/kubelet $KUBELET_OPTS
   Restart=on-failure
   LimitNOFILE=65536
   
   [Install]
   WantedBy=multi-user.target
   EOF
   # 设置开机启动
   systemctl daemon-reload
   systemctl start kubelet
   systemctl enable kubelet
   # 批准kubelet证书申请并加入集群
   [root@k8s-master-01 conf]# kubectl get csr
   NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           CONDITION
   node-csr-6FKC-eMe893MT7MGRIJG-IFwGmKmS614A6Vp5oDZ7xs   48s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
   [root@k8s-master-01 conf]# kubectl certificate approve node-csr-6FKC-eMe893MT7MGRIJG-IFwGmKmS614A6Vp5oDZ7xs
   certificatesigningrequest.certificates.k8s.io/node-csr-6FKC-eMe893MT7MGRIJG-IFwGmKmS614A6Vp5oDZ7xs approved
   ```

   ​	4.3.4 在master节点上生成kube-proxy.kubeconfig配置文件

   ```shell
   # 将之前生成的kube-proxy证书拷贝到/opt/kubernetes/ssl中
   cp /opt/ssl/k8s/kube*.pem /opt/kubernetes/ssl/
   # 生成kube-proxy.kubeconfig配置文件（master节点上操作）
   # KUBE_APISERVER="https://192.168.31.71:6443" 这里之前设置过，如果机器重启请重新设置下
   cd /opt/kubernetes/conf
   kubectl config set-cluster kubernetes \
     --certificate-authority=/opt/kubernetes/ssl/ca.pem \
     --embed-certs=true \
     --server=${KUBE_APISERVER} \
     --kubeconfig=kube-proxy.kubeconfig
   kubectl config set-credentials kube-proxy \
     --client-certificate=/opt/kubernetes/ssl/kube-proxy.pem \
     --client-key=/opt/kubernetes/ssl/kube-proxy-key.pem \
     --embed-certs=true \
     --kubeconfig=kube-proxy.kubeconfig
   kubectl config set-context default \
     --cluster=kubernetes \
     --user=kube-proxy \
     --kubeconfig=kube-proxy.kubeconfig
   kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
   # 设置systemd管理kube-proxy
   cat > /usr/lib/systemd/system/kube-proxy.service << 'EOF'
   [Unit]
   Description=Kubernetes Proxy
   After=network.target
   
   [Service]
   EnvironmentFile=/opt/kubernetes/conf/kube-proxy.conf
   ExecStart=/opt/kubernetes/bin/kube-proxy $KUBE_PROXY_OPTS
   Restart=on-failure
   LimitNOFILE=65536
   
   [Install]
   WantedBy=multi-user.target
   EOF
   # 设置开机启动
   systemctl daemon-reload
   systemctl start kube-proxy
   systemctl enable kube-proxy
   ```

   ​	4.3.5 将生成的bootstrap.kubeconfig文件和ca.pem证书拷贝到192.168.205.152节点上

   ```shell
   cd /opt/kubernetes/conf
   scp bootstrap.kubeconfig 192.168.205.152:/opt/kubernetes/conf/
   scp scp ../ssl/ca.pem 192.168.205.152:/opt/kubernetes/ssl/
   ```

   ​	4.3.6 设置systemd管理kubelet

   ​	node节点192.168.205.152上

   ```shell
   cat > /usr/lib/systemd/system/kubelet.service << 'EOF'
   [Unit]
   Description=Kubernetes Kubelet
   After=docker.service
   
   [Service]
   EnvironmentFile=/opt/kubernetes/conf/kubelet.conf
   ExecStart=/opt/kubernetes/bin/kubelet $KUBELET_OPTS
   Restart=on-failure
   LimitNOFILE=65536
   
   [Install]
   WantedBy=multi-user.target
   EOF
   ```

   ​	4.3.7 设置开机启动

   ```
   systemctl daemon-reload
   systemctl start kubelet
   systemctl enable kubelet
   ```

   4.4 批准kubelet证书申请并加入集群

   ```shell
   在master节点上执行以下命令
   # 查看kubelet证书请求
   [root@k8s-master-01 conf]# kubectl get csr
   NAME                                                   AGE    SIGNERNAME                                    REQUESTOR           CONDITION
   node-csr-e1aTZQ0pc6Oxct0Ew6RZYUCDakDGIdGKPkuTuCvfLr4   2m7s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
   # 批准证书申请
   kubectl certificate approve node-csr-e1aTZQ0pc6Oxct0Ew6RZYUCDakDGIdGKPkuTuCvfLr4
   # 查看节点已经加入进来了
   [root@k8s-master-01 conf]# kubectl get nodes
   NAME          STATUS     ROLES    AGE   VERSION
   k8s-master-01   NotReady   <none>   26s     v1.18.13
   k8s-node-03   NotReady   <none>   16s   v1.18.13
   ```

   4.5 将另一个节点也加入进来

   ```shell
   # 在node节点192.168.205.152上拷贝配置文件及ca证书到192.168.205.154节点上
   cd /opt/kubernetes/conf
   scp bootstrap.kubeconfig kubelet.conf kubelet-config.yml 192.168.205.154:/opt/kubernetes/conf/
   scp ../ssl/ca.pem 192.168.205.154:/opt/kubernetes/ssl/ 
   # 在192.168.205.154节点上修改kubelet.conf的--hostname-override参数为节点主机名
   --hostname-override=k8s-node-03
   [root@k8s-node-03 conf]# cat kubelet.conf 
   KUBELET_OPTS="--logtostderr=false \
   --v=2 \
   --log-dir=/opt/kubernetes/logs \
   --hostname-override=k8s-node-03 \
   --network-plugin=cni \
   --kubeconfig=/opt/kubernetes/conf/kubelet.kubeconfig \
   --bootstrap-kubeconfig=/opt/kubernetes/conf/bootstrap.kubeconfig \
   --config=/opt/kubernetes/conf/kubelet-config.yml \
   --cert-dir=/opt/kubernetes/ssl \
   --pod-infra-container-image=mirrorgooglecontainers/pause-amd64:3.0"
   # 设置systemd管理kubelet
   cat > /usr/lib/systemd/system/kubelet.service << 'EOF'
   [Unit]
   Description=Kubernetes Kubelet
   After=docker.service
   
   [Service]
   EnvironmentFile=/opt/kubernetes/conf/kubelet.conf
   ExecStart=/opt/kubernetes/bin/kubelet $KUBELET_OPTS
   Restart=on-failure
   LimitNOFILE=65536
   
   [Install]
   WantedBy=multi-user.target
   EOF
   
   # 设置开机启动
   systemctl daemon-reload
   systemctl start kubelet
   systemctl enable kubelet
   
   # 在master上批准证书申请
   [root@k8s-master-01 conf]# kubectl get csr
   NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           CONDITION
   node-csr-e1aTZQ0pc6Oxct0Ew6RZYUCDakDGIdGKPkuTuCvfLr4   26m   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued
   node-csr-vpNBs0DrgR1tqDpSJzfSGvp4Y5aPr97_Pn490pLh3Xk   72s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
   [root@k8s-master-01 conf]# kubectl certificate approve node-csr-vpNBs0DrgR1tqDpSJzfSGvp4Y5aPr97_Pn490pLh3Xk
   certificatesigningrequest.certificates.k8s.io/node-csr-vpNBs0DrgR1tqDpSJzfSGvp4Y5aPr97_Pn490pLh3Xk approved
   [root@k8s-master-01 conf]# kubectl get nodes
   NAME          STATUS     ROLES    AGE   VERSION
   k8s-master-01   NotReady   <none>   26s     v1.18.13
   k8s-node-03   NotReady   <none>   19m   v1.18.13
   k8s-node-04   NotReady   <none>   10s   v1.18.13
   ```

   ​	上面查看node节点状态为NotReady是还未安装网络组件。

   4.6 部署kube-proxy

   ```shell
   # 在node上节点创建配置文件
   cat > /opt/kubernetes/conf/kube-proxy.conf << 'EOF'
   KUBE_PROXY_OPTS="--logtostderr=false \
   --v=2 \
   --log-dir=/opt/kubernetes/logs \
   --config=/opt/kubernetes/conf/kube-proxy-config.yml"
   EOF
    # 创建配置文件参数
   cat > /opt/kubernetes/conf/kube-proxy-config.yml << 'EOF'
   kind: KubeProxyConfiguration
   apiVersion: kubeproxy.config.k8s.io/v1alpha1
   bindAddress: 0.0.0.0
   metricsBindAddress: 0.0.0.0:10249
   clientConnection:
     kubeconfig: /opt/kubernetes/conf/kube-proxy.kubeconfig
   hostnameOverride: k8s-node-03
   clusterCIDR: 10.0.0.0/24
   EOF
   
   # 在master将kube-proxy.kubeconfig拷贝到工作节点k8s-node-03上
   cd /opt/kubernetes/conf/
   scp kube-proxy.kubeconfig 192.168.205.152:/opt/kubernetes/conf/
   ```

   ​	4.6.1 设置systemd管理kube-proxy

   ```shell
   cat > /usr/lib/systemd/system/kube-proxy.service << 'EOF'
   [Unit]
   Description=Kubernetes Proxy
   After=network.target
   
   [Service]
   EnvironmentFile=/opt/kubernetes/conf/kube-proxy.conf
   ExecStart=/opt/kubernetes/bin/kube-proxy $KUBE_PROXY_OPTS
   Restart=on-failure
   LimitNOFILE=65536
   
   [Install]
   WantedBy=multi-user.target
   EOF
   ```

   ​	4.6.2 设置开机启动

   ```shell
   systemctl daemon-reload
   systemctl start kube-proxy
   systemctl enable kube-proxy
   
   # 查看启动状态
   [root@k8s-node-03 conf]# systemctl status kube-proxy
   ● kube-proxy.service - Kubernetes Proxy
      Loaded: loaded (/usr/lib/systemd/system/kube-proxy.service; enabled; vendor preset: disabled)
      Active: active (running) since 二 2021-10-26 13:46:04 CST; 12s ago
    Main PID: 56181 (kube-proxy)
       Tasks: 11
      Memory: 11.0M
      CGroup: /system.slice/kube-proxy.service
              └─56181 /opt/kubernetes/bin/kube-proxy --logtostderr=false --v=2 --log-dir=/opt/kubernetes/logs --config=/opt/kubernetes/conf/kube-proxy-config.yml
   
   10月 26 13:46:04 k8s-node-03 systemd[1]: Started Kubernetes Proxy.
   
   ```

   ​	4.6.2 从k8s-node-03节点上将kube-proxy.conf  kube-proxy-config.yml  kube-proxy.kubeconfig 文件拷贝到k8s-node-04节点

   ```shell
   [root@k8s-node-03 conf]# scp kube-proxy* 192.168.205.154:/opt/kubernetes/conf/
   root@192.168.205.154's password: 
   kube-proxy.conf                                                                                                                                                                100%  133    58.4KB/s   00:00    
   kube-proxy-config.yml                                                                                                                                                          100%  259   225.9KB/s   00:00    
   kube-proxy.kubeconfig                                                                                                                                                          100% 6199     7.1MB/s   00:00 
   # 修改kube-proxy-config.yml下面参数
   sed -i '7,/hostnameOverride/s/k8s-node-03/k8s-node-04/g' /opt/kubernetes/conf/kube-proxy-config.yml
   ```

   ​	4.6.3 设置systemd管理kube-proxy并启动

   ```shell
   cat > /usr/lib/systemd/system/kube-proxy.service << 'EOF'
   [Unit]
   Description=Kubernetes Proxy
   After=network.target
   
   [Service]
   EnvironmentFile=/opt/kubernetes/conf/kube-proxy.conf
   ExecStart=/opt/kubernetes/bin/kube-proxy $KUBE_PROXY_OPTS
   Restart=on-failure
   LimitNOFILE=65536
   
   [Install]
   WantedBy=multi-user.target
   EOF
   
   systemctl daemon-reload
   systemctl start kube-proxy
   systemctl enable kube-proxy
   ```

5. 部署CNI网络

   5.1 安装CNI网络组件

   ```shell
   # 各节点下载CNI网络插件
   wget https://github.com/containernetworking/plugins/releases/download/v1.0.1/cni-plugins-linux-amd64-v1.0.1.tgz
   # 解压安装
   mkdir -p  /opt/cni/bin
   tar -zxvf cni-plugins-linux-amd64-v1.0.1.tgz -C /opt/cni/bin
   # master节点安装calico 
   wget https://docs.projectcalico.org/manifests/calico.yaml
   kubectl apply -f calico.yaml
   [root@k8s-master-01 ~]# kubectl get pods -n kube-system
   NAME                                       READY   STATUS    RESTARTS   AGE
   calico-kube-controllers-6c68d67746-xb4r9   1/1     Running   0          6m7s
   calico-node-47wm6                          1/1     Running   0          6m8s
   calico-node-bwjl6                          1/1     Running   0          6m8s
   ```

   5.2 安装calico

   ```shell
   # master节点下载安装
   wget https://docs.projectcalico.org/manifests/calico.yaml
   kubectl apply -f calico.yaml
   [root@k8s-master-01 ~]# kubectl get pods -A
   NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
   kube-system   calico-kube-controllers-6c68d67746-wfsz8   1/1     Running   0          23m
   kube-system   calico-node-9s8r6                          1/1     Running   0          23m
   kube-system   calico-node-g5r9h                          1/1     Running   0          23m
   kube-system   calico-node-zh98g                          1/1     Running   0          23m
   ```

   5.3 授权apiserver访问kubelet

   ```shell
   # master节点执行如下操作
   cat > apiserver-to-kubelet-rbac.yaml << 'EOF'
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     annotations:
       rbac.authorization.kubernetes.io/autoupdate: "true"
     labels:
       kubernetes.io/bootstrapping: rbac-defaults
     name: system:kube-apiserver-to-kubelet
   rules:
     - apiGroups:
         - ""
       resources:
         - nodes/proxy
         - nodes/stats
         - nodes/log
         - nodes/spec
         - nodes/metrics
         - pods/log
       verbs:
         - "*"
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: system:kube-apiserver
     namespace: ""
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: system:kube-apiserver-to-kubelet
   subjects:
     - apiGroup: rbac.authorization.k8s.io
       kind: User
       name: kubernetes
   EOF
   
   kubectl apply -f apiserver-to-kubelet-rbac.yaml
   ```

   

6. 安装kubernetes-dashboard和coredns

   ```shell
   # 下载kubernetes-dashboard
   wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.2.0/aio/deploy/recommended.yaml
   # 将配置文件种Server访问类型改为NodePort类型，官网原配置不是NodePort类型。
   kind: Service
   apiVersion: v1
   metadata:
     labels:
       k8s-app: kubernetes-dashboard
     name: kubernetes-dashboard
     namespace: kubernetes-dashboard
   spec:
     ports:
       - port: 443
         targetPort: 8443
         nodePort: 30101
     selector:
       k8s-app: kubernetes-dashboard
     type: NodePort
     
   [root@k8s-kubernetes-master ~]# kubectl apply -f recommended.yaml
   # 查看pods,和sevice信息
   [root@k8s-master-01 ~]# kubectl get pods,svc -n kubernetes-dashboard -o wide
   NAME                                             READY   STATUS    RESTARTS   AGE   IP               NODE            NOMINATED NODE   READINESS GATES
   pod/dashboard-metrics-scraper-78f5d9f487-6l4t4   1/1     Running   0          50s   172.16.89.129    k8s-node-03     <none>           <none>
   pod/kubernetes-dashboard-577bd97bc-jfdx7         1/1     Running   0          50s   172.16.151.129   k8s-master-01   <none>           <none>
   
   NAME                                TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE   SELECTOR
   service/dashboard-metrics-scraper   ClusterIP   10.0.0.136   <none>        8000/TCP        50s   k8s-app=dashboard-metrics-scraper
   service/kubernetes-dashboard        NodePort    10.0.0.186   <none>        443:30101/TCP   50s   k8s-app=kubernetes-dashboard
   
   ```

   ​	访问集群任意节点（这里以master节点为例）

   ![image-20211027105514278](https://longlizl.github.io/k8s/images/6.png) 

   ​	生成Token认证方式

   ```shell
   1. 授权
   # 创建serviceaccount 
   [root@k8s-kubernetes-master ~]# kubectl create serviceaccount dashboard-serviceaccount -n kube-system
   # 创建clusterrolebinding 
   [root@k8s-kubernetes-master ~]# kubectl create clusterrolebinding dashboard-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-serviceaccount
   2. 获取口令
   [root@k8s-master-01 ~]# kubectl describe secrets -n kube-system $(kubectl get secret -n kube-system  | awk '/dashboard-serviceaccount/{print $1}')
   Name:         dashboard-serviceaccount-token-6hg8r
   Namespace:    kube-system
   Labels:       <none>
   Annotations:  kubernetes.io/service-account.name: dashboard-serviceaccount
                 kubernetes.io/service-account.uid: c7e6ccf6-b30b-4daf-92c4-deb82a30144e
   
   Type:  kubernetes.io/service-account-token
   
   Data
   ====
   ca.crt:     1330 bytes
   namespace:  11 bytes
   token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IkVRNExkdkxMSHMtSTFnQ2FvMERuMnhoS0pZOENPS25ZU1ZQSGlybWJ0RlEifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtc2VydmljZWFjY291bnQtdG9rZW4tNmhnOHIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLXNlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYzdlNmNjZjYtYjMwYi00ZGFmLTkyYzQtZGViODJhMzAxNDRlIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1zZXJ2aWNlYWNjb3VudCJ9.L5ohnz76U5hQB6d9d3CpuUTEiwetm3erbu07aWDjjKi33FNaVdpt3qTFCIIjgWl37HpnOKc0YgeKFxZW51vT-w_EShxa4q-3PgwTp-JxoDiijE9ELpV_c4LSbhuu4hFhZNNSSawDy62auJXarb2LxDdJqfHa6W4roJJ4s_RFjY3GcXmkKF-3RQ_ip2-gR6mFReQ90FdG06jU6pc7kRJgES5--jRtTN3b10e2BGj11CiWKBb90jwkFT7etytl2BHsyNo_QGKv2uq7ICwt9SxZsZDM1W3KuDC9hAm83ZpJHbIDPYAXtKRL0vP4KFKEU7OYJ_qNX_j_mudD2aF5uUGRlA
   ```

   ![image-20211027111549239](https://longlizl.github.io/k8s/images/7.png)

   ​	部署coredns

   ```shell
   # 新建coredns.yaml配置文件（这里我副本写了3个）
   cat > coredns.yaml << 'EOF'
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: coredns
     namespace: kube-system
     labels:
         kubernetes.io/cluster-service: "true"
         addonmanager.kubernetes.io/mode: Reconcile
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     labels:
       kubernetes.io/bootstrapping: rbac-defaults
       addonmanager.kubernetes.io/mode: Reconcile
     name: system:coredns
   rules:
   - apiGroups:
     - ""
     resources:
     - endpoints
     - services
     - pods
     - namespaces
     verbs:
     - list
     - watch
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     annotations:
       rbac.authorization.kubernetes.io/autoupdate: "true"
     labels:
       kubernetes.io/bootstrapping: rbac-defaults
       addonmanager.kubernetes.io/mode: EnsureExists
     name: system:coredns
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: system:coredns
   subjects:
   - kind: ServiceAccount
     name: coredns
     namespace: kube-system
   ---
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: coredns
     namespace: kube-system
     labels:
         addonmanager.kubernetes.io/mode: EnsureExists
   data:
     Corefile: |
       .:53 {
           errors
           health {
              lameduck 5s
           }
           ready
           kubernetes cluster.local in-addr.arpa ip6.arpa {
              pods insecure
              fallthrough in-addr.arpa ip6.arpa
              ttl 30
           }
           prometheus :9153
           forward . /etc/resolv.conf
           cache 30
           loop
           reload
           loadbalance
       }
   ---
   
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: coredns
     namespace: kube-system
     labels:
       k8s-app: kube-dns
       kubernetes.io/cluster-service: "true"
       addonmanager.kubernetes.io/mode: Reconcile
       kubernetes.io/name: "CoreDNS"
   spec:
     # replicas: not specified here:
     # 1. In order to make Addon Manager do not reconcile this replicas parameter.
     # 2. Default is 1.
     # 3. Will be tuned in real time if DNS horizontal auto-scaling is turned on.
     replicas: 3
     strategy:
       type: RollingUpdate
       rollingUpdate:
         maxUnavailable: 1
     selector:
       matchLabels:
         k8s-app: kube-dns
     template:
       metadata:
         labels:
           k8s-app: kube-dns
         annotations:
           seccomp.security.alpha.kubernetes.io/pod: 'docker/default'
       spec:
         serviceAccountName: coredns
         tolerations:
           - key: node-role.kubernetes.io/master
             effect: NoSchedule
           - key: "CriticalAddonsOnly"
             operator: "Exists"
         containers:
         - name: coredns
           image: coredns/coredns:1.6.7
           imagePullPolicy: IfNotPresent
           resources:
             limits:
               memory: 170Mi
             requests:
               cpu: 100m
               memory: 70Mi
           args: [ "-conf", "/etc/coredns/Corefile" ]
           volumeMounts:
           - name: config-volume
             mountPath: /etc/coredns
             readOnly: true
           ports:
           - containerPort: 53
             name: dns
             protocol: UDP
           - containerPort: 53
             name: dns-tcp
             protocol: TCP
           - containerPort: 9153
             name: metrics
             protocol: TCP
           livenessProbe:
             httpGet:
               path: /health
               port: 8080
               scheme: HTTP
             initialDelaySeconds: 60
             timeoutSeconds: 5
             successThreshold: 1
             failureThreshold: 5
           securityContext:
             allowPrivilegeEscalation: false
             capabilities:
               add:
               - NET_BIND_SERVICE
               drop:
               - all
             readOnlyRootFilesystem: true
         dnsPolicy: Default
         volumes:
           - name: config-volume
             configMap:
               name: coredns
               items:
               - key: Corefile
                 path: Corefile
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: kube-dns
     namespace: kube-system
     annotations:
       prometheus.io/port: "9153"
       prometheus.io/scrape: "true"
     labels:
       k8s-app: kube-dns
       kubernetes.io/cluster-service: "true"
       addonmanager.kubernetes.io/mode: Reconcile
       kubernetes.io/name: "CoreDNS"
   spec:
     selector:
       k8s-app: kube-dns
     clusterIP: 10.0.0.2    #dns ip
     ports:
     - name: dns
       port: 53
       protocol: UDP
     - name: dns-tcp
       port: 53
       protocol: TCP
   EOF
   
   # 安装coredns
   kubectl apply -f coredns.yaml
   
   # 查看pods信息
   [root@k8s-master-01 ~]# kubectl get pods -A -o wide
   NAMESPACE              NAME                                         READY   STATUS    RESTARTS   AGE     IP                NODE            NOMINATED NODE   READINESS GATES
   kube-system            calico-kube-controllers-6c68d67746-z9xgg     1/1     Running   0          3h43m   172.16.127.193    k8s-node-04     <none>           <none>
   kube-system            calico-node-497kt                            1/1     Running   0          3h43m   192.168.205.152   k8s-node-03     <none>           <none>
   kube-system            calico-node-jwshf                            1/1     Running   0          3h43m   192.168.205.150   k8s-master-01   <none>           <none>
   kube-system            calico-node-ml7z4                            1/1     Running   0          3h43m   192.168.205.154   k8s-node-04     <none>           <none>
   kube-system            coredns-5d74c5f4b-vg7gr                      1/1     Running   0          7s      172.16.89.131     k8s-node-03     <none>           <none>
   kube-system            coredns-5d74c5f4b-vn5t8                      1/1     Running   0          7s      172.16.127.196    k8s-node-04     <none>           <none>
   kube-system            coredns-5d74c5f4b-xkzds                      1/1     Running   0          7s      172.16.151.132    k8s-master-01   <none>           <none>
   kubernetes-dashboard   dashboard-metrics-scraper-78f5d9f487-6l4t4   1/1     Running   0          3h25m   172.16.89.129     k8s-node-03     <none>           <none>
   kubernetes-dashboard   kubernetes-dashboard-577bd97bc-jfdx7         1/1     Running   0          3h25m   172.16.151.129    k8s-master-01   <none>           <none>
   test-dev               nginx-demo-588b778dc5-c6cc8                  1/1     Running   0          167m    172.16.151.130    k8s-master-01   <none>           <none>
   test-dev               nginx-demo-588b778dc5-clvjr                  1/1     Running   0          167m    172.16.127.194    k8s-node-04     <none>           <none>
   
   # 运行一个pod测试下dns
   [root@k8s-master-01 ~]# kubectl run -it --rm dns-cs --image=busybox:1.28.4 sh
   If you don't see a command prompt, try pressing enter.
   / # nslookup kubernetes
   Server:    10.0.0.2
   Address 1: 10.0.0.2 kube-dns.kube-system.svc.cluster.local
   
   Name:      kubernetes
   Address 1: 10.0.0.1 kubernetes.default.svc.cluster.local
   / # nslookup www.baidu.com
   Server:    10.0.0.2
   Address 1: 10.0.0.2 kube-dns.kube-system.svc.cluster.local
   
   Name:      www.baidu.com
   Address 1: 220.181.38.149
   Address 2: 220.181.38.150
   ```

   到此一个单master集群已经部署完成

 7. 部署master2（192.168.205.151）

    ```shell
    1. 拷贝Master1上所有K8s文件Master2
    scp -r /opt/kubernetes 192.168.205.151:/opt
    scp -r /opt/cni/ 192.168.205.151:/opt
    scp /usr/lib/systemd/system/kube* 192.168.205.151:/usr/lib/systemd/system/
    scp /usr/bin/kubectl  192.168.205.151:/usr/bin
    2. 删除kubelet客户端证书和kubelet.kubeconfig文件
    rm -f /opt/kubernetes/conf/kubelet.kubeconfig
    rm -f /opt/kubernetes/ssl/kubelet*
    3. 修改kube-apiserver.conf,kubelet.conf和kube-proxy-config.yml配置文件为本机IP或主机名
    sed -i '5,7s/192.168.205.150/192.168.205.151/g' /opt/kubernetes/conf/kube-apiserver.conf
    [root@k8s-master-02 conf]# cat kube-apiserver.conf
    KUBE_APISERVER_OPTS="--logtostderr=false \
    --v=2 \
    --log-dir=/opt/kubernetes/logs \
    --etcd-servers=https://192.168.205.150:2379,https://192.168.205.151:2379,https://192.168.205.152:2379 \
    --bind-address=192.168.205.151 \
    --secure-port=6443 \
    --advertise-address=192.168.205.151 \
    ...
    ###
    sed -i 's/k8s-master-01/k8s-master-02/' /opt/kubernetes/conf/kubelet.conf
    [root@k8s-master-02 conf]# cat kubelet.conf
    KUBELET_OPTS="--logtostderr=false \
    --v=2 \
    --log-dir=/opt/kubernetes/logs \
    --hostname-override=k8s-master-02 \
    --network-plugin=cni \
    --kubeconfig=/opt/kubernetes/conf/kubelet.kubeconfig \
    --bootstrap-kubeconfig=/opt/kubernetes/conf/bootstrap.kubeconfig \
    --config=/opt/kubernetes/conf/kubelet-config.yml \
    --cert-dir=/opt/kubernetes/ssl \
    --pod-infra-container-image=mirrorgooglecontainers/pause-amd64:3.0"
    ###
    sed -i 's/k8s-master-01/k8s-master-02/' /opt/kubernetes/conf/kube-proxy-config.yml
    [root@k8s-master-02 conf]# cat kube-proxy-config.yml
    kind: KubeProxyConfiguration
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    bindAddress: 0.0.0.0
    metricsBindAddress: 0.0.0.0:10249
    clientConnection:
      kubeconfig: /opt/kubernetes/conf/kube-proxy.kubeconfig
    hostnameOverride: k8s-master-02
    clusterCIDR: 10.0.0.0/24
    
    4. 批准kubelet证书申请
    [root@k8s-master-02 system]# kubectl get csr
    NAME                                                   AGE    SIGNERNAME                                    REQUESTOR           CONDITION
    node-csr-9UeQ4eDBupEprMmCK5Wm4RvaPVXls_kWOQKgGh9KNHQ   111s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
    [root@k8s-master-02 system]# kubectl certificate approve node-csr-9UeQ4eDBupEprMmCK5Wm4RvaPVXls_kWOQKgGh9KNHQ
    certificatesigningrequest.certificates.k8s.io/node-csr-9UeQ4eDBupEprMmCK5Wm4RvaPVXls_kWOQKgGh9KNHQ approved
    # 查看节点状态
    [root@k8s-master-02 system]# kubectl get nodes
    NAME            STATUS   ROLES    AGE   VERSION
    k8s-master-01   Ready    <none>   24h   v1.18.13
    k8s-master-02   Ready    <none>   63s   v1.18.13
    k8s-node-03     Ready    <none>   29h   v1.18.13
    k8s-node-04     Ready    <none>   29h   v1.18.13
    
    ```

### 四 . 部署SLB负载均衡

---待续---



   

