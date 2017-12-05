---
title: "使用parallel-ssh批量执行远程shell命令"
date: 2017-12-05 11:42:00
tags: shell
---

### 使用parallel-ssh批量执行远程shell命令

#### pssh使用场景
假设现在需要对数百台服务器节点进行配置更新或者执行一些简短command，而目前并没有完备的部署工具软件， 那可以选择向pssh这样的并行登录远程终端并执行指定命令的shell工具。
以前机器节点少的时候，直接用shell写个for循环来执行命令，也没什么问题。当节点数量多了之后，一个shell命令可能要消耗几秒， 这时才能感受到pssh这种并行方式的好处，省时省力。
<!--more-->
#### pssh可选配置参数列表
除了pssh，当需要传递登录密码时，可以用到sshpass命令：
```shell
pintai@MG:~/bak$ pssh --help
Usage: pssh [OPTIONS] command [...]

Options:
  --version             show program's version number and exit
  --help                show this help message and exit
  -h HOST_FILE, --hosts=HOST_FILE
                        hosts file (each line "[user@]host[:port]")
  -H HOST_STRING, --host=HOST_STRING
                        additional host entries ("[user@]host[:port]")
  -l USER, --user=USER  username (OPTIONAL)
  -p PAR, --par=PAR     max number of parallel threads (OPTIONAL)
  -o OUTDIR, --outdir=OUTDIR
                        output directory for stdout files (OPTIONAL)
  -e ERRDIR, --errdir=ERRDIR
                        output directory for stderr files (OPTIONAL)
  -t TIMEOUT, --timeout=TIMEOUT
                        timeout (secs) (0 = no timeout) per host (OPTIONAL)
  -O OPTION, --option=OPTION
                        SSH option (OPTIONAL)
  -v, --verbose         turn on warning and diagnostic messages (OPTIONAL)
  -A, --askpass         Ask for a password (OPTIONAL)
  -x ARGS, --extra-args=ARGS
                        Extra command-line arguments, with processing for
                        spaces, quotes, and backslashes
  -X ARG, --extra-arg=ARG
                        Extra command-line argument
  -i, --inline          inline aggregated output and error for each server
  --inline-stdout       inline standard output for each server
  -I, --send-input      read from standard input and send as input to ssh
  -P, --print           print output as we get it

Example: pssh -h hosts.txt -l irb2 -o /tmp/foo uptime

pintai@MG:~/bak$ sshpass --help
sshpass: invalid option -- '-'
Usage: sshpass [-f|-d|-p|-e] [-hV] command parameters
   -f filename   Take password to use from file
   -d number     Use number as file descriptor for getting password
   -p password   Provide password as argument (security unwise)
   -e            Password is passed as env-var "SSHPASS"
   With no parameters - password will be taken from stdin

   -h            Show help (this screen)
   -V            Print version information
At most one of -f, -d, -p or -e should be used
```


#### 1.使用sshpass传递登录密码
先把需要远程登录的host集中写到一个文件， 比如叫hostlist，写入host列表:
```
192.168.1.11:22
192.168.1.12:22
192.168.1.13:22
```
然后将ssh登录密码写到另一个文件， 比如叫remotepass， 写入密码:
```
yourpassword
```
最后执行相关命令， 直接打印每个节点的输出内容：
```shell
sshpass -f remotepass pssh -h hostlist -l yourloginname -A -i "hostname"
```
输出结果如下
```
Warning: do not enter your password if anyone else has superuser
privileges or access to your account.
[1] 11:23:21 [SUCCESS] 192.168.1.11:22
test1.hostname
[2] 11:23:21 [SUCCESS] 192.168.1.12:22
test2.hostname
[3] 11:23:21 [SUCCESS] 192.168.1.13:22
test3.hostname
```
#### 2.将结果输出到指定文件

如果需要将输出结果收集起来，那么可以通过*-o*选项来指定结果输出目录，比如：
```shell
sshpass -f remotepass pssh -h hostlist -l yourloginname -o outputdir -A "hostname"
```
执行时终端输出:
```
Warning: do not enter your password if anyone else has superuser
privileges or access to your account.
[1] 11:23:21 [SUCCESS] 192.168.1.11:22
[2] 11:23:21 [SUCCESS] 192.168.1.12:22
[3] 11:23:21 [SUCCESS] 192.168.1.13:22
```
而当前目录会生成*outputdir*目录，目录中每个host占一个文件，如：
```
pintai@MG:~/bak$ ls output/
192.168.1.11:22 192.168.1.12:22 192.168.1.13:22
pintai@MG:~/bak$ cat output/*
test1.hostname
test2.hostname
test3.hostname
```

#### 3.  执行sudo命令
有些shell命令可能需要通过sudo权限来执行，一般来说本地可以这么执行
```
echo your_sudo_pass | sudo -S your_command
```
而在pssh中可以这么做：
```shell
sshpass -f remotepass pssh -h hostlist -l yourloginname -o outputdir -A "echo your_sudo_pass | sudo netstat -antup | grep xxx"
```
执行完毕后，具体输出结果可以在outputdir目录下查找。
