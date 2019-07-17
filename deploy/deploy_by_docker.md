## Docker部署Prometheus

Docker本身是用后即焚的，如果需要持久化保存数据，需要映射到本地驱动。

Prometheus需要持久化存储的数据有2种：

- 配置文件：prometheus.yml文件
- 存储数据文件夹： data文件夹

如果你的Prometheus采用了第三方数据库存储，那么存储数据文件夹也无需做持久化存储，仅需映射配置文件即可，这里我们默认采用映射两种文件的方法。



### 安装Docker

安装Docker需要满足以下两个条件

- CentOS 7 64位及以后
- Kernel：3.10及以后



```
# 卸载旧版本
yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-selinux docker-engine-selinux docker-engine

# 安装依赖
yum install -y yum-utils device-mapper-persistent-data lvm2

# 添加Repo
yum-config-manager --add-repo https://mirrors.ustc.edu.cn/docker-ce/linux/centos/docker-ce.repo

# 更新并安装
yum makecache
yum install -y docker-ce

# 启动&自启动
systemctl start docker
systemctl enable docker

# 配置docker镜像加速
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io
systemctl restart docker
```



### 部署Prometheus

首先规划好目录：

- Prometheus.yml —> /ops/prometheus-data/prometheus.yml
- Data/ —> /ops/prometheus-data/



#### Prometheus.yml

预先建立好文件，监控本机数据：

```
# 建立文件
mkdir -p /ops/prometheus-data && cd /ops/prometheus-data
vi /ops/prometheus-data/prometheus.yml

# 添加如下内容后保存
global:
  scrape_interval: 15s
  external_labels:
    monitor: 'codelab-monitor'

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
```



#### Data

因为存储文件夹需要进行文件的创建读写，需要授权不同的权限，新版的Prometheus Docker Image中，将默认user设置为`nobody`，需要预先取出nobody的UID与GID，然后在本地预先授权文件夹权限。

```
# 创建文件夹
mkdir -p /ops/promerheus-data

# 取得nobody的uid
docker run --rm quay.io/prometheus/busybox cat /etc/passwd
···
nobody:x:65534:65534:nobody:/home:/bin/false

# 配置文件夹权限
chown 65534:65534 -R /ops/prometheus-data
```



#### 启动Prometheus

映射配置与存储文件夹，并映射至本机9090端口。

```
# 启动
docker run --name prometheus -d \
-p 9090:9090 \
--restart=always \
-v /ops/prometheus-data:/prometheus-data \
quay.io/prometheus/prometheus \
--config.file=/prometheus-data/prometheus.yml \
--storage.tsdb.path=/prometheus-data/

# 如果配置文件有改动，重启Prometheus容器
docker ps | grep prometheus | awk '{print $1}' | xargs docker restart
```

