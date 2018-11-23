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
这个方法navigationAction中的属性：sourceFrame和targetFrame,它们分别代表action的出处和目标，类型是WKFrameInfo,WKFrameInfo有一个mainFrame属性，这个属性标识这个fame是在主frame还是新开一个frame

如果targetFrame的mainFrame属性为NO,表明这个WKNaivationAction将会新开一个页面。这时候就回调用createWebView这个方法。

开发者实现这个方法，返回一个新的WKWebView,让WKNavigationAction在新的webView中打开，如果你没有设置WKUIDelegate，那么WKWebVie什么事情都不会做，也就是点击按钮没反应。

返回的wkWebView不能和原来的WKWebView是同一个，如果返回了原来的webView,将会抛出异常。


一般可以如下处理这个情况。

```

- (WKWebView *)webView:(WKWebView *)webView createWebViewWithConfiguration:(WKWebViewConfiguration *)configuration forNavigationAction:(WKNavigationAction *)navigationAction windowFeatures:(WKWindowFeatures *)windowFeatures
{
if navigationAction.targetFrame == nil {
            self.urlString = navigationAction.request.url?.absoluteString
            self.loadUrl()
        } else {
            if let isMain = navigationAction.targetFrame?.isMainFrame {
                if !isMain {
                    self.urlString = navigationAction.request.url?.absoluteString
                    self.loadUrl()
                }
            }
        }
        return nil
}

```

在网上还看到另一种方法

```
- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler {
  
  	[webView evaluateJavaScript:@"var a = document.getElementsByTagName('a');for(var i=0;i<a.length;i++){a[i].setAttribute('target','');}" completionHandler:nil];
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
        self.webView.evaluateJavaScript(JSCookieStr, completionHandler: nil)
    }

```

##### 302重定向cookie丢失

先说一下重定向的概念，比如有两个链接 a:http:a.com, b:http:b.com
再请求a的时候，a的响应只有响应投，头里面一个b的链接，这时候浏览器就会去跳转b的地址。因为每一次请求服务器都会生成一个新的cookie给客户端，因为重定向的缘故，导致a链接设置的cookie没有设置成功，因此在到b的时候cookie不是最新的cookie导致服务器认证失败。

我们解决的办法是，不每次生成新的cookie，哈哈，很低级的方案。

也可以由客户端来解决，客户端加载一个本地空的文档，但是地址指向a，因为是本地空文档，会立马调用finsh这个回调，这样就可以设置上去了

```
self.webView loadHtmlString:"" baseUrl:"http:a.com"
```
这个方法我是随意写的，大概如此。


#### js与native交互

以WK为例，交互舍弃了UIWeb基于JSContext这一套机制，JSContext方案给用户的权限太大，很容易被注入js对象，而WK通过WKUserContentController这个对象完成与js交互。

##### js to native
WKUserContentController必须进行注册，注册过程需要传入一个遵循WKScriptMessageHandler协议的对象，与方法名。

```
  let bridge = WKUserContentController.init()
  js.add(WKScriptMessageHandler, name: "test")
      
```
WKUserContentController 会对遵循协议的对象进行强持有

```
js.removeScriptMessageHandler(forName: "test")
```

###### natvie to js
因为是原生调用js限制没有js调原生大，可以注入js代码并调用（如上面设置cookie的），也可以直接调用js有的函数

```
self.webView.evaluateJavaScript(JSCookieStr, completionHandler: nil)
      
```

____

#### WebViewJavascriptBridge 原理介绍
WebViewJavascriptBridge 面向上层的类为WebViewJavascriptBridge和WKWebViewJavascriptBridge，分别对应UIWebView和WKWebView,WebViewJavascriptBridgeBase为native核心处理逻辑，为两个上层类服务。WebViewJavascriptBridge_JS为js端处理逻辑。

##### js端到native端实现分析

##### js端代码为

```
function callHandler(handlerName, data, responseCallback) {
		if (arguments.length == 2 && typeof data == 'function') {
			responseCallback = data;
			data = null;
		}
		_doSend({ handlerName:handlerName, data:data }, responseCallback);
	}
	
	function _doSend(message, responseCallback) {
		if (responseCallback) {
			var callbackId = 'cb_'+(uniqueId++)+'_'+new Date().getTime();
			responseCallbacks[callbackId] = responseCallback;
			message['callbackId'] = callbackId;
		}
		sendMessageQueue.push(message);
		messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
	}
	
```

##### native 进行对应方法的注册，来处理js端的调用

```
- (void)registerHandler:(NSString *)handlerName handler:(WVJBHandler)handler {
    _base.messageHandlers[handlerName] = [handler copy];
}
```

js端调用callhandler的时候，会把handlerName和data保存到sendMessageQueue这个数组中，如果有回调，生成一个callback一并同前两个信息保存到sendMessageQueue中，通过messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;通知native端，js发起了原生调用请求。


##### native收到js端调用


