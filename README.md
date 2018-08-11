#### WKWebView使用总结

##### 关于wk的两个代理

```
 webView.navigationDelegate = self
 webView.uiDelegate = self
```

##### uidelegate传递与视图相关的事件

```

//MARK: - wkuiDelegate
    func webView(_ webView: WKWebView, createWebViewWith configuration: WKWebViewConfiguration, for navigationAction: WKNavigationAction, windowFeatures: WKWindowFeatures) -> WKWebView? {
        if let isMain = navigationAction.targetFrame?.isMainFrame {
            if !isMain {
                self.webView.load(navigationAction.request)
            }
        }
        return nil
    }
    
    func webView(_ webView: WKWebView, runJavaScriptAlertPanelWithMessage message: String, initiatedByFrame frame: WKFrameInfo, completionHandler: @escaping () -> Void) {
        let alert = UIAlertController.init(title: nil, message: message, preferredStyle: .alert)
        alert.addAction(UIAlertAction.init(title: "确定", style: .default, handler: nil))
        self.present(alert, animated: true, completion: nil)
        completionHandler()
    }
    
    func webView(_ webView: WKWebView, runJavaScriptConfirmPanelWithMessage message: String, initiatedByFrame frame: WKFrameInfo, completionHandler: @escaping (Bool) -> Void) {
        let alert = UIAlertController.init(title: nil, message: message, preferredStyle: .alert)
        alert.addAction(UIAlertAction.init(title: "确定", style: .default, handler: { (action:UIAlertAction) in
            completionHandler(true)
        }))
        alert.addAction(UIAlertAction.init(title: "取消", style: .cancel, handler: { (action:UIAlertAction) in
            completionHandler(false)
        }))
        self.present(alert, animated: true, completion: nil)
    }
```

以上三个方法为uidelegate事件
第一个方法createWebViewWithConfiguration，说一下调用这个方法的原因，比如有一个a表情其target='_blank' 表示希望新开一个页面来打开该链接，而不是用原来的页面。判断是否是需要新的页面可以在

```
func webView(_ webView: WKWebView, decidePolicyFor navigationAction: WKNavigationAction, decisionHandler: @escaping (WKNavigationActionPolicy) -> Void) {
        print("test========:\(navigationAction.request.url!),\(navigationAction.request.allHTTPHeaderFields!.debugDescription)")
        if VCJSLinkHandler.shared().handler(navigationAction.request.url, urlRegular: "") {
            decisionHandler(.cancel)
            return
        }
        decisionHandler(.allow)
    }

```
这个方法navigationAction中的属性：sourceFrame和targetFrame,它们分别代表action的出处和目标，类型是WKFrameInfo,WKFrameInfo有一个mainFrame属性，正是这个属性标记着这个fame是在主frame还是新开一个frame

如果targetFrame的mainFrame属性为NO,表明这个WKNaivationAction将会新开一个页面。这时候就回调用createWebView这个方法。

开发者属性这个方法，返回一个新的WKWebView,让WKNavigationAction在新的webView中打开，如果你没有设置WKUIDelegate，那么WKWebVie什么事情都不会做，也就是点击按钮没反应。

返回的wkWebView不能和原来的WKWebView是同一个，如果返回了原来的webView,将会抛出异常。


一般可以如下处理这个情况。

```

- (WKWebView *)webView:(WKWebView *)webView createWebViewWithConfiguration:(WKWebViewConfiguration *)configuration forNavigationAction:(WKNavigationAction *)navigationAction windowFeatures:(WKWindowFeatures *)windowFeatures
{
WKFrameInfo *frameInfo = navigationAction.targetFrame;
if (![frameInfo isMainFrame]) {
[webView loadRequest:navigationAction.request];
}
return nil;
}

```

在网上还看到另一种方法

```
- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler
{
  if (!navigationAction.targetFrame.isMainFrame) {
      [webView evaluateJavaScript:@"var a = document.getElementsByTagName('a');for(var i=0;i<a.length;i++){a[i].setAttribute('target','');}" completionHandler:nil];
  }
  decisionHandler(WKNavigationActionPolicyAllow);
}

```

还有两个关于uidelegate的方法，看方法名就知道个大概不做过来的解释了。


