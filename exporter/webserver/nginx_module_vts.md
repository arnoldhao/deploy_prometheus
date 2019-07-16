## Prometheus监控Nginx

第三方模块`nginx_module_vts`最新版已经支持Prometheus metrics格式数据，重新编译Nginx加入模块即可。

### 编译安装
#### 1.准备编译环境
- 环境依赖：wget、gcc、automake、autoconf、libtool、make
- 编译依赖：pcre、pcre-devel、zlib、zlib-devel、openssl、openssl-devel、libxslt-devel、gd-devel、perl-devel、perl-ExtUtils-Embed、geoip-devel、gperftools



```
# 安装
yum install -y wget gcc automake autoconf libtool \
make pcre pcre-devel zlib zlib-devel openssl openssl-devel \
libxslt-devel gd-devel perl-devel perl-ExtUtils-Embed \
geoip-devel gperftools
```



#### 2.新增Nginx用户、用户组

useradd参数
- -M：不要自动简历用户的登入目录
- -g：指定用户所属的群组，也可以跟GID
- -s：指定用户登入后所使用的shell，默认为/bin/bash
```
groupadd nginx
useradd -M -g nginx -s /sbin/nologin nginx
```

#### 3.编译安装
下载Nginx源码
```
cd /tmp
wget https://nginx.org/download/nginx-1.17.1.tar.gz
tar xvf nginx-1.17.1.tar.gz
```
下载nginx-module-vts源码
```
mkdir /etc/nginx/3rd-modules && cd /etc/nginx/3rd-modules
git clone https://github.com/vozlt/nginx-module-vts
```
编译安装
```
# 进入目录
cd /tmp/nginx-1.17.1
# 配置Makefile

./configure \
--prefix=/usr/share/nginx \
--sbin-path=/usr/sbin/nginx \
--modules-path=/usr/lib64/nginx/modules \
--conf-path=/etc/nginx/nginx.conf \
--error-log-path=/var/log/nginx/error.log \
--http-log-path=/var/log/nginx/access.log \
--http-client-body-temp-path=/var/lib/nginx/tmp/client_body \
--http-proxy-temp-path=/var/lib/nginx/tmp/proxy \
--http-fastcgi-temp-path=/var/lib/nginx/tmp/fastcgi \
--http-uwsgi-temp-path=/var/lib/nginx/tmp/uwsgi \
--http-scgi-temp-path=/var/lib/nginx/tmp/scgi \
--pid-path=/run/nginx.pid \
--lock-path=/run/lock/subsys/nginx \
--user=nginx \
--group=nginx \
--with-file-aio \
--with-ipv6 \
--with-http_auth_request_module \
--with-http_ssl_module \
--with-http_v2_module \
--with-http_realip_module \
--with-http_addition_module \
--with-http_xslt_module=dynamic \
--with-http_image_filter_module=dynamic \
--with-http_geoip_module=dynamic \
--with-http_sub_module \
--with-http_dav_module \
--with-http_flv_module \
--with-http_mp4_module \
--with-http_gunzip_module \
--with-http_gzip_static_module \
--with-http_random_index_module \
--with-http_secure_link_module \
--with-http_degradation_module \
--with-http_slice_module \
--with-http_stub_status_module \
--with-http_perl_module=dynamic \
--with-mail=dynamic \
--with-mail_ssl_module \
--with-pcre \
--with-pcre-jit \
--with-stream \
--with-stream_ssl_module \
--with-google_perftools_module \
--with-debug \
--add-module=/etc/nginx/3rd-modules/nginx-module-vts
```
```
# make && make install
make && make install
```

#### 4.配置nginx.conf
需要做两点修改：
- 修改运行用户
- 添加配置文件目录

```
vim /etc/nginx/nginx.conf

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid
···
http {
  ···
  include /etc/nginx/conf.d/*.conf;
  access_log /var/log/nginx/access.log;
  ···
}
```
这时候可以来测试配置
```
nginx -t
```
但有可能会报错
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: [emerg] mkdir() "/var/lib/nginx/tmp/client_body" failed (2: No such file or directory)
nginx: configuration file /etc/nginx/nginx.conf test failed
```
我们手动来创建文件夹，因为有可能上一层文件夹也不存在，所以需要加`-p`参数
```
mkdir -p /var/lib/nginx/tmp/client_body
```
然后再来测试配置文件，配置通过
```
nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
在检查的时候也许会遇到这样的问题
```
[warn] the "ssl" directive is deprecated, use the "listen ... ssl" directive instead
```
这是因为在nginx 1.15版本之后更改了配置文件的规则，以前是这么写的：
```
server {
    listen 443;
    ···
    ssl on;
}
```
现在需要这么写：
```
server {
    listen 443 ssl;
}
```

#### 5.配置Nginx为Systemd服务
```
# 编辑文件
vi /usr/lib/systemd/system/nginx.service 

[Unit]
Description=nginx
After=network.target

[Service]
Type=forking
ExecStart=/usr/sbin/nginx
ExecReload=/usr/sbin/nginx -s reload
ExecStop=/usr/sbin/nginx -s stop

[Install]
WantedBy=mutil-user.target

# 启动与自启动
systemctl daemon-reload
systemctl start nginx && systemctl enable nginx
```

#### 6.配置nginx-module-vts
因为刚刚我们已经在nginx.conf配置文件中include了一个文件夹，而且是在http下，所以只需要建立如下配置文件即可。
实际在配置中，有时候会出现配置无法生效的情况，配置在`/etc/nginx/nginx.conf`即可。
```
vim /etc/nginx/conf.d/nginx-module-vts.conf 

vhost_traffic_status_zone;
vhost_traffic_status_filter_by_host on; //根据不同server name分开统计
server {
  location /status {
    vhost_traffic_status off; //取消本区域的统计
    vhost_traffic_status_display;
    vhost_traffic_status_display_format html;
  }
}
```
测试后重新载入配置即可。
```
nginx -t
nginx -s reload
```
访问`http://ip/status/format/prometheus`即可获得Prometheus数据。
如果防火墙没有放行，可能还需要执行如下程序：
```
firewall-cmd --add-service=http --permanent
firewall-cmd --reload
```

#### 7.更改Job metrics path
官方文档中，对于默认的metrics path说明可以重写：[Configuration \| Prometheus](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
所以只需要在Prometheus的yaml配置文件中，新增参数修改默认配置即可。
而且默认为80端口也可省略。
```
vi prometheus.yaml

···
scrapy_configs:
 ···
 - job_name: Nginx
   metrics_path: /status/format/prometheus
   static_configs:
   - targets: ['ip']
```
因为我们采用80端口采集数据，所以防火墙不需新建端口，直接重启Prometheus使配置生效。
```
systemctl restart prometheus
```