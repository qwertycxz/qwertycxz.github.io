---
categories: [编程]
tags: [PowerShell, Windows, 随笔]
title: PowerShell中的mklink
date: 2024-07-22 00:36:52
excerpt: 在PowerShell中复现cmd的mklink
---

`mklink`不属于PowerShell的一部分，但我们可以通过自定义函数的方式实现。注意符号链接的创建方式与`mklink`有所不同。

```ps1
function mklink {
	param ([switch]$s, [switch]$h, [switch]$j, $link, $target)
	function newItem {
		param ($item_type)
		return New-Item $link -ItemType $item_type -Value $target | Format-Table LinkType, Target
	}
	if ($s) {
		return newItem SymbolicLink
	}
	if ($h) {
		return newItem HardLink
	}
	if ($j) {
		return newItem Junction
	}
}
```

把这段代码添加到`$Profile`（对于Windows自带的PowerShell，可以是`C:\Users\<user>\Documents\WindowsPowerShell\profile.ps1`，文件不存在则创建之）即可。

---

用法：`mklink <[-s] | [-h] | [-j]> <link> <target>`

`-s`：创建[SymbolicLink](https://learn.microsoft.com/zh-cn/windows/win32/fileio/symbolic-links '符号链接')

`-h`：创建[HardLink](https://learn.microsoft.com/zh-cn/windows/win32/fileio/hard-links-and-junctions#hard-links '硬链接')

`-j`：创建[Junction](https://learn.microsoft.com/zh-cn/windows/win32/fileio/hard-links-and-junctions#junctions '这个有中文名吗？')

`link`：链接文件所在的位置

`target`：链接到哪个文件，使用最新版（而非Windows自带的）PowerShell可以在符号链接上使用相对位置

三个选项若全不选，则函数什么也不做。

若选中多于一个选项，则以符号链接、硬链接、Junction的优先级创建链接。

---

[`New-Item`](https://learn.microsoft.com/zh-cn/powershell/module/microsoft.powershell.management/new-item)的语法太冗长，所以我半小时速成了PowerShell的基本语法造了个轮子。

事实证明，还是mingw-w64附赠的busybox自带的ln.exe更好使，秒杀mklink一百条街。
