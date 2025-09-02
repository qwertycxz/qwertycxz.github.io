---
categories: [编程]
tags: [我的世界, 突破面板服的桎梏]
title: 突破面板服的桎梏 3：守护程序与Telnet
date: 2025-08-31 18:14:27
excerpt: 通过PM2，在面板服中架设群组服与Telnet
---

搭建MC服务器时，我们有多种选择，其中淘宝的面板服租赁业务提供了便捷的搭建方式。可是面板服的自由度极其低下，最多只允许你跑一个基本的Paper或整合包端。

在“突破面板服的桎梏”系列中，我们将探讨如何一步步突破面板服的限制，搭建一个全功能的服务器。本系列难度逐步递增，适合不同水平的服主。

本篇文章为第三期，需要你拥有一定Linux运维知识，掌握Linux常用命令用法。

**本文探讨并架设了PM2，以其为守护程序运行MC服务器与Telnet服务器，解决[无法方便地访问子服务器的控制台](/posts/BreakTheShacklesOfPanelServer1#总结)的问题与无法通过Ctrl+C退出程序的赛博灯泡。**

**推荐阅读：**

- [我的文章](/posts/GnuLinuxIntro)

# 选择守护程序

上一期我们将服务器进程放到后台运行，避免了面板服重启时子服务器跟着重启的问题。但没有守护进程，服务器崩了就只能手动重启了。既然阅读文章的你已经拥有了一定的Linux运维知识，那应该了解守护程序（Daemon）的用处。下面我们就来挑选守护程序以运行我们的MC服务器。

## Init.d / OpenRC / Runit

这是老牌的服务管理方式。但我们没有root权限，所以不好使……

## Systemd用户模式

Systemd是Init.d的替代品，虽然很多人都不知道，但实际上Systemd是有用户模式的（`systemd --user`），藉由此，即便我们没有root权限，也能使用Systemd的功能。

但是我们的极端精简的Ubuntu容器没有Systemd……这些尴尬了。

## OpenRC用户模式 / Runit用户模式

也许可以用吧，但我不熟……

## Docker

容器化是未来的趋势。众所周知，Docker需要root权限，但实际上Docker支持rootless模式。像是Podman，作为Docker的竞品，更是开箱即用地支持rootless模式。

但有个问题，不知道你有没有发现：

```
I have no name!@xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:~$ whoami
whoami: cannot find name for user ID 998
```

我们当前的用户没有一个用户名！查阅`/etc/passwd`发现用户ID为998的用户压根不存在……

```
I have no name!@xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:~$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
```

“这有什么大不了的！”你想。但事实上有很大的问题，让我们来看看Rootless Docker的核心：[RootlessKit](https://github.com/rootless-containers/rootlesskit)的工作原理：

> RootlessKit is a Linux-native implementation of "fake root" using [user_namespaces(7)](https://man7.org/linux/man-pages/man7/user_namespaces.7.html).

英语好的话，推荐戳上面的链接看看“用户命名空间”的原理。

简单点说，Rootless Docker需求`/etc/subuid`和`/etc/subgid`应包含至少65536个**当前用户**的从属UID/GID，但我们的当前用户不存在！`/etc/subuid`和`/etc/subgid`这两个文件是空的！所以，不行，忘了Docker吧。

如果非常幸运地，你的面板服中的`/etc/subuid`和`/etc/subgid`文件已经配置好了，那么恭喜你，你可以使用Rootless Docker了。详见[官方教程](https://docs.docker.com/engine/security/rootless '纯英文')。

## [PM2](https://pm2.keymetrics.io)

PM2是一个以JavaScript编写的进程管理器，以AGPLv3开源于[GitHub](https://github.com/Unitech/pm2)，主要适用于Node.js应用程序，但也可以用于其他类型的应用程序，比如我们的Minecraft服务器。

此方法经本人尝试完全可行，下面详细讲解。

# 安装PM2

参考[PM2官方文档](https://pm2.keymetrics.io/docs/usage/pm2-doc-single-page '纯英文')。

## 安装JavaScript运行时

Node是主流，但这里我们采用更快且支持单二进制文件的[Bun](https://bun.sh)。在[Releases](https://github.com/oven-sh/bun/releases)中下载*bun-linux-x64.zip*，解压后放到`~/bin`目录下，将`~/bin`添加到`$PATH`就能用了。

> 如果你的系统不允许你修改`~/.bashrc`，可以尝试`~/.profile`。
>
> 我给`~/.bashrc`和`~/.profile`创建了一个硬链接，这样总是能用。
>
> 如果都不能用也问题不大，打全名就好了。

为了方便使用Bun，建议在`~/bin`目录下创建一个指向`bun`，名为`node`的符号链接，这样就可以强制使用Node的JS脚本使用Bun了。

## 安装PM2

```sh
bun i -g pm2
```

记得将`~/.bun/bin`添加到`$PATH`。

# 使用PM2启动MC服务器

以第一期的目录结构为例：

```
.
├── worlds
│   ├── creative
│   │   ├── server.properties
│   │   └── fabric.jar
│   └── survival
│       ├── server.properties
│       └── fabric.jar
├── velocity.jar
└── velocity.toml
```

```sh
pm2 start java -n velocity --namespace minecraft -- -Xmx1G -jar velocity.jar
cd worlds/creative
pm2 start java -n creative --namespace minecraft -- -Xmx2G -jar fabric.jar
cd ../survival
pm2 start java -n survival --namespace minecraft -- -Xmx8G -jar fabric.jar
```

执行`pm2 l`：

| id | name | namespace | version | mode | pid | uptime | ↺ | status | cpu | mem | user | watching |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | creative | minecraft | N/A | fork | 238 | 2M | 0 | online | 0% | 1.2gb | con… | disabled |
| 2 | survival | minecraft | N/A | fork | 340 | 2M | 0 | online | 0% | 1.0gb | con… | disabled |
| 0 | velocity | minecraft | N/A | fork | 173 | 2M | 0 | online | 0% | 793.1mb | con… | disabled |

接下来执行`pm2 save`将当前进程列表保存到`~/.pm2/dump.pm2`，在下次重启时可以通过`pm2 resurrect`恢复。

> 考虑将`pm2 resurrect`加入`~/.bashrc`。

# 架设Telnet

如果你想通过`pm2 logs`查看日志，你就又陷进了赛博灯泡……我们的客户端不好使，必须要搞一个伪终端（pseudo terminal）才行。

SSH呢？前文提到，你的用户不存在，所以基于用户登录的SSH完全不可用。成熟的连接方案只有Telnet了。

可是Telnet太老了，现阶段已经找不到专门的Telnet服务器了……吗？

隆重介绍[Busybox](https://busybox.net)！

如果你折腾过路由器，你大概对Busybox并不陌生。它将多个常用的Unix工具集成在一个可执行文件中，非常适合嵌入式设备。当然我们不是嵌入式设备而是一个容器，但这没关系。Busybox静态链接，非常适合我们的应用场合。

在[官方下载站](https://busybox.net/downloads/binaries)下载最新版构建，解压并扔到`~/bin`目录下。运行`busybox`可以看到有极其大量的可用功能，其中就有我们想要的`telnetd`（太多了，我不放了）。

要怎么使用`telnetd`呢？非常简单：创建一个指向`busybox`，名为`telnetd`的符号链接就可以了。现在输入`telnetd -h`，可以看到帮助信息了。

> 你的容器一般会缺少`pstree`、`free`这样的实用功能，而Busybox提供了全套的Linux工具，按照和`telnetd`一样的步骤使用即可。

## 使用PM2启动Telnetd

```sh
pm2 start telnetd -n telnet --namespace shell -- -Fp 12345 -l bash
```

别忘了`pm2 save`。

在防火墙中放行端口，使用你本地的telnet客户端连接到`<你的面板服IP>:12345`，终于，一个完整的伪终端可以用了，再也不用担心赛博灯泡了！

> 如果在服务器面板吞下了赛博灯泡，只需在Telnet中`kill -2`赛博灯泡的进程就可以了，简单又方便！

## 密码登录Telnet

然后你就发现世界上所有人都可以登录你的面板服了，服务器秒变RBQ啊喂……

我们启动Telnetd的指令，通过`-l`参数指定在启动时使用`bash`，进入bash自然不需要密码。那我们需要在启动时启动一个密码认证程序，认证成功后进入bash就可以了。

```sh
#!/bin/bash
read -s password
if [[ $password == "PASSWORD" ]]; then
	bash
fi
```

将这段代码保存为`~/bin/auth.sh`，并赋予可执行权限。将pm2脚本改为：

```sh
# 如果你还没输入上面那个指令
pm2 start telnetd -n telnet --namespace shell -- -Fp 12345 -l ~/bin/auth.sh
# 如果你已经输入了上面的指令
pm2 start telnet -- -Fp 12345 -l ~/bin/auth.sh
```

# 重新利用面板控制台

我们希望服务器面板可以快捷地查看群组服所有服务器的日志，并发送后台命令到各个服务器。但PM2做不到这一点：`pm2 logs`只能查看日志无法发送命令，`pm2 send`只能发送命令无法查看日志，`pm2 attach`虽然既能查看日志也能发送命令，但一次只能服务一个进程。如果有办法将这些功能整合在一起就好了。

所以我就做了[`pm2-logs-attach-input`](https://github.com/qwertycxz/pm2-logs-attach-input)，并以Apache 2.0协议开源~

我也向PM2官方提交了[PR](https://github.com/Unitech/pm2/pull/5981)，不过他们还没理我……

## 安装`pm2-logs-attach-input`

```sh
bun i -g pm2-logs-attach-input
```

## 使用`pm2-logs-attach-input`

```sh
# 不再使用
pm2 logs minecraft --lines 0
# 而是
pm2-logs-attach-input minecraft --lines 0
```

进入交互式界面后，可以实时查看浏览各进程发来的日志。若要发送命令，只需要输入：

```
velocity glist all
creative gamemode creative @a
survival gamerule keepInventory true
```

也就向这些进程输入了想要的命令。

# 总结

本文解决了以下问题：

- 无法方便地访问子服务器的控制台，只能通过RCON访问。现在使用`pm2-logs-attach-input`就可以了。
- 无法通过Ctrl+C退出程序，只能强制关闭赛博灯泡的问题。我们有了Telnet就不用担心这个了。

但又引入了一个新问题：

- Telnet连接不安全。

同时一个旧问题仍然存在：

- 配置不了数据库。

下一期，我将解决剩余的两个顽疾，敬请期待！
