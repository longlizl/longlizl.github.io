## k8s使用kubeadm快速部署(v1.22.2.0)

### 各组件介绍

- etcd 

用于持久化存储集群中所有的资源对象，如Node、Service、Pod、RC、Namespace等；API Server提供了操作etcd的封装接口API，这些API基本上都是集群中资源对象的增删改查及监听资源变化的接口。

- API Server 

提供了资源对象的唯一操作入口，其他所有组件都必须通过它提供的API来操作资源数据，通过对相关的资源数据“全量查询”+“变化监听”，这些组件可以很“实时”地完成相关的业务功能。

- Controller Manager 

集群内部的管理控制中心，其主要目的是实现Kubernetes集群的故障检测和恢复的自动化工作，比如根据RC的定义完成Pod的复制或移除，以确保Pod实例数符合RC副本的定义；根据Service与Pod的管理关系，完成服务的Endpoints对象的创建和更新；其他诸如Node的发现、管理和状态监控、死亡容器所占磁盘空间及本地缓存的镜像文件的清理等工作也是由Controller Manager完成的。

- Scheduler 

集群中的调度器，负责Pod在集群节点中的调度分配。

- Kubelet 

负责本Node节点上的Pod的创建、修改、监控、删除等全生命周期管理，同时Kubelet定时“上报”本Node的状态信息到API Server里。

- KubeProxy 

实现Pod网络代理，维护网络规则和四层负载均衡工作。

### 1.环境准备

一个master节点2个node节点，centos7系统

  1.1 所有节点关掉防护墙firewalld，selinux，swap

```shell
# 禁用防火墙
systemctl stop firewalld && systemctl disable firewalld
# 禁用selinux
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
# 禁用swap
swapoff -a
```

​	  永久禁用swap

