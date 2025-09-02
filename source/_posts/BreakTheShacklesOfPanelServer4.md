---
categories: [编程]
tags: [我的世界, 突破面板服的桎梏]
title: 突破面板服的桎梏 4：SSH与数据库
date: 2025-09-01 23:46:31
excerpt: 通过PRoot，实现SSH与数据库的配置
---

搭建MC服务器时，我们有多种选择，其中淘宝的面板服租赁业务提供了便捷的搭建方式。可是面板服的自由度极其低下，最多只允许你跑一个基本的Paper或整合包端。

在“突破面板服的桎梏”系列中，我们将探讨如何一步步突破面板服的限制，搭建一个全功能的服务器。本系列难度逐步递增，适合不同水平的服主。

本篇文章为第四期，需求扎实的Linux运维经验。

**本文介绍了PRoot，通过其搭建了SSH服务器与数据库。**

# 为何不用Telnet

首先是，Telnet的密码以及一切传输数据都是明文发送的，极端不安全。其次，VSCode不支持Telnet进行远程连接，导致管理服务器很麻烦。

> 也许有人喜欢Vim吧，但是精简的容器内压根就没有Vi，Vim更没有了。

VSCode自带两种远程连接支持，SSH和隧道。

## 隧道

大家应该都很熟悉Remote SSH了，但对于Remote Tunnels不太了解。这种连接方式要求本地运行Code Server，用其连接GitHub和微软的服务器，以纯传出连接的方式进行远程访问。

看似很美好，但是当我们下载好Code Server后，尝试在Remote Tunnels登录时却会显示向GitHub的访问被拒绝。

`nslookup`[^1]一波github.com，发现访问被解析到了127.0.0.1。`cat /etc/hosts`一看：

```
127.0.0.1	raw.githubusercontent.com
127.0.0.1	moddinglegacy.com
127.0.0.1	blamejared.com
127.0.0.1	updates.blamejared.com
127.0.0.1	pastebin.com
127.0.0.1	api.github.com
127.0.0.1	github.com
127.0.0.1	modrinth.com
127.0.0.1	www.curseforge.com
127.0.0.1	curseforge.com
127.0.0.1	discord.com
127.0.0.1	repo1.maven.org
127.0.0.1	www.spigotmc.org
127.0.0.1	gist.githubusercontent.com
127.0.0.1	enigmaticlegacy-common.omniconf
```

这就完蛋了，我们没法改`/etc/hosts`啊。

# 架设“虚拟机”

第三期提到`/etc/passwd`中不存在我们的用户名，因此同样无法搭建SSH和数据库。

