---
title: CentOS7 中文乱码问题
date: 2017/07/23 1:25:42
---

## 修改/etc/locale.conf

```bash
LANG="en_US.UTF-8"
```

## 修改/etc/sysconfig/i18n

```bash
LANG="zh_CN.UTF-8"
```
<!--more-->

## 修改/etc/vimrc或~/.vimrc,添加
```bash
set fileencodings=utf-8,gb2312,gbk,gb18030  
set termencoding=utf-8  
set fileformats=unix  
set encoding=prc
```

## 刷新配置文件

```bash
source /etc/locale.conf
source /etc/sysconfig/i18n
source ~/.vimrc
source /etc/vimrc
```
