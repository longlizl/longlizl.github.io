# linux全局代理

## 1. 设置代理环境变量：

> 环境变量格式:
>
> ```shell
> # 1.1 没有用户和密码的格式
> export <environment variable>=http://<IP address>:<port>
> # 1.2 有用户和密码的格式
> export <environment variable>=https://<user>:<password>@<IP address>:<port>
> #（注意：如果密码中也有一个 “@” 符号，则需要把 “@” 符号转义一下，转义成 %40）
> ```

## 2. Linux 代理环境变量的种类

> ```shell
> （1） http_proxy
> （2） https_proxy
> （3） ftp_proxy
> （4） socket_proxy
> （5） all_proxy
> （6） no_proxy
> # 不走代理： localhost、127.0.0.1 和 ::1
> export no_proxy="localhost, 127.0.0.1, ::1"
> ```

## 3. 配置代理环境变量

> ```shell
> vim /etc/profile
> # 设置http协议走代理,代理服务器类型为socks5代理
> export http_proxy=socks5://test:xxxx@111.111.217.11:8899/
> 
> # 所有协议走代理,代理服务器类型为socks5代理
> export ALL_PROXY=socks5://test:xxxx@111.111.217.11:8899/
> ```
>
> ![image-20210806142823835](https://longlizl.github.io/代理服务/images/7.png)



