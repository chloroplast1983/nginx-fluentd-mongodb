# fluentd 解决方案

---

## 概述

这次考虑使用前端`nginx`输出日志, 然后异步到`fluentd`中来记录. `fluentd`每1秒写入到`redis`中. 这样避免`redis`链接数过高的问题

## 安装

### 下载镜像

这里的镜像先使用前端`php`的镜像, 然后测试的时候修改配置文件. 使用`docker-compose`编排.