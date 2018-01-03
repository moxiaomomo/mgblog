---
title: 'golang并发下载文件'
date: 2018-01-03 17:43:52
tags: golang
---

#### 背景说明
假设有一个分布式文件系统，现需要从该系统中并发下载一部分文件到本地机器。
已知该文件系统的部分节点ip, 以及需要下载的文件fileID列表，并能通过这些信息来拼接下载地址。
其中节点ip列表保存在xx_node.txt， 要下载的fileID保存在xx_fileID.txt中。

<!--more-->
#### 代码示例
```golang
package main

import (
	"bufio"
	"flag"
	"fmt"
	"io"
	"math/rand"
	"net/http"
	"os"
	"time"
)

var (
	clustername = flag.String("clustername", "c1", "download clustername")
)

// 逐行读取文件内容
func ReadLines(fpath string) []string {
	fd, err := os.Open(fpath)
	if err != nil {
		panic(err)
	}
	defer fd.Close()

	var lines []string
	scanner := bufio.NewScanner(fd)
	for scanner.Scan() {
		lines = append(lines, scanner.Text())
	}
	if err := scanner.Err(); err != nil {
		fmt.Fprintln(os.Stderr, err)
	}

	return lines
}

// 实现单个文件的下载
func Download(clustername string, node string, fileID string) string {
	nt := time.Now().Format("2006-01-02 15:04:05")
	fmt.Printf("[%s]To download %s\n", nt, fileID)

	url := fmt.Sprintf("http://%s/file/%s", node, fileID)
	fpath := fmt.Sprintf("/yourpath/download/%s_%s", clustername, fileID)
	newFile, err := os.Create(fpath)
	if err != nil {
		fmt.Println(err.Error())
		return "process failed for " + fileID
	}
	defer newFile.Close()

	client := http.Client{Timeout: 900 * time.Second}
	resp, err := client.Get(url)
	defer resp.Body.Close()

	_, err = io.Copy(newFile, resp.Body)
	if err != nil {
		fmt.Println(err.Error())
	}
	return fileID
}

func main() {
	flag.Parse()
	
	// 从文件中读取节点ip列表
	nodelist := ReadLines(fmt.Sprintf("%s_node.txt", *clustername))
	if len(nodelist) == 0 {
		return
	}
	
	// 从文件中读取待下载的文件ID列表
	fileIDlist := ReadLines(fmt.Sprintf("%s_fileID.txt", *clustername))
	if len(fileIDlist) == 0 {
		return
	}

	ch := make(chan string)
	
	// 每个goroutine处理一个文件的下载
	r := rand.New(rand.NewSource(time.Now().UnixNano()))
	for _, fileID := range fileIDlist {
		node := nodelist[r.Intn(len(nodelist))]
		go func(node, fileID string) {
			ch <- Download(*clustername, node, fileID)
		}(node, fileID)
	}
	
	// 等待每个文件下载的完成，并检查超时
	timeout := time.After(900 * time.Second)
	for idx := 0; idx < len(fileIDlist); idx++ {
		select {
		case res := <-ch:
			nt := time.Now().Format("2006-01-02 15:04:05")
			fmt.Printf("[%s]Finish download %s\n", nt, res)
		case <-timeout:
			fmt.Println("Timeout...")
			break
		}
	}
}
```

#### 小结
下载时没有用到默认的http Client, 并指定了超时时间；
下载文件时调用了系统调用, goroutine会被挂起；
下载文件完成后会唤醒被挂起的goroutine, 该goroutine执行完后面的代码后便退出；
全局超时控制，超时后主线程退出。
