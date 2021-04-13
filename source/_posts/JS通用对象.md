---
title: JS通用对象
date: 2021-04-12 22:53:24
tags:
---

### history

`History`对象存储在`Window`对象的`history`属性中,包含用户在浏览器中访问过的URL信息.

- length: 返回历史列表中的URL数量
- `history.back()`,加载`histroy`列表中的前一个`url`
- `history.forward()`,加载`history`列表中的下一个`url`
- `history.go()`,加载`history`中某个具体的`url`
- `history.pushState()`,添加一条历史记录不刷新界面
- `history.replaceState()`,替换当前历史记录,不刷新界面

`window.onpopstate`历史记录发生改变时触发,但是`pushState、replaceState`不会触发这个事件
`window.onhashchang`当页面`hash`更改时触发


### location

`Location`对象存储在`Window`对象的`Location`属性中,包含当前窗口文档的URL信息.

- href: 返回完整的文档URL.`location.toString() === location.href`
- protocol: 返回当前URL的协议,即`//`前
- host: 返回当前文档URL的主机名和端口
  - hostname: 返回当前文档URL的主机
  - port: 返回当前文档URL的端口
- pathname: 返回URL的路径名
- search: 返回URL的查询部分,即`?`后的部分
- hash: 返回URL的锚点,即`#`后的部分
- `location.assign()`加载新的文档
- `location.reload()`重新加载当前文档 === 刷新
- `location.replace()`用新的文档替换当前的文档

```js
http://www.test.com:8080/api/v3/getInfo?param=1
// protocol: http:
// host: www.test.com:8080
// hostname: www.test.com
// port: 8080
// pathname: /api/v3/getInfo
// search: param=1
http://www.test.com/static/view/index.html#/dashboard
// hash: /dashboard
```

### navigator

`Navigator`对象存储在`window`对象的`navigator`属性中,包含了用户代理的状态和标示信息.

- activeVRDisplays:`navigator.activeVRDisplays`筛选所有的`VRDisplay`对象并以一个数组返回`ispresenting === true`的对象
- appCodeName:`navigator.appCodeName`返回当前浏览器的内部"开发代号"名称
- appName:`navigator.appName`返回当前浏览器的官方名称
- appVersion:`navigator.appVersion`返回当前浏览器的版本
- clipboard:`navigator.clipboard`返回设备的剪切板信息
- connection:`navigator.connection`返回当前设备的网络状态信息
- cookieEnabled:`navigator.cookieEnabled`cookie是否在浏览器中可用
- geolocation:`navigator.geolocation`返回设备的地理信息
- javaEnabled:`navigator.javaEnabled`当前设备是否支持JAVA
- keyboard:`navigator.keyboard`提供当前设备的键盘的访问
- language:`navigator.language`浏览器用户界面的语言(优先级最高)
- languages:`navigator.languages`浏览器中用户已知语言数组,按照优先级排列
- onLine:`navigator.onLine`浏览器是否联网
- oscpu:`navigator.oscpu`返回当前操作系统名
- permissions:`navigator.permissions`返回一个permission对象,用于查询和更新API的权限状态
- platform:`navigator.platform`返回浏览器平台名
- plugins:`navigator.plugins`返回浏览器中安装的插件
- userAgent:`navigator.userAgent`返回当前浏览器的用户代理
- product:`navigator.product === Gecko`
- serviceWorker:`navigator.serviceWorker`返回ServiceWorkerContainer对象用于提供注册、删除、更新以及serviceWorker之间的通信

### screen

`Screen`对象存储在`window`对象的`screen`属性中,包含客户端显示屏幕的相关信息

- height:`screen.height`返回屏幕的总高度
- width:`screen.width`返回屏幕的总宽度
- availHeight:`screen.availHeight`返回屏幕可用高度
- availWidth:`screen.availWidth`返回屏幕可用宽度
- availLeft:`screen.availLeft`距左
- availTop:`screen.availTop`距上
- colorDepth:`screen.colorDepth`返回目标设备或缓冲器上的调色板的比特深度
- pixelDepth:`screen.pixelDepath`返回屏幕的颜色分辨率