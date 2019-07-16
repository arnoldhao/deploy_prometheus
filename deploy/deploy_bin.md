## 二进制部署Prometheus、Node_export、Alertmanager
Prometheus一系列软件是基于Golang编写、编译的软件包，Golang的性质就是没有任何其他依赖，直接运行即可。在Prometheus官网可以直接下载解压后运行。
对于普通二进制程序，可以使用`nohup`、`supervisor`等工具实现后台运行，开机自启动等，但我们部署的服务器为CentOS7，使用全新的`systemd`进行系统管理，对于编写服务文件非常简单，所以直接将Prometheus一系列软件注册为服务，可以非常方面地使用`systemctl`进行开关与自启动管理。

---
### 下载解压
为了便于管理，将新建根目录文件夹`/ops`并将所有文件放入其中。
下载文件后放入目，目前最新版本：
- 截止日期：2019年07月16日
```
# 新建文件夹
mkdir /ops

# Prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.11.1/prometheus-2.11.1.linux-amd64.tar.gz
tar xvf prometheus-2.11.1.linux-amd64.tar.gz -C /ops

# node_exporter
wget https://github.com/prometheus/node_exporter/releases/download/v0.18.1/node_exporter-0.18.1.linux-amd64.tar.gz
tar xvf node_exporter-0.18.1.linux-amd64.tar.gz -C /ops

# Alertmanager
wget https://github.com/prometheus/alertmanager/releases/download/v0.18.0/alertmanager-0.18.0.linux-amd64.tar.gz
tar xvf alertmanager-0.18.0.linux-amd64.tar.gz -C /ops

# 目录
tree /ops

/ops
├── alertmanager-0.18.0.linux-amd64
│   ├── alertmanager
│   ├── alertmanager.yml
│   ├── amtool
│   ├── LICENSE
│   └── NOTICE
├── node_exporter-0.18.1.linux-amd64
│   ├── LICENSE
│   ├── node_exporter
│   └── NOTICE
└── prometheus-2.11.1.linux-amd64
    ├── console_libraries
    │   ├── menu.lib
    │   └── prom.lib
    ├── consoles
    │   ├── index.html.example
    │   ├── node-cpu.html
    │   ├── node-disk.html
    │   ├── node.html
    │   ├── node-overview.html
    │   ├── prometheus.html
    │   └── prometheus-overview.html
    ├── LICENSE
    ├── NOTICE
    ├── prometheus
    ├── prometheus.yml
    └── promtool

5 directories, 22 files
```

### 配置服务
Prometheus启动会自动加载当前文件夹下的`prometheus.yml`文件，且会将数据存储与当前目录下的`data/`
文件夹下，所以需要提前建立文件夹，Alertmanager与Prometheus类似
```
# 建立文件夹
mkdir /ops/prometheus-2.11.1.linux-amd64/data
mkdir /ops/alertmanager-0.17.0.linux-amd64/data
```
Node_exporter启动后默认使用`9100`端口提供metrics信息，没有配置文件。
所以只需要分别新建`prometheus.service`、`node_exporter.service`、`alertmanager.service`放入`/usr/lib/systemd/system/`即可。
#### 配置Prometheus服务
`/usr/lib/systemd/system/prometheus.service`
```
[Unit]
Description=Prometheus

[Service]
Type=simple
ExecStart=/ops/prometheus-2.10.0.linux-amd64/prometheus --config.file="prometheus.yml" --web.listen-address="0.0.0.0:9090" --storage.tsdb.path="data/" 

[Install]
WantedBy=mutil-user.target
```

#### 配置Node_exporter服务
`/usr/lib/systemd/system/node_exporter.service`
```
[Unit]
Description=Node Exporter For Prometheus

[Service]
Type=simple
ExecStart=/ops/node_exporter-0.18.1.linux-amd64/node_exporter

[Install]
WantedBy=mutil-user.target
```

#### 配置Alertmanager服务
`/usr/lib/systemd/system/alertmanager.service`
```
[Unit]
Description=Alertmanager

[Service]
Type=simple
ExecStart=/ops/alertmanager-0.17.0.linux-amd64/alertmanager --config.file="alertmanager.yml" --web.listen-address="0.0.0.0:9093" --storage.path="data/" 

[Install]
WantedBy=mutil-user.target
```

配置完成后需要reload重新载入方可使用。
```
# reload
systemctl reload

# 启动与自启动，node_exporter与alertmanager相同
systemctl start prometheus
systemctl enable prometheus
```

#### 配置防火墙
如果想外部访问，还需配置防火墙放行：
- Prometheus 9090
- Node_exporter 9100
- Alertmanager 9093

CentOS7中使用`firewall-cmd`替换了`iptables`，使用上更为简单

```
# 新增放行端口
firewall-cmd --add-port=9090/tcp --permanent
firewall-cmd --add-port=9100/tcp --permanent
firewall-cmd --add-port=9093/tcp --permanent

# 重新载入配置生效
firewall-cmd --reload
```
现在可以使用`ip:9090`访问Prometheus，`ip:9093`访问Alertmanager，`ip:9100`查看本服务器metrics结构信息。

---
### 使用Supervisor管理服务
除了`systemd`，你还可以使用第三方程序`supervisor`管理服务，如果你是CentOS7之前使用`sysVinit`，设置服务过于复杂，使用`supervisor`更为方便。

Supervisor基础操作：
- reload: supervisorctl update
- start: supervisorctl start xxx
- stop: supervisorctl stop xxx
- restart: supervisorctl restart xxx

#### Supervisor管理Prometheus服务
`/etc/supervisord.d/prometheus.ini`
```
[program:prometheus]
directery=/ops/prometheus-2.10.0.linux-amd64
command=/ops/prometheus-2.10.0.linux-amd64/prometheus --config.file="prometheus.yml" --web.listen-address="0.0.0.0:9090" --storage.tsdb.path="data/" 
user=root
autostart=true
autorestart=true
startsecs=5
startretries=5
redirect_stderr=true
stdout_logfile_maxbytes=20MB
stdout_logfile=/var/log/supervisor/out_prometheus.log
```

#### Supervisor管理node_exporter
`/etc/supervisord.d/node_exporter.ini`
```
[program:node_exporter]
directery=/ops/node_exporter-0.18.1.linux-amd64
command=/ops/node_exporter-0.18.1.linux-amd64/node_exporter
user=root
autostart=true
autorestart=true
startsecs=5
startretries=5
redirect_stderr=true
stdout_logfile_maxbytes=20MB
stdout_logfile=/var/log/supervisor/out_node_exporter.log
```

#### Supervisor管理Alertmanager
`/etc/supervisord.d/alertmanager.ini`
```
[program:altermanager]
directery=/ops/alertmanager-0.17.0.linux-amd64
command=/ops/alertmanager-0.17.0.linux-amd64/alertmanager --config.file="alertmanager.yml" --web.listen-address="0.0.0.0:9093" --storage.path="data/" 
user=root
autostart=true
autorestart=true
startsecs=5
startretries=5
redirect_stderr=true
stdout_logfile_maxbytes=20MB
stdout_logfile=/var/log/supervisor/out_alertmanager.log
```