##### navigationDelegate，处理加载的一系列事件

```
func webView(_ webView: WKWebView, didStartProvisionalNavigation navigation: WKNavigation!) {
        self.startProgress()
    }
    
    func webView(_ webView: WKWebView, decidePolicyFor navigationAction: WKNavigationAction, decisionHandler: @escaping (WKNavigationActionPolicy) -> Void) {
        print("test========:\(navigationAction.request.url!),\(navigationAction.request.allHTTPHeaderFields!.debugDescription)")
        if VCJSLinkHandler.shared().handler(navigationAction.request.url, urlRegular: "") {
            decisionHandler(.cancel)
            return
        }
        decisionHandler(.allow)
    }
    
    func webView(_ webView: WKWebView, decidePolicyFor navigationResponse: WKNavigationResponse, decisionHandler: @escaping (WKNavigationResponsePolicy) -> Void) {
        decisionHandler(.allow)
        self.saveCooike(in: navigationResponse.response, cooikeName: self.requestCookieName())
    }
    
    func webView(_ webView: WKWebView, didReceiveServerRedirectForProvisionalNavigation navigation: WKNavigation!) {
        print("test:重定向")
        self.redirect = true
    }
    
    func webView(_ webView: WKWebView, didFinish navigation: WKNavigation!) {
        self.setDocumentCooike(cooikeName: self.requestCookieName())
    }
```
decidePolicyFor navigationAction 方法决定是否调用某个连接

decidePolicyFor navigationResponse 方法决定是否相应某个事件


#### WKWebView cookie问题

我现在也不是很清楚WKWebView 的 cookie管理机制，只是知道它不会像uiwebView那样帮你带上cookie

##### 第一次情况带不上cookie的问题
可以再请求头上带上本地保存的用户标识，解决第一次无法认证的问题。

##### 后续跳转cookie丢失，ajax请求带不上cookie

decidePolicyFor navigationResponse 方法保存服务器设置的cookie，didFinish中为document.cookie设置cookie

```
func setDocumentCooike(cooikeName:String?) -> Void {
        let jsFuncStr:String =
          """
        function setCookie(name,value,expires)
        {
        var oDate=new Date();
        oDate.setDate(oDate.getDate()+expires);
        document.cookie=name+'='+value+';expires='+oDate+';path=/'
        }
        function getCookie(name)
        {
        var arr = document.cookie.match(new RegExp('(^| )'+name+'=({FNXX==XXFN}*)(;|$)'));
        if(arr != null) return unescape(arr[2]); return null;
        }
        function delCookie(name)
        {
        var exp = new Date();
        exp.setTime(exp.getTime() - 1);
        var cval=getCookie(name);
        if(cval!=null) document.cookie= name + '='+cval+';expires='+exp.toGMTString();
        }
        """
        var JSCookieStr = jsFuncStr
        if let cookies = HTTPCookieStorage.shared.cookies {
            for cookie in cookies {
                if let name = cooikeName {
                    if cookie.name == name {
                        let excuteJS = "setCookie('\(cookie.name)', '\(cookie.value)', 1);"
                        print("save cookie:\(excuteJS)")
                        JSCookieStr.append(excuteJS)
                    }
                } else {
                    let excuteJS = "setCookie('\(cookie.name)', '\(cookie.value)', 1);"
                    print("save cookie:\(excuteJS)")
                    JSCookieStr.append(excuteJS)
                }
            }
        }
        
    }

```

##### 302重定向cookie丢失

先说一下重定向的概念，比如有两个链接 a:http:a.com, b:http:b.com
再请求a的时候，a的响应只有响应投，头里面一个b的链接，这时候浏览器就会去跳转b的地址。因为每一次请求服务器都会生成一个新的cookie给客户端，因为重定向的缘故，导致a链接设置的cookie没有设置成功，因此在到b的时候cookie不是最新的cookie导致服务器认证失败。

我们解决的办法是，不每次生成新的cookie，哈哈，很低级的方案。

也可以由客户端来解决，客户端加载一个本地空的文档，但是地址指向a，因为是本地空文档，会立马调用finsh这个回调，这样就可以设置上去了

```
self.webView loadHtmlString:"" baseUr:"http:a.com"
```
这个方法我是随意写的，大概如此。






