# ssr代理

```shell
#!/bin/bash

wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/[docker-ce-18.09.8-3.el7.x86_64.rpm](https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-18.09.8-3.el7.x86_64.rpm)

yum install -y ./[docker-ce-18.09.8-3.el7.x86_64.rpm](https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-18.09.8-3.el7.x86_64.rpm)

systemctl start docker

systemct enable docker

curl -L https://github.com/docker/compose/releases/download/1.27.4/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

chmod +x /usr/local/bin/docker-compose

docker pull shadowsocks/shadowsocks-libev

mkdir ~/shadowsocks/

cd ~/shadowsocks/

curl -sSLO https://github.com/shadowsocks/shadowsocks-libev/raw/master/docker/alpine/docker-compose.yml

sed -i "s/aes-256-gcm/rc4-md5/g" docker-compose.yml

sed -i "s/9MLSpPmNt/Password@1/g" docker-compose.yml

docker-compose up -d

docker-compose ps
```

