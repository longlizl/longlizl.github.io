使用逻辑备份恢复数据库

## 本页目录：

- [操作场景]()
- 操作步骤
  - [步骤1：下载备份文件]()
  - [步骤2：解包备份文件]()
  - [步骤3：解压备份文件]()
  - [步骤4：导入备份至目标数据库]()

## 操作场景

> 说明：
>
> 为节约存储空间，云数据库 MySQL 的物理备份和逻辑备份文件，都会先经过 qpress 压缩，后经过 xbstream 打包（xbstream 为 Percona 的一种打包/解包工具）进行压缩与打包。

云数据库 MySQL 支持 [逻辑备份](https://cloud.tencent.com/document/product/236/35172) 方式，用户可通过控制台手动备份来生成逻辑备份文件，并下载获取整实例/部分库表的逻辑备份文件，本文为您介绍如何使用逻辑备份文件进行手动还原。

## 操作步骤

### 步骤1：下载备份文件

1. 登录 [MySQL 控制台](https://console.cloud.tencent.com/cdb)，在实例列表，单击实例 ID 或“操作”列的【管理】，进入实例管理页面。

2. 在实例管理页面，选择【备份恢复】>【数据备份列表】页， 选择需要下载的备份，在“操作”列单击【下载】。

3. 在弹出的对话框，推荐您复制下载地址，并

    

   登录到云数据库所在 VPC 下的 CVM（Linux 系统）

   中，运用 wget 命令进行内网高速下载，更高效。

   > 说明：
   >
   > 
   >
   > - 您也可以选择【本地下载】直接下载，但耗时较多。
   > - wget 命令格式：wget -c '备份文件下载地址' -O 自定义文件名.xb

   示例如下：

   ```shell
   wget -c 'https://mysql-database-xxx-bj-001.cos.ap-beijing.myqcloud.com/12427%2Fmysql%2F42d-11ea-b887-6c0b82b%2Fdata%2Fautomatic-delete%2F2019-11-28%2Fautomatic%2Fxtrabackup%2Fbk_204_10385%2Fcdb-1pe7bexs_backup_20191128044644.xb?sign=q-sign-algorithm%3Dsha1%26q-ak%3D1%26q-sign-time%3D1574269%3B1575417469%26q-key-time%3D1575374269%3B1517469%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3Dfb8fad13c4ed&response-content-disposition=attachment%3Bfilename%3D%2141731_backup_20191128044644.xb%22&response-content-type=application%2Foctet-stream' -O t_battery_statu_hour.xb
   ```

### 步骤2：解包备份文件

使用 xbstream 解包备份文件。

> 说明：
>
> 下载安装
>
> ```shell
> wget https://www.percona.com/downloads/XtraBackup/Percona-XtraBackup-2.4.4/\
> binary/redhat/7/x86_64/percona-xtrabackup-24-2.4.4-1.el7.x86_64.rpm
> yum localinstall percona-xtrabackup-24-2.4.4-1.el7.x86_64.rpm
> xbstream -x < t_battery_statu_hour.xb
> ```
>
> 

解包结果如下所示：

```shell
[root@cs t_battery_statu_hour]# ls
cdb-l8jqhzex_backup_20210820110138.sql.qp  t_battery_statu_hour.xb
```

### 步骤3：解压备份文件

1. 通过如下命令下载 qpress 工具。

   ```shell
   curl -O http://www.quicklz.com/qpress-11-linux-x64.tar
   ```

   > 说明：
   >
   > 若 wget 下载提示错误，您可至 [quicklz](http://www.quicklz.com/) 下载 qpress 工具到本地后，再将 qpress 工具上传至 Linux 云服务器，请参见 [通过 SCP 上传文件到 Linux 云服务器](https://cloud.tencent.com/document/product/213/2133)。

2. 通过如下命令解出 qpress 二进制文件。

   ```shell
   tar -xf qpress-11-linux-x64.tar -C /usr/local/bin
   source /etc/profile
   ```

3. 使用 qpress 解压备份文件。

   ```shell
   qpress -d cdb-jp0zua5k_backup_20191202182218.sql.qp .
   # 可以看到目录里多了一个sql文件
   [root@cs t_battery_statu_hour]# ls
   cdb-l8jqhzex_backup_20210820110138.sql  cdb-l8jqhzex_backup_20210820110138.sql.qp  t_battery_statu_hour.xb
   ```
   
   > 说明：
   >
   > 请根据解压时间，找到`.sql.qp`后缀的备份文件，并将`cdb-jp0zua5k_backup_20191202182218`替换为该文件名。
   
   解压结果如下图所示：
   
   

### 步骤4：导入备份至目标数据库

执行如下命令导入 sql 文件至目标数据库：

```
mysql -uroot -P3306 -h127.0.0.1 -p < ccdb-12xfg_backup_20210820110138.sql
```