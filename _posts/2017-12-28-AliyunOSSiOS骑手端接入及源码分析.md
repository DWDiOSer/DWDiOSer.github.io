# AliyunOSSiOS骑手端实践及源码分析

## AliyunOSSiOS简介
#### OSS简介
```
1.阿里云对象存储服务（Object Storage Service，简称 OSS），
是阿里云提供的海量、安全、低成本、高可靠的云存储服务。
2.参考链接
https://help.aliyun.com/document_detail/31817.html?spm=5176.doc32055.6.539.Ra1hlS
```
## 骑手端实战
#### 骑手iOS端封装OSS架构图
###### DWOSSManager管理类介绍
```
1.上传图片管理类，此类为单例，按照业务需求提供了多种上传图片策略
2.目前只支持同一时刻单张图片上传，要上传多张图片请依次上传。
```
###### DWOSSManager上传图片方法
![](http://chuantu.biz/t6/187/1514288939x-1566657763.png)

```
DWOSSObjectTypeDefault 通用图片上传规则。为了兼容之前老版本，
这里保留原有方法，不做调整

- (void)uploadImage:(UIImage *)image;

//是否限制图片大小在100kb
- (void)uploadImage:(UIImage *)image limitSizeOfThe100Kb:(BOOL)isLimit;

//根据imageTypew及params对图片目录及图片名字可自定制
- (void)uploadImage:(UIImage *)image imageType:(DWOSSObjectType)imageType params:(NSDictionary *)params;

//是否限制在100kb, 根据imageTypew及params对图片目录及图片名字可自定制
- (void)uploadImage:(UIImage *)image limitSizeOfThe100Kb:(BOOL)isLimit imageType:(DWOSSObjectType)imageType params:(NSDictionary *)params;
```
###### 骑手端上传图片类型
```
DWOSSObjectType定义了骑手端上传图片类型
typedef NS_ENUM(NSUInteger, DWOSSObjectType) {
    DWOSSObjectTypeDefault,//通用上传规则
    DWOSSObjectTypeTicket,//小票
    DWOSSObjectTypeGoods,//货物图片
    DWOSSObjectTypeIdentityFront,//身份证正面
    DWOSSObjectTypeIdentityBack,//身份证反面
    DWOSSObjectTypeIdentityHold//身份证手持
};
1.默认情况下，采用DWOSSObjectTypeDefault即可，不需要做额外参数配置.
2.其他业务场景，根据具体需求添加新的类型及配置即可.
```
###### 图片上传成功或者失败回调
```
通过block回调：
void(^uploadCallBack)(BOOL isSuccess,NSString *fileUrl)
通过对isSuccess赋值，代表图片上传成功或者失败
isSuccess当为YES时，fileUrl为上传到阿里云OSS的图片完整路径
isSuccess为NO时，表示图片上传失败。
```
###### DWOSSManager内部通过DWOSSObject操作图片上传OSS及回调
```
DWOSSObject通过持有OSSFederationCredentialProvider及
OSSClientConfiguration对象操作图片上传操作
```

## 源码分析
#### 1.OSSClient类
###### 介绍
```
OSSClient是OSS服务的iOS客户端，它为调用者提供了一系列的方法，用于和OSS服务进行交互。
一般来说，全局内只需要保持一个OSSClient，用来调用各种操作
```
![](http://chuantu.biz/t6/187/1514288914x-1566657763.png)
###### OSSClient持有关键类对象
```
/**
 用以收发网络请求
 */
@property (nonatomic, strong) OSSNetworking * networking;

/**
 提供访问所需凭证，非常重要，用户可以通过该类，自实现加签接口的加签器
 */
@property (nonatomic, strong) id<OSSCredentialProvider> credentialProvider;

/**
 客户端设置，可以配置如下参数:最大重试次数、最大并发请求数、请求超时时间、设置代理Host、端口等
 */
@property (nonatomic, strong) OSSClientConfiguration * clientConfiguration;
/**
 任务队列，此类能够实现在给定线程队列里面执行一个block，也就是一个任务
 */
@property (nonatomic, strong, readonly) OSSExecutor * ossOperationExecutor;
```
###### OSSClient对象生成方式
```
OSSClient *client = [[OSSClient alloc] initWithEndpoint:endpoint credentialProvider:self.creDentialProvider clientConfiguration:self.conf];
其中self.creDentialProvider的创建方式为:
- (OSSFederationCredentialProvider *)creDentialProvider{
    
    if (!_creDentialProvider) {
        __weak typeof(self)weakSelf = self;
        _creDentialProvider = [[OSSFederationCredentialProvider alloc] initWithFederationTokenGetter:^OSSFederationToken *{
            
            NSURL * url = [NSURL URLWithString:[NSString stringWithFormat:@"%@%@",STS_WEB_URL,STS_UPLOADPOLICY_PATH]];
            
            NSMutableURLRequest * request = [NSMutableURLRequest requestWithURL:url];
            [request setHTTPMethod:@"POST"];
        
            //设置请求头信息
            NSMutableDictionary *dict = [NSMutableDictionary dictionary];
            [dict setObject:@"file" forKey:@"type"];
            ...
            [request setHTTPBody:[NSJSONSerialization dataWithJSONObject:dict options:NSJSONWritingPrettyPrinted error:nil]];
        
            [request setValue:@"application/json; charset=UTF-8" forHTTPHeaderField:@"content-type"];//请求头
            
            OSSTaskCompletionSource * tcs = [OSSTaskCompletionSource taskCompletionSource];
            NSURLSession * session = [NSURLSession sharedSession];
            NSURLSessionTask * sessionTask = [session dataTaskWithRequest:request
                                                        completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
                                                            if (error) {
                                                                [tcs setError:error];
                                                                return;
                                                            }
                                                            [tcs setResult:data];
                                                        }];
            [sessionTask resume];
            [tcs.task waitUntilFinished];
            if (tcs.task.error) {
                NSLog(@"get token error: %@", tcs.task.error);
                return nil;
            } else {
                NSDictionary *object = [NSJSONSerialization JSONObjectWithData:tcs.task.result
                                                                       options:kNilOptions
                                                                         error:nil];
                //设置OSS请求Token，为了去报安全性,token的tAccessKey及tSecretKey由服务器下发            
                OSSFederationToken * token = [OSSFederationToken new];
                token.tAccessKey = [credentials objectForKey:@"tAccessKey"];
                token.tSecretKey = [credentials objectForKey:@"tSecretKey"];
                token.tToken = [credentials objectForKey:@"SecurityToken"];
                token.expirationTimeInGMTFormat = [credentials objectForKey:@"Expiration"];
                NSLog(@"get token: %@", token);
                //返回OSSFederationToken对象
                return token;
            }
        }];
    }
    return _creDentialProvider;
}


```
**看到上面创建方式有一个回调block了吗，当OSS判断当前Token超过缓存时间或者根本不存在时，会执行block，获取Token，但是token又是通过网络获取的，那么怎么确保网络回来后再执行后面的代码逻辑呢，这里用到了多线程通过条件变量控制线程的等待已回复当前线程，关键代码：**

```
OSSTaskCompletionSource * tcs = [OSSTaskCompletionSource taskCompletionSource];
...
[tcs setResult:data];
OSSTaskCompletionSource里面持有OSSTask对象，
OSSTask对象包含
@property (nonatomic, strong) NSCondition *condition;
通过condition实现线程等待与唤醒

- (void)runContinuations {
    @synchronized(self.lock) {
        [self.condition lock];
        [self.condition broadcast];
        [self.condition unlock];
        for (void (^callback)() in self.callbacks) {
            callback();
        }
        [self.callbacks removeAllObjects];
    }
}

```
**self.conf的创建方式:**

```

-(OSSClientConfiguration *)conf{
    if (!_conf) {
        OSSClientConfiguration * conf = [OSSClientConfiguration new];
        //设置最大重试次数
        conf.maxRetryCount = 2;
        //超时时间
        conf.timeoutIntervalForRequest = 30;
        conf.timeoutIntervalForResource = 24 * 60 * 60;
        _conf = conf;
    }
    return _conf;
}

```
#### 2.OSSPutObjectRequest类
###### 介绍
```
上传Object的请求头
```
###### OSSPutObjectRequest关键属性，包含上传图片数据及其bucketName，objectKey
```
/**
 Bucket名称
 */
@property (nonatomic, strong) NSString * bucketName;

/**
 Object名称
 */
@property (nonatomic, strong) NSString * objectKey;

/**
 从内存中的NSData上传时，通过这个字段设置
 */
@property (nonatomic, strong) NSData * uploadingData;

......

```
#### 3.OSSTask类
###### 介绍
```
1.The consumer view of a Task. A OSSTask has methods to
 inspect the state of the task, and to add continuations to
 be run once the task is complete.
2.该类是图片上传阿里云核心类，管理了任务上传成功及失败的回调
```
###### 对象生成及调用函数
```
OSSClient *client = [[OSSClient alloc] initWithEndpoint:endpoint credentialProvider:self.creDentialProvider clientConfiguration:self.conf];
    
    OSSPutObjectRequest * put = [OSSPutObjectRequest new];
    
    put.bucketName = STS_BUCKETNAME;
    put.objectKey = imageTag;
    self.tag = imageTag;
    
    put.uploadingData = data; // 直接上传NSData
    
    OSSTask * putTask = [client putObject:put];
    
    [putTask continueWithBlock:^id(OSSTask *task) {
        if (!task.error) {
            OSSPutObjectResult * result = task.result;
            NSLog(@"upload object success! %@ result = %@",task,result);
            
            NSString *imagePath = [NSString stringWithFormat:@"%@/%@",weakSelf.destination,weakSelf.tag];
            if ([weakSelf.destination containsString:@"http"] || [weakSelf.destination containsString:@"https"]) {
                //啥也不做
            }else{
               imagePath = [NSString stringWithFormat:@"https://%@/%@",weakSelf.destination,weakSelf.tag];
            }
            NSLog(@"imagePath = %@",imagePath);
            if (weakSelf.delegate && [weakSelf.delegate respondsToSelector:@selector(uploadSuccess:)]) {
                [weakSelf.delegate uploadSuccess: imagePath];
            }
        } else {
            NSLog(@"upload object failed, error: %@" , task.error);
            OSSPutObjectResult * result = task.result;
            NSLog(@"Result - requestId: %@, headerFields: %@, servercallback: %@",
                  result.requestId,
                  result.httpResponseHeaderFields,
                  result.serverReturnJsonString);
            if (weakSelf.delegate && [weakSelf.delegate respondsToSelector:@selector(uploadFailed)]) {
                [weakSelf.delegate uploadFailed];
            }
            
        }
        return nil;
    }];
```
###### OSSTask内部实现原理
```
1.OSSClient分析
OSSClient持有OSSNetworking对象，该对象用于上传图片等网路请求
/**
 用以收发网络请求
 */
@property (nonatomic, strong) OSSNetworking * networking;
2.OSSNetworking对象分析
OSSNetworking持有系统网络请求类，实现数据和服务器的传递，通过网络请求代理，监听请求成功或者失败，将结果通过block回调出去
@property (nonatomic, strong) NSURLSession * dataSession;
@property (nonatomic, strong) NSURLSession * uploadFileSession;

- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)sessionTask didCompleteWithError:(NSError *)error {
    OSSNetworkingRequestDelegate * delegate = [self.sessionDelagateManager objectForKey:@(sessionTask.taskIdentifier)];
    [self.sessionDelagateManager removeObjectForKey:@(sessionTask.taskIdentifier)];

    NSHTTPURLResponse * httpResponse = (NSHTTPURLResponse *)sessionTask.response;
    if (delegate == nil) {
        OSSLogVerbose(@"delegate: %@", delegate);
        /* if the background transfer service is enable, may recieve the previous task complete callback */
        /* for now, we ignore it */
        return ;
    }

    ...
    [[[[OSSTask taskWithResult:nil] continueWithSuccessBlock:^id(OSSTask * task) {
        if (!delegate.error) {
            delegate.error = error;
        }
        if (delegate.error) {
            OSSLogDebug(@"networking request completed with error: %@", error);
            if ([delegate.error.domain isEqualToString:NSURLErrorDomain] && delegate.error.code == NSURLErrorCancelled) {
                return [OSSTask taskWithError:[NSError errorWithDomain:OSSClientErrorDomain
                                                                 code:OSSClientErrorCodeTaskCancelled
                                                             userInfo:[error userInfo]]];
            } else {
                NSMutableDictionary * userInfo = [NSMutableDictionary dictionaryWithDictionary:[error userInfo]];
                [userInfo setObject:[NSString stringWithFormat:@"%ld", (long)error.code] forKey:@"OriginErrorCode"];
                return [OSSTask taskWithError:[NSError errorWithDomain:OSSClientErrorDomain
                                                                 code:OSSClientErrorCodeNetworkError
                                                             userInfo:userInfo]];
            }
        }
        return task;
    }] continueWithSuccessBlock:^id(OSSTask *task) {
        if (delegate.isHttpRequestNotSuccessResponse) {
            if (httpResponse.statusCode == 0) {
                return [OSSTask taskWithError:[NSError errorWithDomain:OSSClientErrorDomain
                                                                 code:OSSClientErrorCodeNetworkingFailWithResponseCode0
                                                             userInfo:@{OSSErrorMessageTOKEN: @"Request failed, response code 0"}]];
            }
            NSString * notSuccessResponseBody = [[NSString alloc] initWithData:delegate.httpRequestNotSuccessResponseBody encoding:NSUTF8StringEncoding];
            OSSLogError(@"http error response: %@", notSuccessResponseBody);
            NSDictionary * dict = [NSDictionary dictionaryWithXMLString:notSuccessResponseBody];

            return [OSSTask taskWithError:[NSError errorWithDomain:OSSServerErrorDomain
                                                             code:(-1 * httpResponse.statusCode)
                                                         userInfo:dict]];
        }
        return task;
    }] continueWithBlock:^id(OSSTask *task) {
        if (task.error) {
            OSSNetworkingRetryType retryType = [delegate.retryHandler shouldRetry:delegate.currentRetryCount
                                                                  requestDelegate:delegate
                                                                         response:httpResponse
                                                                            error:task.error];
            OSSLogVerbose(@"current retry count: %u, retry type: %d", delegate.currentRetryCount, (int)retryType);

            switch (retryType) {

                case OSSNetworkingRetryTypeShouldNotRetry: {
                    delegate.completionHandler(nil, task.error);
                    return nil;
                }

                case OSSNetworkingRetryTypeShouldCorrectClockSkewAndRetry: {
                    /* correct clock skew */
                    [delegate.interceptors insertObject:[OSSTimeSkewedFixingInterceptor new] atIndex:0];
                    break;
                }

                default:
                    break;
            }

            /* now, should retry */
            NSTimeInterval suspendTime = [delegate.retryHandler timeIntervalForRetry:delegate.currentRetryCount retryType:retryType];
            delegate.currentRetryCount++;
            [NSThread sleepForTimeInterval:suspendTime];

            /* retry recursively */
            [delegate reset];
            [self dataTaskWithDelegate:delegate];
        } else {
        
            //这里就是图片上传成功回调地方
            delegate.completionHandler([delegate.responseParser constructResultObject], nil);
        }
        return nil;
    }];
    ...
}
```
**delegate.completionHandler([delegate.responseParser constructResultObject], nil);
回调下面函数的completionHandler**

```
- (OSSTask *)sendRequest:(OSSNetworkingRequestDelegate *)request {
    OSSLogVerbose(@"send request --------");
    if (self.configuration.proxyHost && self.configuration.proxyPort) {
        request.isAccessViaProxy = YES;
    }

    /* set maximum retry */
    request.retryHandler.maxRetryCount = self.configuration.maxRetryCount;

    OSSTaskCompletionSource * taskCompletionSource = [OSSTaskCompletionSource taskCompletionSource];

    __weak OSSNetworkingRequestDelegate * ref = request;
    request.completionHandler = ^(id responseObject, NSError * error) {

        [ref reset];
        if (!error) {
            [taskCompletionSource setResult:responseObject];
        } else {
            [taskCompletionSource setError:error];
        }
    };
    [self dataTaskWithDelegate:request];
    return taskCompletionSource.task;
}

接着继续执行OSSTask的runContinuations方法，执行self.callBacks里面的回调，其中包含将上传图片成功或者失败的block
- (void)runContinuations {
    @synchronized(self.lock) {
        [self.condition lock];
        [self.condition broadcast];
        [self.condition unlock];
        for (void (^callback)() in self.callbacks) {
            callback();
        }
        [self.callbacks removeAllObjects];
    }
}

```






