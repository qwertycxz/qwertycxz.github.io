---
categories: [游戏]
tags: [我的世界, 突破面板服的桎梏]
title: 突破面板服的桎梏 1：群组服
date: 2025-08-29 12:46:32
excerpt: 介绍如何在面板服中搭建群组服
---

搭建MC服务器时，我们有多种选择，其中淘宝的面板服租赁业务提供了便捷的搭建方式。可是面板服的自由度极其低下，最多只允许你跑一个基本的Paper或整合包端。

在“突破面板服的桎梏”系列中，我们将探讨如何一步步突破面板服的限制，搭建一个全功能的服务器。本系列难度逐步递增，适合不同水平的服主。

本篇文章为第一期，需要你有一定Minecraft开服的基本知识，并了解群组服的概念。

**本文将介绍如何在面板服中搭建群组服。**

**推荐阅读：**

- [笨蛋MC开服教程](https://nitwikit.8aka.cn)

# 初见面板服

近来，朋友出资希望我可以帮助他搭建一个Minecraft服务器。经讨论，也是出于成本上的考量，我们决定使用面板服，在淘宝上随机找了一家店买了一年。

> 就我而言，我强烈推荐不要租赁服务器，而是自购一台服务器搭建MC私服。淘宝上面板服超卖现象很严重，哪怕是阿里云我也不信它不超卖。

![面板服控制台](/images/PanelServerConsole.png)

可以看到，控制台提供了简单的MC管理功能，但大部分都只是快捷方式，真正可用于操控的就只有四个地方：启动、重启、停止、输入命令。

## 选择代理端

如果我们想要玩家在不同的世界（如创造世界、生存世界）中自由切换，Paper端是一个好主意，但如果希望服务器对生电友好的话，Paper就指望不上了。我们需要群组服。

群组服事实上就是同时运行多台MC服务器，并通过一个代理端（Proxy）进行玩家的分配。主流的代理端有BungeeCord和Velocity，Velocity更新、性能更好（同时也有另外的原因，见[后文](#impulse服务端管理器)），因此我们选择Velocity。在[Paper的官网](https://papermc.io/downloads/velocity '是的，Velocity是Paper团队做的')下载之。

## 部署代理端

非常幸运的是，我在淘宝租赁的面板服支持SFTP，可以方便地上传和管理文件。它还自带了一个网页版的文件管理器，可以直接在浏览器中操作文件。

![面板服文件管理器](/images/PanelServerExplorerBasic.png)

通过文件管理器，我们可以方便地上传Velocity的jar包。将启动参数中的核心名改为上传的文件的文件名。

> 哪怕是面板服不允许改包名也没关系，反过来把上传的文件的文件名改过来就行了。

![面板服启动参数](/images/PanelServerArgument.png)

在面板服自带的防火墙中放行指定端口：

![面板服网络](/images/PanelServerFirewallBasic.png)

好，接下来启动一次代理端，配置文件就自动生成了，我们修改配置文件，就可以开心地在面板服中玩上群组服了！😀

## 或者，事情没这么简单……

你也许发现，子服务器还没有启动呢！真正的服务器（创造服生存服）还需要另外启动，可是我们没有办法启动这些服务器啊？面板服上只有一个启动按钮，我们只能启动一个服务器，启动了代理端就没法启动创造服生存服，启动了创造服就没法启动代理端生存端，这下尴尬了。难道说就没有办法了？

# [Impulse服务端管理器][Impulse插件]

Impulse是一个Velocity插件，以AGPLv3开源于[GitHub](https://github.com/Arson-Club/Impulse)。它可以通过Velocity指令直接控制子服务器的启动和停止，对于我们这种情况再合适不过了！

> 这个插件本意是通过动态启停子服以达到节省服务器资源的目的，不过在我们的场景中，它可以完美地解决面板服无法同时启动多个服务器的问题。

插件有详实的[文档](https://arson-club.github.io/Impulse)，只不过是纯英文，下面简单过一下：

## 0. 配置代理端和子服务器

这个不用说吧，起码先把这几个配置利索了。本示例配置了一个代理端和两个fabric子服务器。

## 1. 安装Impulse

运行过一次Velocity后，在自动生成的`plugins`目录放入[Impulse插件]，再运行一次Velocity，在`plugins`目录中生成`impulse`目录即安装成功。

### 目录结构

你不一定要完全按照和我一样的结构来组织文件，以下是一个示例结构，后续步骤按照此目录结构进行说明：

```text
.
├── plugins
│   ├── impulse
│   │   └── config.yaml
│   └── impulse.jar
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

## 2. 服务端配置示例

```properties
# worlds/creative/server.properties
server-ip=127.0.0.2
server-port=25565
```

```properties
# worlds/survival/server.properties
server-ip=127.0.0.3
server-port=25565
```

```toml
# velocity.toml
[servers]
creative = "127.0.0.2:25565"
survival = "127.0.0.3:25565"
try = [
	"creative",
	"survival",
]
```

> 如果你想，配成`127.0.0.1:25566`与`127.0.0.1:25567`也行。但比起端口号，我更喜欢使用不同的IP地址来区分不同的服务器。[^1]

## 3. 配置Impulse

```yml
# plugins/impulse/config.yaml
servers:
  - jar:
      jarFile: fabric.jar
      javaFlags:
	    - -Xms1G
        - -Xmx2G
      workingDirectory: /home/container/worlds/creative
    lifecycleSettings:
      allowAutoStop: false
    name: creative
    type: jar
  - jar:
      jarFile: fabric.jar
      javaFlags:
	    - -Xms1G
        - -Xmx8G
      workingDirectory: /home/container/worlds/survival
    lifecycleSettings:
      allowAutoStop: false
    name: survival
    type: jar
```

- 任何通过`java -jar xxx.jar`方式运行的端都可以通过Impulse进行管理，如[Nana Limbo Plugin](https://github.com/bivashy/NanoLimboPlugin 'LibreLogin的必需品')。
- `jarFile`、`javaFlags`和`workingDirectory`需要根据实际情况进行修改。
- `allowAutoStop`设置为`false`已禁用自动停服（以节省服务器资源），若你希望在服务器长时间无玩家在线时自动停服，请将其设置为`true`（默认值）。

## 4. 启动服务器

通过面板服自带的控制台启动Velocity后，即可通过命令控制子服务器的启动和停止。

### 启动命令

```sh
/impulse start <server>
# 如：
/impulse start creative
/impulse start survival
```

### 停止命令

```sh
/impulse stop <server>
# 如：
/impulse stop creative
/impulse stop survival
```

### 启停权限

- 启动：`impulse.server.warm` （对，不是`impulse.server.start`）
- 停止：`impulse.server.stop`

# 总结

至此，我们已经成功搭建了一个支持多世界的Minecraft群组服。你可以随心所欲地增减子服务器并增加插件与模组，就像一个真正的全功能MC服务器。

目前还有三个问题：

1. 当Velocity重启时，所有的子服务器也会跟着重启[^2]；
2. 无法方便地访问子服务器的控制台，只能通过RCON访问[^3]；
3. 配置不了数据库，因此像[LuckPerms](https://luckperms.net)就没办法实现多端同步了，每个子服与代理端必须独立设置权限。

这三个问题的复杂度更高，我将在下一期中进行详细探讨。敬请期待！

[^1]: 小知识：127.x.x.x都是本地回环地址，可以任意使用。

[^2]: <https://github.com/Arson-Club/Impulse/issues/75>

[^3]: <https://github.com/Arson-Club/Impulse/issues/79>

[Impulse插件]: https://modrinth.com/plugin/impulse-server-manager
