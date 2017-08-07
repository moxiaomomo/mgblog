
---
title: '[mysql]python3并发访问分布式mysql表'
date: 2017-08-06 15:22:43
tags: mysql
---

### 场景说明
假设有一个mysql表被水平切分，分散到多个host中，每个host拥有n个切分表。
如果需要并发去访问这些表，快速得到查询结果， 应该怎么做呢？
这里提供一种方案，利用python3的asyncio异步io库及aiomysql异步库去实现这个需求。
<!--more-->

### 代码演示
```python
import logging
import random
import asyncio
from aiomysql import create_pool

# 假设mysql表分散在8个host, 每个host有16张子表
TBLES = {
    "192.168.1.01": "table_000-015", # 000-015表示该ip下的表明从table_000一直连续到table_015
    "192.168.1.02": "table_016-031",
    "192.168.1.03": "table_032-047",
    "192.168.1.04": "table_048-063",
    "192.168.1.05": "table_064-079",
    "192.168.1.06": "table_080-095",
    "192.168.1.07": "table_096-0111",
    "192.168.1.08": "table_112-0127",
}
USER = "xxx"
PASSWD = "xxxx"

# wrapper函数，用于捕捉异常
def query_wrapper(func):
    async def wrapper(*args, **kwargs):
        try:
            await func(*args, **kwargs)
        except Exception as e:
            print(e)
    return wrapper


# 实际的sql访问处理函数，通过aiomysql实现异步非阻塞请求
@query_wrapper
async def query_do_something(ip, db, table):
    async with create_pool(host=ip, db=db, user=USER, password=PASSWD) as pool:
        async with pool.get() as conn:
            async with conn.cursor() as cur:
                sql = ("select xxx from {} where xxxx")
                await cur.execute(sql.format(table))
                res = await cur.fetchall()
                # then do something...


# 生成sql访问队列, 队列的每个元素包含要对某个表进行访问的函数及参数
def gen_tasks():
    tasks = []
    for ip, tbls in TBLES.items():
        cols = re.split('_|-', tbls)
        tblpre = "_".join(cols[:-2])
        min_num = int(cols[-2])
        max_num = int(cols[-1])
        for num in range(min_num, max_num+1):
            tasks.append(
               (query_do_something, ip, 'your_dbname', '{}_{}'.format(tblpre, num))
            )

    random.shuffle(tasks)
    return tasks

# 按批量运行sql访问请求队列
def run_tasks(tasks, batch_len):
    try:
        for idx in range(0, len(tasks), batch_len):
            batch_tasks = tasks[idx:idx+batch_len]
            logging.info("current batch, start_idx:%s len:%s" % (idx, len(batch_tasks)))
            for i in range(0, len(batch_tasks)):
                l = batch_tasks[i]
                batch_tasks[i] = asyncio.ensure_future(
                    l[0](*l[1:])
                )
            loop.run_until_complete(asyncio.gather(*batch_tasks))
    except Exception as e:
        logging.warn(e)

# main方法, 通过asyncio实现函数异步调用
def main():
    loop = asyncio.get_event_loop()

    tasks = gen_tasks()
    batch_len = len(TBLES.keys()) * 5   # all up to you
    run_tasks(tasks, batch_len)

    loop.close()
```