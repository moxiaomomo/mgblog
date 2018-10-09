---
title: Nginx如何配置跨域(多域名)
date: 2018-10-09 09:23:58
tags: nginx
---

假设需要允许来源为`localhost`或`.*.example.com`下所有二级域名的访问，在nginx中只需要类似这样配置即可：
<!--more-->

```nginx
    location / {
		set $match "";
		# 支持http及https
		if ($http_origin ~* 'https?://(localhost|.*\.example\.com)') {
		set $match "true";
		}
		
		if ($match = "true") {
			add_header Access-Control-Allow-Origin "$http_origin";
			add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type';
			add_header Access-Control-Allow-Methods GET,POST,OPTIONS,DELETE;
			add_header Access-Control-Allow-Credentials true;
		}
		# 处理OPTIONS请求
		if ($request_method = 'OPTIONS') {
			return 204;
		}
	}
```

当然通过正则也可以详细指定若干个允许访问的域名来源。
