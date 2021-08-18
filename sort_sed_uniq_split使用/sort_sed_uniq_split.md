# sort,sed,uniq,split常用命令

## 1.Sort排序使用

```shell
1.sort file
```

​	把文件按字母的升序进行排序

```shell
2.sort -r file:
```

​	把文件按字母的降充进行排序

```shell
3.cat file l sort -t: -k 1 -r
```

进行分割后的第一列来倒序排序

## 2.uniq 使用

```shell
1.uniq -c file # 打印紧挨的重复行出现的次数 

2.uniq -d file # 只打印重复的行 

3.awk '{print $1}' /var/log/httpd/access_log  I sort | uniq -c   # 把apache网站的所有访问ip全部统计出来,并打印出统计次数
```



```shell
4.awk '/Failed/{print $(NF-3)}' /var/log/secure  | sort | uniq -c | sort -t' ' -k 1 -nr  #按照访问次数从高到低排序
```

![img](https://longlizl.github.io/sort_sed_uniq_split使用/images/1.png)

```shell
5.打印某一列数据并过滤空行（awk NF）并按照出现频率次数倒序排序
cat serverForkliftStatus_log_2020-12-29.log | grep -i hzszd | awk -F',' '{print $2}' | awk NF | sort | uniq -c | sort -t' ' -k 1 -nr
```

![img](https://longlizl.github.io/sort_sed_uniq_split使用/images/2.png)

##  3. sed行处理

```shell
1 sed -n '2'p file    # 只打印第二行,不打印其它的行

2 sed -n '1,4'p file  #  从第一行到第四行的记录

3 sed -n '/los/p' file  # 打印匹配los的行

4 sed -n '4,/los/'p file # 打印从第四行到匹配los的之间的所有行

5 sed '1,2'd file   # 把第一行和第二行全部删除

7 sed  -i ：直接修改读取的文件内容，而不是输出到终端。

8 sed '/^\s*$/d'  file  # 删除文件里空白行（grep  -Ev '^$' file 或  awk NF file)

9 sed -i 's/原字符串/新字符串/g' /home/1.txt

10 # 匹配文件里ip地址并将匹配的ip地址全部替换成另一个
sed -i -r 's/([0-9]{1,3}\.){3}[0-9]{1,3}/1.1.1.1/g' ip.txt
```

## 11.替换某行数据（c是替换覆盖）

```shell
sed -i "69c bind $IP_ADDR" /opt/redis/conf/redis.conf
sed -i "69c\bind $IP_ADDR" /opt/redis/conf/redis.conf #用\分隔便于区分建议使用
```

## 12.匹配以某字符串开头并将其替换

```shell
sed -i '/^logfile/c\logfile "/opt/redis/log/redis.log"' /opt/redis/conf/redis.conf
```

## 13.将before.sh的内容插入到catalina.sh的第一行之后

```shell
sed -i '1r /srv/tomcat8/bin/before.sh' /srv/tomcat8/bin/catalina.sh
```

## 14.去除行首，行尾，所有空格

```shell
sed 's/^[ ]*//g' test.txt  #去除行首空格

sed 's/[ ]*$//g' test.txt #去除行尾空格

sed 's/[[:space:]]//g'  test.txt  #去除所有空格
```

## 15.插入数据（匹配行前或后插入数据）

```shell
sed -i '/allow linux.com/i\allow linux.cn' the.conf.file #匹配行前插入一行数据

sed -i '/allow linux.com/a\allow linux.cn' the.conf.file #匹配行后插入一行数据
```

## 16.将[]中内容去掉

```shell
echo '[xxx12x3rrt5xxx]' | sed  's/\[[^][]*\]/[]/g'
```

![img](D:\software\youdao_file\weixinobU7Vji2jSDT8WUoQ-GPtcbtUpic\22ad0ac05168498aaae249c0c466a1cc\clipboard.png)

## 17.在多少行前后插入数据

```shell
sed -i '156i\     <listen_host>::</listen_host>' config.xml.bak #156行前

sed -i '156a\     <listen_host>::</listen_host>' config.xml.bak #156行后
```

# split语法

split [--help][--version][-<行数>][-b <字节>][-C <字节>][-l <行数>][要切割的文件][输出文件名]

## 参数说明：

```shell
 -<行数> : 指定每多少行切成一个小文件
 -b<字节> : 指定每多少字节切成一个小文件
- --help : 在线帮助
- --version : 显示版本信息
- -C<字节> : 与参数"-b"相似，但是在切 割时将尽量维持每行的完整性
- [输出文件名] : 设置切割后文件的前置文件名， split会自动在前置文件名后再加上编号
```

**实例**

```shell
使用指令"split"将文件"README"每6行切割成一个文件，输入如下命令：

$ split -6 README       #将README文件每六行分割成一个文件 

以上命令执行后，指令"split"会将原来的大文件"README"切割成多个以"x"开头的小文件。而在这些小文件中，每个文件都只有6行内容。

使用指令"ls"查看当前目录结构，如下所示：

#获得当前目录结构   README xaa xad xag xab xae xah xac xaf xai    

split -5 file1 test  #将文件file1以5行进行分割，分割后的文件以test前缀命名
```

