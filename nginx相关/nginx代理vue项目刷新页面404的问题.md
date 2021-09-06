# nginx中vue项目刷新404问题

```nginx
server {
        listen       80;
        server_name  xxx.xx.com;
        server_name  111.85.70.23;
#       rewrite ^(.*) https://$server_name$1 permanent;
        return 301 https://$server_name$request_uri;
 }

server {
        listen  443 ssl;
        server_name  xxx.xx.com;
        ssl_certificate /etc/nginx/5696469_xxx.xx.com.pem;
        #私钥文件名称
        ssl_certificate_key /etc/nginx/5696469_xxx.xx.com.key;
        ssl_session_timeout 5m;
        #请按照以下协议配置
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
        ssl_prefer_server_ciphers on;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        # 解决刷新页面变成404问题的代码
        try_files $uri $uri/ /index.html; 
    }

    location /ft5otAaUNm.txt {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        default_type    text/html;
        # 直接返回值不需要将‘ft5otAaUNm.txt’文件放到html目录下
        return 200 "0d19a456aa06810d6330a423c176a8c6";
        }
    location /prod-api/ {
          proxy_pass         http://172.16.0.11:8080/;
          proxy_set_header   Host             $host;
          proxy_set_header   X-Real-IP        $remote_addr;
          proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
    }
}
```

