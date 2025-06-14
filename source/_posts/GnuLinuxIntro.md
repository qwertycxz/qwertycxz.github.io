---
categories: [编程]
tags: [Linux]
title: GNU/Linux入门
date: 2023-05-01 18:47:03
excerpt: 补充一点我自己对Linux入门的想法
---

~~嗯，Linux是一个[内核](https://kernel.org 'Linux内核官网')的名字，只是我们通常把它当作GNU/Linux的简称来说。~~

这篇学长写的[文章][Linux入门]已经很完善了，所以我只是补充一点我自己的想法。

# 选择你的Linux

## 平台

### 虚拟机

这是最廉价、最易得的一种Linux入门方式，即使是把虚拟机玩坏了也没关系。

- [WSL2（WSL）](https://learn.microsoft.com/zh-cn/windows/wsl 'Windows Subsystem for Linux')：微软官方的模拟器，只能模拟GNU/Linux。功能不及Vmware，但是对Hyper-V的兼容很好。
- [VMware](https://www.vmware.com)：主流的模拟器，同时支持模拟多种系统，如Windows、DOS、GNU/Linux等，功能强大。新版本的VMware对Hyper-V的兼容也很不错。
- [Hyper-V](https://learn.microsoft.com/zh-cn/windows-server/virtualization/hyper-v)：功能同样很强大，但Windows家庭版如果想用费的力气不会小。

Leverage^[我在[NUIST](https://www.nuist.edu.cn '南京信息工程大学')中经手的主要项目，一个OJ。]采取Hyper-V虚拟机。

### [WSL1](https://learn.microsoft.com/zh-cn/windows/wsl/compare-versions)

和WSL2不同，WSL1更像是Windows版本的[Wine](https://www.winehq.org)~~（WSL1 is not emulator）~~。WSL1内部并没有Linux内核，只是给出了一些GNU/Linux的API，实际上还是Windows系统。这既是优点也是缺点：WSL1可以无缝在两个系统的文件系统间相互访问，但这压根就不是GNU/Linux系统！

想玩玩还是可以的，但真的要用Linux的话，还是试试别的方案吧。

::: 黑幕

据说在WSL1运行`rm -rf /*`，过会你就会得到一个全新的Windows系统！

:::

### 桌面系统

虽说我们通常把GNU/Linux视为没有图形化界面的、纯服务器侧的类分时操作系统，但还是有一小部分极客乐意在自己的PC上使用GNU/Linux作为自己的主力系统。

当然了，出于使用一些仅供Windows的专业软件[游戏]{.黑幕}的需要，大部分 Linuxer 还是会使用双系统（在同一台电脑上装两个系统）以便使用 Windows。但即使如此，我也不建议使用 GNU/Linux 的桌面系统。没必要，真没必要。

### NAS或云服务器

这种做法是最高端、最有效的GNU/Linux学习方法。毕竟在自己的电脑上部署另一个系统总是没什么动力，而自己的服务器上肯定要用GNU/Linux（跑Windows Server怎么想怎么不得劲）。

缺点显而易见：无论是买一台NAS还是去[阿里云](https://aliyun.com)[腾讯云](https://cloud.tencent.com)[华为云](https://www.huaweicloud.com)或者去国外的云服务器提供商购买ECS使用时长，**都要钱**，还恐怕不是一笔小钱。

## 发行版

发行版是什么：<https://zhuanlan.zhihu.com/p/238122351>

不同的发行版可以自行了解，各个发行版有各个发行版的好~~和不好~~。

我在阿里云的ECS使用了[Alibaba Cloud Linux](https://www.aliyun.com/product/alinux)（说是RHEL/CentOS的一个分支？）

Leverage使用了[Ubuntu](https://cn.ubuntu.com)。

Heng-Client^[Leverage的评测机。]使用了[CentOS](https://centos.org)。

## Console、Terminal与Shell

以上这三个词都可以用来指代命令行（Command Line），但他们的具体关系是什么呢：<https://zhuanlan.zhihu.com/p/405527391>

省流：做个不恰当的比喻，Console是姓、Terminal是名、Shell是号，第一个不能换，第二个很少换，所以若要优化你的命令行体验，从Shell入手就对了。

目前常规的GNU/Linux的发行版自带Shell为[Bash](https://www.gnu.org/software/bash)，Windows自带Shell为[PowerShell](https://learn.microsoft.com/zh-cn/powershell)。主要优点是社区丰富，遇到什么问题直接百度一下你就知道。不过论功能强大性肯定比不过第三方软件了。

Leverage使用了[ZSH](https://www.zsh.org)。

[学长的文章][Linux入门]中推荐了[Xshell](https://blog.oi.al/post/basic-use-of-linux#Xshell_36 '学长的文章')

（顺带一提，是个能用的Console就可以使用SSH，只不过使用更强大的Shell可以提高你的效率罢了）

# 命令行的操作

注意大部分命令行基本不支持鼠标操作，这不能称之为一个缺点：事实上不使用鼠标的命令行效率很高，只是上手的门槛比较高。

|              |                                                            |
| ------------ | ---------------------------------------------------------- |
| Ctrl+C       | 终止当前运行的程序（相当于图形化界面中右上角那个红色的叉） |
| Ctrl+Shift+C | 复制                                                       |
| Ctrl+Shift+V | 粘贴                                                       |
| 上下箭头     | 快速选择自己输入过的指令                                   |

当然并不是所有的Shell都完全不支持鼠标操作，这里以Powershell为例：

|          |                                                                  |
| -------- | ---------------------------------------------------------------- |
| 鼠标左键 | 长按选中文字（无论长按还是短按，都不能改变你光标真正的位置）[^1] |
| 鼠标中键 | 复制                                                             |
| 鼠标右键 | 粘贴                                                             |

# 防止自己把自己的服务器删光

## 尽可能避免使用root用户

GNU/Linux拥有着强大的权限管理系统（Windows也有个组策略，当然和家庭版没关系），但是如果你一直使用root用户，那这些权限管理就等于白做。

勤劳一点，都用GNU/Linux了，难道进行完整的权限划分还很麻烦吗。

## 少用rm

个人建议使用其他的删除方式，例如[trash-cli](https://github.com/andreafrancia/trash-cli/blob/master/README_zh-CN.rst)之类的回收站，或者只用图形化界面进行删除操作。

当然这只是个人的习惯，不喜勿喷。

# vi/vim的重要快捷键（指令）

|     |                              |
| --- | ---------------------------- |
| :w  | 若非只读文件，则保存         |
| :w! | 强制保存                     |
| :q  | 若已保存则退出               |
| :wq | 保存并退出                   |
| :q! | 不保存，直接退出（强制退出） |

: [记住这些，起码不会搁键盘上一顿乱敲还退不出去]{.mark .底表题}

# 其他

前文多次提到的，学长的文章：[Linux入门]

[^1]: 用鼠标移动光标这一功能，更多和Terminal有关。个人试用过，感觉极其蹩脚，还是别用了。

[Linux入门]: https://blog.oi.al/post/basic-use-of-linux
