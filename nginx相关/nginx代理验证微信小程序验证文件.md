nginx代理验证小程文件序配置

```nginx
server {
      listen  443 ssl;
      server_name  testwx.***.com.cn;
      ssl_certificate /etc/nginx/1_testwx.***.com.cn_bundle.crt;
      #私钥文件名称
      ssl_certificate_key /etc/nginx/2_testwx.***.com.cn.key;
      ssl_session_timeout 5m;
      #请按照以下协议配置
      ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
      #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
      ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
      ssl_prefer_server_ciphers on;
      #charset koi8-r;
      #access_log  /var/log/nginx/host.access.log  main;
    
      # 8DX997K7df.txt为微信小程序验证文件
      location /8DX997K7df.txt {	
            root   /usr/share/nginx/html;
            index  index.html index.htm;
            default_type    text/html;
            return 200 "566d985213eca2a8a6e47d2f5ba9e92f";  # 文件里字符串，这个操作可以不将验证文件上传至网站根目录
        }

      location / {
            proxy_pass         http://178.12.22.22:8080;
            proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
      }
  }
```