> 理论上可以先在其他计算机做好数据库文件（至少一个密码登录的用户），上传到面板服并禁用[Unix Sockets](https://dev.mysql.com/doc/mysql-shell/9.4/en/mysql-shell-connection-socket.html)/[`peer`](https://www.postgresql.org/docs/current/auth-peer.html)的登录方式以启用数据库，但有点麻烦且有安全隐患，暂不考虑。

如果我们可以修改`/etc/passwd`或`/etc/hosts`就好了，很容易想到使用虚拟机。但是各类传统虚拟机，如KVM等：

- 无Root架设困难；
- 动态链接，而我们所在的容器缺失几乎所有动态链接库；
- 性能损失巨大。

轻量级虚拟机呢？很容易想到Docker，但第三期也说过了，[RootlessKit](https://github.com/rootless-containers/rootlesskit)需求[User Namespace](https://man7.org/linux/man-pages/man7/user_namespaces.7.html)，而你的容器大概率要么没有给你分配Namespace，要么甚至内核中都没有编译User Namespace。

此外还有办法吗？熟悉Linux的你应该想到了[chroot](https://manpages.debian.org/trixie/manpages-zh/chroot.1.zh_CN.html)，但chroot本身就需要root权限……因此考虑chroot的替代品：

- [become-root](https://github.com/giuseppe/become-root)：和RootlessKit原理类似，通过User Namespace达到模拟root权限的目的，用不了。
- [fakeroot](https://wiki.debian.org/FakeRoot)：通过`LD_PRELOAD`劫持系统调用，模拟root权限。然而我并没能找到这个工具的二进制静态编译版本，可能需要自己编译，太麻烦。
- [fakerootng](https://sourceforge.net/projects/fakerootng): 通过`ptrace`劫持系统调用，模拟root权限。但这个项目九年没更新了，可用度不太高的样子。
- [fakechroot](https://github.com/dex4er/fakechroot)：和RootlessKit原理类似，通过User Namespace达到模拟root权限的目的，用不了。
- [proot](https://proot-me.github.io)：通过`ptrace`劫持系统调用，模拟root权限。虽然两年没更新，但是提供了静态链接的二进制可执行文件，可以一用。
- [unshare](https://man7.org/linux/man-pages/man1/unshare.1.html)：和RootlessKit原理类似，通过User Namespace达到模拟root权限的目的，用不了。

# PRoot简介

PRoot使用了[`ptrace`](https://manpages.debian.org/trixie/manpages-dev/ptrace.2.en.html)这一个极其强大的系统调用。这一系统调用可以修改、跟踪主进程fork出的子进程的系统调用，被GDB等调试器广泛使用。

> 这是一个很有用的系统调用，你的系统内核基本不可能修剪掉它。如果你的面板服真的疯到了连`ptrace`都禁用了，那就没救了。

PRoot提供了多个[示例](https://proot-me.github.io/#examples)，展现了其强大的功能。下面报一遍菜名：

- `chroot`等效：顾名思义，让子进程在访问`/`时实际上访问的是`/path/to/new/root`。
- `mount --bind`等效：让子进程在访问`/dir`时实际上访问的是`/path/to/new/dir`，哪怕`/dir`实际存在也给你拐到`/path/to/new/dir`去；不影响其他路径。和上面那个差不多，但是更灵活。
- `su`等效：假装进程是以另一个用户运行的，访问文件时会给出一个虚拟的文件持有者ID。
- `binfmt_misc`等效：使用qemu进行二进制格式转换。这个有点高级，我们也用不着。

真是神器啊！我们甚至不需要用`chroot`等效了，只使用`mount --bind`等效偷偷替换掉`/etc/passwd`文件就可以了~

[下载](https://gitlab.com/proot/proot/-/pipelines)并放到我们的`~bin`目录下即可，静态链接开箱即用。

# SSH

几乎所有发行版都自带SSH服务器，哪怕是默认不安装SSH服务器的也会在包管理器中提供。可我们的面板服显然不带SSH服务器，也自然用不了apt包管理器，只好手动安装SSH服务器。

> 你说nix包管理器？应该不错，你可以试试看。我这就手动装了。

主流的SSH服务器有俩：OpenSSH和Dropbear。OpenSSH是最主流的，但是需要编译安装（至少我没在网上找到预编译好的）。Dropbear则是一个主要由嵌入式设备使用的SSH服务器，缺失scp支持；但我们已经有stcp了，所以这绝对不是什么大问题。

Dropbear官方没有提供预编译的静态链接版本，但有好心人提供了：<https://bin.leommxj.com>，直接下载x86_64的`dropbearmulti`即可。`dropbearmulti`和`busybox`类似，将其上传到服务器的`~/bin`目录下并创建一个指向`dropbearmulti`，名为`dropbear`的符号链接即可使用。

下一步就是配置密钥了，需要两个密钥：

1. 给Dropbear自己用的密钥，可以用`dropbearkey`[^2]手动生成，也可以用`dropbear -R`自动生成；
2. 登录用的密钥（主要是我们也不知道密码，必须用密钥登录），这个用openssh的那个keygen生成一下就好，注意Dropbear可能不支持ED25519，得用ECDSA。

我将登录用的密钥`authorized_keys`、Dropbear用的密钥`dropbear_ecdsa_host_key`与增加了998用户的`passwd`一同放到了`~/.ssh`目录下，权限均为`600`。在`~/.ssh`目录下执行如下PM2启动Dropbear：

```sh
pm2 start proot -n ssh --namespace shell -- -b passwd:/etc/passwd dropbear -Fsp 12345 -r dropbear_ecdsa_host_key
```

使用你PC的OpenSSH连接，成功即算告一段落。

## 解决：SSH服务器关不掉

很古怪的是，当你执行`pm2 stop ssh`时，只有PRoot停止了，Dropbear却依然在运行。默认来说，PM2会连带停止所有子进程的，怎么回事呢？

阅读[代码](https://github.com/Unitech/pm2/blob/master/lib/TreeKill.js#L34)，发现PM2使用`ps --ppid`命令来获取子进程信息，而`dropbear`的`ps`没这个参数……

但没关系！继续阅读[代码](https://github.com/Unitech/pm2/blob/master/lib/TreeKill.js#L22)，发现PM2在FreeBSD系统下通过`pgrep -P`命令获取子进程信息，`dropbear`支持完全体的`pgrep`。

Bun在启动时会读取`./bunfig.toml`配置文件。既然每次我们每次启动服务器都会在`~`运行`pm2 resurrect`，那么我们只需在`~/bunfig.toml`中写入：

```toml
[define]
"process.platform" = "'freebsd'"
```

这样我们就假装运行在FreeBSD系统，而非Linux系统了。接下来重启PM2，再`pm2 stop ssh`，就正常了。

## 解决：VSCode SSH remote卡在“正在上传VSCode服务器”

日志显示：

```
Copying file to remote with "C:\Windows\System32\OpenSSH\scp.exe" "vscode-cli-6f17636121051a53c88d3e605c491d22af2ba755.tar.gz" "vscode-cli-6f17636121051a53c88d3e605c491d22af2ba755.tar.gz.done" "panel-server":"/home/container/.vscode-server"
```

VSCode SSH remote会以两种方式在远程架设VSCode服务器：

1. 从VSCode官方下载。然而`/etc/hosts`被篡改无法下载。
2. 从本地用`scp`上传。然而我们的Dropbear不支持`scp`。

因此有两种方法解决：

1. 在PRoot额外劫持`/etc/hosts`文件，改起来很方便。
2. 使用`sftp`手动上传`vscode-cli-xxx.tar.gz`与`vscode-cli-xxx.tar.gz.done`两个文件[^3]到`~/.vscode-server`目录。注意先上传`.tar.gz`再上传`.tar.gz.done`。

二者任选其一即可。

## 解决：PRoot带来的性能损失

显然PRoot会带有性能损失。而我们的SSH是PRoot的子进程，所以我们使用SSH创建的一切子进程也会受到PRoot的性能损失影响。有没有办法甩掉这个影响呢？

还记得我们上一期搭建的Telnet吗？

是的！虽然Telnet有安全问题，但如果躲在堡垒机后，Telnet就是安全的。而我们的面板服的堡垒机，就是PRoot！

顺便可以把Telnet的密码验证关了，反正外界也连不到只监听本地回环的Telnet。

```sh
pm2 start telnetd -n telnet --namespace shell -- -Fb 127.23.0.1 -l bash
```

### 在VSCode内联终端中快捷访问Telnet

在VSCode设置（远程）中（对应设置文件位于`~.vscode-server/data/Machine/settings.json`），添加这两项：

```json
"terminal.integrated.defaultProfile.linux": "telnet",
"terminal.integrated.profiles.linux": {
	"bash": {
		"icon": "terminal-bash",
		"path": "bash",
	},
	"telnet": {
		"args": [
			"127.23.0.1",
		],
		"icon": "terminal-linux",
		"path": "telnet",
	},
},
```

现在在VSCode新建终端，默认新建的就是Telnet终端而非有性能损失的SSH终端了。

## 解决：使用PC机代理

修改PC的`.ssh/config`。看这个端口号，懂的都懂，这么配就行。

```
Host panel-server
  HostName example.com
  IdentityFile ~/.ssh/panel-server.pem
  Port 12345
  RemoteForward 10809 127.0.0.1:10809
  User container
```

## 解决：VSCode下载插件显示证书异常

显然是因为容器没有内置的系统CA，挂代理也没用。

要么用PRoot劫持系统CA文件，要么在配置文件中增加

```json
"extensions.verifySignature": false
```

这个选项会灰显作用域无效，但实际上完全可用。

# 数据库

具体思路比较简单，先用PRoot的`su`等效模式启动数据库服务端，再用`su`等效模式连接数据库，新建一个允许进行TCP/IP连接的用户，然后不用PRoot也能正常启动数据库服务端并连接了。

以下以[MariaDB](https://mariadb.org/download '随便下一个版本的就行')的`mariadbd-safe`脚本为例。PostgreSQL的步骤类似，在此不再赘述。

```sh
proot -0 bash mariadbd-safe # 以虚拟root用户启动MariaDB
proot -0 mariadb # 以虚拟root用户连接MariaDB
# 进行一系列配置
# ……
pm2 start bash -n mariadb --namespace database -- mariadbd-safe
```

# 总结

通过以上步骤，前三期提到的所有可解决的问题都已解决。可以认为本系列迎来了结局。

目前还有两个问题，恐怕很难解决：

- Docker无法使用：毕竟我们只是虚拟了文件系统，而骗不过内核。除非我们跑一个用户态内核，否则Docker就别想了，可性能损失太大，得不偿失；
- 小于1024的特权端口无法使用。老老实实用高端口吧。

其他问题，像是包管理器的缺失，其实完全可以用PRoot解决，但我懒……

总之，感谢大家的阅读，希望本系列对你们有所帮助！若有任何疑问或建议，欢迎在留言区发言~

![本人成果](/images/PanelServerVscode.png)

[^1]: 用的当然是Busybox的nslookup

[^2]: 指向`dropbearmulti`，名为`dropbearkey`的符号链接

[^3]: 这俩文件在哪？建议用Everything搜。
