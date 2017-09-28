---
title: '[shell]批量对比两个host上的文件'
date: 2017-09-28 19:09:37
tags: shell
---

假设有一个数据集群, 每个集群的目录/data/下面有很多子目录, 子目录内包含很多文件; <br>
每两个节点的/data/目录下所有文件理论上要保持一致(比如fastdfs的两副本模式)。<br>

现在需要快速的对每两台机器上的/data目录下的文件检测是否全部一致, 那么可以怎么做呢?<br>
一个思路是利用shell实现, 每次将两台机器上的文件全部扫描并排序到一个文件内, 然后拉取到本地上进行对比。<br>

具体示例代码(同时解决了sudo密码验证的问题)如下:

```bash
#!/bin/bash

# 每行两个要用于对比文件的ip
ipPairs=(
192.168.1.59 192.168.1.60
192.168.1.81 192.168.1.86
192.168.1.82 192.168.1.87
)
# ssh用户/ssh端口/sudo密码
username="yourname"
port="yourport"
sudoPasswd="yourpassword"

# 检查列表长度是否为偶数
if [[ ${#ipPairs[@]}%2 -eq 1 ]];
then
    echo "invalid pair in ipPairs."
    exit
fi

for((i=0;i<${#ipPairs[@]};i+=2))
do
    ipOne=${ipPairs[$i]}
    ipTwo=${ipPairs[$i+1]}
    echo "ip1:" $ipOne "ip2:" $ipTwo

/usr/bin/expect <<-EOF
    set timeout 300
    spawn ssh -t -p $port -l ${username} $ipOne {sudo find /data/ -type f | sort > /tmp/${ipOne}.log}
    expect {
        "(yes/no)? " { send "yes\n" }
        "password for ${username}: " { send "${sudoPasswd}\n" }
    }

    spawn ssh -t -p $port -l ${username} $ipTwo {sudo find /data/ -type f | sort > /tmp/${ipTwo}.log}
    expect {
        "(yes/no)? " { send "yes\n" }
        "password for ${username}: " { send "${sudoPasswd}\n" }
    }
interact
expect eof
EOF

    scp -P $port ${username}@$ipOne:/tmp/${ipOne}.log ${ipOne}.log
    if [ `ls -l ${ipOne}.log | awk '{ print $5 }'` -lt  1 ]
    then
        echo "${ipOne}.log is empty."
    fi
    
    scp -P $port ${username}@$ipTwo:/tmp/${ipTwo}.log ${ipTwo}.log
    if [ `ls -l ${ipTwo}.log | awk '{ print $5 }'` -lt  1 ]
    then
        echo "${ipTwo}.log is empty."
    fi

    # 对比sha1
    sha1sum ${ipOne}.log ${ipTwo}.log

    # 查找差集
    comm -1 -3 ${ipOne}.log ${ipTwo}.log > ${ipTwo}_${ipOne}.log
done
```