![img](https://longlizl.github.io/k8s/1.png)

  1.2 设置各节点主机名与hosts

```shell
# 设置主机名
hostnamectl set-hostname k8s-kubernetes-master
hostnamectl set-hostname k8s-kubernetes-node1
hostnamectl set-hostname k8s-kubernetes-node2

```

```shell
# 设置hosts
cat >> /etc/hosts <<EOF
192.168.205.128  k8s-kubernetes-master
192.168.205.168  k8s-kubernetes-node1
192.168.205.169  k8s-kubernetes-node2
EOF
```

1.3 **打开net.bridge.bridge-nf-call-ip6tables=1，net.bridge.bridge-nf-call-iptables=1**

```shell
cat  >> /etc/sysctl.conf << EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl -p
```

  执行sysctl -p 时出现：

```shell
[root@localhost ~]# sysctl -p

sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-ip6tables: No such file or directory

sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-iptables: No such file or directory
```

  解决方法：

```shell
[root@localhost ~]# modprobe br_netfilter
```

### 2.各节点安装docker,kubeadm,kubelet,kubectl

  docker,kubeadm,kubelet,kubectl各节点都安装，kubectl工作节点可以不安装master节点安装管理集群即可

  2.1 安装docker-ce

```shell
yum install -y yum-utils
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum -y install docker-ce
systemctl enable docker && systemctl start docker
```

  添加阿里云docker仓库加速器 

```shell
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://fl791z1h.mirror.aliyuncs.com"]
}
EOF

systemctl daemon-reload
systemctl restart docker
```

2.2 添加阿里云kubernetes镜像源

```shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

  2.3 安装kubeadm,kubelet,kubectl

```shell
yum -y install kubelet kubeadm kubectl
systemctl enable kubelet  
```

  2.4 设置Cgroup Driver: systemd

```shell
# 在ExecStart后面加上"--exec-opt native.cgroupdriver=systemd" 参数
vim /usr/lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd
# 重启docker
systemctl daemon-reload
systemctl restart docker
```

### 3. 初始化集群

  3.1 部署kubernetes master

在master(192.168.205.128)节点上执行以下命令：

```shell
kubeadm init \
--kubernetes-version=v1.22.2 \
--apiserver-advertise-address=192.168.205.128 \
--image-repository registry.aliyuncs.com/google_containers \
--pod-network-cidr=10.244.0.0/16 \
--service-cidr=10.1.0.0/16
```

  成功后会显示如下信息：

![image-20211012143557698](https://longlizl.github.io/k8s/images/2.png)

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> /etc/profile
source /etc/profile
```

  3.2 各节点加入集群 

```
kubeadm join 192.168.205.128:6443 --token r0rv6m.ywh0sz9qjnxfb8i5 \
--discovery-token-ca-cert-hash sha256:7b37f5c42b350c3bbd51705007504d0cba758b1b600afacc2cbf8658fae06057
```

   master上查看节点信息 

```shell
[root@k8s-kubernetes-master ~]# kubectl get nodes
NAME                    STATUS     ROLES                  AGE   VERSION
k8s-kubernetes-master   NotReady   control-plane,master   44m   v1.22.2
k8s-kubernetes-node1    NotReady   <none>                 13m   v1.22.2
k8s-kubernetes-node2    NotReady   <none>                 13m   v1.22.2
```

​    可以看到节点状态都是未准备状态，是因为我们还未安装网络组件。下载安装网络组件

  3.3 安装pod网络插件

```shell
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

```shell
# 查看kube-system名称空间pod
[root@k8s-kubernetes-master ~]# kubectl get pods -n kube-system
NAME                                            READY   STATUS    RESTARTS   AGE
calico-kube-controllers-75f8f6cc59-54b7r        1/1     Running   0          2m35s
calico-node-5hdcf                               1/1     Running   0          2m35s
calico-node-7w6rl                               1/1     Running   0          2m35s
calico-node-qwb5l                               1/1     Running   0          2m35s
coredns-7f6cbbb7b8-8kmm2                        1/1     Running   0          61m
coredns-7f6cbbb7b8-v5k5s                        1/1     Running   0          61m
etcd-k8s-kubernetes-master                      1/1     Running   0          61m
kube-apiserver-k8s-kubernetes-master            1/1     Running   0          61m
kube-controller-manager-k8s-kubernetes-master   1/1     Running   0          61m
kube-proxy-6f2j2                                1/1     Running   0          30m
kube-proxy-bbx4c                                1/1     Running   0          61m
kube-proxy-r7555                                1/1     Running   0          30m
kube-scheduler-k8s-kubernetes-master            1/1     Running   0          61m
```

### 4. 安装kubernetes-dashboard

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
```

```shell
[root@k8s-kubernetes-master ~]# kubectl apply -f recommended.yaml
# 查看pods,和sevice信息
[root@k8s-kubernetes-master ~]# kubectl get pods,svc -n kubernetes-dashboard -o wide
NAME                                             READY   STATUS    RESTARTS   AGE   IP               NODE                   NOMINATED NODE   READINESS GATES
pod/dashboard-metrics-scraper-856586f554-f49ql   1/1     Running   0          11s   10.244.195.66    k8s-kubernetes-node1   <none>           <none>
pod/kubernetes-dashboard-78c79f97b4-dqfn2        1/1     Running   0          11s   10.244.130.130   k8s-kubernetes-node2   <none>           <none>

NAME                                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE   SELECTOR
service/dashboard-metrics-scraper   ClusterIP   10.1.58.80     <none>        8000/TCP        11s   k8s-app=dashboard-metrics-scraper
service/kubernetes-dashboard        NodePort    10.1.214.121   <none>        443:30101/TCP   11s   k8s-app=kubernetes-dashboard
```

  浏览器访问

​	https://192.168.205.128:30101/（其他2节点ip均可访问）

![image-20211012170701866](https://longlizl.github.io/k8s/images/3.png)

**Token （令牌） 认证方式**

1. 授权 (所有 namespace )

```shell
# 创建serviceaccount 
[root@k8s-kubernetes-master ~]# kubectl create serviceaccount dashboard-serviceaccount -n kube-system
```

```shell
# 创建clusterrolebinding 
[root@k8s-kubernetes-master ~]# kubectl create clusterrolebinding dashboard-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-serviceaccount
```

2. 获取令牌（用于网页登录）

```shell
# 查看口令列表
[root@k8s-kubernetes-master ~]# kubectl get secret -n kube-system |grep dashboard-serviceaccount-token
dashboard-serviceaccount-token-qqhgm             kubernetes.io/service-account-token   3      2m51s
# 获取口令 
[root@k8s-kubernetes-master ~]# kubectl describe secret dashboard-serviceaccount-token-qqhgm -n kube-system 
Name:         dashboard-serviceaccount-token-qqhgm
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-serviceaccount
              kubernetes.io/service-account.uid: 51d08def-eeb9-4366-b50b-d3d48f760270

Type:  kubernetes.io/service-account-token

Data
====

namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6ImNiSWpKTTM0NXM2TDltbi04Z3RmcENjMHV1dW02cW5tbTdpZS1OMllDTncifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtc2VydmljZWFjY291bnQtdG9rZW4tcXFoZ20iLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLXNlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiNTFkMDhkZWYtZWViOS00MzY2LWI1MGItZDNkNDhmNzYwMjcwIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1zZXJ2aWNlYWNjb3VudCJ9.PT93mR2Mcp-k76PeFJx10k5wEHwBuZDEZqCFLBl2ZBMOpEK8GytWklI9ogXDCDhxDjzix0FsoOc6V7wU55yNQDlDUlgZi_Nh-tZxC2dCzVXB3XpKoCN68ts-jwuFyC1tn_pX-QAHef6nBZCEP_TSxLKDUtFF4nrCuQpJAhKI2izkJpb6xn5ZDcZZ3v45WkdW0a8hvx6W1-i0whVbMzghXctqIncWHsZmc4_vgCQRZHbqJPn2Z01nfDtHuAdwGOLqEcQu6X2J5wkZXqPejOxmBPzwaQ2noFyMpv56deJR4x1iW-5-w3JNRiJNZbe15_AdJHVmay5Fje-AQrBHSRNkuQ
ca.crt:     1099 bytes
```

![image-20211012170852071](https://longlizl.github.io/k8s/images/4.png)

### 5. 服务发布

1. 发布nginx服务：

   我们先创建一个名称空间来用作我们测试应用

   ```shell
   kubectl create namespace test-dev
   ```
   
   编写nginx.yaml配置
   
   vim nginx.yaml

```shell
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx-demo
  namespace: test-dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: /etc/localtime  # 挂载到容器的目录
          name: timezone
      volumes:
      -  hostPath:
           path: /usr/share/zoneinfo/Asia/Shanghai  # 宿主机的目录
         name: timezone
```
​    kubectl apply -f nginx.yaml

2. 编写service资源

   vim nginx-service.yaml

```shell
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx-service
  namespace: test-dev
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 30080
  selector:
    app: nginx
  type: NodePort
```

   kubectl apply -f nginx-service.yaml

查看service资源

```shell
[root@k8s-kubernetes-master ~]# kubectl get pods,svc -n test-dev -o wide
NAME                              READY   STATUS    RESTARTS   AGE    IP               NODE                   NOMINATED NODE   READINESS GATES
pod/nginx-demo-7848d4b86f-92d96   1/1     Running   0          5m8s   10.244.130.132   k8s-kubernetes-node2   <none>           <none>
pod/nginx-demo-7848d4b86f-xzmdr   1/1     Running   0          5m8s   10.244.195.68    k8s-kubernetes-node1   <none>           <none>

NAME                    TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE   SELECTOR
service/nginx-service   NodePort   10.1.246.195   <none>        80:30080/TCP   14s   app=nginx
```

3. 通过各节点ip访问（master和node节点均可）

![image-20211013101158579](https://longlizl.github.io/k8s/images/5.png)







