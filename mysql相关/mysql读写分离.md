mysql-proxy实现读写分离

1、安装mysql-proxy

实现读写分离是有lua脚本实现的，现在mysql-proxy里面已经集成，无需再安装

下载：http://dev.mysql.com/downloads/mysql-proxy/ 一定要下载对应的版本

tar zxvf mysql-proxy-0.8.5-linux-glibc2.3-x86-32bit.tar.gz mv mysql-proxy-0.8.5-linux-glibc2.3-x86-32bit /usr/local/mysql-proxy

 

2、配置mysql-proxy，创建主配置文件

cd /usr/local/mysql-proxy mkdir lua #创建脚本存放目录 mkdir logs #创建日志目录 cp share/doc/mysql-proxy/rw-splitting.lua ./lua #复制读写分离配置文件 cp share/doc/mysql-proxy/admin-sql.lua ./lua #复制管理脚本 vi /etc/mysql-proxy.cnf   #创建配置文件 [mysql-proxy] user=root #运行mysql-proxy用户 admin-username=lin3615 #主从mysql共有的用户 admin-password=123456 #用户的密码 proxy-address=192.168.179.142:4040 #mysql-proxy运行ip和端口，不加端口，默认4040 proxy-read-only-backend-addresses=192.168.179.147 #指定后端从slave读取数据 proxy-backend-addresses=192.168.179.146 #指定后端主master写入数据 proxy-lua-script=/usr/local/mysql-proxy/lua/rw-splitting.lua #指定读写分离配置文件位置 admin-lua-script=/usr/local/mysql-proxy/lua/admin-sql.lua #指定管理脚本 log-file=/usr/local/mysql-proxy/logs/mysql-proxy.log #日志位置 log-level=info #定义log日志级别，由高到低分别有(error|warning|info|message|debug) daemon=true    #以守护进程方式运行 keepalive=true #mysql-proxy崩溃时，尝试重启 #保存退出！ chmod 660 /etc/mysql-porxy.cnf

 

3、修改读写分离配置文件

vim /usr/local/mysql-proxy/lua/rw-splitting.lua if not proxy.global.config.rwsplit then proxy.global.config.rwsplit = {  min_idle_connections = 1, #默认超过4个连接数时，才开始读写分离，改为1  max_idle_connections = 1, #默认8，改为1  is_debug = false } end

4.测试读写分离

.在主服务器创建proxy用户用于mysql-proxy使用，从服务器也会同步这个操作

mysql> grant all on *.* to 'lin3615'@'192.168.179.142' identified by '123456';