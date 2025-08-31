---
layout: page
---

# 前言

这是我使用Python的[AIOHTTP](https://docs.aiohttp.org)编写的简易客户端。这是一个进阶HTTP客户端框架，若你希望了解本工具的技术说明，[请见下](#逐步讲解)。

代码中包含四个`<按需替换>`，你需要抓包获取鉴权信息`pterodactyl_session`、将域名改为你面板服的域名、并将服务器UUID替换为你的服务器UUID。

# 代码

请先`pip install aiohttp`

```py
#!/usr/bin/env python
from asyncio import gather, get_event_loop, run
from typing import TypedDict
from aiohttp import ClientSession, ClientWebSocketResponse

async def transmit(ws: ClientWebSocketResponse):
	loop = get_event_loop()
	while not ws.closed:
		await ws.send_json({
			'args': [
				await loop.run_in_executor(None, input),
			],
			'event': 'send command',
		})

class Response(TypedDict):
	args: list[str]
	event: str
	extra: str

async def receive(ws: ClientWebSocketResponse):
	async for response in ws:
		data: Response = response.json()
		if data['event'] == 'console output':
			print(' '.join(data['args']))

class Handshake(TypedDict):
	socket: str
	token: str

async def main():
	async with ClientSession(cookies = {
		'pterodactyl_session': '<按需替换>',
	}, headers = {
		'Origin': 'https://<按需替换>'
	}) as session:
		data: Handshake
		async with session.get('https://<按需替换>/api/client/servers/<按需替换>/websocket') as request:
			data = (await request.json())['data']
		async with session.ws_connect(data['socket']) as ws:
			await ws.send_json({
				'args': [
					data['token'],
				],
				'event': 'auth',
			})
			await gather(receive(ws), transmit(ws))

if __name__ == '__main__':
	run(main())
```

# 逐步讲解

若要理解这段代码，你需要拥有异步编程和网络编程的基础知识。

1. 首先运行`run(main())`，异步地执行`main()`函数。
2. 在`main()`函数中，我们设置`cookie`与`Origin`，进行HTTP的鉴权。
3. 向`/api/client/servers/<按需替换>/websocket`发送HTTP请求，也就是握手。
4. 建立WebSocket连接。
5. 将`token`发送到WebSocket服务器进行认证。
6. 同时执行发送命令的函数`transmit`和接收输出的函数`receive`（异步就是为了这个）。
7. `receive`函数循环等待服务器返回的消息，并将其打印到控制台。因为是异步等待，所以不会阻塞主线程。
8. `transmit`函数调用[事件循环](https://docs.python.org/zh-cn/3/library/asyncio-eventloop.html)，等待用户输入，并将其发送到服务器。
