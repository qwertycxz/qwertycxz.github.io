---
categories: [游戏]
tags: [Python, 冰与火之舞]
title: 《冰与火之舞》一键二球转三球工具使用说明
date: 2022-07-26 17:11:50
excerpt: https://www.bilibili.com/video/av898458557
---

# 写在前面

在工坊看到各种把二球转换成三球的铺面，觉得这些铺面真的很水，想能不能批量化生产这种铺面，于是做了一个demo发在B站上，意外发现反响不错，就拓展了一下工具的功能，加了个GUI。本教程就是基于这个GUI版本的工具而成的。

Demo展示视频: <https://www.bilibili.com/video/av898458557>

教程视频: <https://www.bilibili.com/video/av856255918>

# 下载

Github仓库: <https://github.com/qwertycxz/ADOFAI-2PlanetsTo3Planets>

百度网盘镜像: <https://pan.baidu.com/s/1_ercvgt14i49QAIHQqrTaA?pwd=DOFI>

推荐在Github下载，若因网络波动等原因无法上Github且能忍受百度网盘的龟速的话，那么这个镜像也勉强能用。

# 用法

![软件截图](/images/Adofai2PlanetsTo3Planets.png)

1. 单击`OpenFile`，加载关卡文件；
2. 在`Start`和`End`输入框输入转换区间，区间需满足$Min \leqslant Start < End \leqslant Max$；
   - $Min = 1$
   - $Max = 关卡的方块总数量$
3. 在`To`输入框输入要转换成多少球^[多于3球？可以也不可以，懂的都懂。]；
4. 单击`Apply New Interval`以应用这一区间；
5. 若有区间不需要，可以右键删除；
6. 重复2~5，直至满意；
7. 点击`Convert As`导出关卡文件。
