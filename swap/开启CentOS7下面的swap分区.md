# 划分swap分区

## 1、查看当前内存和swap使用情况

```shell
free -h
```

## 2、开启swap

```shell
swapon -a
```

## 3、关闭swap

```shell
swapoff -a
```

## 4、swap推荐设置

```
4G以内的物理内存，SWAP 设置为内存的2倍。

4-8G的物理内存，SWAP 等于内存大小。

8-64G 的物理内存，SWAP 设置为8G。

64-256G物理内存，SWAP 设置为16G。
```

## 5、系统使用swap的规则阈值

（实际上，并不是等所有的物理内存都消耗完毕之后，才去使用swap的空间，什么时候使用是由swappiness 参数值控制。）

​	查看当前设置

```shell
cat /proc/sys/vm/swappiness
```

```
swappiness=0 的时候表示最大限度使用物理内存，然后才是 swap空间。

swappiness=100 的时候表示积极的使用swap分区，并且把内存上的数据及时的搬运到swap空间里面。
```

​	临时修改

```shell
sysctl -w vm.swappiness=10
```

​	永久修改，在/etc/sysctl.conf 文件里添加如下参数：

```shell
vm.swappiness=10
```

## 6、使用文件作为swap交换分区

```shell
# 生成swap-file，大小为1G
dd if=/dev/zero of=/swap-file bs=1M count=1024
```

```shell
# 将交换文件格式化为swap分区，记录UUID
mkswap /swap-file
Setting up swapspace version 1, size = 8388604 KiB
no label, UUID=e7d93441-2606-4cd0-a5bd-c983f579d6cb
```

```shell
# 永久生效，配置/etc/fstab，更新UUID，新增/swap-file配置
/swap-file     swap         swap    defaults        0 0
```

## 7、启动swap分区

```shell
chmod 0600 swap-file

swapon /swap-file
```

