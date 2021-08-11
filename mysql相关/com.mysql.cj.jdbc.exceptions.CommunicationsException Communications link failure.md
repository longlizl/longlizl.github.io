**com.mysql.cj.jdbc.exceptions.CommunicationsException: Communications link failure（真实有效）**

**摘要：** 数据库连接失败 在数据库连接失败，经常会有蛮多一系列的问题导致的原因，这个时候一定要多去尝试一下各种方法，并且做好自己的梳理！ 一、例如我在SpringBoot项目中使用了阿里的数据库连接池Driud。

数据库连接失败

在数据库连接失败，经常会有蛮多一系列的问题导致的原因，这个时候一定要多去尝试一下各种方法，并且做好自己的梳理！

一、例如我在SpringBoot项目中使用了阿里的数据库连接池Driud。

有次在启动的时候，会报这样的错：

Caused by: com.mysql.cj.exceptions.CJCommunicationsException: Communications link failure The last packet successfully received from the server was 319 milliseconds ago.  The last packet sent successfully to the server was 319 milliseconds ago.

就是数据库连接失败的问题。

二、定位问题

为什么会出现这样的一个问题呢？ 

出现这样的一个问题，首先确定是不是数据库问题，看看数据库能不能连上。 

如果你的同事或者其他人都能够连上，那么数据库就没有问题。 

看看你能不能上网。 

如果你能上网，你的网络还OK。

三、代理问题

如果你使用了代理，就是哪种能帮助你上谷歌的软件。 

你将它关掉，看看问题是否解决了。

四、增加一个配置

\#下面这两个配置，可以在每次连接的时候判断一些连接是否有效 spring.datasource.druid.test-on-borrow=true spring.datasource.druid.test-while-idle=true

OK，经过以上的一系列操作，问题基本上就能够解决了！