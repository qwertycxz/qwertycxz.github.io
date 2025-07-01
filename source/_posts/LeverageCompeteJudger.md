---
categories: [编程]
tags: [Leverage]
title: Leverage对抗裁判程序编写指南
date: 2023-07-23 14:22:33
excerpt: 如何为对抗功能编写裁判程序
---

裁判程序可以选用[C](https://www.c-language.org '没想到C语言也有官网吧！')（不推荐）、[C++](https://isocpp.org)、[Java](https://www.java.com/zh-CN)（不推荐）、[JavaScript](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript)、[Python](https://python.org)编写，评测机与裁判程序间使用[JSON](https://json.org/json-zh.html)进行通信。

# 输出格式

裁判程序首先需要输出内容给评测机，然后再接受评测机的输入，一个通常的示例如下：

```json
{
	"command": "request",
	"content": {
		"0": "0 90231849 57933430 77682741",
		"2": "2 90231849 57933430 77682741"
	},
	"display": "[90231849,57933430,77682741]"
}
```

`command`的值只能为`request`或`finish`，前者代表向bot发送信息，后者代表游戏结束。

`content`的值为一个对象（散列表），键为bot的名称（`"0"`、`"1"`、`"2"`、`"3"`等，须为字符串）。

如果`command`的值为`request`，则值的内容（字符串）为指定bot的输入。不强制每回合每名bot都必须接受输入，若代表bot的一项不存在，则不给该bot输入。

如果`command`的值为`finish`，则值的内容为bot的得分（整数），每名bot都必须有得分。

`display`的值为一个字符串，内容可自行指定，将会每回合传给前台的回放程序。

讲解一下上述的例子：本回合游戏正常运行，向`0`号bot发送`0 90231849 57933430 77682741`，向`2`号bot发送`2 90231849 57933430 77682741`，向回放程序发送`[90231849, 57933430, 77682741]`。

再看另一个例子：

```json
{
	"command": "finish",
	"display": "{\"gamer\":0,\"x\":9,\"y\":6,\"winner\":0,\"error\":\"\",\"map\":[[...]]}",
	"content": {
		"0": 1,
		"1": 0
	}
}
```

这条输出意味着游戏已结束，bot0得分为`1`，bot1得分为`0`，向回放程序发送`"{\”gamer\”:0,\”x\”:9,\”y\”:6,\”winner\”:0,\”error\”:\”\”,\”map\”:[[…]]}"`

# 输入格式

在向评测机发送request请求后，裁判程序即应立即等待下一次输入。一个输入的例子：

```json
{
	"0": {
		"verdict": "OK",
		"raw": "4 2"
	},
	"2": {
		"verdict": "OK",
		"raw": "-1 4"
	}
}
```

输出的`content`有哪些bot，输入的内容就有哪些bot的输出。

`verdict`为bot的状态，只有为`"OK"`时其状态才正常，如果不是`"OK"`，请将其判负。可能值包括：`TLE`（超时）、`MLE`（超内存）、`NJ`（评测机未输出）、`RE`（运行时错误）、`CE`（编译错误）、`SE`（评测机错误）、`NR`（bot未输出）

`raw`为bot的输出。

# 通信方法

请注意以上的所有例子都进行了格式化，实际上传输的文本是没有这些空白字符的，例如：

```json
输入> {"0":{"verdict":"OK","raw":"6 9"}}
输出> {"command":"request","content":{"0":"4 4","1":"4 4"},"display":"{\"0\":\"4 4\",\"1\":\"4 4\"}"}
```

## JavaScript 与 Python

这两种语言提供了JSON解析的官方库，请参见[MDN的官方文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/JSON)与[Python的官方文档](https://docs.python.org/zh-cn/3/library/json.html)。

## C++

使用了[nlohmann/json](https://github.com/nlohmann/json)用于JSON解析，下载[最新版的json.hpp](https://github.com/nlohmann/json/releases)，将其放于裁判程序目录下的nlohmann子目录内即可使用。

## C与Java

没有用于 JSON 解析的官方库，也没有预装任何的第三方库，因此强烈不建议使用。

# 示例程序

[戳这里](/pages/LeverageJudgerExample)

# 如何调试裁判程序

可以手动输入JSON进行调试：

```json
输出> {"command":"request","display":"{\"gamer\":1,\"x\":-1,\"y\":-1}","content":{"0":"-1 -1"}}
输入> {"0":{"verdict":"OK","raw":"7 7"}}
输出> {"command":"request","display":"{\"gamer\":0,\"x\":7,\"y\":7}","content":{"1":"7 7"}}
输入> {"1":{"verdict":"OK","raw":"6 6"}}
输出> {"command":"request","display":"{\"gamer\":1,\"x\":6,\"y\":6}","content":{"0":"6 6"}}
输入> {"0":{"verdict":"OK","raw":"7 8"}}
输出> {"command":"request","display":"{\"gamer\":0,\"x\":7,\"y\":8}","content":{"1":"7 8"}}
输入> {"1":{"verdict":"OK","raw":"7 6"}}
```

稍后我也许会上线一个自动化的调试模块，敬请期待！
