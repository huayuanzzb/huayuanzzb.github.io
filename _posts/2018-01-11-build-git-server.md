---
layout: post
title:  "搭建 git 服务器"
author: recaton
categories: [ git ]
image: assets/images/12.jpg
featured: true
hidden: true
---

> 公司通常都是使用 gitlab 搭建私有 git 服务，但在一些特殊的场景下，可能仅需要一个只需要命令行交互的 git 服务，这时再使用 gitlab 就显得臃肿。


本文描述搭建一个简单 git 服务器的过程。

#### 安装
1. 安装 git-core
```
apt-get install git-core
```
2. 添加 git 用户，可不设置密码
```
useradd git
```
3. 可选步骤，为方便开发者访问，可以将开发者的 ssh key 加入 git 用户 的authorized_keys
```
echo "xxxxxx" >> ~git/.ssh/authorized_keys
```
安装完成后，即可通过ssh协议访问git仓库。

#### 创建仓库
1. 新建仓库根目录
```
mkdir -p /data/repo
```
2. 新建一个仓库目录，通常以.git结尾
```
mkdir -p /data/repo/test.git
```
3. 初始化空仓库
```
cd /data/repo/test.git
git init --bare
```
4. clone 仓库到本地，必须以 git 用户通过 ssh 连接到服务端，事先需添加 ssh key 到 git 用户的authorized_keys中
```
git clone ssh://git@192.168.10.173/data/repo/test.git
```

#### 配置http访问
使用 nginx 提供 http 服务
1. 安装fcgiwrap，完成后，fcgiwrap会以默认配置自动启动
```shell
root@iZ2ze2gccdk5c3jovkb2tqZ:~# apt-get install fcgiwrap nginx
```
2. 新建配置文件 /etc/nginx/sites-available/git_smart_http.conf，内容如下

```
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
```

3. 启用配置文件
```shell
root@iZ2ze2gccdk5c3jovkb2tqZ:~# ln -s /etc/nginx/sites-available/git_smart_http.conf /etc/nginx/sites-enabled/git_smart_http.conf
```

#### 配置 nginx 用户名密码
1. 安装工具
```shell
root@iZ2ze2gccdk5c3jovkb2tqZ:~# apt install apache2-utils
```
2. 生成用户名密码，/etc/nginx/htpasswd 为 nginx 配置文件中 auth_basic_user_file 的值，test为用户名，在弹出的窗口中输入两次密码即可
```shell
root@iZ2ze2gccdk5c3jovkb2tqZ:~# htpasswd /etc/nginx/htpasswd test
```
3. 重启nginx
```
root@iZ2ze2gccdk5c3jovkb2tqZ:~# service nginx restart
```

4. 测试
```shell
root@iZ2ze2gccdk5c3jovkb2tqZ:~# git clone http://192.168.10.173/git/test.git
Cloning into 'test'...
Username for 'http://192.168.10.173': test
Password for 'http://test@192.168.10.173':
remote: Counting objects: 9, done.
remote: Compressing objects: 100% (4/4), done.
remote: Total 9 (delta 0), reused 0 (delta 0)
Unpacking objects: 100% (9/9), done.
Checking connectivity... done.
root@iZ2ze2gccdk5c3jovkb2tqZ:~# ls
test
```
#### Note
http配置完成后，通过curl访问仓库的url，nginx日志中可能报错，但并不影响git使用
```
2018/10/16 16:41:43 [error] 8551#8551: *4 FastCGI sent in stderr: "Request not supported: '/data/repo/master/ec.git'" while reading response header from upstream, client: 192.168.10.173, server: 192.168.10.173, request: "GET /git/ec.git HTTP/1.1", upstream: "fastcgi://unix:/var/run/fcgiwrap.socket:", host: "192.168.10.173"
```
#### 参考文档
* [How run your own git server](https://www.linux.com/learn/how-run-your-own-git-server)
* [Git smart http](https://git-scm.com/book/zh/v2/%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E7%9A%84-Git-Smart-HTTP)
* [How to serve git through http via nginx with user password](https://stackoverflow.com/questions/6414227/how-to-serve-git-through-http-via-nginx-with-user-password)