```
- (void)WKFlushMessageQueue {
    [_webView evaluateJavaScript:[_base webViewJavascriptFetchQueyCommand] completionHandler:^(NSString* result, NSError* error) {
        if (error != nil) {
            NSLog(@"WebViewJavascriptBridge: WARNING: Error when trying to fetch data from WKWebView: %@", error);
        }
        [_base flushMessageQueue:result];
    }];
}
```

改方法会拉取保存在js端sendMessageQueue中的信息，信息通过flushMessageQueue方法进行处理

```
- (void)flushMessageQueue:(NSString *)messageQueueString{
    if (messageQueueString == nil || messageQueueString.length == 0) {
        NSLog(@"WebViewJavascriptBridge: WARNING: ObjC got nil while fetching the message queue JSON from webview. This can happen if the WebViewJavascriptBridge JS is not currently present in the webview, e.g if the webview just loaded a new page.");
        return;
    }

    id messages = [self _deserializeMessageJSON:messageQueueString];
    for (WVJBMessage* message in messages) {
        if (![message isKindOfClass:[WVJBMessage class]]) {
            NSLog(@"WebViewJavascriptBridge: WARNING: Invalid %@ received: %@", [message class], message);
            continue;
        }
        [self _log:@"RCVD" json:message];
        
        NSString* responseId = message[@"responseId"];
        if (responseId) {
            WVJBResponseCallback responseCallback = _responseCallbacks[responseId];
            responseCallback(message[@"responseData"]);
            [self.responseCallbacks removeObjectForKey:responseId];
        } else {
            WVJBResponseCallback responseCallback = NULL;
            NSString* callbackId = message[@"callbackId"];
            if (callbackId) {
                responseCallback = ^(id responseData) {
                    if (responseData == nil) {
                        responseData = [NSNull null];
                    }
                    
                    WVJBMessage* msg = @{ @"responseId":callbackId, @"responseData":responseData };
                    [self _queueMessage:msg];
                };
            } else {
                responseCallback = ^(id ignoreResponseData) {
                    // Do nothing
                };
            }
            
            WVJBHandler handler = self.messageHandlers[message[@"handlerName"]];
            
            if (!handler) {
                NSLog(@"WVJBNoHandlerException, No handler for message from JS: %@", message);
                continue;
            }
            
            handler(message[@"data"], responseCallback);
        }
    }
}

```
上述方法分处理native调用js，js给的回调，和js调用native两部分

先分析处理js调用native部分，也就是上述else部分
else部分也分两步来看待，处理回调和调用native，关于调用native部分
```
 WVJBHandler handler = self.messageHandlers[message[@"handlerName"]];
 ```
 message中有本次js端调用的方法名，获取方法名从messageHandlers这个字典中获取对应原生注册来处理该方法名的hander

回调部分，从message中获取callbackId,如果获取到了则说明js端还希望收到原生端处理后的通知，


```
responseCallback = ^(id responseData) {
        if (responseData == nil) {
            responseData = [NSNull null];
        }
        WVJBMessage* msg = @{ @"responseId":callbackId, @"responseData":responseData };
        [self _queueMessage:msg];
};
                
```

```
WVJBMessage* msg = @{ @"responseId":callbackId, @"responseData":responseData };
```
携带了js端的callbackId，和原生传过来的信息。通过下面方法调用js端的方法，这个在后面会在native到原生详细介绍
```
[self _queueMessage:msg];
```

没有callbackId说明js端不需要本次调用的回调,直接如下处理
```
responseCallback = ^(id ignoreResponseData) {
                    // Do nothing
                };
```

如果存在responseId:说明该方法是native调用js后，js处理完给的回调。通过responseId找到保存在responseCallbacks中的callback。


##### native调用js

逻辑和js调用native差不多，原生携带本次调用的handlerName,data（如果是回调部分则是responseId，data）,调用js端的_dispatchMessageFromObjC 函数

```

function _dispatchMessageFromObjC(messageJSON) {
		if (dispatchMessagesWithTimeoutSafety) {
			setTimeout(_doDispatchMessageFromObjC);
		} else {
			 _doDispatchMessageFromObjC();
		}
		
		function _doDispatchMessageFromObjC() {
			var message = JSON.parse(messageJSON);
			var messageHandler;
			var responseCallback;

			if (message.responseId) {
				responseCallback = responseCallbacks[message.responseId];
				if (!responseCallback) {
					return;
				}
				responseCallback(message.responseData);
				delete responseCallbacks[message.responseId];
			} else {
				if (message.callbackId) {
					var callbackResponseId = message.callbackId;
					responseCallback = function(responseData) {
						_doSend({ handlerName:message.handlerName, responseId:callbackResponseId, responseData:responseData });
					};
				}
				
				var handler = messageHandlers[message.handlerName];
				if (!handler) {
					console.log("WebViewJavascriptBridge: WARNING: no handler for message from ObjC:", message);
				} else {
					handler(message.data, responseCallback);
				}
			}
		}
	}
```

可以看出与native端处理逻辑差不多。
上面还是蛮绕的，callbackId,reponseId,直接的互相替换。总的来说就是两端的responseId和callbackid是互相对应的，存在responseId的时候说明都是对端函数处理完成后的回调，通过responseId找到对应的handler。

                

       








