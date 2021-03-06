---
title: '[golang]简单文件上传服务'
date: 2017-09-07 18:51:21
tags: golang
---

利用net/http库及gorilla/mux库实现了一个简单的文件上传服务,<br>
示例如下:
<!--more-->

```golang
package main

import (
	"fmt"
	"github.com/gorilla/mux"
	"io"
	"net/http"
	"os"
)

const uploadHTML = `
<html>  
  <head>  
    <title>选择文件</title>
  </head>  
  <body>  
    <form enctype="multipart/form-data" action="/" method="post">  
      <input type="file" name="uploadfile" />  
      <input type="submit" value="上传文件" />  
    </form>  
  </body>  
</html>`

const destLocalPath = "/data/files/"

func index(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte(uploadHTML))
}

func upload(w http.ResponseWriter, r *http.Request) {
	if r.Method == "GET" {
		index(w, r)
		return
	}

	r.ParseMultipartForm(32 << 20) // max memory is set to 32MB
	clientfd, handler, err := r.FormFile("uploadfile")
	if err != nil {
		fmt.Println(err)
		w.Write([]byte("upload failed."))
		return
	}
	defer clientfd.Close()

	localpath := fmt.Sprintf("%s%s", destLocalPath, handler.Filename)
	localfd, err := os.OpenFile(localpath, os.O_WRONLY|os.O_CREATE, 0666)
	if err != nil {
		fmt.Println(err)
		w.Write([]byte("upload failed."))
		return
	}
	defer localfd.Close()

	io.Copy(localfd, clientfd)
	w.Write([]byte("upload finish."))
}

func newRouter() http.Handler {
	hdl := mux.NewRouter()
	hdl.HandleFunc("/", upload)

	return hdl
}

func main() {
	http.ListenAndServe(":8877", newRouter())
}
```

假如需要在接收文件的时候计算文件hash值， 应该如何做呢？<br>
根据io.TeeReader库，可以在文件上传过程中自动计算hash值, 完整代码修改为:

```golang
package main

import (
	"crypto/sha1"
	"encoding/hex"
	"fmt"
	"github.com/gorilla/mux"
	"io"
	"net/http"
	"os"
)

const uploadHTML = `
<html>  
  <head>  
    <title>选择文件</title>
  </head>  
  <body>  
    <form enctype="multipart/form-data" action="/" method="post">  
      <input type="file" name="uploadfile" />  
      <input type="submit" value="上传文件" />  
    </form>  
  </body>  
</html>`

const destLocalPath = "/data/files/"

func index(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte(uploadHTML))
}

func upload(w http.ResponseWriter, r *http.Request) {
	if r.Method == "GET" {
		index(w, r)
		return
	}

	r.ParseMultipartForm(32 << 20) // max memory is set to 32MB
	clientfd, handler, err := r.FormFile("uploadfile")
	if err != nil {
		fmt.Println(err)
		w.Write([]byte("upload failed."))
		return
	}
	defer clientfd.Close()

	localpath := fmt.Sprintf("%s%s", destLocalPath, handler.Filename)
	localfd, err := os.OpenFile(localpath, os.O_WRONLY|os.O_CREATE, 0666)
	if err != nil {
		fmt.Println(err)
		w.Write([]byte("upload failed."))
		return
	}
	defer localfd.Close()

	// 利用io.TeeReader在读取文件内容时计算hash值
	fhash := sha1.New()
	io.Copy(localfd, io.TeeReader(clientfd, fhash))
	hstr := hex.EncodeToString(fhash.Sum(nil))
	w.Write([]byte(fmt.Sprintf("upload finish:%s", hstr)))
}

func newRouter() http.Handler {
	hdl := mux.NewRouter()
	hdl.HandleFunc("/", upload)

	return hdl
}

func main() {
	http.ListenAndServe(":8877", newRouter())
}
```
