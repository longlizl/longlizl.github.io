1.安装httpd-tools工具

```shell
yum install httpd-tools -y
```

2.生成用户名密码文件

```shell
htpasswd -c /etc/nginx/conf.d/.ngpasswd foms
```

回车输入密码

![image-20210526163658110](C:\Users\15507\AppData\Roaming\Typora\typora-user-images\image-20210526163658110.png)

3.安装nginx（略）

配置nginx

```shell
server {
    listen       80;
    server_name  localhost;
    #access_log  /var/log/nginx/host.access.log  main;
    location / {
        auth_basic           "input you user name and password";
        auth_basic_user_file /etc/nginx/conf.d/.ngpasswd;
        proxy_pass http://192.168.205.106:8081;
    }
    }  
```

重启nginx后访问：

![image-20210526164350310](C:\Users\15507\AppData\Roaming\Typora\typora-user-images\image-20210526164350310.png)

![image-20210526164449134](C:\Users\15507\AppData\Roaming\Typora\typora-user-images\image-20210526164449134.png)

也可以配置一个虚拟访问路径(访问时加上虚拟路径)

```shell
server {
    listen       80;
    server_name  localhost;
	location /login/ {
        auth_basic           "input you user name and password";
        auth_basic_user_file /etc/nginx/conf.d/.ngpasswd;
        proxy_pass http://192.168.205.106:8081/;
    }
```

```
http://192.168.205.106/login
```

![image-20210526170259027](C:\Users\15507\AppData\Roaming\Typora\typora-user-images\image-20210526170259027.png)

