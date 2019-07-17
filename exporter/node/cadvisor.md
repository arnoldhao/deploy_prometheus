## cAdvisor

cAdviosr是Google开源的用于Docker监控的镜像，启动时挂载本地几个特定目录，并暴露端口即可。



### 安装Docker

既然要监控Docker，那么Docker肯定已经安装好了。



### 启动cAdvisor

这里随机指定本地端口为48080

```
# 启动cadvisor容器
docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:rw \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --publish=48080:8080 \
  --detach=true \
  --name=cadvisor \
  --restart=always \
  google/cadvisor:latest
  
# 防火墙放行
firewall-cmd --add-port=48080/tcp --permanent
firewall-cmd --reload
```

然后访问`ip:48080/metrics`即可查看metrics数据。



### 添加Prometheus配置

加入需求监控的主机，然后重启Prometheus或者热加载配置。

```
# 修改加入内容
vi prometheus.yml

scrape_configs:
  - job_name: docker
    static_configs:
    - targets: 
      - ip:48080
      labels:
        instance: docker_name
        
# 重启Prometheus
# 如果你是systemd管理
systemctl restart prometheus
如果你是supervisor管理
supervisorctl restart prometheus 
```

