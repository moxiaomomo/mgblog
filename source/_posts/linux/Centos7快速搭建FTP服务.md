---
title: Centos7快速搭建FTP服务
date: 2018-08-16 10:02:57
tags: Centos, DevOps
---


## 通过`yum`安装vsftpd

```shell
yum install -y vsftpd
```

## 修改配置文件`/etc/vsftpd/vsftpd.conf`

增加了一些自定义配置，全部配置详细如下：

<!--more-->

```
# 原有初始配置
local_umask=022
dirmessage_enable=YES
xferlog_enable=YES
connect_from_port_20=YES
xferlog_std_format=YES
tcp_wrappers=YES
local_enable=YES
write_enable=YES
pam_service_name=vsftpd

# 不支持匿名访问
anonymous_enable=NO

# 所有用户都被限制在其主目录下
chroot_local_user=YES
chroot_list_enable=NO
allow_writeable_chroot=YES

# 支持IPv4及IPv6, 监听端口8021
listen=NO
listen_ipv6=YES
listen_port=8021

# 只允许userlist_file文件中的用户可访问ftp
userlist_enable=YES
userlist_deny=NO
userlist_file=/etc/vsftpd/user_list

# ftp用户主目录
local_root=/data/ftp

# passive模式，数据端口范围自定义(6000-6010)，要确保这些端口已开放给外网访问
pasv_enable=YES
pasv_min_port=6000
pasv_max_port=6010
```

## 配置允许登录的用户`/etc/vsftpd/user_list`

```
# vsftpd userlist
# If userlist_deny=NO, only allow users in this file
# If userlist_deny=YES (default), never allow users in this file, and
# do not even prompt for a password.
# Note that the default vsftpd pam config also checks /etc/vsftpd/ftpusers
# for users that are denied.
# 这里为允许登录的用户名，一行一个
ftpUser
```

## 创建ftp登录用户

```
groupadd ftpGroup
useradd -d /opt/reconciliation -s /sbin/nologin -g ftpGroup -G root ftpUser
passwd ftpUser
```

## 创建ftp文件存放目录

```
mkdir -p /data/ftp
chown -R ftpUser /data/ftp
```

## 启动ftp服务

```
service vsftpd start
```

## ftp访问测试

使用filezilla工具进行连接测试，正常情况如下所示：

![这里写图片描述](https://img-blog.csdn.net/20180816094528643?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21veGlhb21vbW8=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
