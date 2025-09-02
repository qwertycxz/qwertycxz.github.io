---
categories: [编程]
tags: [我的世界, 突破面板服的桎梏]
title: 突破面板服的桎梏 2：Bash
date: 2025-08-30 11:29:33
excerpt: 在面板服中使用Linux命令行
---

搭建MC服务器时，我们有多种选择，其中淘宝的面板服租赁业务提供了便捷的搭建方式。可是面板服的自由度极其低下，最多只允许你跑一个基本的Paper或整合包端。

在“突破面板服的桎梏”系列中，我们将探讨如何一步步突破面板服的限制，搭建一个全功能的服务器。本系列难度逐步递增，适合不同水平的服主。

本篇文章为第二期，需要你了解HTTP与Websocket通信，并熟悉使用浏览器F12抓包。

**本文给出了一种在面板服使用Linux命令行的方法，解决[当Velocity重启时，所有的子服务器也会跟着重启](/posts/BreakTheShacklesOfPanelServer1#总结)的问题。**

**推荐阅读：**

- [知乎上一篇简明的Websocket科普](https://www.zhihu.com/question/20215561/answer/40316953)
- [博客园的一篇F12教程](https://www.cnblogs.com/sheepboy/p/16082594.html)

# 访问Bash

[上一篇文章](/posts/BreakTheShacklesOfPanelServer1)中，我们探讨了Impulse服务端管理器的基本使用方法，使用其成功搭建了一个多世界的Minecraft服务器，但这个插件尚有一些不足函待解决。如果我们可以直接访问Shell，那么一切困难就都能解决了。

在面板服自带的文件管理中，我们可以发现当前用户的家目录是`/home/container`，这提示我们的服务器跑在一个Linux容器[^1]中。一个容器可以极端轻量化，但至少也得有个Bash（或者Sh）可供使用。

问题是，所有的shell都是native[^2]的二进制文件，而面板服的启动按钮只能以`java -jar xxx.jar`的形式运行程序，因此一个交互式的shell无法直接启动。

## 方案1：加载可以调用shell的mod/插件

[ExecuteHostCommand](https://modrinth.com/plugin/hostcommand)是一个Velocity插件，可以通过MC命令调用shell命令，从而达到访问Bash的目的。

然而实战中，这个插件不太好用……它在处理字符串时的策略不太对，想通过`bash -c`执行指令极其困难。而这个插件又不支持输入stdin，因此交互式终端也没辙。

这个方法似乎走不通……

## 方案2：自己编写一个“端”

市面上各种服务端很多，什么Paper啊Fabric啊Velocity啊，一个共同的特点是都是可以通过`java -jar xxx.jar`的形式启动的。那我自己编一个端，直接在里面调用Bash不就好了？

隆重介绍——[BashJar](https://github.com/qwertycxz/BashJar)端！这是我个人自编的服务端，以Apache 2.0协议开源，代码寥寥数行：

```java
import java.io.IOException;
public class Bash {
	public static void main(String[] args) throws InterruptedException, IOException {
		new ProcessBuilder("bash").inheritIO().start().waitFor();
	}
}
```

如果不会Java也不用担心，[Releases](https://github.com/qwertycxz/BashJar/releases)中可以直接下载`.jar`文件，当作一个MC服务端运行就可以了~

这个端在网页控制台与Bash之间建立了桥梁，现在我们终于可以在网页控制台用Bash了！

![BashJar在网页控制台的效果](/images/PanelServerConsoleBash.png)

可以看到，`ls -Ahl /`、`df -h`与`uname -a`等命令正常运行。

可是当我们运行`cd /`时，问题出现了：![该命令已被禁止？！](/images/PanelServerConsoleBanned.png)

这什么情况？

### 禁用原因分析

开F12，发现网页控制台是通过[Websocket](https://developer.mozilla.org/zh-CN/docs/Web/API/WebSocket)进行通信的，而我们的`cd`命令并没有被发送到后端，也就是说这个限制是在前端进行的。

在F12中翻代码，发现：

```js
for (let i in [
	'sudo ',
	'su ',
	'root',
	'./',
	'sh ',
	'bash ',
	'wget ',
	'curl ',
	'apt ',
	'unzip ',
	'tar ',
	'cd ',
]) {
	if (r.includes(i))
		return ne({
			key: 'server:setup',
			type: 'error',
			message: '该命令已被禁止',
		})
}
y.send('send command', r)
```

含`cd`、`wget`、`curl`等常用关键命令的语句被禁止，其他语句则通过Websocket发送。

假如这个限制是纯前端的，我们只需要重新实现一遍这个前端，就可以绕过这个纯前端的限制了。

> 另一种做法是通过油猴脚本等方法禁用这里的限制，但我对油猴脚本不太熟悉，pass。

### 我的实现

[戳这里](/pages/PanelServerShell)

替换必要的内容，并用Python运行：

```sh
> python main.py
I have no name!@xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:~$ cd /
ls -Ahl
I have no name!@xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:/$ ls -Ahl
total 48K
lrwxrwxrwx   1 root root    7  6月 12  2024 bin -> usr/bin
drwxr-xr-x   2 root root 4.0K  1月 29  2024 boot
drwxr-xr-x   5 root root  360  6月 23 22:23 dev
-rwxr-xr-x   1 root root    0  6月 23 22:23 .dockerenv
-rw-r--r--   1 root root  534  2月 17  2025 entrypoint.sh
drwxr-xr-x   1 root root 4.0K  6月 23 22:23 etc
drwxr-xr-x   1 root root 4.0K  2月 21  2025 home
lrwxrwxrwx   1 root root    7  6月 12  2024 lib -> usr/lib
lrwxrwxrwx   1 root root    9  6月 12  2024 lib64 -> usr/lib64
drwxr-xr-x   2 root root 4.0K  6月 12  2024 media
drwxr-xr-x   2 root root 4.0K  6月 12  2024 mnt
drwxr-xr-x   2 root root 4.0K  6月 12  2024 opt
dr-xr-xr-x 955 root root    0  6月 23 22:23 proc
drwx------   2 root root 4.0K  6月 12  2024 root
drwxr-xr-x   3 root root 4.0K  6月 12  2024 run
lrwxrwxrwx   1 root root    8  6月 12  2024 sbin -> usr/sbin
drwxr-xr-x   2 root root 4.0K  6月 12  2024 srv
dr-xr-xr-x  13 root root    0  6月 23 22:23 sys
drwxrwxrwt   2 root root  220  6月 28 23:13 tmp
drwxr-xr-x   1 root root 4.0K  2月 21  2025 usr
drwxr-xr-x   1 root root 4.0K  6月 12  2024 var
```

很好，现在我们终于有全功能的Bash用了！

> **警告**
>
> 我们没法通过这个客户端发送Ctrl+C、Ctrl+D这样的命令，因此不要进入那些只能通过Ctrl+C退出的程序！假如一不小心吃了这样的赛博灯泡，就只能在控制面板强制关服了。

# 运行服务器

最简单的方法，就是命令行结尾加个&：

```sh
java -Xmx4G -jar server.jar &
```

我们的BashJar不会关闭（除非手动输入`exit`或者在控制面板强制关服），因此使用&将进程放到后台运行是绝对可行的，完美解决了“当Velocity重启时，所有的子服务器也会跟着重启”的问题。

这是最简单的办法，没有守护程序。假如意外崩端了，我们就得手动再启动服务器。

也可以看看Wiki上介绍的这些[服务器启动脚本](https://zh.minecraft.wiki/w/Tutorial:服务器启动脚本)。没用过，不知道好不好用。

> **再次警告**
>
> 某些启动脚本可能只依赖Ctrl+C退出，不能使用这种脚本。

# 总结

本文通过一系列手段解决了上一篇文章中提到的第一个问题，但仍有两个问题尚未解决：

1. 无法方便地访问子服务器的控制台，只能通过RCON访问[^3]；
2. 配置不了数据库，因此像[LuckPerms](https://luckperms.net)就没办法实现多端同步了，每个子服与代理端必须独立设置权限。

这两个更复杂的问题，就留到下一期再说吧~

# 补充

我的面板服限制很严苛，但如果你的面板服限制比我的还要严苛，就可能遭遇下面三种情况：

## 我的容器内沒有Bash（运行BashJar报错bash不存在）

一般的服务器都是Ubuntu镜像，但假如是Alpine，一般就只有sh可用了。欢迎留言，我可以帮助构建一个sh版本的BashJar。

## 我面板服的命令禁用限制是在后端执行的（使用那个Python脚本发送命令仍然返回给我此命令已被禁止）

市面上的面板服技术栈都差不多，但如果真有面板服做了黑科技阻止你这么做，我们就需要将这些限制通过别名绕过。

这是一篇介绍别名的[教程](https://www.junmajinlong.com/shell/script_course/shell_alias)。将别名添加到`/home/xxx/.bashrc`中即可生效。

> `.bashrc`的内容会在Bash启动时自动执行。

## 我没权限修改`/home/xxx/.bashrc`

真是不可思议……但是有解！欢迎留言，我可以帮助构建不使用`.bashrc`也可使用别名的BashJar。

[^1]: 容器可以看成一种轻量级虚拟机，具有独立的文件系统和网络栈，但不包含系统内核。

[^2]: 比如C语言编译出来的产物。
