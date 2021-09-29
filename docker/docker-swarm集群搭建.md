## 一、系统环境

系统 centos 7.6

主机 4台  （1管理节点+3工作节点）

docker版本 19.03.13

禁用防火墙

开启以下配置：

```shell
cat  >> /etc/sysctl.conf << EOF

net.bridge.bridge-nf-call-ip6tables = 1

net.bridge.bridge-nf-call-iptables = 1

EOF

sysctl -p  
```



## 二、集群部署

1）选一台主机为master作为管理节点  在master创建Swarm（要保存初始化后token，因为在节点加入时要使用token作为通讯的密钥）

```shell
[root@v7-01 ~]# docker swarm init --advertise-addr 172.16.2.22
```

![img](https://longlizl.github.io\docker\images\1.png)

2）添加节点到swarm集群中

在其他工作节点上执行此操作（本列用了3台工作节点）

```shell
docker swarm join --token SWMTKN-1-3ta7uc5whuhjyvy08uvxn1mtcy07bow67c05cyr0ayi7wzzdio-3rghhvhjuj3l1lnsge14v0mx8 172.16.2.22:2377
```

在master上查看集群节点的状态

```shell
docker node ls
```

![img](https://longlizl.github.io\docker\images\2.png)

## 三、在Swarm中部署服务

```shell
docker service create --replicas 4 -p 9999:80 --name nginx nginx
```

查看nginx分布得节点及副本数

```shell
docker service ps nginx

docker service ls
```

![img](https://longlizl.github.io\docker\images\3.png)

查看运行的服务分布在哪些节点上运行：

```shell
docker service ls -q | xargs docker service ps | grep -i running
```

![img](https://longlizl.github.io\docker\images\4.png)

## 四.服务扩容

```shell
docker service scale nginx=5
```

更新service副本数：

```shell
docker service update –replicas 3 nginx
```

## 五.部署多服务集群

docker-compose.yml 为提前编排好的配置文件

```shell
# 格式：docker stack deploy -c <docker-compose.yml> <service_name>
docker stack deploy -c docker-compose.yml nginx-demo
```
## 六.Swarm持久化共享存储

1. 选择一台主机安装NFS服务

   yum -y install nfs-utils  rpcbind

   mkdir /opt/nfs_share

   vim /etc/exports
   
   ```shell
   /opt/nfs_share 192.168.205.0/24(rw,no_root_squash,sync) 
   ```

   systemctl start rpcbind

   systemctl start nfs
   
   systemctl enable nfs
   
2. swarm所有节点安装nfs-utils

   ```shell
   yum -y install nfs-utils
   systemctl start nfs
   systemctl enable nfs
   ```

   

3. 持久化创建服务

   3.1  命令行创建（参考的官网命令）
   
   [docker-sewram-volume](https://docs.docker.com/storage/volumes/)

```shell
$ docker service create \
	-p 80:80 \
    --mount 'type=volume,src=nginxhtml,dst=/usr/share/nginx/html,volume-nocopy=true,volume-driver=local,volume-opt=type=nfs,volume-opt=device=192.168.205.135:/opt/nfs_share,"volume-opt=o=addr=192.168.205.135,vers=4,soft,timeo=180,bg,tcp,rw"'
    --name nginx-demo \
    nginx:latest
```

​      3.2   docker-compose.yml文件编排

```shell
version: '3.8'
services:
  nginx:
    image: nginx:latest
    deploy:
      mode: replicated
      replicas: 3
      restart_policy:
        condition: on-failure
    ports:
      - "88:80"
    networks:
      ng_vol:
    volumes:
      - "nginxhtml:/usr/share/nginx/html"

volumes:
  nginxhtml:
    driver: local
    driver_opts:
      type: "nfs"
      o: "addr=192.168.205.135,rw"
      device: ":/opt/nfs_share"

networks:
  ng_vol:
    driver: overlay
```

部署服务

```
docker stack deploy -c docker-compose.yml  nginx-demo
```

查看服务运行的节点分布

```shell
[root@swarm-master _data]# docker service ls | grep nginx-demo | awk '{print $1}' | xargs docker service ps | grep -i running
kkecr966hg1d   nginx-demo_nginx.1       nginx:latest   swarm-master   Running         Running 16 minutes ago                                       
gli71ut5nru1   nginx-demo_nginx.2       nginx:latest   swarm-master   Running         Running 15 minutes ago                                       
ldd5r3csu7lp   nginx-demo_nginx.3       nginx:latest   swarm-node1    Running         Running 16 minutes ago
```

查看NFS共享目录及挂载卷是否

```shell
# NFS共享目录
[root@swarm-master nfs_share]# ls
123  50x.html  index.html
# work节点挂载卷目录（由于服务运行节点有2个都被分配到主节点上所以node2节点暂时没数据）
[root@swarm-node1 _data]# ls
123  50x.html  index.html
# 现在我们把运行容器任务更新到6个看看
[root@swarm-master _data]# docker service update --replicas 6 nginx-demo_nginx
nginx-demo_nginx
overall progress: 6 out of 6 tasks 
1/6: running   [==================================================>] 
2/6: running   [==================================================>] 
3/6: running   [==================================================>] 
4/6: running   [==================================================>] 
5/6: running   [==================================================>] 
6/6: running   [==================================================>] 
verify: Service converged 
[root@swarm-master _data]# docker service ls
ID             NAME               MODE         REPLICAS   IMAGE          PORTS
r7iem955bk5y   nginx              replicated   3/3        nginx:latest   *:9999->80/tcp
wcbf9dpwxwki   nginx-demo_nginx   replicated   6/6        nginx:latest   *:88->80/tcp
[root@swarm-master _data]# docker service ls | grep nginx-demo | awk '{print $1}' | xargs docker service ps | grep -i running
kkecr966hg1d   nginx-demo_nginx.1       nginx:latest   swarm-master   Running         Running 50 minutes ago                                          
gli71ut5nru1   nginx-demo_nginx.2       nginx:latest   swarm-master   Running         Running 49 minutes ago                                          
ldd5r3csu7lp   nginx-demo_nginx.3       nginx:latest   swarm-node1    Running         Running 50 minutes ago                                          
fu90vw0u1l0t   nginx-demo_nginx.4       nginx:latest   swarm-node1    Running         Running 8 minutes ago                                           
3vguvr71lsch   nginx-demo_nginx.5       nginx:latest   swarm-node2    Running         Running about a minute ago                                      
cphhmb1dr44z   nginx-demo_nginx.6       nginx:latest   swarm-node2    Running         Running about a minute ago  # 可以看到每个节点都有2个容器在运行了，在看下挂载卷已经有数据了 
[root@swarm-node2 _data]# ls
123  50x.html  index.html
```

测试访问下 看：

```html
[root@swarm-node2 _data]# curl http://192.168.205.137:88

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>


<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>

</body>
</html>
```


## Swarm相关命令

```shell
1) 初始化swarm manager并制定网卡地址

docker swarm init --advertise-addr 182.48.115.237

2) 删除集群，强制退出需要加–force (针对manager节点). 到各个节点上执行退出集群的命令

docker node rm swarm-node1    

# manager节点退出集群,需要加--force
docker swarm leave --force     

3) 查看swarm worker的连接令牌

docker swarm join-token worker

例如:

[root@manager-node ~]# docker swarm init --advertise-addr 182.48.115.237
Swarm initialized: current node (1gi8utvhu4rxy8oxar2g7h6gr) is now a manager.
To add a worker to this swarm, run the following command:
    docker swarm join \
    --token SWMTKN-1-4roc8fx10cyfgj1w1td8m0pkyim08mve578wvl03eqcg5ll3ig-f0apd81qfdwv27rnx4a4y9jej \
    182.48.115.237:2377
To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

4) 加入docker swarm集群,作为worker节点
利用上面执行结果中的命令放在其他节点上执行,则该节点加入到swarm集群中作为worker节点

[root@node1 ~]# docker swarm join --token SWMTKN-1-4roc8fx10cyfgj1w1td8m0pkyim08mve578wvl03eqcg5ll3ig-f0apd81qfdwv27rnx4a4y9jej 182.48.115.237:2377
This node joined a swarm as a worker.

5) 查看swarm manager的连接令牌

docker swarm join-token manager

例如:

[root@swarm-manager-node ~]# docker swarm join-token manager
To add a manager to this swarm, run the following command:
    docker swarm join \
    --token SWMTKN-1-075gaitl18z3v0p37sx7i5cmvzjjur0fbuixzp4tun0xh0cikd-0y8ttp5h0g54j10amn670w6su \
    172.16.60.220:2377

6) 加入docker swarm集群,作为manager节点
利用上面执行结果中的命令放在其他节点上执行,则该节点加入到swarm集群中作为manager管理节点,状态为reachable.

[root@swarm-manager-node2 ~]# docker swarm join --token SWMTKN-1-075gaitl18z3v0p37sx7i5cmvzjjur0fbuixzp4tun0xh0cikd-0y8ttp5h0g54j10amn670w6su 172.16.60.220:2377
This node joined a swarm as a manager.
[root@swarm-manager-node2 ~]# docker node ls
ID                                                HOSTNAME                  STATUS      AVAILABILITY  MANAGER STATUS
rpbey5t1v14olke2mgtc430de     swarm-node2                 Ready        Active       
u6gkfr4j19gq16ddyb76fxsl3       swarm-node1                 Ready        Active       
vwbb0imil512a1le04bnkx98u *   swarm-manager-node    Ready       Active                      Leader
ybjvaszg838upeqvvzswhq0tt       swarm-manager-node2  Ready       Active                      Reachable

如果之前的leader状态的manager管理节点挂了后(假如systemctl stop docker, 然后再systemctl start docker),
则新加入的manager节点状态由reachable变为leader, 之前的manager节点状态为unreachable.

[root@swarm-manager-node2 ~]# docker node ls
ID                                                HOSTNAME                  STATUS      AVAILABILITY  MANAGER STATUS
rpbey5t1v14olke2mgtc430de     swarm-node2                 Ready        Active       
u6gkfr4j19gq16ddyb76fxsl3       swarm-node1                 Ready        Active       
vwbb0imil512a1le04bnkx98u *   swarm-manager-node    Ready       Active                      Unreachable
ybjvaszg838upeqvvzswhq0tt       swarm-manager-node2  Ready       Active                      Leader

7) 使旧令牌无效并生成新令牌

docker swarm join-token --rotate

8) 查看集群中的节点

docker node ls

9) 查看集群中节点信息

docker node inspect swarm-node1 --pretty

10) 调度程序可以将任务分配给节点

docker node update --availability active swarm-node1

11) 调度程序不向节点分配新任务，但是现有任务仍然保持运行

docker node update --availability pause swarm-node1

12) 调度程序不会将新任务分配给节点。调度程序关闭任何现有任务并在可用节点上安排它们. 也就是线下节点,不参与任务分配.

docker node update --availability drain swarm-node1

13) 添加节点标签

docker node update --label-add label1 --label-add bar=label2 swarm-node1

14) 删除节点标签

docker node update --label-rm label1 swarm-node1

15) 将worker节点升级为manager节点

docker node promote swarm-node1

16) 将manager节点降级为worker节点

docker node demote swarm-manager-node

17) 查看服务列表

docker service ls

18) 查看服务的具体信息

# docker service ps my-test

19) 创建一个不定义name，不定义replicas的服务. (如下的nginx是docker的nginx镜像名称,不是服务名称)

# docker service create nginx

20) 创建一个指定name的服务

# ocker service create --name my-nginx nginx

21) 创建一个指定name、run cmd的服务

# docker service create --name my-nginx nginx ping www.baidu.com

22) 创建一个指定name、version、run cmd的服务

# docker service create --name my-redis redis:3.0.6

# docker service create --name my-nginx nginx:1.8 /bin/bash

23) 创建一个指定name、port、replicas的服务

# docker service create --name my-nginx --replicas 3 -p 80:80 nginx

24) 为指定的服务更新一个端口

# docker service update --publish-add 80:80 my-nginx

25) 为指定的服务删除一个端口

# docker service update --publish-rm 80:80 my-nginx

26) 将redis:3.0.6更新至redis:3.0.7

# docker service update --image redis:3.0.7 redis

27) 配置运行环境，指定工作目录及环境变量

# docker service create --name my-nginx --env MYVAR=myvalue --workdir /data/www --user my_user nginx 

28) 创建一个my-nginx的服务

# docker service create --name my-nginx nginx 

29) 更新my-nginx服务的运行命令

# docker service update --args "ping www.baidu.com" my-nginx

30) 删除一个服务

# docker service rm my-nginx

31) 在每个群组节点上运行web服务

# docker service create --name tomcat --mode global --publish mode=host,target=8080,published=8080 tomcat:latest

32) 创建一个overlay网络

# docker network create --driver overlay my-network

# docker network create --driver overlay --subnet 10.10.10.0/24 --gateway 10.10.10.1 haha-network

33) 创建服务并将网络添加至该服务

# docker service create --name my-test --replicas 3 --network my-network redis

34) 删除群组网络

# docker service update --network-rm my-network my-test

35) 更新群组网络

# docker service update --network-add haha-network my-test

36) 创建群组并配置cpu和内存

# docker service create --name my_nginx --reserve-cpu 2 --reserve-memory 512m --replicas 3 nginx

37) 更改所分配的cpu和内存

# docker service update --reserve-cpu 1 --reserve-memory 256m my_nginx

38) 创建服务时自定义的几个参数
指定每次更新的容器数量
--update-parallelism
指定容器更新的间隔
--update-delay
定义容器启动后监控失败的持续时间
--update-monitor
定义容器失败的百分比
--update-max-failure-ratio
定义容器启动失败之后所执行的动作
--update-failure-action
比如:创建一个服务并运行3个副本，同步延迟10秒，10%任务失败则暂停

# docker service create --name mysql_5_6_36 --replicas 3 --update-delay 10s --update-parallelism 1 --update-monitor 30s --update-failure-action pause --update-max-failure-ratio 0.1 -e MYSQL_ROOT_PASSWORD=123456 mysql:5.6.36

39) 回滚至之前版本

# docker service update --rollback mysql

自动回滚
如果服务部署失败，则每次回滚2个任务，监控20秒，回滚可接受失败率20%

# docker service create --name redis --replicas 6 --rollback-parallelism 2 --rollback-monitor 20s --rollback-max-failure-ratio .2 redis:latest

40) 创建服务并将目录挂在至container中

# docker service create --name mysql --publish 3306:3306 --mount type=bind,src=/data/mysql,dst=/var/lib/mysql --replicas 3 -e MYSQL_ROOT_PASSWORD=123456 mysql:5.6.36

需要注意使用bind绑定宿主机目录会带来的风险

- 绑定的主机路径必须存在于每个集群节点上，否则会有问题;
- 调度程序可能会在任何时候重新安排运行服务容器，如果目标节点主机变得不健康或无法访问;
- 主机绑定数据不可移植，当你绑定安装时，不能保证你的应用程序开发方式与生产中的运行方式相同;
41) 添加swarm配置

# echo "this is a mysql config" | docker config create mysql -

42) 查看配置

# docker config ls

查看配置详细信息

# docker config inspect mysql

43) 删除配置

# docker config rm mysql

44) 添加配置

# docker service update --config-add mysql mysql

45) 删除配置

# docker service update --config-rm mysql mysql

46) 添加配置

# docker config create kevinpage index.html

47) 启动容器的同时添加配置(target如果报错,就使用dst或destination)

# docker service create --name nginx --publish 80:80 --replicas 3 --config src=kevinpage,target=/usr/share/nginx/html/index.html nginx
```

