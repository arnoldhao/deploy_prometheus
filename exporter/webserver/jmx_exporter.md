## Tomcat数据采集

**JMX**（英语：Java Management Extensions，即Java管理扩展）是Java平台上为应用程序、设备、系统等植入管理功能的框架。

JMX_exporter搭配不同的配置文件可提供不同系统的metrics数据。



### 下载

本次我们监控tomcat，所以只需要准备`jmx_exporter`与`tomcat.yml`即可。

```
# 创建目录
mkdir -p /ops/jmx_exporter && cd /ops/jmx_exporter

# 下载jmx_exporter
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.12.0/jmx_prometheus_javaagent-0.12.0.jar

# 下载tomcat.yml
wget https://raw.githubusercontent.com/prometheus/jmx_exporter/master/example_configs/tomcat.yml

# 目录结构
tree /ops
/ops
└── jmx_exporter
    ├── jmx_prometheus_javaagent-0.12.0.jar
    └── tomcat.yml

1 directory, 2 files
```



### 启动

JMX_exporter支持jar包启动直接添加`javaagent`，但是这里监控的是tomcat，一般是修改`path/bin/catalina.sh`添加`JAVA_OPTS`，让jar包跟随tomcat启动。

根据tomcat启动顺序，不建议直接修改`catalina.sh`，可以新建`path/bin/setenv.sh`写入内容，启动tomcat会直接加载，这里我们监控tomcat使用38080端口，如果有多个tomcat实例，请注意每一个实例需要有不同的监控端口。

```
# 新建添加内容，path为你tomcat目录
vi path/bin/setenv.sh

export JAVA_OPTS="-javaagent:/ops/jmx_exporter/jmx_prometheus_javaagent-0.12.0.jar=38080:/ops/jmx_exporter/tomcat.yml"

# 重新启动tomcat
cd path/bin
sh shutdown.sh
sh startup.sh

# 防火墙放行
firewall-cmd --add-port=38080/tcp --permanent
firewall-cmd --reload
```

这时候访问`ip:38080`即可查看metrics数据，后面接任何路径不影响数据展示，如`ip:38080/metrics`，所以在Pormetheus配置文件中，无需修改`metrics_path`默认路径。



### 添加Prometheus配置

加入需求监控的主机，然后重启Prometheus或者热加载配置。

```
# 修改加入内容
vi prometheus.yml

-scrape_configs:
  - job_name: JMX
    static_configs:
    - targets:
      - ip:38080
      labels:
        instance: Tomcat_name
        
# 重启Prometheus
# 如果你是systemd管理
systemctl restart prometheus
如果你是supervisor管理
supervisorctl restart prometheus
```



