---
title: Jekyll博客部署迁移到Linux（Centos7）且支持https教程
layout: post
guid: urn:uuid:85cd4b63-e9f0-4459-8ca7-fbdsdc2a2fad
tags:
  - Jekyll
  - https
---

之前一直把博客托管到Github pages上，虽然方便但是也有几个弊端，比如：

- 1.有时访问速度过慢
- 2.国内的一些搜索引擎无法检索到

刚好自己有一个服务器，所以准备将博客迁移到自己的服务器上，当然推荐将内容在github也保存一份，以防万一数据丢失。

操作前提：

1.申请一个自己的服务器和域名(国内的有些服务器用80端口需要备案，如使用请先备案)
2.Jekyll博客安装
3.有自己的github

### 步骤

#### 1.编译自己的博客

执行命令：

```jekyll
jekyll build
```
jekyll将编译完的文件放在工程的*_site*目录下，部署的时候可以单独上传这个文件夹就行，但这里我们会将自己的博客工程存储在github中，所以我们将工程的所有内容都上传到github上了。

#### 2.上传到github

如何跟将项目跟github上的Repositories绑定请自行参考google。

执行命令进行上传：

```git
git add -f *
git commit -m "update blog"
git push
```

#### 3.服务器安装git并clone工程项目

执行命令：

```shell
yum install git

//clone你自己的项目

git clone https://github.com/godpan/godpan.github.io.git
```

#### 4.安装nginx并修改配置

```shell
yum install nginx
```
修改nginx配置

```shell
vi /etc/nginx/nginx.conf

//修改以下内容，其他不变

user root;

server {
    listen       80 default_server;
    listen       [::]:80 default_server;
    server_name  _;
    root         /root/godpan.github.io/_site; #博客_site文件目录

    # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;

    location / {
    }

    error_page 404 /404.html;
        location = /40x.html {
    }

    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
}
```
保存文件，并启动nginx并开启80端口

```
systemctl start nginx

firewall-cmd --add-port=80/tcp --permanent
firewall-cmd --reload
```
然后直接访问你服务器的ip，不出意外应该能直接看到你自己博客的内容了，接下来我们就将域名与ip绑定。

#### 5.绑定域名

我这里操作的是阿里云上的域名，仅供参考，其他平台的自己按照规则设置，配置如下：

![domain-settings](/media/images/2019/09/domain-settings.png)

等域名生效（通常不超过10分钟）就可以直接用域名访问你的博客了。

如果你不想给网站配置为https的话，到这里就大功告成了。

#### 6.配置https

这里我们使用Certbot来生成https需要的证书，先安装Certbot：

```shell
yum install certbot
```
生成证书：
```shell
certbot certonly --webroot -w /root/godpan.github.io/_site -d godpan.me -d www.godpan.me
```
证书生成的目录在：

```shell
/etc/letsencrypt/live/godpan.me  #后面为你自己的域名
```
修改nginx对应的配置：

```shell
vi /etc/nginx/nginx.conf

#修改443端口对应的server内容：

server {
    listen       443 ssl http2 default_server;
    listen       [::]:443 ssl http2 default_server;
    server_name  _;
    root         /root/godpan.github.io/_site; #博客_site文件目录

    ssl_certificate "/etc/letsencrypt/live/godpan.me/fullchain.pem";  #证书的路径
    ssl_certificate_key "/etc/letsencrypt/live/godpan.me/privkey.pem"; #证书密钥的路径
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout  10m;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;

    location / {
    }

    error_page 404 /404.html;
        location = /40x.html {
    }

    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
}
```
开启443端口，并重启nginx：

```shell
firewall-cmd --add-port=443/tcp --permanent
firewall-cmd --reload
systemctl restart nginx
```
这时候你就可以用https访问你的网站了，比如https://godpan.me，但因为certbot证书有3个月的有效期，所以我们需要定时去更新证书，不过它提供了更新证书的命令：
```
certbot renew
```
但手动更新的话可以能会忘，所以我们用系统自身的定时任务去更新：

创建一个定时任务：

```shell
vi certbot-auto-renew-crontab

#输入以下内容进行保存，含义是每个月第一天的一点凌晨会去更新一次证书

0 1 1 * * certbot renew --pre-hook "systemctl stop nginx" --post-hook "systemctl start nginx"
```
最后我们crontab来启动这个任务：

```shell
crontab certbot-auto-renew-crontab
```
至此大功告成，除非你的服务器崩溃或者被封了😂😂😂。



