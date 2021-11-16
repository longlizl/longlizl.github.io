基本环境:

centos7.4，OpenLDAP使用2.4.44版本

## 一. 安装OpenLDAP

```shell
yum -y install openldap compat-openldap openldap-clients openldap-servers openldap-servers-sql openldap-devel migrationtools
```

查看OpenLDAP版本，使用如下命令：

```shell
[root@ldap-master ~]# slapd -VV
@(#) $OpenLDAP: slapd 2.4.44 (Sep 30 2020 17:16:39) $
	mockbuild@x86-02.bsys.centos.org:/builddir/build/BUILD/openldap-2.4.44/openldap-2.4.44/servers/slapd

```

## 二. 配置OpenLDAP

注意:从OpenLDAP2.4.23版本开始所有配置数据都保存在/etc/openldap/slapd.d/中，建议不再使用slapd.conf作为配置文件。

### 2.1 配置OpenLDAP管理员密码

设置OpenLDAP的管理员密码生成一个加密的秘钥:

```shell
[root@ldap-master ~]# slappasswd -s 123456
{SSHA}iuteV+jmpnQul9byuSdckSPK4olCz2ax(此密钥后面会用到)
```

### 2.2 修改olcDatabase={2}hdb.ldif文件

```shell
vim /etc/openldap/slapd.d/cn=config/olcDatabase\=\{2\}hdb.ldif
olcSuffix: dc=test,dc=com
olcRootDN: cn=admini,dc=test,dc=com
olcRootPW: {SSHA}iuteV+jmpnQul9byuSdckSPK4olCz2ax
```

