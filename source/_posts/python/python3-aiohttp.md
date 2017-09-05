---
title: '[python3]使用aiohttp'
date: 2017-09-05 18:10:02
tags: python
---

python3下asyncio可实现异步并发处理IO逻辑, 而aiohttp可实现异步请求http逻辑,<br>
因此可结合asyncio及aiohttp库实现异步并发请求http的需求,
直接上示例代码:
<!--more-->

```python
#! /usr/bin/python
# -*- coding:utf-8 -*-

import aiohttp
import asyncio
import async_timeout


async def test(url):
    async with aiohttp.ClientSession() as session:
         with async_timeout.timeout(10):
            async with session.post(url) as resp:
                r = await resp.text()
                print(r)
                print(resp.headers.get('Content-Type'))
                # TODO: do something...


def test_main():
    tasks = []
    tasks.append(
        asyncio.ensure_future(
            test("http://hostname1.com/xxx")
        )
    )
    tasks.append(
        asyncio.ensure_future(
            test("http://hostname2.com/xxx")
        )
    )
    loop.run_until_complete(asyncio.gather(*tasks))


if __name__ == "__main__":
    loop = asyncio.get_event_loop()
    test_main()
    loop.close()
```
