# Dockerfile语法简介

```
Dockerfile是由一系列命令和参数构成的脚本，一个Dockerfile里面包含了构建整个image的完整命令。Docker通过docker build执行Dockerfile中的一系列命令自动构建image。

# :代表注释
```

## 1. FROM

```
Syntax:

FROM <image>[:<tag> | @<digest>] [AS <name>]

- FROM指定一个基础镜像，且必须为Dockerfile文件开篇的每个非注释行,至于image则可以是任何合理存在的image镜像
- FROM可以在一个Dockerfile中出现多次，以便于创建混合的images。如果没有指定tag,latest将会被指定为要使用的基础镜像版本。
- AS name,可以给新的构建阶段赋予名称。该名称可用于后续FROM 和 COPY --from=<name | index>说明可以引用此阶段中构建的镜像
```

 

## 2. LABEL

```shell
为镜像生成元数据标签信息

Syntax:

LABEL <KEY>=<VALUE> \ <KEY>="XXXX"

多个标签写成一行，避免在镜像中额外增加layer

```

## 3. MAINTAINER

```shell
作者信息,写在FROM后

Syntax:

MAINTAINER "auth <email>"
```

## 4. COPY

当复制一个目录时，并不会复制目录本身，而是会递归复制其下子目录 至目标目录下

```shell
Syntax:

COPY data /data/

文件复制准则

- <src>必须是build上下言文中的路径，不能是其父目录中的文件
- 如果<src>是目录，则其内部文件或子目录会被递归复制，但<src>目录自身不会被复制
- 如果指定了多个<src>,或在<src>中使用了通配符，则<dest>必须是一个目录，且必须以/结尾
- 如果<dest>事先不存在，它将会被自动创建，这包括其父目录路径。
```

 

## 5. ADD

```shell
ADD指令类似于COPY指令，ADD支持使用TAR文件和URL路径

Syntax:

ADD <src>...<dest> ADD ["<src>",..."<dest>"]

操作准则

- 如果<src>为URL且<dest>不以/结尾，则<src>指定的文件将被下载并直接被创建为<dest>;如果<dest>以/结尾，则文件名URL指定的文件将被直接下载并保存为<dest>/<filename>
- 如果<src>是一个本地文件系统上的压缩格式的tar文件，它将被展开为一个目录，其行为类似于"tar -x"命令;然而，通过URL获取到的tar文件将不会自动展开。
- 如果<src>有多个，或其间接或直接使用了通配符，则<dest>必须是一个以/结尾的目录路径;如果<dest>不以/结尾，则其被视作一个普通文件，<src>内容将被直接写入到<dest>
- 为了让镜像尽量小，最好不要使用 ADD 指令从远程 URL 获取包，而是使用 curl 和 wget。这样你可以在文件提取完之后删掉不再需要的文件来避免在镜像中额外添加一层。

```

## 6. WORKDIR

```shell
用于为Dockerfile中所有RUN、CMD、ENTRYPOINT、COPY和ADD指令设定工作目录

Syntax:

WORKDIR <dirpath>

语法格式:
WORKDIR /var/log 
WORKDIR $STATEPATH
```

## 7. RUN

