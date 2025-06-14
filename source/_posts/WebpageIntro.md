---
categories: [编程]
tags: [博客]
title: 架设自己的网站（步骤）
date: 2023-05-06 13:26:21
excerpt: 架设网站的步骤（乱写的）
---

# 原理（先看这个）

<https://blog.oi.al/post/web-page-related>

# 服务器

[上一篇文章](/posts/GnuLinux.md)中推荐了几种GNU/Linux的玩法，但作为一个网站，免费的那几种方法就不好使了。着重推荐云服务器（ECS），去阿里云或者腾讯云之类的网站上看看价位，通常新人优惠和学生优惠力度极大，不会费太多钱。

也可以考虑国外的云服务器提供商，除了架设网站还可以架设*那个*，但是访问速度基本上就跪了。

架设Windows Server也可以，但是价格（每月的价格！）要贵很多，仅限富哥。

# Web Server

购买了ECS服务后，系统这方面就不用操心了（前提是别用Linux From Scratch这种闲人专用发行版）。接下来就该Web Server了。

Web Server也有很多种，Leverage采用[Nginx](https://nginx.org)，本博客的Web服务器使用[Apache](https://httpd.apache.org)搭建（但目前正有迁移至Nginx的打算）。

各个 Web Server 有各自的优点，可以自行搜索了解。

另外，推荐尽可能使用包管理器进行各类软件的安装，[CentOS Stream](https://www.centos.org/centos-stream)自带的[dnf](https://docs.fedoraproject.org/zh_Hans/quick-docs/dnf)、[Ubuntu](https://cn.ubuntu.com)自带的[apt](https://documentation.ubuntu.com/server/how-to/software/package-management)都是强大而好用的包管理器。

# 数据库

没学过也听说过数据库吧？

Leverage使用[MariaDB](https://mariadb.org)，本博客（的上一个[WordPress](https://cn.wordpress.org)版本）使用[MySQL](https://www.mysql.com/cn)。这两款数据库没有本质上的区别，教程都是通用的，还有很多其他数据库，这里不再赘述。

下载使用包管理器就好，配置方法自己搜搜。

# 网页服务

当你的Web Server搞好了之后，你就可以直接往目录放一个.html文件作为静态页面了。

不过一个静态页面实在是太草率了，所以我们需要更复杂的网页服务，例如WordPress。

服务器有六万多个端口，一个端口对应着一个服务，但是只有一个80对应HTTP，一个443对应HTTPS，如果我有多个服务要部署怎么办？这时候就需要反向代理了。这是Nginx的老本行，但是其他Web Server也可以做到。

反向代理就是在你访问`http://foo.com/api/bar`时，实际上是访问`http://foo.com:8000/bar`，此过程又称URL重写，只不过这个重写过程在你的浏览器这边看不出来。这样做还有一个好处就是不公开你的服务端口，增加安全性。

例如，我在`qwertycxz.top:5232`（此链接不可访问）部署了一个[CalDaV](https://www.rfc-editor.org/rfc/rfc4791)服务，但是5232端口不好使用HTTPS，为了提高安全性，我把另一个（当然不会告诉你）的域反向代理到了5232端口，这样即可使用443端口的HTTPS协议访问这个CalDav服务了。

# 域名

然后你只能用IP地址访问你的网站，输一堆数字真的很丑。

所以你需要[域名](https://blog.oi.al/post/web-page-related#_207)，别忘了还要搞DNS。

# SSL证书

这很重要（虽说Leverage目前的SSL证书爆炸了），有效防范中间人攻击。如果你的域名提供商也赠送免费的SSL证书，那再好不过了。如果不，那就去[Let’s Encrypt](https://letsencrypt.org/zh-cn)和[ZeroSSL](https://zerossl.com)领点免费的就行。

[HTTP](https://blog.oi.al/post/web-page-related#http_166)+[SSL](https://blog.oi.al/post/web-page-related#ssl__217)=HTTPS

# 备案

如果你的域名或服务器在国内，那么开放80/443端口还有重要的一步：备案。

请参照服务商的指引（如[阿里云](https://help.aliyun.com/zh/icp-filing/basic-icp-service/user-guide/icp-filing-application-overview)）进行，不会有什么太大的挫折。唯一的问题就是网站名称不能写博客啊啥的，你看现在这个网站叫做“qwerty吃小庄的个人博客”，其实备案名称叫做“橙色的桌子”。(⊙﹏⊙)

完成 ICP 备案以及网安备案后，你要把备案号放到网站首页的底部，然后就齐活了。
