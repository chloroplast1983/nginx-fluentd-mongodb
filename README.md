# fluentd 解决方案

---

## 概述

这次考虑使用前端`nginx`输出日志, 然后异步到`fluentd`中来记录. `fluentd`每间隔一段时间写入到`mongodb`中. 这样避免`redis`链接数过高的问题

## 安装

### 下载镜像

* 使用`nginx`官方最新镜像.
* 使用`registry.cn-hangzhou.aliyuncs.com/marmot/fluentd`镜像
* 使用`registry.cn-hangzhou.aliyuncs.com/marmot-mongo/mongo-3.6`镜像(如果有必要)

### 配置文件

#### nginx

##### docker-compose.yml

```
nginx:
  image: "nginx"
  ports:
    - "需要映射的端口号:80"
  volumes:
   - ./config/nginx.conf:/etc/nginx/nginx.conf
   - ./html:/var/www/html/
  log_driver: "fluentd"
  log_opt:
    fluentd-address: "127.0.0.1:24224"
  container_name: nginx
```

##### nginx.conf

具体参数根据服务器配置调整.

```
user  nginx;
worker_processes  4;

#error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

worker_rlimit_nofile 65535;

events {
     use epoll;
    worker_connections  102400;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';


    log_format touch '$arg_key';
    access_log  /var/log/nginx/access.log  touch;
    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    #include /etc/nginx/conf.d/*.conf;

    server {
        listen       80;
        server_name  localhost;
	location / {
	    root   /var/www/html/;
            index  index.html index.htm;
        }
    }
}
```

##### 需要额外调整的参数

```
net.ipv4.tcp_syncookies = 0
```

#### fluentd

```
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

<match *>
    @type copy
    <store>
    @type mongo
    host mongo
    port 27017

    database log
    collection record

    user 用户名
    password 密码
    
    time_key time

    flush_interval 2s
    </store>
</match>
```

如果需要进行调试, 在配置文件添加, 输出日志:

```
<store>
@type stdout
</store>
```

## 服务器需求

### 配置

* 一台ecs服务器, 需要装载`nginx`.
	* `centos 7`
	* 系统盘: 100G
	* 数据盘: 100G
* 阿里云的`mongodb`存储, 先暂时买低配就好, 调试完成在看.

如果不买`mongodb`的话, 则额外给我一台服务器我自己搭建一个`mongodb`即可.

### 需要提供

* 服务器`ip`地址
* 服务器`root`账户
* 服务器`root`密码
* `mongodb`
	* 地址
	* 账户
	* 密码
