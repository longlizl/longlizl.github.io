# Nmap使用

## Nmap可以完成以下任务：

- 主机探测
- 端口扫描
- 版本检测
- 系统检测
- 支持探测脚本的编写

- Nmap在实际中应用场合如下：
- 通过对设备或者防火墙的探测来审计它的安全性
- 探测目标主机所开放的端口
- 通过识别新的服务器审计网络的安全性
- 探测网络上的主机

## 常见服务对应端口号：

| 服务                                         | 端口号 |
| -------------------------------------------- | ------ |
| HTTP                                         | 80     |
| HTTPS                                        | 443    |
| Telnet                                       | 23     |
| FTP                                          | 21     |
| SSH（安全登录）、SCP（文件传输）、端口重定向 | 22     |
| SMTP                                         | 25     |
| POP3                                         | 110    |
| WebLogic                                     | 7001   |
| TOMCAT                                       | 8080   |
| WIN2003远程登录                              | 3389   |
| Oracle数据库                                 | 1521   |
| MS SQL* SEVER数据库sever                     | 1433   |
| MySQL 数据库sever                            | 3306   |

## Nmap进行完整全面的扫描

```shell
nmap –T4 –A –v

-A选项用于使用进攻性（Aggressive）方式扫描；

-T4指定扫描过程使用的时序（Timing），总有6个级别（0-5），级别越高，扫描速度越快，但也容易被防火墙或IDS检测并屏蔽掉，在网络通讯状况良好的情况推荐使用T4；

-v表示显示冗余（verbosity）信息，在扫描过程中显示扫描的细节，从而让用户了解当前的扫描状态。
```

## Nmap用于主机发现的一些用法

```shell
-sL: List Scan 列表扫描，仅将指定的目标的IP列举出来，不进行主机发现。     

-sn: Ping Scan 只进行主机发现，不进行端口扫描。     

-Pn: 将所有指定的主机视作开启的，跳过主机发现的过程。     

-PS/PA/PU/PY[portlist]: 使用TCPSYN/ACK或SCTP INIT/ECHO方式进行发现。     

-PE/PP/PM: 使用ICMP echo, timestamp, and netmask 请求包发现主机。

-PO[protocollist]: 使用IP协议包探测对方主机是否开启。     

-n/-R: -n表示不进行DNS解析；-R表示总是进行DNS解析。     

--dns-servers <serv1[,serv2],...>: 指定DNS服务器。     

--system-dns: 指定使用系统的DNS服务器     

--traceroute: 追踪每个路由节点 
```

## Nmap用于端口扫描的一些用法

### 1、扫描方式选项

```shell
-sS/sT/sA/sW/sM:指定使用 TCP SYN/Connect()/ACK/Window/Maimon scans的方式来对目标主机进行扫描。     

-sU: 指定使用UDP扫描方式确定目标主机的UDP端口状况。      

-sN/sF/sX: 指定使用TCP Null, FIN, and Xmas scans秘密扫描方式来协助探测对方的TCP端口状态。      

--scanflags <flags>: 定制TCP包的flags。      

-sI <zombiehost[:probeport]>: 指定使用idle scan方式来扫描目标主机（前提需要找到合适的zombie host）      

-sY/sZ: 使用SCTP INIT/COOKIE-ECHO来扫描SCTP协议端口的开放的情况。      

-sO: 使用IP protocol 扫描确定目标机支持的协议类型。      

-b <FTP relay host>: 使用FTP bounce scan扫描方式  
```

### 2、 端口参数与扫描顺序

