---
title: nginx搭建rtmp直播流
tags: [多媒体]
categories: 多媒体
toc: true
---
使用Nginx和nginx-rtmp-module搭建简单的rtmp直播流服务器。
<!--more-->
# nginx rtmp安装
```
#!/bin/bash
sudo apt-get install build-essential libpcre3 libpcre3-dev libssl-dev unzip
mkdir ~/working
cd ~/working
wget http://nginx.org/download/nginx-1.7.5.tar.gz
wget https://github.com/arut/nginx-rtmp-module/archive/master.zip
tar -zxvf nginx-1.7.5.tar.gz
unzip master.zip
cd nginx-1.7.5
./configure --with-http_ssl_module --add-module=../nginx-rtmp-module-master
make -j2
sudo make install
sudo wget https://raw.github.com/JasonGiedymin/nginx-init-ubuntu/master/nginx -O /etc/init.d/nginx
sudo chmod +x /etc/init.d/nginx
sudo update-rc.d nginx defaults
sudo service nginx start
#sudo service nginx stop
```
# 配置nginx
```
sudo nano /usr/local/nginx/conf/nginx.conf
```
附加上如下内容
```
rtmp {
    server {
            listen 1935;
            chunk_size 4096;

            application live {
                    live on;
                    record off;
            }
    }
```
保存,然后重启服务
```
sudo service nginx restart
```
## 安全考虑
如果有配置防火墙，则需要确保TCP 1935端口是授予权限的.

上面的配置允许所有人查看直播流，可以通过在各个路由中加入如下语句进行ip的过滤
```
allow publish 127.0.0.1;
allow publish 0.0.0.0; #这里改成你自己的电脑ip地址
deny publish all;
```
## 访问的url
上面配置启动后，直播流的URL组成如下：
```
rtmp://your.server.ip/live/stream-key-you-set
Field 1: rtmp://your.server.ip/live/
Field 2: stream-key-you-set
```
# 安装ffmpeg
```
sudo apt-get update
sudo apt-get install ffmpeg
```
# 测试
## 测试Nginx
打开浏览器，输入htpp://localhost确保nginx运行正常
## 测试rtmp
### 用ffmpeg产生一个直播流
```
ffmpeg -re -i Caminandes.mp4 -vprofile baseline -vcodec copy -acodec copy -strict -2 -f flv rtmp://localhost/live/test2
```
### 用ffplay播放直播流
```
ffplay rtmp://localhost/live/test2
```