![image-20211116142703342](https://longlizl.github.io/images/1.png)

### 2.3 修改olcDatabase={1}monitor.ldif文件

```shell
vim /etc/openldap/slapd.d/cn=config/olcDatabase\=\{1\}monitor.ldif
```

![image-20211116142801019](https://longlizl.github.io/images/2.png)

注意：该修改中的dn.base是修改OpenLDAP的管理员的相关信息的。

验证OpenLDAP的基本配置，使用如下命令：

```shell
[root@ldap-master ~]# slaptest -u
61934b66 ldif_read_file: checksum error on "/etc/openldap/slapd.d/cn=config/olcDatabase={1}monitor.ldif"
61934b66 ldif_read_file: checksum error on "/etc/openldap/slapd.d/cn=config/olcDatabase={2}hdb.ldif"
config file testing succeeded
```

启动OpenLDAP服务，使用如下命令：

```shell
systemctl enable slapd
systemctl start slapd
systemctl status slapd
```

![image-20211116141230808](https://longlizl.github.io/images/3.png)

OpenLDAP默认监听的端口是389，下面我们来看下是不是389端口，如下：

```shell
[root@ldap-master ~]# ss -ntupl | grep -w "389"
tcp    LISTEN     0      128       *:389                   *:*                   users:(("slapd",pid=57587,fd=8))
tcp    LISTEN     0      128    [::]:389                [::]:*                   users:(("slapd",pid=57587,fd=9))
```

### 2.4 配置OpenLDAP数据库

OpenLDAP默认使用的数据库是BerkeleyDB，现在来开始配置OpenLDAP数据库，使用如下命令：

```shell
cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
chown ldap:ldap -R /var/lib/ldap
chmod 700 -R /var/lib/ldap
```

ll /var/lib/ldap/

```shell
-rwx------. 1 ldap ldap     2048 1月   6 2021 alock
-rwx------. 1 ldap ldap   286720 1月   7 2021 __db.001
-rwx------. 1 ldap ldap    32768 1月   7 2021 __db.002
-rwx------. 1 ldap ldap    93592 1月   7 2021 __db.003
-rwx------. 1 ldap ldap      845 1月   6 2021 DB_CONFIG
-rwx------. 1 ldap ldap 10485760 1月   6 2021 log.0000000001
```

### 2.5 导入基本Schema

导入基本Schema，使用如下命令：

```shell
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```

### 2.6 修改migrate_common.ph文件

migrate_common.ph文件主要是用于生成ldif文件使用，修改migrate_common.ph文件，如下：

```shell
vim /usr/share/migrationtools/migrate_common.ph +71
```

![image-20211116143251773](https://longlizl.github.io/images/4.png)

到此OpenLDAP的配置就已经全部完毕，下面我们来开始添加用户到OpenLDAP中。

如果这时候直接使用ldap admin工具去连接会报错，原因是没基础组织结构需要将下面

(第四步）的基础数据库导入

**----------------------------------------------------------------------------------------------------------------------------**

## 三. 添加用户及用户组

使用命令行添加，后续介绍工具连接到ldap进行添加

默认情况下OpenLDAP是没有普通用户的，但是有一个管理员用户。管理用户就是前面我们刚刚配置的admin。

现在我们把系统中的用户，添加到OpenLDAP中。为了进行区分，我们现在新加两个用户test1和test2，和两个用户组cs1和cs2，如下：

添加用户组，使用如下命令：

```shell
groupadd cs1
groupadd cs2
```

添加用户并设置密码，使用如下命令：

```shell
useradd -g cs1 test1
useradd -g cs2 test2
echo "123456" | passwd --stdin test1
echo "123456" | passwd --stdin test2
```

把刚刚添加的用户和用户组提取出来，这包括该用户的密码和其他相关属性，如下：

```shell
grep ":10[0-9][0-9]" /etc/passwd > /root/users
grep ":10[0-9][0-9]" /etc/group > /root/groups
```

根据上述生成的用户和用户组属性，使用migrate_passwd.pl文件生成要添加用户和用户组的ldif，如下：

```shell
/usr/share/migrationtools/migrate_passwd.pl /root/users > /root/users.ldif
/usr/share/migrationtools/migrate_group.pl /root/groups > /root/groups.ldif
```

注意ldif文件格式规范（参考关于ldif文件创建格式和规范）

以/root/users.ldif文件为例，groups.ldif同users.ldif

![image-20211116154317855](https://longlizl.github.io/images/5.png)

**四.导入用户及用户组到OpenLDAP数据库**

配置openldap基础的数据库，如下：

```shell
cat > /root/base.ldif << EOF

dn: dc=test,dc=com
o: test com
dc: test
objectClass: top
objectClass: dcObject
objectclass: organization

dn: cn=admin,dc=test,dc=com
cn: admin
objectClass: organizationalRole
description: Directory Manager

dn: ou=People,dc=test,dc=com
ou: People
objectClass: top
objectClass: organizationalUnit

dn: ou=Group,dc=test,dc=com
ou: Group
objectClass: top
objectClass: organizationalUnit

EOF
```

注base.ldif文件格式如下若格式不对请更改有效格式

![image-20211116144428792](https://longlizl.github.io/images/6.png)

导入基础数据库，使用如下命令：（"123456"为最初设置的root管理员密码）

```shell
ldapadd -x -w  "123456" -c -D "cn=admin,dc=test,dc=com"  -f /root/base.ldif
```

导入用户到数据库，使用如下命令：

```shell
ldapadd -x -w  "123456" -c -D "cn=admin,dc=test,dc=com"  -f /root/users.ldif
```

导入用户组到数据库，使用如下命令：

```shell
ldapadd -x -w  "123456" -c -D "cn=admin,dc=test,dc=com"  -f /root/groups.ldif
```

**参数说明：**

- ```shell
  -x：进行简单的验证
  -D：用来绑定服务器的DN
  -w：绑定DN的密码
  -b：要查询的根节点
  -c：出错后继续执行程序不终止，默认出错即停止
  -f：从文件内读取信息还原，而不是标准输入
  ```

  

查看BerkeleyDB数据库文件，使用如下命令：

```shell
ll /var/lib/ldap/
```

```shell
-rwx------. 1 ldap ldap     2048 1月   6 2021 alock
-rw-------. 1 ldap ldap     8192 1月   6 2021 cn.bdb
-rwx------. 1 ldap ldap   286720 1月   7 2021 __db.001
-rwx------. 1 ldap ldap    32768 1月   7 2021 __db.002
-rwx------. 1 ldap ldap    93592 1月   7 2021 __db.003
-rwx------. 1 ldap ldap      845 1月   6 2021 DB_CONFIG
-rwx------. 1 ldap ldap     8192 1月   6 2021 dn2id.bdb
-rw-------. 1 ldap ldap     8192 1月   6 2021 givenName.bdb
-rwx------. 1 ldap ldap    32768 1月   6 2021 id2entry.bdb
-rwx------. 1 ldap ldap 10485760 1月   6 2021 log.0000000001
-rw-------. 1 ldap ldap     8192 1月   6 2021 objectClass.bdb
-rw-------. 1 ldap ldap     8192 1月   6 2021 ou.bdb
-rw-------. 1 ldap ldap     8192 1月   6 2021 sn.bdb
```

可以很明显的看到此时BerkeleyDB数据库文件中多了cn.bdb、sn.bdb、ou.bdb等数据库文件。

## 五. 查询OpenLDAP的相关信息

用户和用户组全部导入完毕后，我们就可以查询OpenLDAP的相关信息。

查询OpenLDAP全部信息，使用如下命令：

```shell
ldapsearch -x -b "dc=test,dc=com" -H ldap://127.0.0.1
```

![image-20211116143417872](https://longlizl.github.io/images/7.png)

查询添加的OpenLDAP用户信息，使用如下命令：

```shell
[root@ldap-master ~]# ldapsearch -LLL -x -D "cn=admin,dc=test,dc=com" -w "123456" -b "dc=test,dc=com" "uid=test1"
dn: uid=test1,ou=People,dc=test,dc=com
uid: test1
cn: test1
sn: test1
mail: test1@test.com
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: top
objectClass: shadowAccount
userPassword:: e2NyeXB0fSQ2JGx6Z0xsUWVOJHNZVUtrVEt2Y1ZyNXE5ZDFCaG5abjhWeEpHWkV
 mcnVLc25WbVpLUTdrVlk1dmVpc3ZsV0FkMkt2bnByQ1BPU0VJREE0czVGUzZ3SThDWHdLQWl0Tncw
shadowLastChange: 18947
shadowMin: 0
shadowMax: 99999
shadowWarning: 7
loginShell: /bin/bash
uidNumber: 1002
gidNumber: 1000
homeDirectory: /home/test1
```

查询添加的OpenLDAP用户组信息，如下：

```shell
[root@ldap-master ~]# ldapsearch -LLL -x -D "cn=admin,dc=test,dc=com" -w "123456" -b "dc=test,dc=com" "cn=cs1"
dn: cn=cs1,ou=Group,dc=test,dc=com
objectClass: posixGroup
objectClass: top
cn: cs1
userPassword:: e2NyeXB0fXg=
gidNumber: 1000
```

**五.把OpenLDAP用户加入到用户组**

尽管我们已经把用户和用户组信息，导入到OpenLDAP数据库中了。但实际上目前OpenLDAP用户和用户组之间是没有任何关联的。

如果我们要把OpenLDAP数据库中的用户和用户组关联起来的话，我们还需要做另外单独的配置。

现在我们要把ldapuser1用户加入到ldapgroup1用户组，需要新建添加用户到用户组的ldif文件，如下：

```shell
cat > add_user_to_groups.ldif <<  EOF
dn: cn=cs1,ou=Group,dc=test,dc=com
changetype: modify
add: memberuid
memberuid: test1
EOF
```

```shell
[root@ldap-master ~]# ldapadd -x -w  "123456" -c -D "cn=admin,dc=test,dc=com"  -f add_user_to_groups.ldif 
modifying entry "cn=cs1,ou=Group,dc=test,dc=com"
```

查询添加的OpenLDAP用户组信息，如下：

```shell
[root@ldap-master ~]# ldapsearch -LLL -x -D "cn=admin,dc=test,dc=com" -w "123456" -b "dc=test,dc=com" "cn=cs1"
dn: cn=cs1,ou=Group,dc=test,dc=com
objectClass: posixGroup
objectClass: top
cn: cs1
userPassword:: e2NyeXB0fXg=
gidNumber: 1000
memberUid: test1
```

我们可以很明显的看出test1用户已经加入到cs1户组了。

**六.使用LDAPADMIN工具（windows客户端）**

![image-20211116152453827](https://longlizl.github.io/images/8.png)

![image-20211116161612567](https://longlizl.github.io/images/9.png)

**七.OpenLDAP客户端配置，实现用户认证**

准备一台linux作为ldap认证登录的客户端

1）安装依赖环境

```shell
yum -y install nss-pam-ldapd openldap-clients openldap 
```

nss-pam-ldapd，是pam模块和nss模块的集合，主要作用是使存在于服务端ldap数据库中的用户，进行ssh登陆客户端时，可以通过pam方式进行验证，而这种情况下此用户是不存在于客户端的服务器上的。

openldap-clients，就是OpenLDAP的客户端软件包，此软件包安装后，不需要像服务端一样运行起来，他的作用主要是集成了类似ldapsearch，ldapadd之类的命令，可以用户验证服务端或者对服务端数据库中的用户信息进行查询，添加等。

openldap，主要包含了OpenLDAP所必须的库文件，当通过pam验证时，这些库文件是必须有的。

2）设置ldap认证（使用可视化比较方便不用改很多配置文件）

```shell
authconfig-tui
```

![img](https://longlizl.github.io/images/10.png)

![img](https://longlizl.github.io/images/11.png)

可以验证下配置文件都是否生效

```shell
vim /etc/openldap/ldap.conf
```

![img](https://longlizl.github.io/images/12.png)

```shell
vim /etc/nsswitch.conf
```

![img](https://longlizl.github.io/images/13.png)

```shell
vim /etc/pam.d/system-auth
```

![img](https://longlizl.github.io/images/14.png)

```shell
vim /etc/pam.d/password-auth
```

![img](https://longlizl.github.io/images/15.png)

```shell
vim /etc/sysconfig/authconfig
```

![img](https://longlizl.github.io/images/16.png)

增加下面一行让ldap认证的用户自己创建家目录

```shell
vim /etc/pam.d/sshd
session    required     pam_mkhomedir.so
```

![img](https://longlizl.github.io/images/17.png)

```shell
systemctl restart nslcd
```

通过xhell连接访问验证ldap连接

![img](https://longlizl.github.io/images/18.png)

![img](https://longlizl.github.io/images/19.png)

![img](https://longlizl.github.io/images/20.png)