```shell
[plain] view plain copy -p <port ranges>: 扫描指定的端口     

实例: -p22; -p1-65535; -p U:53,111,137,T:21-25,80,139,8080,S:9（其中T代表TCP协议、U代表UDP协议、S代表SCTP协议）     

-F: Fast mode – 快速模式，仅扫描TOP 100的端口     

-r: 不进行端口随机打乱的操作（如无该参数，nmap会将要扫描的端口以随机顺序方式扫描，以让nmap的扫描不易被对方防火墙检测到）。     

--top-ports <number>:扫描开放概率最高的number个端口（nmap的作者曾经做过大规模地互联网扫描，以此统计出网络上各种端口可能开放的概率。以此排列出最有可能开放端口的列表，具体可以参见文件：nmap-services。默认情况下，nmap会扫描最有可能的1000个TCP端口）     

--port-ratio <ratio>: 扫描指定频率以上的端口。与上述--top-ports类似，这里以概率作为参数，让概率大于--port-ratio的端口才被扫描。显然参数必须在在0到1之间，具体范围概率情况可以查看nmap-services文件。
```

  

### 3、 版本侦测的用法

```shell
版本侦测方面的命令行选项比较简单。 [plain] view plain copy 

-sV: 指定让Nmap进行版本侦测     

--version-intensity <level>: 指定版本侦测强度（0-9），默认为7。数值越高，探测出的服务越准确，但是运行时间会比较长。     

--version-light: 指定使用轻量侦测方式 (intensity 2)     

--version-all: 尝试使用所有的probes进行侦测 (intensity 9)     

--version-trace: 显示出详细的版本侦测过程信息。 
```

## 具体操作演示如下

### 1、用Nmap扫描特定IP地址

```shell
┌──(root💀kali)-[~]
└─# nmap 192.168.205.106       
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-19 10:41 CST
Nmap scan report for 192.168.205.106
Host is up (0.00010s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 00:0C:29:42:8A:04 (VMware)

Nmap done: 1 IP address (1 host up) scanned in 0.47 seconds

```

### 2、用-vv对结果进行详细输出

```shell
┌──(root💀kali)-[~]
└─# nmap -vv 192.168.205.106
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-19 10:43 CST
Initiating ARP Ping Scan at 10:43
Scanning 192.168.205.106 [1 port]
Completed ARP Ping Scan at 10:43, 0.06s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 10:43
Completed Parallel DNS resolution of 1 host. at 10:43, 0.02s elapsed
Initiating SYN Stealth Scan at 10:43
Scanning 192.168.205.106 [1000 ports]
Discovered open port 80/tcp on 192.168.205.106
Discovered open port 22/tcp on 192.168.205.106
Completed SYN Stealth Scan at 10:43, 0.07s elapsed (1000 total ports)
Nmap scan report for 192.168.205.106
Host is up, received arp-response (0.000088s latency).
Scanned at 2021-08-19 10:43:43 CST for 0s
Not shown: 998 closed ports
Reason: 998 resets
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 64
80/tcp open  http    syn-ack ttl 64
MAC Address: 00:0C:29:42:8A:04 (VMware)

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 0.27 seconds
           Raw packets sent: 1001 (44.028KB) | Rcvd: 1001 (40.036KB)
```

### 3、自行设置端口范围进行扫描

```shell
┌──(root💀kali)-[~]
└─# nmap -p1-1024 192.168.205.106
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-19 10:45 CST
Nmap scan report for 192.168.205.106
Host is up (0.0060s latency).
Not shown: 1022 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 00:0C:29:42:8A:04 (VMware)

Nmap done: 1 IP address (1 host up) scanned in 0.68 seconds
```

### 4、指定端口号进行扫描

```shell
┌──(root💀kali)-[~]
└─# nmap -p22,23,80,25 192.168.205.106
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-19 10:46 CST
Nmap scan report for 192.168.205.106
Host is up (0.00029s latency).

PORT   STATE  SERVICE
22/tcp open   ssh
23/tcp closed telnet
25/tcp closed smtp
80/tcp open   http
MAC Address: 00:0C:29:42:8A:04 (VMware)

Nmap done: 1 IP address (1 host up) scanned in 0.23 seconds
```

### 5、对目标进行Ping扫描

```shell
格式：namp -sP <target ip>
```

