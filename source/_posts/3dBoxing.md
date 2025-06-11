---
categories: [Coding]
tags: [Algorithm]
title: 记录一段关于三维装箱问题的算法
date: 2022-07-11 22:53:15
excerpt: 写了段关于三维装箱问题的算法,特此记录一下。
---

本代码基于[组合启发式算法][这篇论文]，利用模拟退火的思想进行结果的拟合。

代码分为四个部分：

1. 判定两个物体是否重叠；
2. 将指定顺序、给定方向（长、宽、高固定）的物体排列进指定长、宽、高的容器内；
3. 模拟退火，获得最佳装箱方案；
4. 输出图像（利用[matplotlib库](https://matplotlib.org '官网')）。

# 重叠判定算法

基本全盘照抄的[CSDN的一段代码](https://blog.csdn.net/weixin_40929065/article/details/113243751)。

主体思想是分别判断两个物体的正视图、侧视图和俯视图是否重叠,若三者有二者甚至三者发生重叠,则这两个物体一定重叠。

```py
if item1[0] + position[0] <= box_length and item1[1] + position[1] <= box_width and item1[2] + position[2] <= box_height: # 先特判一下该物品是否在容器内
	for item2 in putted_items: # 也许有更高效的方法吧,但我暂且就暴力遍历了
		if item2['position'] == position: # 特判两物体重合的情况
			return False
		overlap = 0
		if item2['position'][0] < item1[0] + position[0] and position[0] < item2['position'][0] + item2['volume'][0] and item2['position'][1] < item1[1] + position[1] and position[1] < item2['position'][1] + item2['volume'][1]: # 判定xOy面是否重合
			overlap = 1
		if item2['position'][0] < item1[0] + position[0] and position[0] < item2['position'][0] + item2['volume'][0] and item2['position'][2] < item1[2] + position[2] and position[2] < item2['position'][2] + item2['volume'][2]: # 判定xOz面是否重合
			overlap += 1
		if item2['position'][1] < item1[1] + position[1] and position[1] < item2['position'][1] + item2['volume'][1] and item2['position'][2] < item1[2] + position[2] and position[2] < item2['position'][2] + item2['volume'][2]: # 判定yOz面是否重合
			overlap += 1
		if overlap > 1: # 若多于一面重合,则这两个物体在三维上是相交的
			return False
	return True
```

# 定向装箱算法

主要参考自[这篇论文]。

为简便起见，下称物体的坐标为$(x_0, y_0, z_0)$，物体的长、宽、高为$l, w, h$。

首先设一个可放置点为$(0,0,0)$，之后每成功放置一个物体，删除这个物体的放置点并增加三个可放置点$(x_0 + l, y_0, z_0)(x_0, y_0 + w, z_0)(x_0, y_0, z_0 + h)$。

设两条参考线$rx$、$rz$重合于$x$轴、$z$轴；优先考虑将物体放置于两条参考线内，若无法放置则再尝试拓展参考线，若仍无法放置则放置失败。

## 拓展参考线的四种情况

1. 两条线与坐标轴重合（初始情况）时，判断物体能否放置于坐标原点，若能，拓展两条参考线为$x = l$与$z = h$；
2. x轴可进行拓展时，优先拓展x轴，并拓展$rx$为$x = x_0 + l$；
3. $rx$再怎么拓展也无法放下新物体时，判断物体能否放置于$(0, 0, rz)$，若能，拓展$rz$为$z = z_0 + h$
4. $rz$再怎么拓展也无法放下新物体时，拓展两条参考线为最大值（容器的长与高）。若此时依旧无法放下新物体，则装箱告终。

```py
# 判定物品是否能在参考线内容纳下
def isInnerReference(item):
	global x, y, z
	for position in available_points: # 遍历所有可用点
		if item[0] + position[0] <= x_reference and item[2] + position[2] <= z_reference and isRoomEnough(item, position) == True: # 判定该点是否可用且不与任何参考线重合
			x, y, z = position[0], position[1], position[2]
			return True
# 拓展参考线,并判定物品能否被容纳
def expandReference(item):
	global x_reference, z_reference, x, y, z
	if x_reference == 0 or x_reference == box_length: # 如果这是第一个物品或者x参考线已经最大,那么这个物体就有放不下的可能
		if isRoomEnough(item, (0, 0, z_reference)) == True: # 若z轴上能放物体,就不用大刀阔斧地修改参考线
			x, y, z, x_reference = 0, 0, z_reference, item[0]
			z_reference += item[2]
			return True
		elif z_reference < box_height: # 如果z参考线还能动的话,直接把俩参考线加到最大
			x_reference, z_reference = box_length, box_height
			return isInnerReference(item) or expandReference(item)
	else:
		for position in available_points: # 遍历所有可用点
			if position[0] == x_reference and position[1] == 0 and item[2] + position[2] <= z_reference and isRoomEnough(item, position) == True: # 判定所有在x参考线上的点是否可用且不与z参考线重合
				x, y, z = position[0], position[1], position[2]
				x_reference += item[0]
				return True
		x_reference = box_length
		return isInnerReference(item) or expandReference(item)
# 将指定方向的一批物品以固定顺序放入容器中
def pack(items):
	global available_points, putted_items, x_reference, z_reference
	available_points = {(0, 0, 0)}
	positions = []
	putted_items = []
	sizes = []
	item_volume = x_reference = z_reference = 0
	for item in items: # 顺序遍历所有物体
		if (isInnerReference(item) or expandReference(item)) and isRoomEnough(item, (x, y, z)): # 若一切顺利,放下物体
			putted_items.append({
				'position': (x, y, z),
				'volume': item,
			}) # putted_items同时储存有物体的位置和大小
			available_points.remove((x, y, z))
			available_points.add(((x + item[0], y, z)))
			available_points.add(((x, y + item[1], z)))
			available_points.add(((x, y, z + item[2])))
			positions.append((x, y, z))
			sizes.append(item)
			item_volume += item[0] * item[1] * item[2]
			'''print('有一个长', item[0], '宽', item[1], '高', item[2], '的物品成功放置于', (x, y, z))
		else:
			print('有一个长', item[0], '宽', item[1], '高', item[2], '的物品放置失败')'''
	return item_volume / box_length / box_width / box_height, positions, sizes
```

# 模拟退火算法

还是参考自[这篇论文]。

装箱时，装箱的顺序与物体的方向均会对装箱的填充率产生影响，因此在退火时随机打乱这两个值进行计算。

```py
for i in range(2): # 2次退火
	temperature, area_length = 0.92, items_kind
	while temperature >= 0.01: # 在温度降至足够低之前不断循环
		for j in range(area_length): # 循环次数由邻域长度决定
			new_items = items
			for k, item in enumerate(new_items): # tuple不能直接shuffle,所以要先化成list再shuffle再化回tuple
				l = list(item)
				shuffle(l)
				new_items[k] = tuple(l)
			shuffle(new_items)
			new_filling_rate, new_positions, new_sizes = pack(new_items)
			if new_filling_rate > filling_rate or exp((new_filling_rate - filling_rate) * 10 / temperature) > random(): # 若满足更优解或满足Metropolis准则,则选取该解,这是模拟退火算法的核心
				filling_rate, items, positions, sizes = new_filling_rate, new_items, new_positions, new_sizes
				if new_filling_rate > max_filling_rate: # 记录最优解
					max_filling_rate, best_items, best_positions, best_sizes = new_filling_rate, new_items, new_positions, new_sizes
		area_length += items_kind
		temperature *= 0.92
	print(i + 1, '次退火填充率:', max_filling_rate, '放置数量:', len(best_sizes))
```

# 图像输出算法

参考自[StackOverflow的一篇问答](https://stackoverflow.com/questions/49277753)。

其实我看不懂这段鬼画符写的是啥,就不献丑了。

```py
subplot = figure().add_subplot(projection='3d')
subplot.set_box_aspect((box_length, box_width, box_height))
arr = []
for position, size in zip(positions, sizes): # 其实我完全看不懂这一段……这段代码是从StackOverFlow上偷来的
	a = array([[[0, 1, 0], [0, 0, 0], [1, 0, 0], [1, 1, 0]], [[0, 0, 0], [0, 0, 1], [1, 0, 1], [1, 0, 0]], [[1, 0, 1], [1, 0, 0], [1, 1, 0], [1, 1, 1]], [[0, 0, 1], [0, 0, 0], [0, 1, 0], [0, 1, 1]], [[0, 1, 0], [0, 1, 1], [1, 1, 1], [1, 1, 0]], [[0, 1, 1], [0, 0, 1], [1, 0, 1], [1, 1, 1]]]).astype(float)
	for i in range(3):
		a[:, :, i] *= size[i]
	arr.append(a + array(position))
subplot.add_collection3d(Poly3DCollection(concatenate(arr), edgecolor='k', facecolors=repeat(sample(CSS4_COLORS.keys(), item_number), 6)))
subplot.set_xlim([0, box_length])
subplot.set_ylim([0, box_width])
subplot.set_zlim([0, box_height])
show()
```

## 一些坑点

1. `figure().gca`的参数即将被淘汰，因此改用`figure().add_subplot()`；
2. 要按照容器的长、宽、高设置图像的比例，否则图像会变形。

# 总结

好吧也没啥可总结的，这篇写于15年前的论文确实帮了我很大的忙，本文一半都是我重新复述一遍这篇论文的话，剩下一半是从CSDN与StackOverflow上偷的代码，稍微改吧改吧（配合我奇怪的码风），就是自己的了。

谨此作为一个记录，防止出现那种“写的时候只有我和上帝看得懂，现在只有上帝能看懂”的悲剧。

# 全文代码

```py
from math import exp
from matplotlib.colors import CSS4_COLORS
from matplotlib.pyplot import figure, show
from mpl_toolkits.mplot3d.art3d import Poly3DCollection
from numpy import array, concatenate, repeat
from random import random, sample, shuffle
available_points = {} # 可供放置物体的点
putted_items = [] # 已放置的物体的大小与位置
x_reference, z_reference, box_length, box_width, box_height, x, y, z = 0, 0, 0, 0, 0, 0, 0, 0
# 判定空间是否足够容纳下物品
def isRoomEnough(item1, position):
	if item1[0] + position[0] <= box_length and item1[1] + position[1] <= box_width and item1[2] + position[2] <= box_height: # 先特判一下该物品是否在容器内
		for item2 in putted_items: # 也许有更高效的方法吧,但我暂且就暴力遍历了
			if item2['position'] == position: # 特判两物体重合的情况
				return False
			overlap = 0
			if item2['position'][0] < item1[0] + position[0] and position[0] < item2['position'][0] + item2['volume'][0] and item2['position'][1] < item1[1] + position[1] and position[1] < item2['position'][1] + item2['volume'][1]: # 判定xOy面是否重合
				overlap = 1
			if item2['position'][0] < item1[0] + position[0] and position[0] < item2['position'][0] + item2['volume'][0] and item2['position'][2] < item1[2] + position[2] and position[2] < item2['position'][2] + item2['volume'][2]: # 判定xOz面是否重合
				overlap += 1
			if item2['position'][1] < item1[1] + position[1] and position[1] < item2['position'][1] + item2['volume'][1] and item2['position'][2] < item1[2] + position[2] and position[2] < item2['position'][2] + item2['volume'][2]: # 判定yOz面是否重合
				overlap += 1
			if overlap > 1: # 若多于一面重合,则这两个物体在三维上是相交的
				return False
		return True
# 判定物品是否能在参考线内容纳下
def isInnerReference(item):
	global x, y, z
	for position in available_points: # 遍历所有可用点
		if item[0] + position[0] <= x_reference and item[2] + position[2] <= z_reference and isRoomEnough(item, position) == True: # 判定该点是否可用且不与任何参考线重合
			x, y, z = position[0], position[1], position[2]
			return True
# 拓展参考线,并判定物品能否被容纳
def expandReference(item):
	global x_reference, z_reference, x, y, z
	if x_reference == 0 or x_reference == box_length: # 如果这是第一个物品或者x参考线已经最大,那么这个物体就有放不下的可能
		if isRoomEnough(item, (0, 0, z_reference)) == True: # 若z轴上能放物体,就不用大刀阔斧地修改参考线
			x, y, z, x_reference = 0, 0, z_reference, item[0]
			z_reference += item[2]
			return True
		elif z_reference < box_height: # 如果z参考线还能动的话,直接把俩参考线加到最大
			x_reference, z_reference = box_length, box_height
			return isInnerReference(item) or expandReference(item)
	else:
		for position in available_points: # 遍历所有可用点
			if position[0] == x_reference and position[1] == 0 and item[2] + position[2] <= z_reference and isRoomEnough(item, position) == True: # 判定所有在x参考线上的点是否可用且不与z参考线重合
				x, y, z = position[0], position[1], position[2]
				x_reference += item[0]
				return True
		x_reference = box_length
		return isInnerReference(item) or expandReference(item)
# 将指定方向的一批物品以固定顺序放入容器中
def pack(items):
	global available_points, putted_items, x_reference, z_reference
	available_points = {(0, 0, 0)}
	positions = []
	putted_items = []
	sizes = []
	item_volume = x_reference = z_reference = 0
	for item in items: # 顺序遍历所有物体
		if (isInnerReference(item) or expandReference(item)) and isRoomEnough(item, (x, y, z)): # 若一切顺利,放下物体
			putted_items.append({
				'position': (x, y, z),
				'volume': item,
			}) # putted_items同时储存有物体的位置和大小
			available_points.remove((x, y, z))
			available_points.add(((x + item[0], y, z)))
			available_points.add(((x, y + item[1], z)))
			available_points.add(((x, y, z + item[2])))
			positions.append((x, y, z))
			sizes.append(item)
			item_volume += item[0] * item[1] * item[2]
			'''print('有一个长', item[0], '宽', item[1], '高', item[2], '的物品成功放置于', (x, y, z))
		else:
			print('有一个长', item[0], '宽', item[1], '高', item[2], '的物品放置失败')'''
	return item_volume / box_length / box_width / box_height, positions, sizes
# 读取容器的长、宽、高
def read():
	try: # 如果输错了就再输一遍
		box_length, box_width, box_height = map(int, input('请依次输入箱子的长、宽、高(整数),以空格分隔:').split())
		if box_length > 0 and box_width > 0 and box_height > 0: # 确保输入的是正数
			return box_length, box_width, box_height
		else:
			print('输入了非正数')
	except ValueError:
		print('输入的不是三个数')
	return read()
while True: # main()
	items = [] # 一组顺序固定、方向固定的物体
	items_kind = 0
	box_length, box_width, box_height = read()
	while True: # 读取物体的长、宽、高、个数
		try: # 多组输入
			items_length, items_width, items_height, number = map(int, (input('请依次输入物品的长、宽、高、个数(整数),以空格分隔。每行一组物品。若输入完毕请输入Ctrl+Z并回车:').split()))
			if items_length > 0 and items_width > 0 and items_height > 0 and number > 0: # 确保正数
				item = tuple(sorted((items_length, items_width, items_height), reverse=True))
				for i in range(number): # 放入number个物体
					items.append(item)
				items_kind += 1
			else:
				print('输入了非正数')
		except EOFError:
			break
		except ValueError:
			print('输入的不是四个数')
	items.sort(key=lambda item: item[0] * item[1] * item[2], reverse=True)
	filling_rate, positions, sizes = pack(items)
	print('0 次退火填充率:', filling_rate, '放置数量:', len(sizes))
	best_items, best_positions, best_sizes, max_filling_rate = items, positions, sizes, filling_rate
	for i in range(2): # 2次退火
		temperature, area_length = 0.92, items_kind
		while temperature >= 0.01: # 在温度降至足够低之前不断循环
			for j in range(area_length): # 循环次数由邻域长度决定
				new_items = items
				for k, item in enumerate(new_items): # tuple不能直接shuffle,所以要先化成list再shuffle再化回tuple
					l = list(item)
					shuffle(l)
					new_items[k] = tuple(l)
				shuffle(new_items)
				new_filling_rate, new_positions, new_sizes = pack(new_items)
				if new_filling_rate > filling_rate or exp((new_filling_rate - filling_rate) * 10 / temperature) > random(): # 若满足更优解或满足Metropolis准则,则选取该解,这是模拟退火算法的核心
					filling_rate, items, positions, sizes = new_filling_rate, new_items, new_positions, new_sizes
					if new_filling_rate > max_filling_rate: # 记录最优解
						max_filling_rate, best_items, best_positions, best_sizes = new_filling_rate, new_items, new_positions, new_sizes
			area_length += items_kind
			temperature *= 0.92
		print(i + 1, '次退火填充率:', max_filling_rate, '放置数量:', len(best_sizes))
	item_number = len(best_sizes)
	if item_number > 0: # 若解存在,则画图展示
		subplot = figure().add_subplot(projection='3d')
		subplot.set_box_aspect((box_length, box_width, box_height))
		arr = []
		for position, size in zip(positions, sizes): # 其实我完全看不懂这一段……这段代码是从StackOverFlow上偷来的
			a = array([[[0, 1, 0], [0, 0, 0], [1, 0, 0], [1, 1, 0]], [[0, 0, 0], [0, 0, 1], [1, 0, 1], [1, 0, 0]], [[1, 0, 1], [1, 0, 0], [1, 1, 0], [1, 1, 1]], [[0, 0, 1], [0, 0, 0], [0, 1, 0], [0, 1, 1]], [[0, 1, 0], [0, 1, 1], [1, 1, 1], [1, 1, 0]], [[0, 1, 1], [0, 0, 1], [1, 0, 1], [1, 1, 1]]]).astype(float)
			for i in range(3):
				a[:, :, i] *= size[i]
			arr.append(a + array(position))
		subplot.add_collection3d(Poly3DCollection(concatenate(arr), edgecolor='k', facecolors=repeat(sample(CSS4_COLORS.keys(), item_number), 6)))
		subplot.set_xlim([0, box_length])
		subplot.set_ylim([0, box_width])
		subplot.set_zlim([0, box_height])
		show()
```

[这篇论文]: http://www.jos.org.cn/1000-9825/18/2083.pdf
