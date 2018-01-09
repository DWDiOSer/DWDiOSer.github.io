---
layout: post
title: WebSocket初步探究
date: 2017-12-26
---

# pxl

* **使用背景：**

由于现在我们骑手端在消息通知或骑手的各种状态修改时，要么是用push，要么使用轮询。现在轮询已经承担了一部分的消息通知，从现在来看，轮询接口已经增加了太多的字段（25+），有些字段使用频率很低，但每次都会返回，感觉轮询接口已经被滥用了，而且轮询是15s一次，消息通知往往都会有一定程度的延迟，达不到及时性的要求。现在我们希望通过接入WebSocket来减轻轮询的压力，并且提高消息通知的及时性，提升骑手APP的用户体验。

## 一、WebSocket简介
1、现在我们APP里消息轮询是如下图：
![](https://upload-images.jianshu.io/upload_images/1194012-ce4df238336909a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

* 这种方式下，是不适合获取实时信息的，客户端和服务器之间会一直进行连接，每隔一段时间就询问一次。客户端会轮询，有没有新消息。这种方式连接数会很多，一个接受，一个发送。我们APP里是15s一次，次数一多，会很耗流量，耗电量，也会消耗CPU的利用率。


2、**WebSocket** 

* WebSocket是 HTML5 一种新的协议。它实现了浏览器与服务器全双工通信，能更好的节省服务器资源和带宽并达到实时通讯，它建立在 TCP 之上，同 HTTP 一样通过 TCP 来传输数据；

* 它和 HTTP 最大不同是：WebSocket 是一种双向通信协议，在建立连接后，WebSocket 服务器能主动的向client发送或接收数据，就像 Socket 一样.


> **Websocket的数据传输是frame形式传输的，比如会将一条消息分为几个frame，按照先后顺序传输出去。这样做会有几个好处：**

 1 大数据的传输可以分片传输，不用考虑到数据大小导致的长度标志位不足够的情况。
 2 和http的chunk一样，可以边生成数据边传递消息，即提高传输效率。


```
/* From RFC:

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-------+-+-------------+-------------------------------+
 |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
 |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
 |N|V|V|V|       |S|             |   (if payload len==126/127)   |
 | |1|2|3|       |K|             |                               |
 +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
 |     Extended payload length continued, if payload len == 127  |
 + - - - - - - - - - - - - - - - +-------------------------------+
 |                               |Masking-key, if MASK set to 1  |
 +-------------------------------+-------------------------------+
 | Masking-key (continued)       |          Payload Data         |
 +-------------------------------- - - - - - - - - - - - - - - - +
 :                     Payload Data continued ...                :
 + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
 |                     Payload Data continued ...                |
 +---------------------------------------------------------------+
 */
 
 *引至SocketRocket库中代码*
 
FIN 1bit 表示信息的最后一帧，flag，也就是标记符
RSV 1-3 1bit each 以后备用的 默认都为 0
Opcode 4bit 帧类型，稍后细说
Mask 1bit 掩码，是否加密数据，默认必须置为1
Payload 7bit 数据的长度 (2^7 -1 最大到127)
Masking-key 1 or 4 bit 掩码 //用来编码数据
Payload data (x + y) bytes 数据 
Extension data x bytes 扩展数据
Application data y bytes 程序数据
```

* [数据帧注解](https://chenjianlong.gitbooks.io/rfc-6455-websocket-protocol-in-chinese/content/section5/section5.html)

* [websocket协议翻译](https://github.com/zhangkaitao/websocket-protocol)


## 二、iOS端WebSocket引入

#### 1、导入SocketRocket库

1、cocoaPods导入

```
pod 'SocketRocket', 	'~> 0.5.1'
```

2、导入头文件

```
#import "SRWebSocket.h"
```
* 注：由于该库只有几个文件，并没有什么frameWork，所以哪里有用到直接导入头文件即可。


#### 2、创建WebSocket管理类

**1、首先创建一个单例类来统一管理WebSocket，WebSocket的开始和关闭都统一调用，方便管理。**

```
@interface WebSocketManager : NSObject
+ (id)shareManager;
//开启
- (void)webSocketOpen;
//关闭
- (void)webSocketClose;
@end
```

**2、开启/关闭**

```
//开始连接
- (void)webSocketOpen {
    if (self.webSocket) {
        return;
    }
    
    self.webSocket = [[SRWebSocket alloc] initWithURLRequest:[NSURLRequest requestWithURL:[NSURL URLWithString:@"ws://localhost:3000"]]];
    self.webSocket.delegate = self;
    [self.webSocket open];
}

//关闭连接
- (void)webSocketClose {
    if (self.webSocket){
        [self.webSocket close];
        self.webSocket = nil;
        //断开连接时销毁心跳
        [self cancelHeartBeat];
    }
}
```

* 在开启和关闭WebSocket前都要判断一下当前是否有WebSocket对象存在，防止在开启时重复创建，在关闭时，没有WebSocket对象可以关闭。开始链接主要是创建一个WebSocket对象，并设置好它的代理，同时打开进行连接。关闭的时候主要是在关闭连接的同时要将WebSocket对象给销毁掉，再将心跳包也销毁掉（下面会讲到）。

**3、设置好代理方法**

```
#pragma mark - SRWebSocketDelegate
- (void)webSocket:(SRWebSocket *)webSocket didReceiveMessage:(id)message {
    NSLog(@"接收到信息");
}

- (void)webSocketDidOpen:(SRWebSocket *)webSocket {
    NSLog(@"已经连接");
}

- (void)webSocket:(SRWebSocket *)webSocket didFailWithError:(NSError *)error {
    NSLog(@"连接出错");
}

- (void)webSocket:(SRWebSocket *)webSocket didReceivePong:(NSData *)pongPayload {
    NSLog(@"pong返回");
}

- (void)webSocket:(SRWebSocket *)webSocket didCloseWithCode:(NSInteger)code reason:(NSString *)reason wasClean:(BOOL)wasClean {
    NSLog(@"已经关闭");
}
```

* SocketRocket对WebSocket提供了这个5个主要的代理方法，主要有收到服务端信息、已经连接、连接出错、pong返回、已经关闭等功能，可以根据不同的场景去做不同的操作。其中第一个代理方法是必须实现的，否则会报错。

**4、心跳机制**

```
//初始化心跳
- (void)createHeartBeat {
    dispatch_main_async_safe(^{
        [self cancelHeartBeat];
        __weak typeof(self) weakSelf = self;
        self.heartBeat = [NSTimer scheduledTimerWithTimeInterval:3*60 repeats:YES block:^(NSTimer * _Nonnull timer) {
            NSLog(@"heart");
            [weakSelf sendData:@"heart"];
        }];
        [[NSRunLoop currentRunLoop]addTimer:self.heartBeat forMode:NSRunLoopCommonModes];
    })
}

//取消心跳
- (void)cancelHeartBeat {
    dispatch_main_async_safe(^{
        if (self.heartBeat) {
            [self.heartBeat invalidate];
            self.heartBeat = nil;
        }
    })
}

//发送数据
- (void)sendData:(id)data {
    
    __weak __typeof(self)weakSelf = self;
    dispatch_queue_t queue =  dispatch_queue_create("zy", NULL);
    
    dispatch_async(queue, ^{
        if (weakSelf.webSocket != nil) {
            if (weakSelf.webSocket.readyState == SR_OPEN) {
                //发送数据
                [weakSelf.webSocket send:data];
            } else if (weakSelf.webSocket.readyState == SR_CONNECTING) {
                //重连
                [self reConnect];
            } else if (weakSelf.webSocket.readyState == SR_CLOSING || weakSelf.webSocket.readyState == SR_CLOSED) {
                //websocket断开了，重连
                [self reConnect];
            }
        } else {
            NSLog(@"没网络，发送失败");
        }
    });
}
```

* 心跳机制主要是为了保证服务端和客户端之间的连接能够正常通信，通过每隔一段时间给服务端发送数据来确认连接是否正常。这边主要是通过定时器来定时发送心跳包。**最后关闭WebSocket的时候一定要销毁掉心跳包**。

**5、重连机制**

```
//重连机制
- (void)reConnect {
    [self webSocketClose];
    if (self.reConnectTime > 64) {
        return;
    }
    
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(self.reConnectTime * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        self.webSocket = nil;
        [self webSocketOpen];
        NSLog(@"重连");
    });
    
    if (self.reConnectTime == 0) {
        self.reConnectTime = 2;
    }else{
        self.reConnectTime *= 2;
    }
}
```

* 重连机制主要是为了防止断开连接或断网情况下，客户端与服务端之间无法正常通信。在这个时候通常会隔一段时间去重新开始连接，我这边暂时设置的是重连5次之后就不再重连了。一般多次之后还是无法连接，表示服务端可能已经出现严重问题，就不要做无用功，继续重连了。


**注：WebSocket在iOS端接入的一些主要代码就是上面这些，这些代码开起来比较简单，但是每一个部分都是非常重要的，都有自己独立的功能，可以说说是缺一不可。通过这些，对WebSocket的轻便、简单及实用有了新的理解。**

## 三、WebSocket实践

#### 1、创建一个简单的WebSocket服务器
我们用npm管理器来创建，在终端输入

```
npm init --yes
```
后面再安装一些工具

```
1、express: npm install --save --save-exact express
2、ws(websocket的node封装): npm install --save --save-exact ws bufferutil utf-8-validate
```

#### 2、网页输入内容，点击按钮客户端做出响应
##### 1、新建一个名为index.html的本地网页，代码如下：

```
<html>
<head>
	<title>dianwoda</title>
</head>
<body>
<input type="text" name="name" placeholder="请输入" id="message" />
<button onclick = "send()">发送</button>
<script>
	var HOST = location.origin.replace(/^http/, 'ws');
	var ws = new WebSocket(HOST);
	ws.onmessage = function (event) {
		// alert(event.data);
	}
	function send() {
		var e = document.getElementById("message").value;
		ws.send(e);
	};
</script>
</body>
</html>>
```
* 这个网页的主要作用是将文本框里的输入内容转发给WebSocket服务器（下面会讲到），进而将内容发送给客户端。

##### 2、创建一个名为server.js的文件，来充当WebSocket服务器，代码如下：

```
'use strict';

const express = require('express');
const SocketServer = require('ws').Server;
const path = require('path');
const PORT = process.env.PORT || 3000;
const INDEX = path.join(__dirname, 'index.html');
const server = express()
	.use((req, res) => res.sendFile(INDEX) )
	.listen(PORT, () => console.log(`Listening on ${ PORT }`));

const wss = new SocketServer({ server });
wss.on('connection', (ws) => {
	// ws.send('revice your connect!');
	console.log(ws);
	ws.on('message', (message) => {
		wss.clients.forEach((client) => {
			client.send(message);
		})
		// ws.send(message);
	});
});
```
* 这server.js的主要作用就是将网页的内容转发给客户端，客户端通过连接这个服务，来做出相对应的操作。

##### 3、iOS连接WebSocket服务器，接受消息并做出响应

 1、连接WebSocket服务器，并设置好代理（同第二部分的开启/关闭）


```
self.webSocket = [[SRWebSocket alloc] initWithURLRequest:[NSURLRequest requestWithURL:[NSURL URLWithString:@"ws://localhost:3000"]]];
self.webSocket.delegate = self;
[self.webSocket open];
```

2、根据接收到WebSocket服务器传回来的消息，做出响应

```
#pragma mark - SRWebSocketDelegate
- (void)webSocket:(SRWebSocket *)webSocket didReceiveMessage:(id)message {
    NSString *msg = (NSString *)message;
    showMsgTimeView(TOAST_TIME, msg, nil);
    NSLog(@"收到消息  %@", message);
    
    if ([msg isEqualToString:@"new"]) {
        //新订单语音
        [[DWDPlayVoiceManager shared] playVoiceWithOrderType:1];
    }else if ([msg isEqualToString:@"grab"]) {
        //抢单语音
        [[NSNotificationCenter defaultCenter] postNotificationName:kReceiveServersShowNewOrderPromptViewNotification object:nil];
    }else if ([msg isEqualToString:@"logout"]) {
        //退出
        [[NSNotificationCenter defaultCenter] postNotificationName:NOTIFICATION_OF_TOKEN_FAILFURE object:nil];
    }
}
```

* 客户端连接上WebSocket服务器之后一定要通过发心跳包来保证与服务器之间的通信，如果出现断开或者断网的情况，要适当的进行重连操作（上面有提到）。

**注：最终我们就实现了客户端和服务端的双通信，不用再去使用轮询，提高了效率，节省了性能。**

## 四、SRWebSocket源码解析
下面用几个主要的方法来进行一些解析：

#### 1、初始化

```
- (void)_SR_commonInit;
{
    NSString *scheme = _url.scheme.lowercaseString;
    assert([scheme isEqualToString:@"ws"] || [scheme isEqualToString:@"http"] || [scheme isEqualToString:@"wss"] || [scheme isEqualToString:@"https"]);
    
    if ([scheme isEqualToString:@"wss"] || [scheme isEqualToString:@"https"]) {
        _secure = YES;
    }
    
    _readyState = SR_CONNECTING;
    _consumerStopped = YES;
    _webSocketVersion = 13;
    
    //初始化工作的队列，串行
    _workQueue = dispatch_queue_create(NULL, DISPATCH_QUEUE_SERIAL);
    
    // Going to set a specific on the queue so we can validate we're on the work queue
    dispatch_queue_set_specific(_workQueue, (__bridge void *)self, maybe_bridge(_workQueue), NULL);
    
    //设置代理queue为主队列
    _delegateDispatchQueue = dispatch_get_main_queue();
    sr_dispatch_retain(_delegateDispatchQueue);
    
    //读取缓冲区
    _readBuffer = [[NSMutableData alloc] init];
    //输出缓冲区
    _outputBuffer = [[NSMutableData alloc] init];
    
    _currentFrameData = [[NSMutableData alloc] init];

    _consumers = [[NSMutableArray alloc] init];
    
    _consumerPool = [[SRIOConsumerPool alloc] init];
    
    _scheduledRunloops = [[NSMutableSet alloc] init];
    
    [self _initializeStreams];
    
    // default handlers
}
```

1. 包括对schem进行断言，只支持ws/wss/http/https四种。
2. 当前socket状态，是正在连接，还是已连接、断开等等。
3. 初始化工作队列，以及流回调线程等等。
4. 初始化读写缓冲区：_readBuffer、_outputBuffer。

#### 2、输入输出流的创建及绑定

```
- (void)_initializeStreams;
{
    assert(_url.port.unsignedIntValue <= UINT32_MAX);
    //拿到端口
    uint32_t port = _url.port.unsignedIntValue;
    if (port == 0) {
        if (!_secure) {
            //https 或 wss
            port = 80;
        } else {
            //http 或 ws
            port = 443;
        }
    }
    NSString *host = _url.host;
    
    CFReadStreamRef readStream = NULL;
    CFWriteStreamRef writeStream = NULL;
    
    CFStreamCreatePairWithSocketToHost(NULL, (__bridge CFStringRef)host, port, &readStream, &writeStream);
    
    _outputStream = CFBridgingRelease(writeStream);
    _inputStream = CFBridgingRelease(readStream);
    
    _inputStream.delegate = self;
    _outputStream.delegate = self;
}
```
在这里，我们根据传进来的url，类似ws://localhost:80,进行输入输出流CFStream的创建及绑定。

![](http://chuantu.biz/t6/187/1514293197x-1566657763.png)

#### 3、开启

```
- (void)open;
{
    assert(_url);
    //如果状态是正在连接，直接断言出错
    NSAssert(_readyState == SR_CONNECTING, @"Cannot call -(void)open on SRWebSocket more than once");

    _selfRetain = self;
    
    //判断超时时长
    if (_urlRequest.timeoutInterval > 0)
    {
        dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, _urlRequest.timeoutInterval * NSEC_PER_SEC);
        dispatch_after(popTime, dispatch_get_main_queue(), ^(void){
            if (self.readyState == SR_CONNECTING)
                [self _failWithError:[NSError errorWithDomain:@"com.squareup.SocketRocket" code:504 userInfo:@{NSLocalizedDescriptionKey: @"Timeout Connecting to Server"}]];
        });
    }
    
    //开始建立连接
    [self openConnection];
}
```
* open方法定义了一个超时，如果超时了还在SR_CONNECTING，则报错，并且断开连接，清除一些已经初始化好的参数。

这个库.h文件暴露出来的方法就那么几个，而.m里却用了2000多行的代码来构造这些方法，所以他里面有很多方法还是值得我们去学习，去解读的，后续我会继续对他方法进行解析。

## 五、总结

* WebSocket实现了服务端和客户端的双通信，而且它和socket也是有区别的，具体的区别待后续去研究一下；WebSocket的功能也远远不止这些，他也能实现IM即时通讯等，这些后面都可以去研究研究。