```shell
接受命令作为参数并用于创建镜像,在之前的commit层上形成新的层。

Syntax:

RUN <command>(如同执行shell命令 /bin/sh -c) RUN ["executable","param1","param2"]

- RUN 指令将在当前image中执行任意合法命令并提交执行结果。命令执行提交后，就会自动执行Dockerfile中的下一个指令。
- 分层RUN指令和生成提交符合Docker的核心概念，其中提交很轻量，可以从image将用于Dockerfile中的下一步。
- exec形式使得可以避免shell字符串变化，以及使用不包含指定的shell可执行文件的基本image来运行RUN命令。
- 在shell形式中，可以使用\（反斜杠）将单个RUN指令继续到下一行。例如:

RUN yum install -y  openssl  pcre-devel  zlib

- 第二种语法格式中的参数是一个JSON格式的数组，其中<executable>为要运行的命令，后面的<paramN>为传递给命令的选项或对数;然而，此种格式指定的命令不会以"/bin/sh -c"来发起，因此常见的shell操作如变量替换以及通配符(?,*等)替换将不会进行;不过，如果要运行的命令依赖于此shell特性的话，可以将其替换为类似下面的格式。

RUN ["/bin/bash","-c","<executable>","<param1>"]

RUN 指令的缓存在下一次构建期间不会自动失效。用于诸如：yum repolist 之类的指令的缓存将在下一次构建期间被重用。可以通过--no-cache 参数来使RUN指令的缓存无效，例如： docker build --no-cache

管理命令

某些RUN 命令依赖于使用管道字符( | )将管道输出到另一个命令功能

RUN wget -O - http://www.baidu.com/index.html | wc -l > /app/html/baidu.html

Docker使用 /bin/sh -c 解释执行这些命令，解释器只评估管道中最后一个操作的退出代码以确定成功。在上面的例子中，只要wc -l 命令成功，即使wget 命令失败，该构建步骤也会成功并生成新的镜像。

由于管道中任何阶段的错误而导致命令失败，请预先 set -o pipefail && 确保意外错误可防止构建无意中成功。例如：

set -o pipefail : 表示在管道连接的命令序列中，只要有任何一个命令返回非0值，则整个管道返回非0值，即使最后一个命令返回0.

RUN set -o pipefail && wget -O - http://www.baidu.com/index.html | wc -l > /app/html/baidu.html

注意：

并非所有的shell都支持 -o pipefail 选项。在这种情况下(例如 dash shell，这是基于Debian的映像上的默认shell），请考虑使用exec形式RUN来明确选择一个支持该pipefail选项的shell。如：

RUN ["/bin/bash","-c","set -o pipefail && wget -O - http://www.baidu.com/index.html | wc -l > /app/html/baidu.html"]
```

## 8. CMD

```shell
类似于RUN指令，CMD指令也可用于运行任何命令或应用程序，不过，二者的运行时间点不同

- RUN 指令运行于映像文件构建过程中，而CMD指令运行于基于Dockerfile构建出的新镜像文件启动一个容器时。
- CMD指令的首要目的在于为启动的容器指定默认要运行的程序，且其运行结束后，容器也将终止;不过，CMD指定的命令其可以被docker run的命令行选项所覆盖
- 在Dockerfile中可以存在多个CMD指令，但仅最后一个生效

Syntax:

CMD <command> //支持命令展开，但是不支持传递信号 
CMD ["<executable>","<param1>","<param2>"] //相当于容器的第一个命令，可以接受信号 
CMD ["param1","param2"] 前两种语法格式的意义同RUN 第三种则用于为ENTRYPOINT指令提供默认参数

CMD会在启动容器的时候执行，build时不执行，而RUN只是在构建镜像的时候执行，后续镜像构建完成之后，启动容器就与RUN无关了。这个命令就相当于在/etc/rc.d/rc.local中写命令 
```

## 9. ENTRYPOINT

```shell
类似CMD指令的功能，用于为容器指定默认运行程序，从而使得容器像是一具单独的可执行程序

与CMD不同的是，由ENTRYPOINT启动的程序不会被docker run命令行指定的参数所覆盖，而且，这些命令行参数会被当作参数传递给ENTRYPOINT指定的程序。不过，docker run 命令的--entrypoint 选项的参数可覆盖ENTRYPOINT指令指定的程序

Syntax：

ENTRYPOINT <command> //这种方式能接受shell命令行展开 
ENTRYPOINT ["<executable>","param1"] //展开不了，但能接收到信号

docker run命令传入的命令参数会覆盖CMD指令的内容并且附加到ENTRYPOINT命令最后做为其参数使用。Dockerfile文件中也可以存在多个ENTRYPOINT指令，但仅有最后一个会生效
 
```

## 10. EXPOSE

```shell
用来指定端口，使容器内的应用可以通过端口和外界交互。

Syntax:

EXPOSE <port> [<port>...]

告诉Docker服务端容器对外映射的本地端口，需要在docker run 的时候使用-p 或者 -P 选项生效。

# 此端口只是申明容器里应用端口为80，如果你容器应用端口为8080 使用-p 80:80是无法访问的
EXPOSE 80/tcp
```shell

 

## 11. ENV

```shell
ENV指令可以用于docker容器设置环境变量

Syntax:

ENV <key> <value> ENV <key>=<value> ...

指定一个环境变量，会被后续RUN指令使用，并在容器运行时保留。

ENV设置的环境变量，可以使用 docker inspect 命令来查看。同时还可以使用 docker run --env <key>=<value>来修改环境变量
```shell

 

