---
title: '[shell]curl结果获取http header的问题'
date: 2017-10-24 12:17:05
tags: shell
---

在通过curl请求http获取response header时， 发现字符串拼接一个问题。
比如以下程序:
```shell
hadoop@1:~$ ct=$(curl -s -I http://www.baidu.com | grep Content-Type | awk '{print $2}')
hadoop@1:~$ echo $ct
text/html
hadoop@1:~$ echo $ct"_postfix"
_postfixl
```
最后一行输出竟然是: `_postfixl`, 而不是期待的`text/html_postfix`
细看结果应该是`_postfix` 直接覆盖了`text/html` 前面的内容。
http header是以`\r\n` 作为行分割符，与echo机制有什么冲突?
原因未解， 不过找到这样一种替代方法(如test.sh):
<!--more-->

```bash
shopt -s extglob # 识别正则, 用于后面去掉空格

function grepCT()
{
    while IFS=':' read key value; do
        # 去掉"value"内的空格
        value=${value##+([[:space:]])}; value=${value%%+([[:space:]])}
        case "$key" in
            Server) SERVER="$value"
                    ;;
            Content-Type) CT="$value"
                    ;;
            HTTP*) read PROTO STATUS MSG <<< "$key{$value:+:$value}"
                    ;;
         esac
    done < <(curl -sI $1)
    echo $CT
}
ct=$(grepCT http://www.baidu.com)
echo $ct"_postfix"
```
执行shell文件:
```shell
hadoop@1:~$ bash test.sh
text/html_postfix
```
