# nginx+waf防火墙

## 1.官网下载nginx源码包（nginx-1.20.0.tar.gz）

```shell
新建nginx安装目录

mkdir -p /opt/nginx

新增nginx运行用户

useradd -s /sbin/nologin -M nginx
```

## 2.安装依赖

```shell
yum -y install wget unzip gcc gcc-c++ make automake autoconf pcre pcre-devel zlib zlib-devel openssl openssl-devel libtool
```

## 3.安装lua相关包（waf配置需要）

下载luajit 2.0并安装 

```shell
wget http://luajit.org/download/LuaJIT-2.0.5.tar.gz

tar -xf tar -zxvf LuaJIT-2.0.5.tar.gz

cd LuaJIT-2.0.5

make && make install
```

安装ngx_devel_kit（nginx development kit）模块是一个拓展nginx服务器核心功能的模块，第三方模块开发可以基于它来快速实现。

```shell
wget https://github.com/simplresty/ngx_devel_kit/archive/v0.3.0.tar.gz

tar -xf ngx_devel_kit-0.3.0.tar.gz #nginx编译安装时需要此模块

安装nginx_lua_module

tar -xf lua-nginx-module-0.10.13.tar.gz #nginx编译安装时需要此模块
```

## 4.导入环境变量：

```shell
echo "export LUAJIT_LIB=/usr/local/lib" >> /etc/profile
echo "export LUAJIT_INC=/usr/local/include/luajit-2.0" >> /etc/profile
source /etc/profile
```

## 5.nginx编译安装

```shell
tar -zxvf nginx-1.20.0.tar.gz
cd nginx-1.20.0
./configure --prefix=/opt/nginx --user=nginx --group=nginx --add-module=/root/ngx_devel_kit-0.3.0 --add-module=/root/lua-nginx-module-0.10.13 --with-ld-opt="-Wl,-rpath,$LUAJIT_LIB"
make && make install
cd ../
chown -R nginx:nginx nginx
```

新建/opt/nginx/logs/hack/攻击日志目录，并赋予nginx用户对该目录的写入权限。

```shell
mkdir -p /opt/nginx/logs/hack/
chown -R nginx.nginx /opt/nginx/logs/hack/
chmod -R 755 /opt/nginx/logs/hack/
```

## 6.下载安装waf

```shell
wget https://github.com/loveshell/ngx_lua_waf/archive/master.zip

unzip master.zip -d /opt/nginx/conf/

cd /opt/nginx/conf/ 

mv ngx_lua_waf-master waf

chown -R nginx:nginx waf/
```

## 7.nginx配置如下

```nginx
# httpm
lua_package_path "/opt/nginx/conf/waf/?.lua";
lua_shared_dict limit 10m;
init_by_lua_file  /opt/nginx/conf/waf/init.lua;
access_by_lua_file /opt/nginx/conf/waf/waf.lua;
```

![image-20210601171442802](https://longlizl.github.io/nginx相关images/1.png)

配置config.lua里的waf规则目录(一般在waf/conf/目录下)：

```shell
cd /opt/nginx/conf/waf/conf
vim config.lua
```

![image-20210601172128491](https://longlizl.github.io/nginx相关images/2.png)



## 8.启动nginx服务完成配置

