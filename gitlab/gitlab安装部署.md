# GitLab部署

1. 使用官方给出脚本添加 GitLab 包存储库

```shell
curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
```

2. 安装指定版本

```shell
yum install -y gitlab-ce-13.9.3-ce.0.el7.x86_64
```

![img](https://longlizl.github.io/gitlab/images/1.png)

3. 修改配置文件（配置访问地址及邮件服务）

```shell
vim /etc/gitlab/gitlab.rb
```

![img](https://longlizl.github.io/gitlab/images/2.png)

```shell
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "xxxxx.exmail.qq.com"
gitlab_rails['smtp_port'] = 25
gitlab_rails['smtp_user_name'] = "xxxxx@xxxxx.com.cn"
gitlab_rails['smtp_password'] = "xxxxx"
gitlab_rails['smtp_domain'] = "xxxxx.qq.com"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = false
gitlab_rails['gitlab_email_from'] = "xxxxxx@xxxxx.com.cn"
user["git_user_email"] = "xxxxx@xxxxx.com.cn"
```

4. gitlab重新加载配置文件

![img](https://longlizl.github.io/gitlab/images/3.png)

```shell
gitlab-ctl reconfigure
```

5. 启动服务

```shell
gitlab-ctl start
```

![img](https://longlizl.github.io/gitlab/images/4.png)

5. 邮件测试

```shell
gitlab-rails console
Notify.test_email('1550789579@qq.com','Message Subject','message Body').deliver_now
```

![img](https://longlizl.github.io/gitlab/images/5.png)

![img](https://longlizl.github.io/gitlab/images/11.png)

6. 浏览器访问http://192.168.205.250/

![image-20210907095020705](https://longlizl.github.io/gitlab/images/6.png)

7. 给访问加自签名证书提供https连接访问

   新建ssl目录存放证书文件

```shell
mkdir -p /etc/gitlab/ssl
cd /etc/gitlab/ssl
openssl genrsa -out git.com.cn.key 2048 openssl req -new -x509 -sha256 -key git.com.cn.key -out git.com.cn.crt -days 3650
```

生成证书过程会填写相关信息，在your name or your hostname那项填写域名：git.com.cn

​	修改配置文件：

```shell
vim /etc/gitlab/gitlab.rb
external_url 'https://git.com.cn'
nginx['redirect_http_to_https'] = true
# nginx['redirect_http_to_https_port'] = 80 （80端口可以不用设置）
nginx['ssl_certificat'] = "/etc/gitlab/ssl/git.com.cn.crt"
nginx['ssl_certificate_ke'] = "/etc/gitlab/ssl/git.com.cn.key"
```

​	gitlab重新加载配置文件

```shell
gitlab-ctl reconfigure
```

​	启动服务

```shell
gitlab-ctl start
```

在客户端添加hosts解析 192.168.205.250 git.com.cn

访问gitlab服务：

![img](https://longlizl.github.io/gitlab/images/7.png)

当使用git通过https协议拉取仓库代码报错

![img](https://longlizl.github.io/gitlab/images/8.png)

由于是自签名证书我们需要关掉验证

```shell
git config --global http.sslVerify false
```

![img](https://longlizl.github.io/gitlab/images/9.png)

# 项目备份与恢复

1. 修改备份路径及备份保留的时间（路径可以默认不用更改）

```shell
vim /etc/gitlab/gitlab.rb
gitlab_rails[‘backup_keep_time’] = 604800   # 单位S
```

2. 备份

   默认备份路径为：/var/opt/gitlab/backups

   ```shell
   gitlab-rake gitlab:backup:create
   ```

   ![image-20210625092258242](https://longlizl.github.io/gitlab/images/10.png)

   定时任务备份

   ```shell
   crontab -e
   0 2 * * * /opt/gitlab/bin/gitlab-rake gitlab:backup:create > /dev/null 2>&1 # 每天凌晨2点备份
   ```

   

3. 恢复

   在gitlab服务器上停止相关数据连接服务，命令如下：

   ```shell
   gitlab-ctl stop unicorn
   gitlab-ctl stop sidekiq
   ```

   恢复之前备份的项目

   ```shell
   gitlab-rake gitlab:backup:restore BACKUP=1624584033_2021_06_25_13.9.3 # 取备份文件时间戳
   ```

4. 重启服务

   ```shell
   gitlab-ctl restart
   ```

   