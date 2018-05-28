---
title: 'shell curl 与 python requests'
date: 2018-04-19 13:48:45
tags: python
---

### shell curl 与 python requests

偶然发现了curl和requests库的一个区别。场景是这样的--

这样使用curl去发起post请求：
```
curl -v -X POST http://api.xx.com/api/yy.php --data 'params={"sign":"xxxx","data":[{"uid":110,"remark":"just4test"}]}'
```

后端接口服务能正常解析params参数并响应结果。但是当我改用python去请求时：

```python
import json
import requests

data = {"sign":"xxxx","data":[{"uid":110,"remark":"just4test"}]}
reqbody = ('params=%s'%json.dumps(data))
res = requests.post(
    'http://api.xx.com/api/yy.php',
    data=reqbody)
```
接口无法解析params并返回错误。
<!--more-->
后来认真对比了一下headers和body，发现有可能是header设置不同的原因。
仔细查看，curl中的header有一行是：
```
Content-Type: application/x-www-form-urlencoded
```
但在python requests中并没有对应的设置，因此加上了这一个header，其后请求能正常返回。

```
import json
import requests

headers = {'Content-Type': 'application/x-www-form-urlencoded'}
data = {"sign":"xxxx","data":[{"uid":110,"remark":"just4test"}]}
reqbody = ('params=%s'%json.dumps(data))
res = requests.post(
    'http://api.xx.com/api/yy.php',
    data=reqbody，
    headers=headers)
```
