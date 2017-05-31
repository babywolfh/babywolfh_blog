---
title: 使用Hexo+openresty+github搭建博客
date: 2017-05-03 11:35:33
tags: [hexo,openresty,github,博客]
categories: 其他
toc: true
---

本文主要记录自己使用hexo+openresty+github在VPS上搭建个人博客的方法，以免遗忘。
<!--more-->
# hexo
## hexo安装
### 安装node
参考[Node和NPM的安装](http://babywolfh.win/2017/05/31/node%E5%92%8Cnpm%E5%AE%89%E8%A3%85/)
### 安装hexo
``` shell
$ npm install -g hexo-cli
```
### 创建博客目录
``` shell
$ hexo init <folder>
$ cd <folder>
$ npm install
```
新建完成后，指定文件夹的目录如下：
```
.
├── _config.yml   # 网站的配置文件
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes        # 网站的主题目录，Hexo会根据主题来生成静态页面。
```
## hexo配置
您可以在_config.yml 中修改大部份的配置。参考[官网文档](https://hexo.io/zh-cn/docs/configuration.html).
这里只列出感兴趣的部分：

| 参数 | 描述 |
| --- | --- |
|title | 网站标题 |
|subtitle	| 网站副标题 |
|description	| 网站描述，主要用于SEO|
|author	|您的名字，用于主题显示文章的作者|
|language	|网站使用的语言，中文：zh-Hans|
|timezone	|网站时区。比如说：America/New_York, Japan, 和 UTC 。|
| url |	网址 |
| theme	| 当前主题名称。值为false时禁用主题|

## hexo常用命令
| 命令 | 功能 | 缩写 |
| --- | --- | --- |
| hexo init [folder] | 新建一个网站 ||
|hexo new [layout] \<title\> | 新建一篇文章||
|hexo generate | 生成静态文件 |hexo g|
|hexo deploy|部署网站 |hexo d|
|hexo server | 启动服务器 ||
| hexo clean|清除缓存文件 (db.json) 和已生成的静态文件 (public)||
|hexo version|显示 Hexo 版本||

# openresty
因为hexo的server常用于debug和自测场景，在实际上线环境下，还是使用nginx比较好。因为后面的github webhook需要使用到lua脚本来简化工作，所以我这里直接使用openresty。
## 安装
建议参考[官网安装](http://openresty.org/en/installation.html),我这里列出centos的安装步骤。
``` shell
$ sudo yum install yum-utils
$ sudo yum-config-manager --add-repo https://openresty.org/package/centos/openresty.repo
$ sudo yum install openresty
```


## 防火墙
在centos7下面，防火墙由iptables变更为firewalld，测试时需要增加相应的80端口，或者暂时关闭防火墙。
配置firewalld 使用firewall-cmd，默认需要安装：
``` shell
$ yum install firewalld firewalld-config
# 查看当前开放zone，端口，服务：
$ firewall-cmd --get-active-zones
$ firewall-cmd --zone=public --list-ports
$ firewall-cmd --zone=public --list-service
# 增加80端口：
$ firewall-cmd --zone=public --add-port=80/tcp
```
## 开机自启动
### 第一种方法：SYSV方式
```
$ chkconfig --add openresty
$ chkconfig openresty on
```
### 第二种方法：systemd
编辑配置文件/usr/lib/systemd/system/nginx.service
```
[Unit]
Description=The nginx HTTP and reverse proxy server
After=syslog.target network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/usr/local/openresty/nginx/logs/nginx.pid
ExecStartPre=/usr/local/openresty/nginx/sbin/nginx -t
ExecStart=/usr/local/openresty/nginx/sbin/nginx
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
然后执行如下命令启动服务和使能自启动
```shell
$ systemctl enable nginx.service # 使能服务开启启动
$ systemctl start nginx.service  # 手动启动服务
```
## 配合hexo使用的配置文件
修改/usr/local/openresty/nginx/conf/nginx.conf文件
```
http {
  server {
    ...
    location /webhooks { # 实现github的webhooks发送的post请求处理
        content_by_lua_file /usr/local/openresty/nginx/blog_hook.lua;
    }

    location / {
        #root   html;
        root   /path/to/hexo/blog_dir/public; # 指向hexo init生成的目录下的public目录
        index  index.html index.htm;
    }
    ...    
  }
}
```
新建立/usr/local/openresty/nginx/blog_hook.lua文件
``` lua
local signature = ngx.req.get_headers()["X-Hub-Signature"]
local key = "xxxxx" --和你在github webhook上配置的相同
if signature == nil then
    return ngx.exit(404)
end
ngx.req.read_body()
local t = {}
for k, v in string.gmatch(signature, "(%w+)=(%w+)") do
    t[k] = v
end
local str = require "resty.string"
local digest = ngx.hmac_sha1(key, ngx.req.get_body_data())
if not str.to_hex(digest) == t["sha1"] then
    return ngx.exit(404)
end
os.execute("bash /path/to/hexo/blog_dir/blog_update.sh");--改成你的真实目录路径
ngx.say("OK")
ngx.exit(200)
```
新建/path/to/hexo/blog_dir/blog_update.sh文件
``` shell
#! /bin/bash
blog_dir=/path/to/hexo/blog_dir/source/_posts #改成你的真实目录路径
blog_hexo_dir=/path/to/hexo/blog_dir #改成你的真实目录路径
git=/usr/bin/git
hexo=/usr/bin/hexo
branch=master

cd $blog_dir

$git reset --hard origin/$branch
$git clean -f
$git pull

cd $blog_hexo_dir
$hexo clean
$hexo d -g
```
# github webhook
## 配置
参考[这篇文章](https://aotu.io/notes/2016/01/07/auto-deploy-website-by-webhooks-of-github/)的"配置github webhooks"这一节

# hexo优化
## hexo主题
常见的主题有：

## 有用的插件

# 编写博客
## 文章组成
## 常见语法
## 支持数学公式
