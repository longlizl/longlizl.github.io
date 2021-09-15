1. 安装邮件服务

   ```shell
   yum -y install postfix
   ```

2. 安装邮件发送工具mailx 在配置文件后面加入以下配置

   ```shell
   yum -y install mailx
   ```

   vim /etc/mail.rc

```shell
set  from=alarm@xxx.com.cn    			 # 发送者邮箱
set  smtp=smtp.exmail.qq.com 		 	 # 企鹅SMTP服务器
set  smtp-auth-user=alarm@xxx.com.cn 	  # 发送者邮箱
set  smtp-auth-password=xxxxx			 # 密码
smtp-auth=login						    # 登录
```

3. 编写告警监控脚本

   > 定时任务
   
   ```shell
   */1 * * * * /root/sh/flink_status.sh > /dev/null 2>&1 &
   ```
   
   > 邮件告警脚本
   >
   > 下面脚本中jq命令是将数据解析为json格式,定时任务为每分钟检测一次，且告警发送2次后不在发送

```shell
#!/bin/bash
#source /etc/profile
ip_addr=`ip addr | egrep 'ens33|eth0' | grep inet | awk '{print $2}' | awk -F/ '{print $1}'`
sent_date=`date +'%F %T'`
task_run_count=`curl -s  http://172.16.0.17:8081/jobs |jq | egrep -v 'jobs|{|}|]' | awk '!(NR%2)' | awk -F'"' '{print $(NF-1)}' | grep -i 'running' | wc -l`
# 定义初始化计数器为alter_count为0，新建alter_js文件将内容设置0
alter_count=`awk NF /root/sh/alter_js`
# 定义接收邮件地址
mail_re_addr='155078xxx@qq.com,qinzhengquan@xxx.com.cn,zhangzhiwei@xxx.com.cn'
alter_mail_sent () {
	 echo -e "公司:x\n时间:$sent_date\nIP地址为:$ip_addr的主机：" | mail -s "flink任务运行故障,请及时查看" $mail_re_addr > /dev/null 2>&1
}

if [ $task_run_count -lt 1 ];then
	alter_count=$[$alter_count+1]
	echo "$alter_count" > /root/sh/alter_js
	if [ $alter_count -le 2 ];then
		alter_mail_sent	
	fi
fi

```

   