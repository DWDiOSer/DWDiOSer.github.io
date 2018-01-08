---
layout: post
title: JavaScriptCore 探究（1）
date: 2017-12-29 
name: yucy
---
##<center>JavaScriptCore 探究（1）</center>

Apple 在 iOS7 中加入了 JavaScriptCore 框架。由于是封装了 JavaScript 和 Objective-C 桥接的 API， JavaScriptCore 框架的加入让 Objective-C 和 JavaScript 代码直接交互变得更加简单方便。如下是 JavaScriptCore.h 文件：
```
#ifndef JavaScriptCore_h
#define JavaScriptCore_h

#include <JavaScriptCore/JavaScript.h>
#include <JavaScriptCore/JSStringRefCF.h>

#if defined(__OBJC__) && JSC_OBJC_API_ENABLED

#import "JSContext.h"
#import "JSValue.h"
#import "JSManagedValue.h"
#import "JSVirtualMachine.h"
#import "JSExport.h"

#endif

#endif
```

#### JSContext
JSContext 是 JavaScript 的运行环境，他主要作用是执行 JavaScript 代码和注册 native 方法接口。所有的 JSValue 都是捆绑在一个 JSContext 上的。

#### JSValue
JSValue 可以说是 JavaScript 和 Object-C之间互换的桥梁，它提供了多种方法使 JavaScript 数据类型与 Objective-C 数据类型互换。并且每个JSValue都是强引用一个context。(参考 Apple 文档)
![score](http://chuantu.biz/t6/190/1514519130x-1566657763.png)

#### JSManagedValue
JavaScript 和 OC 对象的内存管理辅助对象。由于 JavaScript 是使用垃圾回收机制的内存管理方式（JS 中的对象都是强引用），而OC是引用计数机制。如果双方相互引用，势必会造成循环引用，而导致内存泄露。JSManagedValue 能帮助引用计数和垃圾回收这两种内存管理机制进行正确的转换。需要注意的是，如果在 block 中使用 JSValue，那么 JSvalue 就会被 block 强引用，从而造成循环引用。

#### JSVirtualMachine
JavaScript 运行的虚拟机对象空间，有自己独立的堆空间和垃圾回收机制。

#### JSExport
协议，用来将原生对象导出给JavaScript，OC对象只要实现JSExport协议，JavaScript 对象就可以直接直接调用OC对象里面的方法和属性

#### WebView中Objective-C与JavaScript交互
######block 代码示例
将Object-C block 注册给 JScontext:
```
JSContext *context = [[JSContext alloc] init];
context[@"show"] = ^(NSString *string){
        return string;
    };
```

JavaScriptCore 将将这个 block 包装成一个 JavaScript 方法
```
function show(string)
{
    return string;
}

```
JScontext 通过`evaluateScript`调用 JavaScript 方法
```
JSValue *value = [context evaluateScript:@"show('hello word')"]; 
NSLog(@"%@", value);
```
输出:`hello word`

######协议 代码示例
简单的 H5 代码如下
```
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="UTF-8">
            <title>just try</title>
            <script>
                function test() {
                    NativeJSObject.NativeGoToXXXXXX(); // 在 script 中直接调用已经注册的类调用类的方法
                }
            </script>
    </head>
    <body>
        <button id="btnClick" onclick="test()">
            dian-dian-dian-dian
        </button>
    </body>
</html>
```
1. 在 .h 文件中创建、实现协议
```
@interface WebViewController : UIViewController
@end
    //  首先创建一个实现了JSExport协议的协议
@protocol JSObjectProtocol <JSExport>
- (void)NativeGoToXXXXXX;
@end

@interface NativeJSObject : NSObject<JSObjectProtocol>
@end
```
2. 在 .m 文件实现具体的协议方法
```
@interface WebViewController ()
@end

@implementation WebViewController
- (void)viewDidLoad 
{
    [super viewDidLoad];
    [self addWebView];
}

- (void)addWebView
{
    UIWebView *webView = [[UIWebView alloc] initWithFrame:self.view.bounds];
    webView.delegate = self;
    [self.view addSubview:webView];
    
    //  方便测试直接将 html 文件放在工程中
    NSString *filePath = [[NSBundle mainBundle]pathForResource:@"js" ofType:@"html"];
    NSString *htmlString = [NSString stringWithContentsOfFile:filePath encoding:NSUTF8StringEncoding error:nil];
    [webView loadHTMLString:htmlString baseURL:nil];
}

- (void)webViewDidFinishLoad:(UIWebView *)webView 
{
    // 获取 JSContext
    JSContext *jsContext = [webView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];
    
    // 注册 native 接口
    NativeJSObject *testJO = [NativeJSObject new];
    jsContext[@"NativeJSObject"] = testJO;
}

@implementation NativeJSObject
- (void)NativeGoToXXXXXX
{
    // 本地执行的代码 
}
@end
```
声明继承自 JSExport 的自定义协议，把实现这个协议的类的对象暴露给 JavaScript 时，JavaScript 中会生成一个对应的 JavaScript 对象，然后 JavaScriptCore 会按照这个协议中声明的内容遍历实现这个协议的类，把协议中声明的方法转换成 JavaScript 函数（对象中的属性，实质上是转换成getter 和 setter 方法）。
转换方法和之前说的 block 类似，创建一个 JavaScript 方法包装着 Object-C 中的方法，然后协议中声明的方法转换成 JavaScript 对象上的实例方法。
参考：[http://www.cocoachina.com/ios/20170720/19958.html](http://www.cocoachina.com/ios/20170720/19958.html)

