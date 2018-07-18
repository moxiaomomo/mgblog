---
title: 微服务开发之单点登录
date: 2018-07-18 20:07:05
tags: microservice
---

### 关于CAS

CAS是一种单点登录开源框架，遵循apache2.0协议，代码托管在[github.com/apereo/cas](https://github.com/apereo/cas)上。<br>
而单点登录(SSO, Single Sign On)可简单理解为当用户在一个应用上登录了，其他被授权信任的关联应用不用再登录。<br>
比如在同一个浏览器中登录了天猫，再打开淘宝网站时会自动登录，无需单独输用户名密码或扫二维码。以下简单说明下CAS的部署与测试结果。
<!--more-->
### 本地测试环境

```
jdk8
tomcat8.5
gradle-4.3.1
cas4.2.7
```

### 下载及编译cas

```shell
$wget "https://github.com/apereo/cas/archive/v4.2.7.tar.gz"
$tar -zxf v4.2.7.tar.gz
$cd cas-4.2.7/cas-server-webapp
$gradle build
```

编译完成后可在当前路径的`build/libs/`找到编译打包出来的war文件`cas-server-webapp-4.2.7.war`；将其重命名为`cas.war`后，复制到tomcat的web工作目录`webapps/`下。

### tomcat ssl证书配置

具体流程可参考网上资料，这里暂不展开讨论。配置好ssl证书后，在tomcat目录下的`conf/server.xml`增加类似如下配置：

```xml
    <Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
            maxThreads="150" SSLEnabled="true" scheme="https" secure="true"
            clientAuth="false" sslProtocol="TLS"
            keystoreFile="/Users/apple/Documents/cas/castestkey"
            keystorePass="xxxx"/>
```

### 修改hosts

假设测试环境为本机，我们除了启动一个cas server，还将启动启动两个application server用于单点登录测试。以下为这三个节点的本地host配置：
```
127.0.0.1 sso.cas.com
127.0.0.1 app1.cas.com
127.0.0.1 app2.cas.com
```

### 访问cas服务

tomcat起来后，我们可以这样访问cas服务:`https://sso.cas.com:8443/cas/login`<br>
登录界面如下所示：
![这里写图片描述](https://img-blog.csdn.net/2018071608483387?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21veGlhb21vbW8=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### 编译并运行测试服务

我的测试代码基于这源码项目进行了部分修改:[https://github.com/willwu1984/springboot-cas-shiro](https://github.com/willwu1984/springboot-cas-shiro)。
对项目通过`mvn package`命令完成打包后, 在`》/target`下找到可执行文件`cas-0.0.1-SNAPSHOT.jar`。
启动测试服务：
```shell
$java -jar target/cas-0.0.1-SNAPSHOT.jar
// ...
2018-07-13 15:32:04.542  INFO 72755 --- [           main] s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat started on port(s): 18091 (http)
2018-07-13 15:32:04.547  INFO 72755 --- [           main] cas.Application                          : Started Application in 5.087 seconds (JVM running for 5.65)
```

这时我们浏览器访问 `http://app1.cas.com:18091/` , 会发现页面首先被重定向到cas登录认证页面：

```
https://sso.cas.com:8443/cas/login?service=http://app1.cas.com:18091/shiro-cas
```

因为当前并没真正接入数据库，可输入默认用户名:密码(casuser:Mellon)进行测试, 这时会跳转回到测试服务的主页：
![这里写图片描述](https://img-blog.csdn.net/20180716085032483?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21veGlhb21vbW8=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

这时我们运行另外一个测试应用服务，假设地址为 `http://app2.cas.com:18092/`。
在同一个浏览器中访问该地址时，会发现已经自动完成登录并进入主界面。
