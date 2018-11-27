# CN21JSSDK 设计方案

## 下面是 CN21JSSDK 容器的几个重要组成部分：

- **API 管理器主要管理 JS SDK**：JS SDK 中已经提供许多内置的 JS API 暴露给Web前端开发者使用的接口`wx`，即[jok-0.0.1.js](http://gitlab.tech.21cn.com/liub/CN21JSSDK/raw/master/demo/html/jok-0.0.1.js)，是对底层接口_OkJSBridge`的封装，需要Web前端开发自己引入。提供的服务有操控 UI，显示对话框和 Toast，以及使用网络 RCP 服务等，详细信息请查看 《21CN JS-SDK说明文档》。
- **插件管理器主要管理 Plugin**：如果现有的 JS API 无法满足你的业务需求，你也可以选择创造一个新的插件。你只需把原生代码打包在插件中，在管理器里注册该插件，便可在 Javascript 层使用新的 JS API 了
- **JS Bridge 是连接原生层和 JavaScript 的沟通桥梁**：有Native客户端直接注入到WebView，它将 JavaScript 代码转译成能在系统框架运行的字节码，同时也把原生层的数据结构转成 JavaScript 对象使其能在 JavaScript 层处理。



## 代码一些说明

Web与Webview之间的交互采用这个项目（ [OneKitDemo-Android](http://gitlab.tech.21cn.com/liub/CN21JSSDK/tree/master/demo/Android/) , [OneKitDemo-IOS](http://gitlab.tech.21cn.com/liub/CN21JSSDK/tree/master/demo/iOS) ）提供的交互框架，具体原理可查看原文说明。</br>
在Web端的实现，参考[j-ok-demo.html](http://mp.weixin.qq.com/wiki/7/aaa137b55fb2e0456bf8dd9148dd613f.html)的调用方式。



## 几种JS Native相互通信方式的介绍

大家可能看了很多大框架源码，无论是[cordova](http://cordova.axuer.com/)还是[WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge)他们核心的通信方式就都是 **假跳转请求拦截**

但其实JS与Native通信并不止一种方式，还有很多种通信方式，尤为重要的是，不同的通信方式有着不同的特点，有的甚至虽然受限于安卓/苹果平台差异不通用，但独有的优点却是 **假跳转请求拦截** 无法比拟的

## JS 调用 Native 的几种通信方案

- 假跳转的请求拦截
- 弹窗拦截
  - alert()
  - prompt()
  - confirm()
- JS上下文注入
  - 苹果JavaScriptCore注入
  - 安卓addJavascriptInterface注入
  - 苹果scriptMessageHandler注入
- console.log 的方式

## Native 调用 JS 的几种通信方案

JS是一个脚本语言，在设计之初就被设计的任何时候都可以执行一段字符串js代码，换句话说，任何一个js引擎都是可以在任意时机直接执行任意的JS代码，我们可以把任何Native想要传递的消息/数据直接写进JS代码里，这样就能传递给JS了

- evaluatingJavaScript 直接注入执行JS代码

大家在PC上用电脑，用Chrome的时候都知道，可以直接用’javascript:xxxx’来简单的执行一些JS代码，弹个框，这个方法只有安卓可以用，因为iOS必须先将url字符串生成Request再交给webview去load，这种’javascript:xxxx’生成request会失败

- loadUrl 浏览器用’javascript:’+JS代码做跳转地址

WKWebView官方提供了一个Api，可以让WebView在加载页面的时候，自动执行注入一些预先准备好的JS

- WKUserScript WKWebView的addUserScript方法，在加载时机注入



# JS 调用 Native 的几种通信方案

## 假跳转的请求拦截

何谓 **假跳转的请求拦截** 就是由网页发出一条新的跳转请求，跳转的目的地是一个非法的压根就不存在的地址比如

```
//常规的Http地址
https://wenku.baidu.com/xxxx?xx=xx
//假的请求通信地址
wakaka://wahahalalala/action?param=paramobj
```

看我下面写的那条假跳转地址，这么一条什么都不是的扯淡地址，直接放到浏览器里，直接扔到webview里，肯定是妥妥的什么都打不开的，而如果在经过我们改造过的hybrid webview里，进行拦截不进行跳转

url地址分为这么几个部分

- 协议：也就是http/https/file等，上面用了wakaka
- 域名：上面的 wenku.baidu.com 和 wahahalalala
- 路径：上面的 xxxx?或action？
- 参数：上面的 xx=xx或param=paramobj

如果我们构建一条假url

- 用协议与域名当做通信识别
- 用路径当做指令识别
- 用参数当做数据传递

客户端会无差别拦截所有请求，真正的url地址应该照常放过，只有协议域名匹配的url地址才应该被客户端拦截，拦截下来的url不会导致webview继续跳转错误地址，因此无感知，相反拦截下来的url我们可以读取其中路径当做指令，读取其中参数当做数据，从而根据约定调用对应的native原生代码

以上其实是一种 **协议约定** 只要JS侧按着这个约定协议生成假url，native按着约定协议拦截/读取假url，整个流程就能跑通。

完全可以不用按着我写的这种方式约定协议，可以任意另行约定协议比如，协议当做通信识别，域名当做模块识别，路径当做指令识别，参数当做数据传递等等，协议协议，任何一种合理的约定都可以，都可以正常的让JS与Native进行通信

**假跳转的请求拦截-JS发起调用**

JS其实有很多种方式发起假请求，跟发起一个新请求没啥两样，只要按着 **协议约定** 生成假请求地址，正常的发起跳转即可，任何一种方式都可以让客户端拦截住

- A标签跳转

```
//在HTML中写上A标签直接填写假请求地址
<a href="wakaka://wahahalalala/action?param=paramobj">A标签A标签A标签A标签</a>
```

- 原地跳转

```
//在JS中用location.href跳转
location.href = 'wakaka://wahahalalala/action?param=paramobj'
```

- iframe跳转

```
//在JS中创建一个iframe，然后插入dom之中进行跳转
$('body').append('<iframe src="' + 'wakaka://wahahalalala/action?param=paramobj' + '" style="display:none"></iframe>');
```

**假跳转的请求拦截-客户端拦截**

- 安卓的拦截方式 `shouldOverrideUrlLoading`

```
@Override
public boolean shouldOverrideUrlLoading(WebView view, String url) {
    //1 根据url，判断是否是所需要的拦截的调用 判断协议/域名
    if (是){
      //2 取出路径，确认要发起的native调用的指令是什么
      //3 取出参数，拿到JS传过来的数据
      //4 根据指令调用对应的native方法，传递数据
      return true;
    }
    return super.shouldOverrideUrlLoading(view, url);
}
```

- iOS的UIWebView的拦截方式 `webView:shouldStartLoadWithRequest:navigationType:`

```
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType
{
    //1 根据url，判断是否是所需要的拦截的调用 判断协议/域名
    if (是){
      //2 取出路径，确认要发起的native调用的指令是什么
      //3 取出参数，拿到JS传过来的数据
      //4 根据指令调用对应的native方法，传递数据
      return NO;
      //确认拦截，拒绝WebView继续发起请求
    }    
    return YES;
}
```

- iOS的WKWebView的拦截方式 `webView:decidePolicyForNavigationAction:decisionHandler:`

```
- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler {
    //1 根据url，判断是否是所需要的拦截的调用 判断协议/域名
    if (是){
      //2 取出路径，确认要发起的native调用的指令是什么
      //3 取出参数，拿到JS传过来的数据
      //4 根据指令调用对应的native方法，传递数据

      //确认拦截，拒绝WebView继续发起请求
        decisionHandler(WKNavigationActionPolicyCancel);
    }else{
        decisionHandler(WKNavigationActionPolicyAllow);
    }
    return YES;
}
```

## 弹窗拦截

前端可以发起很多种弹窗包含

- alert() 弹出个提示框，只能点确认无回调
- confirm() 弹出个确认框（确认，取消），可以回调
- prompt() 弹出个输入框，让用户输入东西，可以回调

每种弹框都可以由JS发出一串字符串，用于展示在弹框之上，而此字符串恰巧就是可以用来传递数据，我们把所有要传递通讯的信息，都封装进入一个js对象，然后生成字典，最后序列化成json转成字符串

通过任意一种弹框将字符串传递出来，交给客户端就可以进行拦截，从而实现通信

**弹窗拦截 - JS发起调用**

其实alert/confirm/prompt三种弹框使用上没任何区别和差异，这里只取其中一种举例，可以选一个不常用的当做管道进行JS通信，这里用prompt举例

```
var data = {
    action:'xxxx',
    params:'xxxx',
    callback:'xxxx',
};
var jsonData = JSON.stringify([data]);
//发起弹框
prompt(jsonData);
```

**弹窗拦截 - 客户端拦截**

- 安卓的拦截 `onJsPrompt`（其他的两个弹框也有）

```
@Override
public boolean onJsPrompt(WebView view, String url, String message, String defaultValue, JsPromptResult result) {
    //1 根据传来的字符串反解出数据，判断是否是所需要的拦截而非常规H5弹框
    if (是){
      //2 取出指令参数，确认要发起的native调用的指令是什么
      //3 取出数据参数，拿到JS传过来的数据
      //4 根据指令调用对应的native方法，传递数据
      return true;
    }
    return super.onJsPrompt(view, url, message, defaultValue, result);
}
```

- iOS的WKWebView `webView:runJavaScriptTextInputPanelWithPrompt:balbala`

```
- (void)webView:(WKWebView *)webView runJavaScriptTextInputPanelWithPrompt:(NSString *)prompt defaultText:(nullable NSString *)defaultText initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(NSString * _Nullable result))completionHandler{
    //1 根据传来的字符串反解出数据，判断是否是所需要的拦截而非常规H5弹框
    if (是){
        //2 取出指令参数，确认要发起的native调用的指令是什么
        //3 取出数据参数，拿到JS传过来的数据
        //4 根据指令调用对应的native方法，传递数据
        //直接返回JS空字符串
        completionHandler(@"");
    }else{
        //直接返回JS空字符串
        completionHandler(@"");
    }
}
```

- iOS的UIWebView

UIWebView不支持截获任何一种弹框，因此这条路走不通

> 经过好心人提醒，UIWebView也存在一种利用Undocumented API（只是未公开API，但是否处于被禁止的私有API不一定）的方式来拦截弹框。

> 原理是可以自行创建一个categroy，在里面实现一个未出现在任何UIWebView头文件里的delegate，就能拦截弹框了（这个Undocumented的delegate长得和WKWebView的拦截delegate一个样子）

[iOS–UIWebView 屏蔽 alert警告框](http://blog.csdn.net/zhuzhiqiang_zhu/article/details/53336547)

## JS上下文注入

说道JS上下文注入，做iOS的都会了解到iOS7新增的一整个JavaScriptCore这个framework，这个framework被广泛使用在了JSPatch，RN等上面，但这个东西一般用法都是完全脱离于WebView，只有一个JS上下文，这个JS上下文里，没有window对象，没有dom，严格意义上讲这个和我们所关注的依赖WebView的Hybrid框架是有很大差异的，就不在这篇文章里多说了

- 苹果UIWebview JavaScriptCore注入
- 安卓addJavascriptInterface注入
- 苹果WKWebView scriptMessageHandler注入

虽然某种意义上讲上面三种方式，他们都可以被称作JS注入，他们都有一个共同的特点就是，不通过任何`拦截`的办法，而是直接将一个native对象（or函数）注入到JS里面，可以由web的js代码直接调用，直接操作

但这三种注入方式都操作差异还是很大，并且各自的局限性各不相同，我们下面一一说明

### 苹果UIWebview JavaScriptCore注入

UIWebView可以通过KVC的方法，直接拿到整个WebView`当前`所拥有的JS上下文

```
documentView.webView.mainFrame.javaScriptContext
```

拿到了JSContext，一切的使用方式就和直接操作JavaScriptCore没啥区别了，我们可以把任何遵循JSExport协议的对象直接注入JS，让JS能够直接控制和操作

所以在介绍如何JS与Native操作的时候换个顺序，先介绍客户端如何把bridge函数注入到JS，在介绍JS如何使用

**苹果UIWebview JavaScriptCore注入 - 客户端注入**

```
//拿到当前WebView的JS上下文
JSContext *context = [webview valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];
//给这个上下文注入callNativeFunction函数当做JS对象
context[@"callNativeFunction"] = ^( JSValue * data )
{
    //1 解读JS传过来的JSValue  data数据
    //2 取出指令参数，确认要发起的native调用的指令是什么
    //3 取出数据参数，拿到JS传过来的数据
    //4 根据指令调用对应的native方法，传递数据
    //5 此时还可以将客户端的数据同步返回！
}
```

通过上面的方法可以拿到当前WebView的JS上下文`JSContext`，然后就要准备往这个JSContext里面注入准备好的block，而这个准备好的block，负责解读JS传过来的数据，从而分发调用各种native函数指令

TIPS:
这种注入不止可以把block注入，在JS里成为一个JS函数，还可以把字符/数字/字典等数据直接注入到JS全局对象之中，可以让JS访问到Native才能获取的全局对象，甚至还可以注入任何NSObject对象，只要这个NSObject对象遵循`JSExport`OC的协议，相当于JS可以直接调用访问OC的内存对象

**苹果UIWebview JavaScriptCore注入 - JS调用**

```
//准备要传给native的数据，包括指令，数据，回调等
var data = {
    action:'xxxx',
    params:'xxxx',
    callback:'xxxx',
};
//直接使用这个客户端注入的函数
callNativeFunction(data);
```

在没经过客户端注入的时候，直接使用调用callNativeFunction()会报 `callNativeFunction is not defined`这个错误，说明此时JS上下全文全局，是没有这个函数的，调用无效

当执行完客户端注入的时候，此时JS上下文全局global下面，就拥有了这个callNativeFunction的函数对象，就可以正常调用，从而传递数据到Native

## 安卓addJavascriptInterface注入

安卓的WebView有一个接口addJavascriptInterface，可以在loadUrl之前提前准备一个对象，通过这个接口注入给JS上下文，从而让JS能够操作，这个操作方式很类似`苹果UIWebview JavaScriptCore注入`，整个机制也差别不离，但有个很重大的区别，后面在详述优缺点对比的时候，会重点描述

**安卓addJavascriptInterface注入 - 客户端注入**

使用安卓官方的API接口即可，并且可以在loadUrl之前WebView创建之后，即可配置相关注入功能，这个和UIWebView-JSContext的使用差异非常之大，后面会说

```
// 通过addJavascriptInterface()将Java对象映射到JS对象
//参数1：Javascript对象名
//参数2：Java对象名
mWebView.addJavascriptInterface(new AndroidtoJs(), "nativeObject");
```

其中AndroidtoJs这个是一个自定义的安卓对象，他们里面有个函数callFunction，AndroidtoJs这个对象的其他函数方法JS都可以调用

**安卓addJavascriptInterface注入 - JS调用**

刚才注入的js对象叫`nativeObject`，所以JS中可以在全局任意使用

```
nativeObject.callFunction("js调用了android中的hello方法");
```

我不是很熟悉android，以上很多安卓代码都取自 [Android：你要的WebView与 JS 交互方式 都在这里了](http://blog.csdn.net/carson_ho/article/details/64904691)，后面也会纳入参考文献之中

## 苹果WKWebView scriptMessageHandler注入

苹果在开放WKWebView这个性能全方位碾压UIWebView的web组件后，也大幅更改了JS与Native交互的方式，提供了专有的交互API`scriptMessageHandler`

因为这是苹果的API，使用方式搜一下一搜一大堆，我并不详细解释了，直接展示一下代码

**苹果WKWebView scriptMessageHandler注入 - 客户端注入**

```
//配置对象注入
[self.webView.configuration.userContentController addScriptMessageHandler:self name:@"nativeObject"];
//移除对象注入
[self.webView.configuration.userContentController removeScriptMessageHandlerForName:@"nativeObject"];
```

需要说明一下，addScriptMessageHandler就像安卓的addJavascriptInterface一样，可以在WKWebView loadUrl之前即可进行相关配置

但不一样的是，如果当前WebView没用了，需要销毁，需要先移除这个对象注入，否则会造成内存泄漏，WebView和所在VC循环引用，无法销毁。

**苹果WKWebView scriptMessageHandler注入 - JS调用**

刚才注入的js对象叫`nativeObject`，但不像前边两个注入一样，直接注入到JS上下文全局Global对象里，addScriptMessageHandler方法注入的对象被放到了，全局对象下一个Webkit对象下面，想要拿到这个对象需要这样拿

```
window.webkit.messageHandlers.nativeObject
```

并且和之前的两种注入也不同，前两种注入都可以让js任意操作所注入自定义对象的所有方法，而`addScriptMessageHandler`注入其实只给注入对象起了一个名字`nativeObject`，但这个对象的能力是不能任意指定的，只有一个函数`postMessage`，因此JS的调用方式也只能是

```
//准备要传给native的数据，包括指令，数据，回调等
var data = {
    action:'xxxx',
    params:'xxxx',
    callback:'xxxx',
};
//传递给客户端
window.webkit.messageHandlers.nativeObject.postMessage(data);
```

**苹果WKWebView scriptMessageHandler注入 - 客户端接收调用**

前两种注入方式，都是在注入的时候，就指定了对应的接收JS调用的Native函数，但是这次不是，在苹果的API设计里，当JS开始调用后，会调用到指定的iOS的delegate里

```
-(void)userContentController:(WKUserContentController *)userContentController didReceiveScriptMessage:(WKScriptMessage *)message{
    //1 解读JS传过来的JSValue  data数据
    NSDictionary *msgBody = message.body;
    //2 取出指令参数，确认要发起的native调用的指令是什么
    //3 取出数据参数，拿到JS传过来的数据
    //4 根据指令调用对应的native方法，传递数据
}
```

# Native 调用 JS 的几种通信方案

说完了JS调用Native，我们再聊聊Native发起调用JS

## evaluatingJavaScript 执行JS代码

上面也简单说了一下，JS是一个脚本语言，可以在无需编译的情况下，直接输入字符串JS代码，直接运行执行看结果，这也是为什么在Chrome里，在网页运行的时候打开控制台，可以输入各种JS指令的看结果的。

也就是说当Native想要调用JS的时候，可以由Native把需要数据与调用的JS函数，通过字符串拼接成JS代码，交给WebView进行执行

说明一下，Android/iOS-UIWebView/iOS-WKWebView，都支持这种方法，这是目前最广泛运用的方法，甚至可以说，Chrome的DevTools控制台也是用的同样的方式。

假如JS网页里已经有了这么一个函数

```
function calljs(data){
    console.log(JSON.parse(data)) 
    //1 识别客户端传来的数据
    //2 对数据进行分析，从而调用或执行其他逻辑  
}
```

那么客户端此时要调用他需要在客户端用OC拼接字符串，拼出一个js代码，传递的数据用json

```
//不展开了,data是一个字典，把字典序列化
NSString *paramsString = [self _serializeMessageData:data];
NSString* javascriptCommand = [NSString stringWithFormat:@"calljs('%@');", paramsString];
//要求必须在主线程执行JS
if ([[NSThread currentThread] isMainThread]) {
    [self.webView evaluateJavaScript:javascriptCommand completionHandler:nil];
} else {
    __strong typeof(self)strongSelf = self;
    dispatch_sync(dispatch_get_main_queue(), ^{
        [strongSelf.webView evaluateJavaScript:javascriptCommand completionHandler:nil];
    });
}
```

其实我们拼接出来的js只是一行js代码，当然无论多长多复杂的js代码都可以用这个方式让webview执行

```
calljs('{data:xxx,data2:xxx}');
```

TIPS:安卓4.4以上才可以使用evaluatingJavaScript这个API

## loadUrl 执行JS代码

安卓在4.4以前是不能用evaluatingJavaScript这个方法的，因此之前安卓都用的是webview直接loadUrl，但是传入的url并不是一个链接，而是以”javascript:”开头的js代码，从而达到让webview执行js代码的作用

其实这个过程和evaluatingJavaScript没啥差异

还按着刚才举例，假如JS网页里已经有了这么一个函数

```
function calljs(data){
    console.log(JSON.parse(data)) 
    //1 识别客户端传来的数据
    //2 对数据进行分析，从而调用或执行其他逻辑  
}
```

我不太熟悉安卓，就不写安卓的字典数据json序列化的逻辑了

```
mWebView.loadUrl("javascript:callJS(\'{data:xxx,data2:xxx}\')");
```

最终实际上相当于执行了一条js代码

```
calljs('{data:xxx,data2:xxx}');
```

## WKUserScript 执行JS代码

对于iOS的WKWebView，除了evaluatingJavaScript，还有WKUserScript这个方式可以执行JS代码，他们之间是有区别的

- evaluatingJavaScript 是在客户端执行这条代码的时候立刻去执行当条JS代码
- WKUserScript 是预先准备好JS代码，当WKWebView加载Dom的时候，执行当条JS代码

很明显这个虽然是一种通信方式，但并不能随时随地进行通信，并不适合选则作为设计bridge的核心方案。但这里也简单介绍一下

```
//在loadurl之前使用
//time是一个时机参数，可选dom开始加载/dom加载完毕，2个时机进行执行JS
//构建userscript
WKUserScript *script = [[WKUserScript alloc]initWithSource:source injectionTime:time forMainFrameOnly:mainOnly];
WKUserContentController *userController = webView.userContentController;
//配置userscript
[userController addUserScript:script]
```

# 几种通信方式的优缺点对比

说完了JS主动调用Native，也说完了Native主动调用JS，有很多很多的方案我们来聊聊这么些个方案都有哪些局限性，是否值得我们选择

## 假请求的通信拦截的问题 – 当下最不该选择的通信方式

假通信拦截请求这种方式看起来是使用最广泛的，知名的[WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge) 和[cordova](http://cordova.axuer.com/)

为什么这些知名框架选用假请求通信拦截其实有很多原因，但我想说的是，基于眼下设计自己的Hybrid框架，最不应该选择的通信方式就是`假请求通信拦截`

**先说说他为数不多的优点：**

- 版本兼容性好：iOS6及以前只有这唯一的一种方式

cordova的前身是phonegap，随手搜一下大概能知道这个框架有多老，也可以看下WebViewJavascriptBridge，最早第一次提交是在5年前，在没有iOS7的时候，有切只有这唯一的一种通信方式，因此他们都选用了他，但看一眼现在已经iOS11了，再看看iOS6及以下的占有度，呵呵，一到iOS7就有更好的全方位碾压的bridge方式了

- webview支持度好：简单地说框架的开发者容易偷懒

这是所有JS call Native的通信方式里，唯一同时支持安卓webview/苹果UIWebView/苹果WKWebView的一种通信方式，这也就是为什么WebViewJavascriptBridge在即便苹果已经推出了更好的WKWebView并且准备了专属的通信API`messageHandler`的时候，还依然选择继续沿用假请求通信拦截的原因，代码不用重写了，适合写那种兼容iOS7以下的UIWebView，在iOS8以上换WKWebView的代码，但看一眼现在的版本占有度？没有任何意义

**多说两句：**

即便是老项目还在使用UIWebView，要计划升级到WKWebView的时候，既然是升级就应该全面升级到新的WK式通信，做什么妥协和折中方案？

而且最重要的一点，想要做到同时支持多个WebView兼容支持并不需要选择妥协方案，在开发框架的时候完全可以在框架侧解决。想要屏蔽这种webview通信差异，通过在Hybrid框架层设计，抽象统一的调用入口出口，把通信差异在内部消化，这样依然能做到统一对外业务代码流程和清晰的代码逻辑，想要做到代码统一不应该以功能上牺牲和妥协的方面去考虑。

要知道[cordova](http://cordova.axuer.com/)都专门为WKWebView开发了独有的[cordova-plugin-wkwebview](https://www.npmjs.com/package/cordova-plugin-wkwebview)插件来专门适配WKWebView的更优的官方通信API，而不是像[WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge)进行妥协，UI与WK都采取同一种有功能性问题的通信方案

**再说说他最严重的缺点：**

- 丢消息！ 一个通信方案，结果他最大的问题是丢失通信消息！

```
location.href = 'wakaka://wahahalalala/callNativeNslog?param=1111'

location.href = 'wakaka://wahahalalala/callNativeNslog?param=2222'
```

上面是一段JS调用Native的代码，可以靠字面意思猜一下，JS此时的诉求是在同一个运行逻辑内，快速的连续发送出2个通信请求，用客户端本身IDE的log，按顺序打印111，222，那么实际结果是222的通信消息根本收不到，直接会被系统抛弃丢掉。

原因：因为假跳转的请求归根结底是一种模拟跳转，跳转这件事情上webview会有限制，当JS连续发送多条跳转的时候，webview会直接过滤掉后发的跳转请求，因此第二个消息根本收不到，想要收到怎么办？JS里将第二条消息延时一下

```
//发第一条消息
location.href = 'wakaka://wahahalalala/callNativeNslog?param=1111'

//延时发送第二条消息
setTimeout(500,function(){
    location.href = 'wakaka://wahahalalala/callNativeNslog?param=2222'
})
```

这根本治标不治本好么，这种框架设计下决定了JS在任何通信逻辑都得考虑是否这个时间有其他的JS通信代码刚交互过，导致消息丢失？是否页面加载完毕的时候不能同时发送`页面加载完毕`和`其他具体业务需要的Native消息`，是否任何一个AJax网络请求回来后立刻发起的Native消息，都要谨慎考虑与此同时是否有别的SetTimeout也在发Native消息导致冲突？这TM根本是一个天坑，这么设计绝对是客户端开发舒坦省事写bridge框架代码，坑死天天做活动上线的前端同学的。

如果想继续使用假跳转请求，又不想换方案怎么办？前端同学在JS框架层包一层队列，所有JS代码调用消息都先进入队列并不立刻发送，然后前端会周期性比如500毫秒，清空flush一次队列，保证在很快的时间内绝对不会连续发2次假请求通信，这种通信队列的设计不光运用解决丢消息的问题，就连RN根本没丢消息问题的JSCore式的通信，也采用了这种方式，归根结底他能减少通信开销，但是！但是！给假通信请求做队列你将面临第二个根本没法解决的问题

- URL长度限制

假跳转请求归根结底他还是一个跳转，抛给客户端被拦截的时候都已经被封装成一个request了，那么如果url超长了呢？那么这个request里的url的内容还是你想要传递的原内容么？不会丢内容么？尤其是当你采用了队列控制，一次性发送的是多条消息组成的数组数据的时候。

假跳转是现在这个时候最不该使用的通信方式！！！

假跳转是现在这个时候最不该使用的通信方式！！！

假跳转是现在这个时候最不该使用的通信方式！！！

重要的事情说三遍

## 弹窗拦截

这个方式其实没啥不好的，而且confirm还可以用更简单的方式处理callback回调，因为confirm天然是需要返回JS内容的，但callback其实也可以用其他的方式实现，也许更好，因此这里按住不表，第二篇文章会整体聊聊，基于这么多种通信手段，如何设计一个自己的Hybrid框架

- UIWebView不支持，但没事UIWebView有更好的JS上下文注入的方式，JSContext不仅支持直接传递对象无需json序列化，还支持传递function函数给客户端呢(借助隐藏的API也可以支持)
- 安卓一切正常，不会出现丢消息的情况
- WKWebView一切正常，也不会出现丢消息的情况，但其实WKWebView苹果给了更好的API，何不用那个，至少用这个是可以直接传递对象无需进行json序列化的

唯一需要注意的一点，如果你的业务开发中经常希望在前端代码里使用系统alert()/confirm()/prompt()那么，你还是挑一个不常用的进行hook，以免干扰常规业务

> 修订补充优点！
>
> 弹窗拦截也可以支持同步返回！
>
> prompt( ) 拦截在客户端需要执行confirm(data)从而用同步的方式给客户端返回数据到JS

```
//同步JS调用Native  JS这边可以直接写 =  !!!
var nativeNetStatus = nativeObject.getNetStatus();
//异步JS调用Native JS只能这么写
nativeObject.getNetSatus(callback(net){
    console.log(net)
})
```
## console.log 的方式

在 Android 里，js 调用 native 的通讯是通过 console.log 的方式，这个和其他容器实现不一样，其他容器一般通过 prompt 的方式来实现的，但是使用 prompt 的方式，有两个弊端：

使用 prompt 会阻断整个浏览器的进程，如果 native 处理时间一长，会导致页面假死。
prompt 是在 UI 层面上会弹出一个模态窗口，在 native 没有对其进行捕获处理的话，会造成一个问题。一旦这个页面放在非此容器的环境下，就会出现一个很诡异的 prompt 弹窗。在支付宝内，曾经出现过这个问题，天猫页面在支付宝 app 里的时候，由于容器机制不同，页面中 bridge 脚本没有判断环境，导致页面中 js 调用 API 的时候，在页面上出现了 prompt 的模态对话框，严重影响了用户体验，但是如果使用 console.log 的话，就不会出现这个问题。console 的方式避免了不兼容环境的体验问题和同时也避免了页面的假死。



jsbridge 注入的时机，由于业务逻辑要依赖 bridge，所以业务的所有逻辑都会在 bridge ready 之后才会触发，而 bridge js 本身运行是要一定的时间的，因此注入的时机对于性能的影响显得非常的重要。但由于 H5 页面的生命周期和容器的生命周期是相互独立的，因此在 H5 生命周期的哪个阶段注入这段 bridgejs，对于性能的影响就显得异常重要。


现在在支付宝内使用的方式为监听 H5 生活周期的事件，比如说当 Webview 设置 title 结束之后，Android 会放出一个 onReceivedTitle、shouldInterceptRequest 等事件，iOS 会尝试在 webViewDidStartLoad 事件，在监听到这些事件之后，立即注入 bridgejs，让其在 H5 生命周期尽早运行。通过这种方式的注入，经过测试，最早能在页面加载开始后， 50ms 以内就能成功注入 bridgejs。


## JS上下文注入

JS上下文注入其实一共3种情况，这3种情况每个情况都不同，我会一一进行优缺点说明

### UIWebView的JSContext注入

说实话这是我觉得最完美的一种交互方式了，苹果在iOS7开放了JavaScriptCore这个框架，支撑起了RN，Weex这么牛逼的摆脱了WebView的深度混合框架，他的能力是最完美的。

**牛逼的优点:**

- 支持JS同步返回！

要知道我们看到的所有JS通信框架设计的都是异步返回，包括RN（这有设计原因，但不代表JSC不支持同步返回），都是设计了一套callback机制，一条通信消息到达Native后，如果需要返回数据，需要调用这个callback接口由Native反向通知JS，他们在JS侧写代码可是差异非常非常非常之大的！

```
//同步JS调用Native  JS这边可以直接写=  !!!
var nativeNetStatus = nativeObject.getNetStatus();

//异步JS调用Native JS只能这么写
nativeObject.getNetSatus(callback(net){
    console.log(net)
})
```

- 支持直接传递对象，无需通过字符串序列化

一个JS对象在JS代码中如果想通过假跳转/弹窗拦截等方式，那么必须把JS对象搞成json，然后才能传递给端，端拿到后还要反解成字典对象，然后才能识别，但是JS上下文注入不需要（其实他本质上是框架层帮你做了这件事情，就是JSValue这个iOS类的能力）

- 支持传递JS函数，客户端能够直接快速调用callback

在JS里如果是一个function，可以直接当做参数发送给客户端，在客户端得到一个JSValue，可以通过JSValue的callWithParmas的方式直接当做函数去调用

- 支持直接注入任意客户端类，客户端对象，JS可以直接向调用客户端

JavaScriptCore有一种使用方法，是可以让任意iOS对象，遵循`<JSExport>`协议，就可以直接把一整个Native对象直接注入，让JS可以直接操作这个对象，读取这个对象的属性，调用这个对象的方法

**有点尴尬的缺点：**

- only UIWebView

这一点简直是最大的遗憾，只有UIWebView可以用KVC取到JSContext，取到了JSContext才能发挥JavaScriptCore的牛逼能力，但是如果为了更好的性能升级到了WKWebView，那就得忍痛，我依稀记得曾几何时我在哪看到过通过私有API，让WKWebView也能获取JSContext，但我找不到了，希望知道的同学能给我点指引。但我有一个看法 **为了WKWebView的性能提升，舍弃JSContext的优点，值得！**

- JSContext获取时机

UIWebView的JSContext是通过iOS的kvc方法拿到，而非UIWebView的直接接口API，因此UIWebView-JSContext注入使用上要非常注意注入时机

- UIWebView-JSContext 在loadUrl之前注入无效
- UIWebView-JSContext 在FinishLoad之后注入有效但有延迟

因为WebView每次加载一个新地址都会启用一个新的JSContext，在loadUrl之前注入，会因为旧的JSContext已被舍弃导致注入无效，若在WebView触发FinishLoad事件的时候注入，又会导致在FinishLoad之前执行的JS代码，是无法调用native通信的

曾经写过一篇文章[UIWebView代码注入时机与姿势](http://awhisper.github.io/2017/09/09/injectUIWebView/)，可以参考看看，有私有API解决办法，不在这里多言

如果你还在使用UIWebView，真的应该彻底丢弃什么假跳转，直接使用这个方案（iOS7.0现在已经不是门槛了吧），并且深度开发JavaScriptCore这么多牛逼优势所带来的一些黑科技（我感觉会在第三篇文章里提这个）

如果你还在使用UIWebView，就用JSContext吧！不要犹豫!

如果你还在使用UIWebView，就用JSContext吧！不要犹豫!

如果你还在使用UIWebView，就用JSContext吧！不要犹豫!

### 安卓的addJavascriptInterface注入

**我不太了解安卓，因此这粗略写一写，此处如果有错误非常希望大家帮我指出**

安卓的addJavascriptInterface注入，其实原理机制几乎和UIWebView的JSContext注入一样，所以UIWebView的JSContext注入的有点他其实都有

- 可以同步返回
- 无需json化透传数据
- 可以传递函数（不确定）
- 可以注入Native对象

但是安卓的addJavascriptInterface没有注入时机这个缺点（类比-UIWebView的JSContext获取时机），原因是UIWebView缺失一个时机由内核通知外围，当前JSContext刚刚创建完毕，还未开始执行相关JS，导致在iOS下无法在这个最应该进行注入的时机进行注入，除非通过私有API，但安卓没事，安卓系统提供了个API来让外围获得这个最佳时机 `onResourceloaded`，详细说明见 [UIWebView代码注入时机与姿势](http://awhisper.github.io/2017/09/09/injectUIWebView/)

### WKWebView的scriptMessageHandler注入

苹果iOS8之后官方抓们推出的新一代webview，号称全面优化，性能大幅度提升，是和safari一样的web内核引擎，带着光环出生，而scriptMessageHandler正是这个新WKWebView钦点的交互API

**优点:**

- 无需json化传递数据

是的，`webkit.messageHandlers.xxx.postMessage()`是支持直接传递json数据，无需前端客户端字符串处理的

- 不会丢消息

我们团队的以前老代码在丢消息上吃了无数的大亏，导致我对这个事情耿耿于怀，怨念极深！真是坑了好几代前端开发，叫苦不堪

**缺点：**

- 版本要求iOS8

我们舍弃了，不是问题

- 不支持JSContext那样的同步返回

丧失了很多黑科技黑玩法的想象力！但我觉得还是有可能有办法哪怕用私有API的方式想办法找回来的，希望知道的朋友提供更多信息

如果你已经上了WKWebView，就用它，不需要考虑

如果你已经上了WKWebView，就用它，不需要考虑

如果你已经上了WKWebView，就用它，不需要考虑

## evaluatingJavaScript 直接执行JS代码

说完了JS主动调用Native，我们再说说Native主动调用JS，evaluatingJavaScript是一个非常非常通用普遍的方式了，原因也在介绍里解释过，js的脚本引擎天然支持，直接扔字符串进去，当做js代码开始执行

也没啥优缺点可以说的，除了有个特性需要在介绍WKUserScript的时候在多解释一下

安卓/UIWebView/WKWebView都支持

## loadUrl 跳转javascript地址执行JS代码

具体的使用方式不详细介绍了，直说一个优点

- 版本支持

在安卓4.4以前是没有evaluatingJavaScript API的，因此通过他来执行JS代码，但本质上和evaluatingJavaScript区别不大

## WKUserScript 执行JS代码

这里要特别说明一下WKUserScript并不适合当做Hybrid Bridge的通信手段，原因是这种Native主动调用JS，只能在WebView加载时期发起，并不能在任意时刻发起通信

WKUserScript不能采用在Hybrid设计里当做通信手段

WKUserScript不能采用在Hybrid设计里当做通信手段

WKUserScript不能采用在Hybrid设计里当做通信手段

但WKUserScript却有一点值得说一下，上文也提到的UIWebView的注入时机，如果你想在恰当时机让JS上下文执行一段JS代码，在UIWebView你是找不到一个合适的加载时机的，除非你动用私有API，但WKWebView解决了这个问题，在构造WKUserScript的时候可以选择dom load start的时候执行JS，也可以选择在dom load end的时候执行JS。但这个有点其实与设计Hybrid框架的核心通信方案，关系不大，但预加载JS预加载CSS也是一个Hybrid框架的扩展功能，后面第二篇会介绍的。

# 横向对比

如果我们要自主设计一个Hybrid框架，通信方案到底该如何取舍？

## JS主动调用Native的方案

| 通信方案           | 版本支持          | 丢消息 | 支持同步返回 | 传递对象 | 注入原生对象 | 数据长度限制 |
| ------------------ | ----------------- | ------ | ------------ | -------- | ------------ | ------------ |
| 假跳转             | 全版本全平台      | 会丢失 | 不支持       | 不支持   | 不支持       | 有限制       |
| 弹窗拦截           | UIWebView不支持   | 不丢失 | 支持         | 不支持   | 不支持       | 无限制       |
| JSContext注入      | 只有UIWebView支持 | 不丢失 | 支持         | 支持     | 支持         | 无限制       |
| 安卓interface注入  | 安卓全版本        | 不丢失 | 支持         | 支持     | 支持         | 无限制       |
| MessageHandler注入 | 只有WKWebView支持 | 不丢失 | 不支持       | 不支持   | 不支持       | 无限制       |

## Native主动调用JS的方案

- iOS: evaluatingJavaScript
- 安卓: 其实2个区别不大，使用方法差异也不大
  - 4.4以上 evaluatingJavaScript
  - 4.4以下 loadUrl

这样对比优缺点，再根据自己项目需要支持的版本号，可以比较方便的选择合适的通信方案，进一步亲自设计一个Hybrid框架