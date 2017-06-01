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
|hexo new [layout] &lt;title&gt; | 新建一篇文章||
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
### 手动添加MathJax支持
要让网页支持MathJax，其实只要将以下JS代码添加进你的HTML即可，但是我们不能每次生成完静态网页后自己手动添加，那样子会累死人的。
```
<!-- mathjax config similar to math.stackexchange -->
<script type="text/x-mathjax-config">
MathJax.Hub.Config({
    jax: ["input/TeX", "output/HTML-CSS"],   # 可以更改mathjax的默认输出样式
    tex2jax: {
        inlineMath: [ ['$', '$'] ],
        displayMath: [ ['$$', '$$']],
        processEscapes: true,
        skipTags: ['script', 'noscript', 'style', 'textarea', 'pre', 'code']
    },
    messageStyle: "none",
    "HTML-CSS": { preferredFont: "TeX", availableFonts: ["STIX","TeX"] }
});
</script>
<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"></script>
```
为了能让Hexo自动为需要mathjax的文章添加该js代码，我们首先进入主题的layout/_partial/文件夹，创建一个叫做mathjax.ejs的文件，代码内容就是上面的JavaScript。

接下来，因为Hexo渲染你的文章的时候是通过layout/post.ejs进行的，我们只需要在代码的最后添加一个判断语句，如果该文章需要MathJax，我们就加载上面的mathjax.ejs即可。具体来说，将下面的代码添加到post.ejs尾部，注意不要添加到别的判断语句里面了。
```
<% if (page['math']){ %>
<%- partial('_partial/mathjax') %>
<% } %>
```
这样，就完成了Hexo 对 MathJax的支持，接下来再写文章的时候，只需要在文件的头部添加mathjax：true 即可对该文章开启公式渲染，在文章里面你就可以尽情的写公式了。

### hexo-math插件
在博客的根目录执行下面的命令安装插件
```
npm install hexo-math --save
```
修改博客根目录下的站点配置文件_config.xml添加如下内容。**一定要注意如果不修改默认配置，则下面的参数一定一律注释掉，否则会覆盖掉默认配置，造成渲染失败，这个参考网上所有文章和官方github的README，都没说这一点，害得我折腾了2个小时。**
```
math:
  engine: 'mathjax' # or 'katex'
  # 一定要注意如果不修改默认配置，则下面的参数一定一律注释掉，否则会覆盖掉默认配置，造成渲染失败，这个参考网上所有文章和官方github的README，都没说这一点，害得我折腾了2个小时。
  # mathjax:
  #  src: custom_mathjax_source
  #  config:
  #    # MathJax config
  #katex:
  #  css: custom_css_source
  #  js: custom_js_source # not used
  #  config:
  #    # KaTeX config
```
### 例子
- 行内
-
```
Simple inline $a = b + c$.
```

Simple inline $a = b + c$.

- 块
-
```
$$\frac{\partial u}{\partial t}
= h^2 \left( \frac{\partial^2 u}{\partial x^2} +
\frac{\partial^2 u}{\partial y^2} +
\frac{\partial^2 u}{\partial z^2}\right)$$
```

$$\frac{\partial u}{\partial t}
= h^2 \left( \frac{\partial^2 u}{\partial x^2} +
\frac{\partial^2 u}{\partial y^2} +
\frac{\partial^2 u}{\partial z^2}\right)$$


- 单行标签

```
This equation {% math %}\cos 2\theta = \cos^2 \theta - \sin^2 \theta =  2 \cos^2 \theta - 1 {% endmath %} is inline.
```

This equation {% math %}\cos 2\theta = \cos^2 \theta - \sin^2 \theta =  2 \cos^2 \theta - 1 {% endmath %} is inline.

- 多行标签

```
{% math %}
\begin{aligned}
\dot{x} & = \sigma(y-x) \\
\dot{y} & = \rho x - y - xz \\
\dot{z} & = -\beta z + xy
\end{aligned}
{% endmath %}
```

{% math %}
\begin{aligned}
\dot{x} & = \sigma(y-x) \\
\dot{y} & = \rho x - y - xz \\
\dot{z} & = -\beta z + xy
\end{aligned}
{% endmath %}

## 转义字符
在使用 Markdown 编写 Hexo 的 Blog 时，对于特殊字符使用“\”转义有时会不成功，最好的方式是直接使用特殊字符的编码，对应如下：

| 字符 | 编码 | 含义 |
| --- | --- | --- |
| - |&\#45; '&minus';|  减号 |
| ! |&\#33;        |  惊叹号Exclamation mark |
| ” |&\#34; '&quot'; |双引号Quotation mark|
| # |&\#35; |  数字标志Number sign |
| $ |&\#36;  | 美元标志Dollar sign|
| % |&\#37;        | 百分号Percent sign|
| & |&\#38; '&amp';  |Ampersand|
| ‘ |&\#39;        | 单引号Apostrophe|
| ( |&\#40;       | 小括号左边部分Left parenthesis|
| ) |&\#41;        | 小括号右边部分Right parenthesis|
| * |&\#42;        | 星号Asterisk|
| + |&\#43;        | 加号Plus sign|
| < |&\#60; '&lt';   |小于号Less than|
| = |&\#61;        | 等于符号Equals sign||
| > |&\#62; '&gt';   |大于号Greater than|
| ? |&\#63;       | 问号Question mark|
| @ |&\#64;       | Commercial at|
| [ |&\#91;       | 中括号左边部分Left square bracket|
| \ |&\#92;        | 反斜杠Reverse solidus (backslash)|
| ] |&\#93;        | 中括号右边部分Right square bracket|
| { |&\#123;       | 大括号左边部分Left curly brace|
| | |&\#124;       | 竖线Vertical bar|
| } |&\#125;       | 大括号右边部分Right curly brace |
