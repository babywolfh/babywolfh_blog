---
title: Node和NPM的安装
tags: [nodejs,npm,nvm]
categories: JavaScript
toc: true
---

# 安装
## Node.js
通过nvm安装
- 安装nvm

``` shell
$ curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.2/install.sh | bash
或者
$ wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.33.2/install.sh | bash
```
修改profile文件(~/.bashrc)
``` shell
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh" # This loads nvm
```
然后使得环境变量生效
``` shell
$ source ~/.bashrc
# 确认安装成功
$ command -v nvm
```
- 安装node

``` shell
$ nvm install 6
# 安装成功后，查看node和npm的版本
$ node -v
$ npm -v
# 中国境内的npm加速方案
$ npm config set registry=http://registry.npm.taobao.org
```
