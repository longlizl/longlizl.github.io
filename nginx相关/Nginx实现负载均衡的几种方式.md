# nginx负载均衡的五种策略

nginx的upstream目前支持的5种方式的分配

## 1、轮询（默认）

每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除。

```nginx
upstream backserver {

server 192.168.0.14;

server 192.168.0.15;

}
```



## 2、指定权重

指定轮询几率，weight和访问比率成正比，用于后端服务器性能不均的情况。

```nginx
upstream backserver {

server 192.168.0.14 weight=10;

server 192.168.0.15 weight=10;

}
```



## 3、IP绑定 ip_hash

每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器，可以解决session的问题。

```nginx
upstream backserver {

ip_hash;

server 192.168.0.14:88;

server 192.168.0.15:80;

}
```



## 4、fair（第三方）

按后端服务器的响应时间来分配请求，响应时间短的优先分配。

```nginx
upstream backserver {

server server1;

server server2;

fair;

}
```



## 5、url_hash（第三方）

按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，后端服务器为缓存时比较有效。

```nginx
upstream backserver {

server squid1:3128;

server squid2:3128;

hash $request_uri;

hash_method crc32;

}
```

eg:

```nginx
server {
    listen       80;     
    server_name  test.com.cn;       
    access_log  /var/log/nginx/blog.log main;  
location / {
    proxy_pass 	http://backserver/;                             
    proxy_set_header   Host             $host;                    
    proxy_set_header   X-Real-IP        $remote_addr;                                     
    proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;         
    proxy_next_upstream error timeout invalid_header http_404 http_500 http_502 http_504;  #当后端服务器中一台返回error timeout或错误码404、500...时，将请求分配给一下台服务器继续处理
	}
}

# 在需要使用负载均衡的server中增加
upstream backserver{
# 根据客户端IP地址的hash值分配固定的后端服务器，否则将使用轮询方式转发数据
ip_hash; 
# down 表示单前的server暂时不参与负载
server 127.0.0.1:9090 down; 
# weight 默认为1.weight越大，负载的权重就越大
server 127.0.0.1:8080 weight=2; 
# max_fails，允许请求失败的次数默认为1.当超过最大次数时，则认为服务器处理无效状态，fail_timeout定义连接后端服务器超时时间
server 127.0.0.1:6060 weight=4 max_fails=2 fail_timeout=30s; 
# 其它所有的非backup机器down或者忙的时候，请求backup机器
server 127.0.0.1:7070 backup; 
}
```