## 12. USER

```shell
用于指定运行image时的或运行Dockerfile中任何RUN、CMD或ENTRYPOINT指令指定的程序时的用户名或UID

默认情况下，container的运行身份为root用户
Syntax:

USER <UID>|<UserName>

需要注意的是,<UID>可以为任意数字，但实践中其必须为/etc/passwd中某用户的有效UID，否则,docker run命令将运行失败
```

 

## 13. ONBUILD

```shell
用于在Dockerfile中定义一个触发器

Dockerfile用于build映像文件，此映像文件亦可作为base image被另一个Dockerfile用作FROM指令的参数，并以之构建新的映像文件

在后的这个Dockerfile中的FROM指令在build过程中被执行时，将会“触发”创建其base image的Dockerfile文件中的ONBUILD指令定义的触发器

Syntax:

ONBUILD <INSTRUCTION>

注意：

尽管任何指令都可注册成为触发器指令，但ONBUILD不能自我嵌套，且不会触发FROM和MAINTAINER指令

使用包含ONBUILD指令的Dockerfile构建的镜像应该使用特殊的标签，例如ruby:2.0-onbuild

在ONBUILD指令中使用ADD或COPY指令应该格外小心，因为新构建过程和上下文在缺少指定的源文件时会失败。
```

 

## 14. HEALTHCHECK

```shell
Docker 1.12版本后引入的判断容器状态是否正常

Syntax:
# 设置检查容器健康状况的命令
HEALTHCHECK [OPTION] CMD <command>  
# 如果基础镜像有健康检查指令，使用这行可屏蔽掉其健康检查指令
HEALTHCHECK NONE 

在没HEALTHCHECK指令前，Docker只能通过容器内主进程是否退出来判断容器是否状态异常。很多情况下这没问题，但是如果程序进入死锁状态，或者死循环状态，应用进程并不退出，但是该容器已经无法提供服务了。虽然后端的程序可以通过前端的检测工具来检查状态信息。但是最前端的服务就需要本身的检测机制加上监控，就可以做到出现问题解决问题。

当在一个镜像指定了 HEALTHCHECK 指令后，用其启动容器，初始状态会为 starting，在 HEALTHCHECK 指令检查成功后变为 healthy，如果连续一定次数失败，则会变为 unhealthy。

HEALTHCHECK支持下列选项：

- --interval=<间隔> ： 两次健康检查间隔，默认30秒
- --timeout=<时长> : 健康检查命令运行超时时间，如果超过这个时间，本次健康检查就被视为失败，默认为30秒
- --retries=<次数> ：当连续失败指定次数后，则将容器状态视为unhealthy,默认3次。

和CMD、ENTRYPOINT一样，HEALTHCHECK只可以出现一次，如果写了多个，只有最后一个生效。CMD 后面的命令也分为shell和exec格式。命令的返回值决定了该次检查的成功与否： 0表示成功;1表示失败;2保留。

ex:
HEALTHCHECK --interval=5s --timeout=3s \ 
CMD curl -fs http://127.0.0.1/ || exit 1
```

## nginx镜像构建例子：

```shell
1.下载源代码软件包

mkdir /root/nginx_demo/

nginx-1.6.0.tar.gz 

2.编辑dockerfile文件

vim  /root/nginx/Dockerfile

FROM centos MAINTAINER li 
RUN yum -y install gcc* make pcre-devel zlib-devel 
ADD nginx-1.6.0.tar.gz /usr/src/ WORKDIR /usr/src/nginx-1.6.0/ 
RUN useradd -s /sbin/nologin -M nginx 
RUN ./configure --prefix=/usr/local/nginx --user=nginx --group=nginx --with-http_stub_status_module && make && make install 
RUN ln -s /usr/local/nginx/sbin/* /usr/local/sbin/ 
EXPOSE 80 
WORKDIR / 
RUN nginx CMD ["nginx", "-g", "daemon off;"]

3.执行dockerfile文件

docker build -t centos-nginx:test ./

4.创建nginx的容器,并做端口映射

docker run -d --name nginx1 -p 80:80 9b9494bee7cc
```

