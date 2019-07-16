## Prometheus监控MySQL
MySQL的监控比较简单，因为官方提供了exporter，只需要到数据库内新建一个用户授予几个权限即可。

### 授权
```
CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'exporter' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
```

### 配置exporter
```
# 下载
cd /tmp
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.12.0/mysqld_exporter-0.12.0.linux-amd64.tar.gz
tar xvf mysqld_exporter-0.12.0.linux-amd64.tar.gz -C /ops

# 编写systemd service 文件
vi /usr/lib/systemd/system/mysqld_exporter.service 

[Unit]
Description=mysqld_exporter
After=network.target
[Service]
Type=simple
Environment=DATA_SOURCE_NAME=exporter:exporter@(localhost:3306)/
ExecStart=/ops/mysqld_exporter-0.12.0.linux-amd64/mysqld_exporter --web.listen-address=:9104
Restart=on-failure
[Install]
WantedBy=multi-user.target

# 启动与自启动
systemctl daemon-reload
systemctl start mysqld_exporter
systemctl enable mysqld_exporter
```
默认端口为9104，还需防火墙放行：
```
firewall-cmd --add-port=9104/tcp --permanent
firewall-cmd --reload
```

### 配置Prometheus
```
vi prometheus.yaml

scrapy_confings:
···
  - job_name: MySQL
    static_configs:
    - targets:
      - ip:9104
      labels:
        instance: DB
```
重启Prometheus生效
```
# 如果你已注册了prometheus服务
systemctl restart prometheus
# 如果你是使用supervisor管理prometheus的话
supervisorctl restart prometheus
```