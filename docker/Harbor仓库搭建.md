# 一、Harbor介绍

Docker容器应用的开发和运行离不开可靠的镜像管理，虽然Docker官方也提供了公共的镜像仓库，但是从安全和效率等方面考虑，部署私有环境内的Registry也是非常必要的。Harbor是由VMware公司开源的企业级的Docker Registry管理项目，它包括权限管理(RBAC)、LDAP、日志审核、管理界面、自我注册、镜像复制和中文支持等功能

# 二、环境准备

Harbor的所有服务组件都是在Docker中部署的，所以官方安装使用Docker-compose快速部署，所以需要安装Docker、Docker-compose。由于Harbor是基于Docker Registry V2版本，所以就要求Docker版本不小于1.10.0，Docker-compose版本不小于1.6.0

## 1、安装并启动Docker

```shell
安装所需的包。yum-utils提供了yum-config-manager 效用，并device-mapper-persistent-data和lvm2由需要 devicemapper存储驱动程序
```

## 2、安装并启动Docker

安装所需的包。yum-utils提供了yum-config-manager 效用，并device-mapper-persistent-data和lvm2由需要 devicemapper存储驱动程序

```shell
 dnf  install -y yum-utils device-mapper-persistent-data lvm2
```

设置docker-ce源

```shell
 yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo  #安装Docker CE

dnf install -y docker-ce 
```

下载安装docker-compose

```shell
curl -L https://github.com/docker/compose/releases/download/1.27.4/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
```

对二进制文件赋可执行权限

```
[root@localhost ~]# chmod +x /usr/local/bin/docker-compose
```

![img](https://longlizl.github.io\docker\images\12.png)

# 三、Harbor服务搭建及启动

下载Harbor安装文件

```shell
从GitHub上https://github.com/goharbor/harbor/releases下载指定版本的安装包

wget https://github.com/goharbor/harbor/releases/download/v2.1.0/harbor-online-installer-v2.1.0.tgz

[root@centos8_sc ~]# tar -zxvf harbor-online-installer-v2.1.0.tgz

[root@centos8_sc ~]# mv harbor /usr/local/

[root@centos8_sc ~]# cd /usr/local/harbor/

cp harbor.yml.tmpl  harbor.yml
```

这里没有配置https证书先取消

![img](https://longlizl.github.io\docker\images\13.png)

执行安装  ./install.sh  拉取镜像

![img](https://longlizl.github.io\docker\images\14.png)

查看容器启动状态

![img](https://longlizl.github.io\docker\images\15.png)

启动完成后，访问刚设置的hostname即可，默认是80端口，如果端口占用，可以去修改docker-compose.yml文件中，对应服务的端口映射

# 四、Harbor仓库使用

## 1、登录Web Harbor

![img](https://longlizl.github.io\docker\images\16.png)

使用admin用户登录，密码为111111

![img](https://longlizl.github.io\docker\images\17.png)

## 1、上传镜像到Harbor仓库

```shell
我们新建一个名称为test的项目，设置不公开。当项目设为公开后，任何人都有此项目下镜像的读权限。命令行用户不需要docker login就可以拉取此项目下的镜像。

新建项目后，使用admin用户提交本地nginx镜像到Harbor仓库
```

# 2、登录

使用docker login出现如下问题：

```
[root@centos8_sc harbor]# docker login http://192.168.205.178 Username: admin Password: 
```

![img](https://longlizl.github.io\docker\images\18.png)

解决方法：

```
vim /etc/docker/daemon.json

{

"insecure-registries":["http://192.168.205.178"]

}

重启Docker服务

systemctl daemon-reload

systemctl restart docker
```

![img](https://longlizl.github.io\docker\images\19.png)

## 3、给镜像打tag

![img](https://longlizl.github.io\docker\images\20.png)

```
docker tag goharbor/nginx-photon:v2.1.0 192.168.205.178/test/nginx:latest
```

![img](https://longlizl.github.io\docker\images\21.png)

## 4、push到仓库

```
docker push 192.168.205.178/test/nginx:latest
```

![img](https://longlizl.github.io\docker\images\22.png)

**登录查看test项目下镜像**

![image-20210907094036720](https://longlizl.github.io\docker\images\23.png)

