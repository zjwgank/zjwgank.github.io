---
title: hybrid通信
date: 2021-08-10 21:17:51
tags:
---

### hybrid通信

hybrid是通过JS + Native实现的一种开发方式，那JS与Native之间要如何实现交互呢？传统的原生Native开发，都是需要通过页面控件调用Android或者IOS提供的类方法实现交互功能；而JS要怎么实现呢，例如发起请求，调用相机等。这里就涉及到了JS与Native之间的通信了。

#### Native与JS通信

在进行交互之前，Native首先要知道JS要执行什么操作，在执行完成之后JS要如何知道Native执行成功，这个要怎么实现呢?

##### JS调用Native

- 假跳转请求拦截
- 弹窗拦截(alert/prompt/confirm)
- JS上下文注入(Android-addJavaScriptInterface/IOS-[JavaScriptCore,ScriptMessageHandler])

##### Native响应JS

JS是一个脚本语言，任何一个JS引擎都是可以在任意时机直接执行任意JS代码，Native把想要传输的数据或者消息直接写入JS。

- evaluatingJavaScript:直接注入执行JS代码
- loadUrl: `javascript:expression`
- addUserScript: WKUserScript的webview提供addUserScript，在加载时注入

#### JSInterface

- h5页面
  
```js
<html>
  <body>
    <button onClick="javascript:test.run(text)">测试</button>
  </body>
</html>
```

- Android布局

```java
<LinearLayout tools:context="TestActivity">
 <WebView></WebView>
</LinearLayout>

public class TestActivity extends AppCompatActivity{
  onCreate(){
    webView = findViewById(id) // 获取webview
    webSettings = webView.getSettings() // 获取设置
    webSettings.setJavaScriptEnabled(true) // 开启JS
    webView.addJavaScriptInterface(new JSInterface(),'test') // 注入JS
    webView.loadUrl(html) // 加载h5页面
  }

  public class JSInterface{
    run(){
      Toast.makeText(text) // 调用Android布局元素
      webView.loadUrl('javascript:console.log("----")') // loadUrl在h5页面执行log
    }
  }
}
```

- ios布局

```js
@interface WKWebViewVC()<WKScriptMessageHandler>
@implementation WKWebViewVC

viewDidLoad{
  [super viewDidLoad];
  configuration = [WKWebviewConfiguration init]
  configuration.userContentController = [WKUserContentController init]
  userCC = configuration.userContentController
  [userCC addScriptMessageHandler:self name:@"test"]
}
```

#### JSBridge

JSBridge是Js和Native之间的桥梁，它的核心是构建Native和非Native间的消息通信的双向通道。由于JS是运行在一个单独的JS上下文中(webview的JS引擎或者JSCore)与Native运行环境天然隔离，我们可以将JS与Native的每次交互作为一次RPC通信，由JS作为客户端发起请求，由Native作为服务端接收请求处理并响应，而JSBridge作为每次request。

##### 弹窗拦截

在Android的webview中存在一个WebChromeClient类用来监听一些WebView中的事件，其中存在`onJsPrompt`、`onJsAlert`、`onJsConfirm`三个方法。因此我们可以通过继承重写这些方法来拦截弹窗的数据

```java
android - class myChromeClinet extends WebChromeClient {
  public onJsPrompt(){
    ...
  }
}
ios - webview runJavaScriptTextInputPanelWithPrompt(){
  ...
}
```

#### 假请求拦截

我们知道一个完整的请求包括协议、主机、域名、端口、路径、参数即`url== protocol://host.domain:port/path?query`.那么我们可以通过进行协议约定，正常的URL可以在客户端请求访问，但是符合约定的请求对其进行拦截处理。

```java
android - class myChromeClient extends WebChromeClient{
  public shouldOverrideUrlLoading(){
    ...
  }
}
ios - webview shouldStartLoadWithRequest(){
  ...
}
```

#### 优缺点

- 假请求拦截
  - 版本兼容性好、webview支持度好
  - 丢消息、url长度限制
- 弹窗拦截
  - UIWebView不支持
- JS上下文注入
  - 支持JS同步返回、支持同步传递对象、支持传递JS函数、支持注入任意客户端
  - 获取时机