# Linux搭建Socks5代理服务器

## 1、首先，编译安装SS5需要先安装一些依赖组件

```shell
yum -y install gcc gcc-c++ automake make pam-devel openldap-devel cyrus-sasl-devel openssl-devel
```

## 2、去官网下载SS5最新版本的源代码

```shell
wget -c https://nchc.dl.sourceforge.net/project/ss5/ss5/3.8.9-8/ss5-3.8.9-8.tar.gz
```

## 3、解压后开始编译安装：

```shell
  tar zxvf ./ss5-3.8.9-8.tar.gz
　cd ss5-3.8.9
  ./configure
  make
  make install
```

## 4、设置ss5开机启动启动

```shell
  chmod +x /etc/init.d/ss5
  chkconfig --add ss5
  chkconfig --level 345 ss5 on
```

## 5、在/etc/opt/ss5/ss5.conf中找到auth和permit两行，按照下面的格式进行修改则不需要验证

![img](https://longlizl.github.io/代理服务/images/3.png)

![img](https://longlizl.github.io/代理服务/images/4.png)

 6、ss5 默认使用1080端口，如果要修改默认端口，请修改/etc/sysconfig/ss5　

```shell
在/etc/sysconfig/ss5这个文件中，添加下面这一行命令，-b后面的参数代表监听的ip地址和端口号
　　# Add startup option here
　　SS5_OPTS=" -u root -b 0.0.0.0:8080"
```

7、启动ss5

```shell
  service ss5 start
```

8、云服务器一定要记得配置安全组开放SS5监听的端口

9、使用QQ代理测试：

   

![img](https://longlizl.github.io/代理服务/images/5.png)

 

## 10，如果需要配置访问权限，请按如下修改：

```shell
a、开启用户名密码验证机制 
 vim /etc/opt/ss5/ss5.conf
 在ss5.conf中找到auth和permit两行，按照下面的格式进行修改
 auth   0.0.0.0/0    -     u
 permit u    0.0.0.0/0    -    0.0.0.0/0    -    -    -    -    -

 b 、设置用户名和密码 
 vim /etc/opt/ss5/ss5.passwd
  一行一个账号，用户名和密码之间用空格间隔，例如：
  user1 111111
  user2 111111

c、重启服务生效
   service ss5 restart
   
```

## 11，创建用户分组，以方便给不同的用户分配不同的访问权限：

```shell
在/etc/opt/ss5目录中创建以用户分组名命名的文件，然后在相应的组用户文件中添加相应的用户。
 需要创建两组用户：
-  不受限制用户组:ulimit
-  受限制用户组:limit
在/etc/opt/ss5目录里面创建ulimit和limit这两个文件，然后在这两个文件中分别填入 /etc/opt/ss5/ss5.passwd中已添加的用户，格式依旧是每行一个用户（不需要填写用户密码）。请注意！/etc/opt/ss5目录下的这些文件必须能被Ss5服务的执行用户有读取权限（Ss5服务的默认执行用户是nobody）。 
```

## 12，设置不同用户组的访问控制：

```shell
# ulimit组用户不受限制
permit u     0.0.0.0/0    -    0.0.0.0/0    -    -    ulimit -    -
# limit组用户限制流量为64k
permit u     0.0.0.0/0    -    0.0.0.0/0    -    -    limit 64000 -
# 拒绝所有人ip访问google
deny u     0.0.0.0/0    -    www.google.cn    -    -    - - -
```

​	可以看到访问google的请求被拒绝了

![image-20210722161149302](https://longlizl.github.io/代理服务/images/6.png)

