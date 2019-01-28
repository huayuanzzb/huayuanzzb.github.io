---
layout: post
title:  oids
date:   2019-01-28 08:49:26 +0800
nav_order: 2
parent: others
categories: others
---

> 往往使用gitlab搭建私有git服务，但在一些特殊的场景下，可能仅需要一个简单的git服务，这时再使用gitlab就显得臃肿。

#### 安装
```
# 安装git-core
root@iZ2ze2gccdk5c3jovkb2tqZ:~# apt-get install git-core
  
# 添加git用户，可不设置密码
root@iZ2ze2gccdk5c3jovkb2tqZ:~# useradd git
# 可选步骤，为方便开发者访问，可以将开发者的key加入git用户的authorized_keys
root@iZ2ze2gccdk5c3jovkb2tqZ:~# echo "xxxxxx" >> ~git/.ssh/authorized_keys
```
安装完成后，即可通过ssh协议访问git仓库。
```
# 新建git仓库根目录
root@iZ2ze2gccdk5c3jovkb2tqZ:~# mkdir -p /data/repo
# 新建一个仓库目录，通常以.git结尾
root@iZ2ze2gccdk5c3jovkb2tqZ:~# mkdir -p /data/repo/test.git
# 初始化一个空仓库
root@iZ2ze2gccdk5c3jovkb2tqZ:~# cd /data/repo/test.git
root@iZ2ze2gccdk5c3jovkb2tqZ:/data/repo/test.git# git init --bare
# 在客户端使用,客户端必须可以以git用户ssh到服务端，如果不能，添加key到git用户的authorized_keys
root@iZ2ze2gccdk5c3jovkb2tqZ:~# git clone ssh://git@192.168.10.173/data/repo/test.git
```
#### 配置http访问
```
# 使用nginx为静态服务器必须先安装fcgiwrap，安装完成后，fcgiwrap会以默认配置自动启动
root@iZ2ze2gccdk5c3jovkb2tqZ:~# apt-get install fcgiwrap
# 安装nginx
root@iZ2ze2gccdk5c3jovkb2tqZ:~# apt-get install nginx
# 添加配置
root@iZ2ze2gccdk5c3jovkb2tqZ:~# vim /etc/nginx/sites-available/git_smart_http.conf
# Only for git smart http running on localhost.
 
server {
  listen  192.168.10.173:80;
  auth_basic            "Restricted";
  auth_basic_user_file  /etc/nginx/htpasswd;
 
  location ~ /git(/.*) {
    # Set chunks to unlimited, as the body's can be huge
        client_max_body_size            0;
 
        include     fastcgi_params;
        fastcgi_param   SCRIPT_FILENAME     /usr/lib/git-core/git-http-backend;
        fastcgi_param   GIT_HTTP_EXPORT_ALL "";
        # GIT_PROJECT_ROOT 为git仓库根目录
        fastcgi_param   GIT_PROJECT_ROOT    /data;
        fastcgi_param   PATH_INFO       $1;
 
        # Forward REMOTE_USER as we want to know when we are authenticated
        fastcgi_param   REMOTE_USER     $remote_user;
        fastcgi_pass    unix:/var/run/fcgiwrap.socket;
    }
}
root@iZ2ze2gccdk5c3jovkb2tqZ:~# ln -s /etc/nginx/sites-available/git_smart_http.conf /etc/nginx/sites-enabled/git_smart_http.conf
# 配置nginx用户名密码
## 1. 安装工具
root@iZ2ze2gccdk5c3jovkb2tqZ:~# apt install apache2-utils
## 2. 生成用户名密码，/etc/nginx/htpasswd为nginx配置文件中auth_basic_user_file的值，test为用户名，在弹出的窗口中输入两次密码即可
root@iZ2ze2gccdk5c3jovkb2tqZ:~# htpasswd /etc/nginx/htpasswd test
# 重启nginx
root@iZ2ze2gccdk5c3jovkb2tqZ:~# service nginx restart
# 测试
root@iZ2ze2gccdk5c3jovkb2tqZ:~# git clone http://192.168.10.173/git/ec.git
Cloning into 'ec'...
Username for 'http://192.168.10.173': test
Password for 'http://test@192.168.10.173':
remote: Counting objects: 9, done.
remote: Compressing objects: 100% (4/4), done.
remote: Total 9 (delta 0), reused 0 (delta 0)
Unpacking objects: 100% (9/9), done.
Checking connectivity... done.
root@iZ2ze2gccdk5c3jovkb2tqZ:~# ls
ec
```
#### Note
http配置完成后，通过curl访问仓库的url，nginx日志中可能报错，但并不影响git使用
```
2018/10/16 16:41:43 [error] 8551#8551: *4 FastCGI sent in stderr: "Request not supported: '/data/repo/master/ec.git'" while reading response header from upstream, client: 192.168.10.173, server: 192.168.10.173, request: "GET /git/ec.git HTTP/1.1", upstream: "fastcgi://unix:/var/run/fcgiwrap.socket:", host: "192.168.10.173"
```
#### 参考文档
https://www.linux.com/learn/how-run-your-own-git-server

https://git-scm.com/book/zh/v2/%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E7%9A%84-Git-Smart-HTTP

https://stackoverflow.com/questions/6414227/how-to-serve-git-through-http-via-nginx-with-user-password