```shell
┌──(root💀kali)-[~]
└─# nmap -sP 192.168.205.106
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-19 10:48 CST
Nmap scan report for 192.168.205.106
Host is up (0.00088s latency).
MAC Address: 00:0C:29:42:8A:04 (VMware)
Nmap done: 1 IP address (1 host up) scanned in 0.13 seconds
```

### 6、路由跟踪

```shell
nmap -traceroute <target ip>
```

```shell
┌──(root💀kali)-[~]
└─# nmap -traceroute www.baidu.com  
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-19 10:49 CST
Nmap scan report for www.baidu.com (14.215.177.39)
Host is up (0.0017s latency).
Other addresses for www.baidu.com (not scanned): 14.215.177.38
Not shown: 996 filtered ports
PORT    STATE SERVICE
25/tcp  open  smtp
80/tcp  open  http
110/tcp open  pop3
443/tcp open  https

TRACEROUTE (using port 80/tcp)
HOP RTT     ADDRESS
1   0.23 ms 192.168.205.2
2   0.29 ms 14.215.177.39

Nmap done: 1 IP address (1 host up) scanned in 51.46 seconds
```

### 7、扫描一个网段的主机在线状况

#### 1. 被扫描主机与扫描端主机同在一个网段

```shell
nmap -sP <network address > </CIDR>
或：
nmap -sn <network address > </CIDR>
nmap -Pn <network address > </CIDR> # 即扫描主机在线情况，也显示端口开放情况
```

```shell
┌──(root💀kali)-[~]
└─# nmap -sP 192.168.205.0/24
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-19 11:16 CST
Nmap scan report for 192.168.205.1
Host is up (0.00011s latency).
MAC Address: 00:50:56:C0:00:08 (VMware)
Nmap scan report for 192.168.205.2
Host is up (0.00026s latency).
MAC Address: 00:50:56:E3:62:AA (VMware)
Nmap scan report for 192.168.205.106
Host is up (0.0020s latency).
MAC Address: 00:0C:29:42:8A:04 (VMware)
Nmap scan report for 192.168.205.254
Host is up (0.00038s latency).
MAC Address: 00:50:56:F5:DF:34 (VMware)
Nmap scan report for 192.168.205.173
Host is up.
Nmap done: 256 IP addresses (5 hosts up) scanned in 6.24 seconds
```

#### 2. 如果扫描主机与被扫描主机不在同一网段

```shell
nmap -sP -PE <network address > </CIDR>
或：
nmap -sn -PE <network address > </CIDR>
```

```shell
┌──(root💀kali)-[~]
└─# nmap -sP -PE 172.16.1.0/24
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-19 11:46 CST
Nmap scan report for 172.16.1.1
Host is up (0.017s latency).
Nmap scan report for 172.16.1.5
Host is up (0.017s latency).
Nmap scan report for 172.16.1.8
Host is up (0.017s latency).
Nmap scan report for 172.16.1.101
Host is up (0.0039s latency).
Nmap scan report for 172.16.1.102
Host is up (0.0039s latency).
Nmap scan report for 172.16.1.103
Host is up (0.010s latency).
Nmap scan report for 172.16.1.104
Host is up (0.11s latency).
Nmap scan report for 172.16.1.106
Host is up (0.0084s latency).
Nmap scan report for 172.16.1.108
Host is up (0.0082s latency).
Nmap scan report for 172.16.1.111
Host is up (0.0018s latency).
Nmap scan report for 172.16.1.113
Host is up (0.13s latency).
Nmap scan report for 172.16.1.114
Host is up (0.012s latency).
Nmap scan report for 172.16.1.122
Host is up (0.0069s latency).
Nmap scan report for 172.16.1.124
Host is up (0.0068s latency).
Nmap scan report for 172.16.1.126
Host is up (0.0066s latency).
Nmap scan report for 172.16.1.131
Host is up (0.055s latency).
Nmap done: 256 IP addresses (16 hosts up) scanned in 2.28 seconds
```



