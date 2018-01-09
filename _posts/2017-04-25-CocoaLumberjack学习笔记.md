---
layout: post
title: 开源工程 CocoaLumberjack 学习笔记
date: 2017-04-26 16:36:24.000000000 +09:00
---

# WangYijian

### 1.1 基本介绍

CocoaLumberjack是一个开源工程，为Xcode提供分级打印的策略，源码地址就是：[CocoaLumeberjack](https://github.com/CocoaLumberjack/CocoaLumberjack) 在git上有将近9k的star，这个库包含的几个对象可以按需求把Log输出到不同的地方：
* Console.app
* Xcode控制台
* file文件
* DataBase 等

然后可以通过自己的ddLogLevel来定义日志的打印等级

```
typedef NS_ENUM(NSUInteger, DDLogLevel){
 
    DDLogLevelOff       = 0,
    DDLogLevelError     = (DDLogFlagError),
    DDLogLevelWarning   = (DDLogLevelError   | DDLogFlagWarning), 
    DDLogLevelInfo      = (DDLogLevelWarning | DDLogFlagInfo),
    DDLogLevelDebug     = (DDLogLevelInfo    | DDLogFlagDebug),
    DDLogLevelVerbose   = (DDLogLevelDebug   | DDLogFlagVerbose),
    DDLogLevelAll       = NSUIntegerMax
};

```
对应不同的级别，级别之间的联系是向下包含的。
同时也定义了相应的Log级别的宏

```
#define DDLogError(frmt, ...)   LOG_OBJC_MAYBE(LOG_ASYNC_ERROR,   LOG_LEVEL_DEF, LOG_FLAG_ERROR,   0, frmt, ##__VA_ARGS__)
#define DDLogWarn(frmt, ...)    LOG_OBJC_MAYBE(LOG_ASYNC_WARN,    LOG_LEVEL_DEF, LOG_FLAG_WARN,    0, frmt, ##__VA_ARGS__)
#define DDLogInfo(frmt, ...)    LOG_OBJC_MAYBE(LOG_ASYNC_INFO,    LOG_LEVEL_DEF, LOG_FLAG_INFO,    0, frmt, ##__VA_ARGS__)
#define DDLogDebug(frmt, ...)   LOG_OBJC_MAYBE(LOG_ASYNC_DEBUG,   LOG_LEVEL_DEF, LOG_FLAG_DEBUG,   0, frmt, ##__VA_ARGS__)
#define DDLogVerbose(frmt, ...) LOG_OBJC_MAYBE(LOG_ASYNC_VERBOSE, LOG_LEVEL_DEF, LOG_FLAG_VERBOSE, 0, frmt, ##__VA_ARGS__)
```

其中LOG_OBJC_MAYBE的宏的定义是

```
#define LOG_MACRO(isAsynchronous, lvl, flg, ctx, atag, fnct, frmt, ...) \
        [DDLog log : isAsynchronous                                     \
             level : lvl                                                \
              flag : flg                                                \
           context : ctx                                                \
              file : __FILE__                                           \
          function : fnct                                               \
              line : __LINE__                                           \
               tag : atag                                               \
            format : (frmt), ## __VA_ARGS__]

#define LOG_MAYBE(async, lvl, flg, ctx, fnct, frmt, ...)                       \
        do { if(lvl & flg) LOG_MACRO(async, lvl, flg, ctx, nil, fnct, frmt, ##__VA_ARGS__); } while(0)

#define LOG_OBJC_MAYBE(async, lvl, flg, ctx, frmt, ...) \
        LOG_MAYBE(async, lvl, flg, ctx, __PRETTY_FUNCTION__, frmt, ## __VA_ARGS__)
```


所有的 log 都会发给 DDLog 对象，其运行在自己的一个GCD队列(GlobalLoggingQueue)，之后，DDLog 会将 log 分发给其下注册的一个或多个 Logger，这步在多核下是并发的，效率很高。每个 Logger 处理收到的 log 也是在它们自己的 GCD队列下（loggingQueue）做的，它们询问其下的 Formatter，获取 Log 消息格式，然后最终根据 Logger 的逻辑，将 log 消息分发到不同的地方。

因为一个 DDLog 可以把 log 分发到所有其下注册的 Logger 下，也就是说一个 log 可以同时打到控制台，打到远程服务器，打到本地文件。

### 1.2 具体代码部分

Lumberjack的核心是DDLog,也是其必须实现的，其基本实现也在DDLog这文件中我们在其中找到了第一节当中介绍的那个宏的方法

```
+ (void)log:(BOOL)asynchronous
      level:(DDLogLevel)level
       flag:(DDLogFlag)flag
    context:(NSInteger)context
       file:(const char *)file
   function:(const char *)function
       line:(NSUInteger)line
        tag:(id __nullable)tag
     format:(NSString *)format, ... NS_FORMAT_FUNCTION(9,10)
```
这里介绍下这几个参数：

`asynchronous` BOOL值是否进行异步操作，具体的是指其在加入logger类时的同步和异步性这里不同的logger类是在最开始的时候说的将log信息传到不同地方的logger类；

`level`就是日志等级；

`flag`日志标记，通过标记来和日志等级比较 error最高，verbose最低，flag > level就输出，否则不输出；

`context`上下文，这里我的理解是给用户一个自定义环境的作用，例如区分不同网络环境；

`file`所在那个类的文件名；

`function`所在那个方法的方法名；

`line` 所在代码的行数；

`tag` 可能会用到的tag值；

`format` log的相关参数，或者说log的格式

这里传递参数的方式是用了va_list，然后DDLog类又创建了一个DDLogMessage来封装log信息里面的各个参数

```
 DDLogMessage *logMessage = [[DDLogMessage alloc] initWithMessage:message
                                                               level:level
                                                                flag:flag
                                                             context:context
                                                                file:[NSString stringWithFormat:@"%s", file]
                                                            function:[NSString stringWithFormat:@"%s", function]
                                                                line:line
                                                                 tag:tag
                                                             options:(DDLogMessageOptions)0
                                                           timestamp:nil];
```

这里我把这个message类理解为model层，来储存必要的参数。

```
- (instancetype)initWithMessage:(NSString *)message
                          level:(DDLogLevel)level
                           flag:(DDLogFlag)flag
                        context:(NSInteger)context
                           file:(NSString *)file
                       function:(NSString *)function
                           line:(NSUInteger)line
                            tag:(id)tag
                        options:(DDLogMessageOptions)options
                      timestamp:(NSDate *)timestamp {
    if ((self = [super init])) {
        BOOL copyMessage = (options & DDLogMessageDontCopyMessage) == 0;
        _message      = copyMessage ? [message copy] : message;
        _level        = level;
        _flag         = flag;
        _context      = context;

        BOOL copyFile = (options & DDLogMessageCopyFile) != 0;
        _file = copyFile ? [file copy] : file;

        BOOL copyFunction = (options & DDLogMessageCopyFunction) != 0;
        _function = copyFunction ? [function copy] : function;

        _line         = line;
        _tag          = tag;
        _options      = options;
        _timestamp    = timestamp ?: [NSDate new];

        if (USE_PTHREAD_THREADID_NP) {
            __uint64_t tid;
            pthread_threadid_np(NULL, &tid);
            _threadID = [[NSString alloc] initWithFormat:@"%llu", tid];
        } else {
            _threadID = [[NSString alloc] initWithFormat:@"%x", pthread_mach_thread_np(pthread_self())];
        }
        _threadName   = NSThread.currentThread.name;

        // Get the file name without extension
        _fileName = [_file lastPathComponent];
        NSUInteger dotLocation = [_fileName rangeOfString:@"." options:NSBackwardsSearch].location;
        if (dotLocation != NSNotFound)
        {
            _fileName = [_fileName substringToIndex:dotLocation];
        }
        
        // Try to get the current queue's label
        if (USE_DISPATCH_CURRENT_QUEUE_LABEL) {
            _queueLabel = [[NSString alloc] initWithFormat:@"%s", dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL)];
        } else if (USE_DISPATCH_GET_CURRENT_QUEUE) {
            #pragma clang diagnostic push
            #pragma clang diagnostic ignored "-Wdeprecated-declarations"
            dispatch_queue_t currentQueue = dispatch_get_current_queue();
            #pragma clang diagnostic pop
            _queueLabel = [[NSString alloc] initWithFormat:@"%s", dispatch_queue_get_label(currentQueue)];
        } else {
            _queueLabel = @""; // iOS 6.x only
        }
    }
    return self;
}

```

Lumberjack又分别定义了 `DDFileLogger` ` DDOSLogger` `DDTTYLogger` `DDASLLoger`来分别实现往不同地方输出Log的Logger类。可以见使用部分。

### 1.3 使用方法 

示例代码：
我们在全局某处增加log等级之后就可以添加相应的Logger,它的DDLog语法上和NSLog完全相同。

```

  //添加输出到Xcode控制台
  [[DDTTYLogger sharedInstance] setLogFormatter:formatter];
  [DDLog addLogger:[DDTTYLogger sharedInstance]];
  
  //添加输出到Console
  [[DDASLLogger sharedInstance] setLogFormatter:formatter];
  [DDLog addLogger:[DDASLLogger sharedInstance]];

  //添加文件输出
    DDFileLogger *fileLogger = [[DDFileLogger alloc] init];
    fileLogger.rollingFrequency = 60 * 60 * 24; // 一个LogFile的有效期长，有效期内Log都会写入该LogFile
    fileLogger.logFileManager.maximumNumberOfLogFiles = 7;//最多LogFile的数量
    [fileLogger setLogFormatter:formatter];
    [DDLog addLogger:fileLogger];

    //添加数据库输出
    DDAbstractDatabaseLogger *dbLogger = [[DDAbstractDatabaseLogger alloc] init];
    [fileLogger setLogFormatter:formatter];
    [DDLog addLogger:dbLogger];
```



