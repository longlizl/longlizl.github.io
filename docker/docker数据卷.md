# 数据卷Volumes

介绍2种数据卷持久化挂载方式

> 1. 宿主机目录挂载方式
> 2. 卷名挂载方式

## 1. 宿主机目录挂载方式

```shell
# docker run -d -p port:port --name 自定义容器名 -v 宿主机路径:容器内路径 镜像名
ex: 
docker run -d -p 8080:8080 --name tomcat-test -v /opt/apps:/usr/local/tomcat/webapps tomcat
```

执行下面命令可以看到宿主机的挂载目录

```shell
[root@cs ~]# docker inspect -f '{{(index .Mounts 0).Source}}' tomcat-test
/opt/apps
```

![image-20210918105724800](https://longlizl.github.io\docker\images\5.png)

在/opt/apps下新建个ROOT根目录再新建个index.html文件测试

```shell
mkdir /opt/apps/ROOT
echo 'hello tomcat' > /opt/apps/ROOT/index.html
```

![image-20210918111116624](https://longlizl.github.io\docker\images\6.png)

## 2. 卷名挂载方式

```shell
# docker run -d -p 8081:8080 -v 卷名:容器内路径 镜像名
注：此方式挂载如果容器目录里有数据会复制一份到宿主机目录，下面使用镜像原本目录是没任何文件
ex:
docker run -d -p 8081:8080 --name tomcat-01 -v v_tomcat:/usr/local/tomcat/webapps tomcat
```

可以看到下图已经自动新建好了一个卷名（不必单独去新建卷名，容器被删后数据还在重新起个容器挂载进去就好）

![image-20210918111350257](https://longlizl.github.io\docker\images\7.png)

我们在看下持久化目录在哪里 

```shell
[root@cs apps]# docker inspect -f '{{(index .Mounts 0).Source}}' tomcat-01
/var/lib/docker/volumes/v_tomcat/_data
```

做同样测试

```shell
[root@cs apps]# mkdir /var/lib/docker/volumes/v_tomcat/_data/ROOT
[root@cs apps]# echo 'hello tomcat' > /var/lib/docker/volumes/v_tomcat/_data/ROOT/index.html
```

![image-20210918112847235](https://longlizl.github.io\docker\images\8.png)

我们试着删除此容器，重新挂载到新启动的容器中

```shell
[root@cs apps]# docker rm -f tomcat-01
[root@cs apps]# docker run -d -p 9090:8080 --name tomcat-02 -v v_tomcat:/usr/local/tomcat/webapps tomcat
[root@cs apps]# docker ps
```

![image-20210918113559354](https://longlizl.github.io\docker\images\9.png)

查看挂载信息

![image-20210918113856946](https://longlizl.github.io\docker\images\10.png)

访问结果

![image-20210918113725776](https://longlizl.github.io\docker\images\11.png)