### 8、操作系统探测

```shell
nmap -O <target ip>
```

```shell
┌──(root💀kali)-[~]
└─# nmap -O 192.168.205.106
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-19 11:18 CST
Nmap scan report for 192.168.205.106
Host is up (0.00099s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 00:0C:29:42:8A:04 (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop

OS detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 1.75 seconds
```

### 9、万能开关扫描

```shell
nmap -A <target ip>
```

```shell
# 可以看到此参数输出结果包含了很多详细信息
┌──(root💀kali)-[~]
└─# nmap -A 192.168.205.106
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-19 11:19 CST
Nmap scan report for 192.168.205.106
Host is up (0.00080s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 4c:0f:08:30:1a:34:64:ed:be:c0:14:da:70:91:dc:94 (RSA)
|   256 88:e3:33:6b:c8:95:62:84:ac:79:2e:e6:e4:58:5e:17 (ECDSA)
|_  256 86:e0:f5:d9:f1:2f:ed:3d:da:93:d6:72:32:69:10:8b (ED25519)
80/tcp open  http    nginx 1.20.1
|_http-server-header: nginx/1.20.1
|_http-title: Welcome to nginx!
MAC Address: 00:0C:29:42:8A:04 (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop

TRACEROUTE
HOP RTT     ADDRESS
1   0.80 ms 192.168.205.106

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.33 seconds
```



### 10、其他扫描方式

​	**SYN扫描**：利用基本的SYN扫描方式测试其端口开放状态

```shell
namp -sS -T4 <target ip>
```

```shell
┌──(root💀kali)-[~]
└─# nmap -sS -T4 192.168.205.106
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-19 11:22 CST
Nmap scan report for 192.168.205.106
Host is up (0.000093s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 00:0C:29:42:8A:04 (VMware)

Nmap done: 1 IP address (1 host up) scanned in 0.31 seconds
```

**FIN扫描**：利用FIN扫描方式探测防火墙状态。FIN扫描方式用于识别端口是否关闭，收到RST回复说明该端口关闭，否则说明是open或filtered状态

```shell
namp -sF -T4 <target ip>
```

​	**ACK扫描**：利用ACK扫描判断端口是否被过滤。针对ACK探测包，为被过滤的端口（无论打开或关闭）会回复RST包

```shell
namp -sA -T4 <target ip>
```

```shell
┌──(root💀kali)-[~]
└─# nmap -sA -T4 192.168.205.106
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-19 11:26 CST
Nmap scan report for 192.168.205.106
Host is up (0.000071s latency).
All 1000 scanned ports on 192.168.205.106 are unfiltered
MAC Address: 00:0C:29:42:8A:04 (VMware)

Nmap done: 1 IP address (1 host up) scanned in 0.28 seconds
```

扫描前不进行Ping扫描测试

```shell
nmap -Pn <target ip>
```

```shell
┌──(root💀kali)-[~]
└─# nmap -Pn 192.168.205.106  
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-19 11:55 CST
Nmap scan report for 192.168.205.106
Host is up (0.00015s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 00:0C:29:42:8A:04 (VMware)

Nmap done: 1 IP address (1 host up) scanned in 0.59 seconds
```

如果有一个ip地址列表，将这个保存为一个txt文件，和nmap在同意目录下，扫描这个txt的所有主机，命令为

```shell
nmap -iL target.txt
```

版本检测扫描

```shell
nmap -sV <target ip>
```

```shell
# 显示各端口所在服务及版本信息
┌──(root💀kali)-[~]
└─# nmap -sV 192.168.205.106
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-19 11:56 CST
Nmap scan report for 192.168.205.106
Host is up (0.0014s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4 (protocol 2.0)
80/tcp open  http    nginx 1.20.1
MAC Address: 00:0C:29:42:8A:04 (VMware)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.56 seconds
```

其他参数请自行查阅 man nmap



