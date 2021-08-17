# clickhouse监控部署

## 1. 各节点安装node_exporter，并使用systemd管理

## 1.1 官网下载node_exporter

```shell
wget https://github.com/prometheus/node_exporter/releases/download/v1.2.2/node_exporter-1.2.2.linux-amd64.tar.gz
```

## 1.2 安装node_exporter

```shell
# 解压安装
tar -zxvf node_exporter-1.2.2.linux-amd64.tar.gz && mv node_exporter-1.2.2.linux-amd64 /opt/node_exporter
# 设置systemd管理
vim /usr/lib/systemd/system/node_exporter.service
[Unit]
Description=https://prometheus.io
[Service]
Restart=on-failure
ExecStart=/opt/node_exporter/node_exporter
[Install]
WantedBy=multi-user.target
# 启动服务
systemctl start node_exporter
# 查看服务状态及端口
systemctl status node_exporter
```

![image-20210816135622782](https://longlizl.github.io/clickhouse/images/12.png)

​	其余节点按照上述步骤安装

## 2. clickhouse配置文件打开服务内置监控参数(所有节点打开)

```xml
# 将下面一段注释取消
vim /etc/clickhouse-server/config.xml
    <prometheus>
        <endpoint>/metrics</endpoint>
        <port>9363</port>
        <metrics>true</metrics>
        <events>true</events>
        <asynchronous_metrics>true</asynchronous_metrics>
        <status_info>true</status_info>
    </prometheus>
```

​	重启clickhouse

```shell
systemctl restart clickhouse-server
```

## 3. 监控端安装[Prometheus](https://prometheus.io/)和[Grafana](https://grafana.com/grafana/)

### 3.1 安装并配置Prometheus

```shell
wget https://github.com/prometheus/prometheus/releases/download/v2.29.1/prometheus-2.29.1.linux-amd64.tar.gz
mv prometheus-2.29.1.linux-amd64 /opt/prometheus
# 配置systemd管理
vim /usr/lib/systemd/system/prometheus.service
[Unit]
Description=https://prometheus.io
[Service]    
Restart=on-failure
# 启动路径
ExecStart=/opt/prometheus/prometheus --config.file=/opt/prometheus/prometheus.yml  
[Install]
WantedBy=multi-user.target 
```

### 3.2 配置监控

```yaml
# 打开配置文件下面加入如下配置：
vim /opt/prometheus/prometheus.yml
scrape_configs:
  - job_name: "hosts-groups"
    static_configs:
      - targets:
        - "192.168.205.190:9100"
        - "192.168.205.191:9100"
        - "192.168.205.192:9100"
        - "192.168.205.193:9100"
        - "192.168.205.194:9100"
        - "192.168.205.195:9100"

  - job_name: "clickhouse-cluster"
    static_configs:
      - targets:
        - "192.168.205.190:9363"
        - "192.168.205.191:9363"
        - "192.168.205.192:9363"
        - "192.168.205.193:9363"
        - "192.168.205.194:9363"
        - "192.168.205.195:9363"
```

### 3.3 启动服务

```shell
systemctl start prometheus
```



### 3.2 安装配置Grafana

```shell
wget https://dl.grafana.com/oss/release/grafana-8.1.1-1.x86_64.rpm
yum localinstall grafana-8.1.1-1.x86_64.rpm
systemctl start grafana-server
```

​	查看服务端口：

```shell
[root@ckh-06 ~]# ss -ntupl | grep grafana
tcp    LISTEN     0      128      :::3000                 :::*                   users:(("grafana-server",pid=20142,fd=9))
```

## 4. 配置grafana可视化监控界面

### 4.1  访问grafana

​	浏览器打开http://192.168.205.195:3000（默认用户名密码均为admin）,登录后提示修改密码：

![image-20210816175236905](https://longlizl.github.io/clickhouse/images/13.png)

### 4.2 配置数据源

​	![image-20210816175500628](https://longlizl.github.io/clickhouse/images/14.png)

![image-20210816175702453](https://longlizl.github.io/clickhouse/images/15.png)

#### 4.2.1  从grafana官网下载node_expoter相关监控模板导入

![image-20210816180421084](https://longlizl.github.io/clickhouse/images/16.png)

![image-20210816180205949](https://longlizl.github.io/clickhouse/images/17.png)

#### 4.2.1  从grafana官网下载ClickHouse internal exporter metrics相关监控模板导入

![image-20210816180906562](https://longlizl.github.io/clickhouse/images/18.png)

