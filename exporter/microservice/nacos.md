## Prometheus监控Nacos
Nacos是阿里巴巴出品的微服务注册中心，在0.8.0版本完善了对Prometheus的支持，原生支持通过`8848`端口暴露metrics数据，默认路径为`ip:8848/nacos/actuator/prometheus`。

### 集群
Nacos集群比较简单，在`path/conf/cluster.conf`配置即可，与hosts概念很像，所以对于同一集群内的Nacos服务器，每一台均需开启Prometheus metrics。

### 配置文件
取消每一台Nacos配置文件中暴露Prometheus metrics的注释：
```
vi path/conf/application.properties

management.endpoints.web.exposure.include=*
```
然后重启Nacos服务，一般为如下操作：
```
sh path/bin/shutdown.sh
sh path/bin/startup.sh
```
确认防火墙已放行8848端口后，访问`ip:8848/nacos/actuator/prometheus`确认有metrics数据输出。

### 配置prometheus.yml

```
scrape_configs:
- job_name: Nacos
  metrics_path: /nacos/actuator/prometheus
  static_configs:
  - targets:
    - ip1:8848
    labels:
      instance: Nacos_node1
  - targets:
    - ip2:8848
    labels:
      instance: Nacos_node2
  - targets:
    - ip:8848
    labels:
      instance: Nacos_node3
```
重启Prometheus即可采集Nacos集群数据。