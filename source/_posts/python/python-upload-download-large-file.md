---
title: python upload/download large file
date: 2017-08-19 22:39:53
tags: python
---


对于大文件上传下载，可以通过requests库来流式处理，示例如下:

```python

import requests

def upload(url, local_fpath):
    headers = {
        'content-type': 'application/octet-stream',
        # ...
    }
    with open(local_fpath, 'rb') as f:
        r = requests.post(url, headers=headers, data=f)
        # deal with r.headers.xxx

def download(url, local_fpath):
    headers = {
        # ...
    }
    r = requests.get(url, stream=True, headers=headers)
    with open(local_fpath, "wb") as fd:
        for chunk in r.iter_content(chunk_size=10240):
            if chunk:
                fd.write(chunk)
```
