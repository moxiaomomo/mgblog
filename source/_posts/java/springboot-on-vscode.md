---
title: 'VS Code下创建SpringBoot应用'
date: 2018-06-23 23:36:36
tags: ["java","springboot"]
---

## 在vscode中安装相关扩展包

主要选择了`Java Extension Pack`,`Spring Boot Extension Pack`这两个扩展包，如图：
![](/img/vscdep.png)

<!--more-->
## 创建spring boot项目

`Ctrl + Shift + P`打开命令选项板，输入`Spring Initializr`开始生成Maven或Gradle项目。这里以Gradle为例，其中在选择依赖包一步，可以选择web和devTools这两个，创建好后打开对应工作区，可以看到初始化的项目结构是这样子的(HelloController.java是后来所建)：
![](/img/vscstruct.png)

## 创建一个controller类

这里创建了`HelloController`, 用以注册请求url及对应的处理方法，主要代码如下：

```java
package com.example.demo.controller;  
  
import org.springframework.web.bind.annotation.RequestMapping;  
import org.springframework.web.bind.annotation.RestController;  
  
@RestController
public class HelloController {  

    @RequestMapping("/hello")  
    String greeting() {  
        return "Hello my friend!";
    }
}
```

## 通过gradlew直接启动服务

在终端环境中，cd到项目根目录下，可直接通过 `chmod +x gradlew && ./gradlew bootRun` 来启动springboot程序：

```shell
$./gradlew bootRun
Starting a Gradle Daemon, 1 busy Daemon could not be reused, use --status for details
> Task :bootRun22:53:25.501 [main] DEBUG org.springframework.boot.devtools.settings.DevToolsSettings - Included patterns for restart : []
22:53:25.505 [main] DEBUG org.springframework.boot.devtools.settings.DevToolsSettings - Excluded patterns for restart : [/sprin
g-boot-actuator/target/classes/, /spring-boot-devtools/target/classes/, /spring-boot/target/classes/, /spring-boot-starter-[\w-
]+/, /spring-boot-autoconfigure/target/classes/, /spring-boot-starter/target/classes/]
22:53:25.505 [main] DEBUG org.springframework.boot.devtools.restart.ChangeableUrls - Matching URLs for reloading : [file:/data/
apps/javaproc/secondprocj/build/classes/java/main/, file:/data/apps/javaproc/secondprocj/build/resources/main/]
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.0.3.RELEASE)

2018-06-23 22:53:25.993  INFO 37795 --- [  restartedMain] com.example.demo.DemoApplication         : Starting DemoApplication on AppledeMBP.lan with PID 37795 (/data/apps/javaproc/secondprocj/build/classes/java/main started by apple in /data/apps/javaproc/secondprocj)
// ... 这里忽略部分日志
2018-06-23 22:53:29.863  INFO 37795 --- [  restartedMain] com.example.demo.DemoApplication         : Started DemoApplication in 4.334 seconds (JVM running for 5.009)
2018-06-23 22:53:41.140  INFO 37795 --- [nio-8080-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring Framew
orkServlet 'dispatcherServlet'
2018-06-23 22:53:41.140  INFO 37795 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : FrameworkServlet 'dispatcherServlet': initialization started
2018-06-23 22:53:41.165  INFO 37795 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : FrameworkServlet 'dispatcherServlet': initialization completed in 24 ms
<=========----> 75% EXECUTING [3m 50s]
> :bootRun
```

服务起来后， 我们去打开浏览器，输入 `http://localhost:8080/hello`，将会得到如下响应结果：
![](/img/vscsbshow.png)

### 打包成可执行jar文件

当然我们可以把项目源码编译成可执行jar包，再部署到对应的位置并启动:

```shell
$./gradlew build
Starting a Gradle Daemon (subsequent builds will be faster)

> Task :test
2018-06-23 23:30:54.215  INFO 39578 --- [       Thread-6] o.s.w.c.s.GenericWebApplicationContext   : Closing org.springframewor
k.web.context.support.GenericWebApplicationContext@3dfafd95: startup date [Sat Jun 23 23:30:49 CST 2018]; root of context hierarchy


BUILD SUCCESSFUL in 23s
5 actionable tasks: 4 executed, 1 up-to-date
```
这时候我们可以看到在`./build/libs/`下会生成对应的jar包，我们将它拷到某个位置后，再启动测试一下:

```shell
$java -jar demo-0.0.1-SNAPSHOT.jar

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.0.3.RELEASE)

2018-06-23 23:33:24.134  INFO 39652 --- [           main] com.example.demo.DemoApplication         : Starting DemoApplication on AppledeMBP.lan with PID 39652 (/data/apps/demo-0.0.1-SNAPSHOT.jar started by apple in /data/apps)
// ... 这里忽略部分日志
2018-06-23 23:33:28.515  INFO 39652 --- [           main] com.example.demo.DemoApplication         : Started DemoApplication in 5.352 seconds (JVM running for 6.077)
```
在浏览器中重新刷新`http://localhost:8080/hello`， 同样可以得到预期的响应内容。
