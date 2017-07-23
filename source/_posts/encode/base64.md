---
title: python之encode/decode
date: 2017/7/23 12:46:25
---

## python之encode/decode

```python
# coding: utf-8

import base64

to_enc = 'sampledata'
enc_res = base64.b64encode(to_enc)
print enc_res

dec_res = base64.b64decode(enc_res)
print dec_res
```
运行效果如下:

```bash
root@XIAOMO:/tmp# python27 b64.py
c2FtcGxlZGF0YQ==
sampledata
```
