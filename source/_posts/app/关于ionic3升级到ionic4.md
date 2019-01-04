---
title: 关于ionic3升级到ionic4
date: 2019-01-04 08:36:24
tags: ionic
---

## 参考资料

```
https://moduscreate.com/blog/upgrading-an-ionic-3-application-to-ionic-4/
https://www.joshmorony.com/my-method-for-upgrading-from-ionic-3-to-ionic-4/
https://www.joshmorony.com/the-ionic-4-migration-survival-guide/
```

## 库导入方面的差异

| ionic3| ionic4 | 备注 |
| ------ | ------ | ------ |
| ionic-angular | @ionic/angular |  |
| import { Observable } from 'rxjs/Observable';| import { Observable } from 'rxjs'; |  |
| import 'rxjs/add/operator/map';<br>import 'rxjs/add/operator/catch';<br>import 'rxjs/add/operator/do';<br>import 'rxjs/add/operator/finally';<br> |无||
|import {IonicPage} from "ionic-angular";|无|可直接去掉IonicPage|
|import {IonicPageModule} from "ionic-angular";|import { RouterModule } from '@angular/router';|ionic4使用RouterModule管理路由规则|
|`import { Network } from '@ionic-native/network';`|`import { Network } from '@ionic-native/network/ngx';`||
|`import { SplashScreen } from "@ionic-native/splash-screen";`|`import { SplashScreen } from "@ionic-native/splash-screen/ngx";`||
|import { StatusBar } from '@ionic-native/status-bar';|import { StatusBar } from '@ionic-native/status-bar/ngx';||
|import { AppMinimize } from '@ionic-native/app-minimize';|import { AppMinimize } from '@ionic-native/app-minimize/ngx';||


| echarts3| echarts4 | 备注 |
| ------ | ------ | ------ |
|`import ECharts from "echarts";`|`import * as ECharts from "echarts";`||

## 一些坑

- 关于*router.navigate* 和 *router.navigateByUrl*的区别

```
https://stackoverflow.com/questions/45025334/how-to-use-router-navigatebyurl-and-router-navigate-in-angular
https://angular.io/api/router/Router
```

- 关于*HttpInterceptor*的使用

ionic4中通过`import { HttpClient } from '@angular/common/http';`的请求才会被拦截器拦截；

通过`import { Http } from '@angular/http';`发起的请求不会触发拦截器。
