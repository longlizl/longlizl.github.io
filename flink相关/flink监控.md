# flink监控

flink监控需要一个网关组件PushGateway然后Prometheus 从网关拉取数据

1. 下载安装PushGateway

   ```shell
   # 下载经安装
   curl -L -O  https://github.com/prometheus/pushgateway/releases/download/v1.4.1/pushgateway-1.4.1.linux-amd64.tar.gz
   tar -zxvf pushgateway-1.4.1.linux-amd64.tar.gz
   mv pushgateway-1.4.1.linux-amd64 /opt/pushgateway
   # 设置systemd启动
   vim /usr/lib/systemd/system/pushgateway.service
   [Unit]
   Description=https://prometheus.io
   [Service]
   Restart=on-failure
   ExecStart=/opt/pushgateway/pushgateway
   [Install]
   WantedBy=multi-user.target
   
   systemctl start pushgateway.service
   systemctl enable pushgateway.service
   ```

   1. flink配置文件加入如下监控参数

   ```shell
   metrics.reporter.promgateway.class: org.apache.flink.metrics.prometheus.PrometheusPushGatewayReporter
   # 这里写PushGateway的主机名与端口号
   metrics.reporter.promgateway.host: 172.16.2.29
   metrics.reporter.promgateway.port: 9091
   # Flink metric在前端展示的标签（前缀）与随机后缀
   metrics.reporter.promgateway.jobName: flink-metrics
   metrics.reporter.promgateway.randomJobNameSuffix: true
   metrics.reporter.promgateway.deleteOnShutdown: false
   metrics.reporter.promgateway.interval: 10 SECONDS
   ```

   

2. 配置prometheus

   prometheus和grafana安装省略

   ```yaml
   # 在scrape_configs:配置区域后门加入如下配置
     - job_name: 'PushGateway'
       static_configs:
       - targets: ['172.16.2.36:9091']
   ```

   ![image-20210830113348926](C:\Users\15507\AppData\Roaming\Typora\typora-user-images\image-20210830113348926.png)

3. 打开grafana配置flink模板

   官网下载导入flink相关模板

   

   ![image-20210830170006588](C:\Users\15507\AppData\Roaming\Typora\typora-user-images\image-20210830170006588.png)

下面显示监控数据

![image-20210830170111301](C:\Users\15507\AppData\Roaming\Typora\typora-user-images\image-20210830170111301.png)