# nginx配置ssl证书

```nginx
server {

      listen  443 ssl;

      server_name  test.hezhisheng.com.cn;

      ssl_certificate /etc/nginx/1_test.com.cn_bundle.crt;

     #私钥文件名称

      ssl_certificate_key /etc/nginx/2_test.com.cn.key;

      ssl_session_timeout 5m;

      #请按照以下协议配置

       ssl_protocols TLSv1 TLSv1.1 TLSv1.2;

      #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。 

       ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;

       ssl_prefer_server_ciphers on;

      #charset koi8-r;

      #access_log  /var/log/nginx/host.access.log  main;

          location / {

               proxy_pass         http://test.com.cn:6300;

               proxy_set_header   Host             $host;

               proxy_set_header   X-Real-IP        $remote_addr;

               proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;

     }

}
```

