---
categories: [编程]
tags: [算法, 随笔]
title: Matlab实现香农编码
date: 2023-11-13 16:48:23
excerpt: 四行代码解决香农编码
---

我挺好奇为啥网上的香农编码实现都如此的冗长……明明四行就完事了：

```matlab
%shannon - 对信源编香农编码
%	语法
%		shannon(...)
%	输入参数
%		... - 信源的概率分布
%			总和为1的正数
%	示例
%		shannon(0.25, 0.25, 0.2, 0.15, 0.1, 0.05)

%# ok<*NOPRT>
function shannon(varargin)
	probabilities = sort(cell2mat(varargin), 'descend')
	prefix_sum = [0, cumsum(probabilities(1 : end - 1))]
	code_length = -floor(log2(probabilities))
	extractAfter(string(dec2bin(prefix_sum .* 2 .^ code_length))', max(code_length) - code_length)
end
```

四行解决一切= =

本页代码适用[Unlicense](https://unlicense.org)协议。
