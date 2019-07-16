## Prometheus监控Redis
Redis监控与MySQL监控基本相同，而且更为简单。
目前的`redis_exporter`为第三方，并非官方提供。

### 查看Redis配置
首先确认redis有无密码:
```
cat /etc/redis.conf | grep -v '#' | grep requirepass
```
如果密码默默记下，等下需要用，如无密码则略过。

### 配置exporter
```
# 下载
cd /tmp 
wget https://github.com/oliver006/redis_exporter/releases/download/v1.0.3/redis_exporter-v1.0.3.linux-amd64.tar.gz
tar xvf redis_exporter-v1.0.3.linux-amd64.tar.gz
mv xvf redis_exporter-v1.0.3.linux-amd64 /redis_exporter

# 编写systemd service文件
vi /usr/lib/systemd/system/redis_exporter.service

[Unit]
Description=Redis exporter

[Service]
Type=simple
ExecStart=/redis_exporter/redis_exporter --redis.password=密码

[Install]
WantedBy=mutil-user.target

# 启动与自启动
systemctl daemon-reload
systemctl start redis_exporter
systemctl enable redis_exporter
```
默认端口为9121，还需防火墙放行：
```
firewall-cmd --add-port=9121/tcp --permanent
firewall-cmd --reload
```

### 配置Prometheus
```
vi prometheus.yaml

scrapy_confings:
···
  - job_name: Redis
    static_configs:
    - targets:
      - ip:9121
      labels:
        instance: Redis
```
重启Prometheus生效
```
# 如果你已注册了prometheus服务
systemctl restart prometheus
# 如果你是使用supervisor管理prometheus的话
supervisorctl restart prometheus
```