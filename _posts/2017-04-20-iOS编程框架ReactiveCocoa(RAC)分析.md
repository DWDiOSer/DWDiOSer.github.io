---
layout: post
title: iOS编程框架ReactiveCocoa(RAC)分析
date: 2017-04-20 14:36:24.000000000 +09:00
---

# pxl

## RAC简介
>ReactiveCocoa(简称RAC),是一个函数响应式编程框架，是由GitHub开源的一个应用于iOS和OS开发的新框架，Cocoa是苹果整套框架的简称，因此很多苹果框架喜欢以Cocoa结尾。
>ReactiveCocoa组合了集中编程范式：
 >1. Functional Programming (函数式编程)通过高阶函数实现编码，例如接受其他函数座位函数的参数。
 2. Reactive Programming（响应式编程）通常专注于数据流和数据的改变。
>所以，经常会以函数响应式编程框架来描述ReactiveCocoa。

-------
>ReactiveCocoa v2.5是公认的OC最稳定的版本，因此被广大的以OC为主要语言的客户端选中使用。ReactiveCocoa v3.0及以后版本主要是基于Swift的版本。所以将以ReactiveCocoa v2.5版本做为例子，分析一下OC版的RAC具体实现。

#### ReactiveCocoa主要解决了一下问题
* UI数据绑定

> UI控件通常需要绑定一个事件，RAC可以很方便的绑定任何数据流到控件上。

* 用户交互事件绑定

>RAC为可交互的UI控件提供了一系列 能发送Signal信号的方法。这些数据流会在用户交互中相互传递。

* 解决状态以及状态之间依赖过多的为题

>有了RAC的绑定之后，可以不用再关心各种复杂的状态，及依赖关系，也解决了这些状态在后期很难维护的问题。

* 消息传递机制的大统一

>OC中原来的消息传递机制有以下几种：Delegate、Block、Target-Action、Timer、KVO等，有了RAC以后，以上这5中方式都可以统一用RAC来处理了。

## RAC v2.5源码分析
### 一、关于常见类
#### 1、 RACSignal 的使用

~~~
/**
 RACSiganl简单使用
 */
- (void)testRACSignal {
    //1、创建信号
    RACSignal *signal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        //3、1 发送Next信号
        [subscriber sendNext:@"RACSiganl"];
        //3、2 发送信号完成,并取消订阅
        [subscriber sendCompleted];
        //4、用于取消订阅时清理资源用，比如释放一些资源
        return [RACDisposable disposableWithBlock:^{
            NSLog(@"信号被销毁");
        }];
    }];
    
    //2、1 订阅Next信号
    [signal subscribeNext:^(id x) {
        NSLog(@"接收到的数据: %@",x);
    }];
    
    //2、2 订阅Error信号
    [signal subscribeError:^(NSError *error) {
        NSLog(@"error");
    }];
    
    //2、3 订阅Completed信号
    [signal subscribeCompleted:^{
        NSLog(@"Completed");
    }];
}
~~~

* 完成一个信号的生命周期分为四步：

**1、 创建信号**

>有上面代码块可知，创建信号类方法中传入了一个返回值是RACDisposable类型，参数是遵循RACSubscriber协议的名为subscriber的block。


~~~
+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe {
    //创建的是RACDynamicSignal类型的信号
	return [RACDynamicSignal createSignal:didSubscribe];
}
~~~

>这里首先先创建了一个 RACDynamicSignal 类型的信号，然后将传入的名为 didSubscribe 的block保存在创建的信号的 didSubscribe 属性中，此时仅仅是保存并未触发。

~~~
+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe {
	RACDynamicSignal *signal = [[self alloc] init];
    //重点：保存block
	signal->_didSubscribe = [didSubscribe copy];
	return [signal setNameWithFormat:@"+createSignal:"];
}
~~~

* 创建信号本质就是创建了一个 RACDynamicSignal 类型的信号，并将传入的代码块保存起来，留待以后调用。
      

**2、订阅信号**

>有三种订阅信号的方式，分别是订阅 Next 、Error 、 completed。内部实现如下

~~~
+ (instancetype)subscriberWithNext:(void (^)(id x))next error:(void (^)(NSError *error))error completed:(void (^)(void))completed {
	RACSubscriber *subscriber = [[self alloc] init];
    //将传入的block分别保存到相应的属性
	subscriber->_next = [next copy];
	subscriber->_error = [error copy];
	subscriber->_completed = [completed copy];

	return subscriber;
}
~~~

* 很明显，创建订阅者的实质是，创建一个订阅者，并保存相应的block，比如 next、error、或者complete，此时仅仅是保存并未触发！

>执行订阅命令

~~~
- (RACDisposable *)subscribe:(id<RACSubscriber>)subscriber {
	NSCParameterAssert(subscriber != nil);

	RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];
	subscriber = [[RACPassthroughSubscriber alloc] initWithSubscriber:subscriber signal:self disposable:disposable];

	if (self.didSubscribe != NULL) {
		RACDisposable *schedulingDisposable = [RACScheduler.subscriptionScheduler schedule:^{
            //执行了信号保存的didSubscribe block，并传入了刚生成的订阅者
			RACDisposable *innerDisposable = self.didSubscribe(subscriber);
			[disposable addDisposable:innerDisposable];
		}];

		[disposable addDisposable:schedulingDisposable];
	}
	
	return disposable;
}
~~~

* 订阅信号本质就是创建了一个 RACPassthroughSubscriber 类型的订阅者，并将传入的代码块保存起来，留待以后调用，同时调用了第一步创建信号中保存的代码块，并传入创建的订阅者。


**3、发送信号**

>发送信号就是执行订阅信号时对应的block。

~~~
- (void)sendNext:(id)value {
	@synchronized (self) {
		void (^nextBlock)(id) = [self.next copy];
		if (nextBlock == nil) return;

		nextBlock(value);
	}
}

- (void)sendError:(NSError *)e {
	@synchronized (self) {
		void (^errorBlock)(NSError *) = [self.error copy];
		[self.disposable dispose];//先取消订阅，再执行相应的block

		if (errorBlock == nil) return;
		errorBlock(e);
	}
}

- (void)sendCompleted {
	@synchronized (self) {
		void (^completedBlock)(void) = [self.completed copy];
		[self.disposable dispose];//先取消订阅，再执行相应的block

		if (completedBlock == nil) return;
		completedBlock();
	}
}
~~~

* 第一、发送信号就是执行相应block，此处执行的就是第二步中保存的相应的block。
* 第二、对于 sendError 和 sendCompleted 都是先取消订阅，再执行相应的代码块，而 sendNext 并未使订阅结束，这样的话，对之后讨论的各种组合方法中必须写上 sendCompleted来结束订阅的做法就好理解了。
* 第三、我们也看到三种方法中，假如信号没有相应的block代码块保存，即没有经过第二步去订阅保存代码块，就算是发送了信号也不会执行，此时也就是冷热信号的区别，当然用 RACSubject 来解释更容易理解。

**4、取消订阅**

>这里我们会遇到RAC的四大组件之一的调度器RACScheduler，本质上，它就是用 GCD 的串行队列来实现的，并且支持取消操作。也只是对 GCD 的简单封装而已。

~~~
- (RACDisposable *)schedule:(void (^)(void))block {
	NSCParameterAssert(block != NULL);
    //执行订阅的代码块实际上是放到相应的调度器中去执行的
	if (RACScheduler.currentScheduler == nil) return [self.backgroundScheduler schedule:block];

	block();
	return nil;
}

- (RACDisposable *)schedule:(void (^)(void))block {
	NSCParameterAssert(block != NULL);

	RACDisposable *disposable = [[RACDisposable alloc] init];
    //放在调度中，加入调度到这里，disposed，则不再执行
	dispatch_async(self.queue, ^{
		if (disposable.disposed) return;
		[self performAsCurrentScheduler:block];
	});

	return disposable;
}
~~~

* 取消订阅就是把订阅信号获得的disposable 进行dispose即可在调度器调度该部分代码之前禁止调用。

#### 2、RACSubject 的使用

>RACSubject 代表的是可以手动控制的信号，我们可以把它看作是 RACSignal 的可变版本，就好比 NSMutableArray 是 NSArray 的可变版本一样。RACSubject 继承自 RACSignal ，所以它可以作为信号源被订阅者订阅，同时，它又实现了 RACSubscriber 协议，所以它也可以作为订阅者订阅其他信号源


```
- (void)RACSubjectUsed {
    //创建信号
    RACSubject *subject = [RACSubject subject];
    
    //订阅信号
    [subject subscribeNext:^(id x) {
        NSLog(@"第一个订阅者: %@",x);
    }];
    
    [subject subscribeNext:^(id x) {
        NSLog(@"第二个订阅者: %@",x);
    }];
    
    //发送信号
    [subject sendNext:@1];
    [subject sendNext:@12];
}
```

**1、创建信号**
>初始化相应的一个disposable及订阅者数组

```
+ (instancetype)subject {
	return [[self alloc] init];
}

- (id)init {
	self = [super init];
	if (self == nil) return nil;

	_disposable = [RACCompoundDisposable compoundDisposable];
	_subscribers = [[NSMutableArray alloc] initWithCapacity:1];
	
	return self;
}
```

**2、订阅信号**
>RACSinal订阅的时候直接执行了订阅命令，执行的是创建时保存的代码块，但RACSubject的做法是将订阅者保存到初始化时生成的那个订阅者数组内，此时每个订阅者都包含其对应的代码块（比如：next、error、complete）。
>由此可以知道，RACSubject是可以多次订阅的，多次订阅就是把相应包含代码块的订阅者放入订阅者数组内。
>另外，此处对于取消订阅做了一些清理工作，将保存的订阅者都移除。
>

```
- (RACDisposable *)subscribe:(id<RACSubscriber>)subscriber {
	NSCParameterAssert(subscriber != nil);

	RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];
	subscriber = [[RACPassthroughSubscriber alloc] initWithSubscriber:subscriber signal:self disposable:disposable];

	NSMutableArray *subscribers = self.subscribers;
	@synchronized (subscribers) {
		[subscribers addObject:subscriber];
	}
	
	return [RACDisposable disposableWithBlock:^{
		@synchronized (subscribers) {
			// Since newer subscribers are generally shorter-lived, search
			// starting from the end of the list.
			NSUInteger index = [subscribers indexOfObjectWithOptions:NSEnumerationReverse passingTest:^ BOOL (id<RACSubscriber> obj, NSUInteger index, BOOL *stop) {
				return obj == subscriber;
			}];

			if (index != NSNotFound) [subscribers removeObjectAtIndex:index];
		}
	}];
}
```

* 订阅信号就是将代码块保存到订阅者中，并将订阅者添加到信号的订阅者数组中。

**3、发送信号**

>RACSubject的发送信号就是遍历自己的订阅者数组，然后分别发送信号。
>这也是对月RACsubjec为什么假如有多个订阅者，发送一个信号所有的订阅者都收到的原因。


```
- (void)sendNext:(id)value {
	[self enumerateSubscribersUsingBlock:^(id<RACSubscriber> subscriber) {
		[subscriber sendNext:value];
	}];
}
```

* 发送信号就是遍历自己的订阅者数组，然后分别发送信号，并执行该订阅者保存的代码块。

#### 3、RACCommand 的使用
>这个类比较复杂，因为他是基于信号的一个对象，里面绑定了很多相关的信号，只是讲一讲他的执行原理，及常用的一些方法的分析。



```
- (void)testRACCommand {
    
    //1、创建命令（返回的是信号）
    RACCommand *command = [[RACCommand alloc] initWithSignalBlock:^RACSignal *(id input) {
        return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
            [subscriber sendNext:input];
            [subscriber sendCompleted];
            return [RACDisposable disposableWithBlock:^{
                
            }];
        }];
    }];
    
    //2、订阅命令发出的信号
    [command.executionSignals.switchToLatest subscribeNext:^(id x) {
        NSLog(@"command   %@",x);
    }];
    
    //3、判断命令是否正在执行
    [command.executing subscribeNext:^(id x) {
        NSLog(@"executing   %@",x);
    }];
    
    //4、执行命令
    [command execute:@"dianwoda"];
}
```

**1、创建命令**

```
- (id)initWithSignalBlock:(RACSignal * (^)(id input))signalBlock {
	return [self initWithEnabled:nil signalBlock:signalBlock];
}

- (id)initWithEnabled:(RACSignal *)enabledSignal signalBlock:(RACSignal * (^)(id input))signalBlock {
	NSCParameterAssert(signalBlock != nil);

	self = [super init];
	if (self == nil) return nil;
    //额外创建了一个名为 _activeExecutionSignals 的信号数组，并把传入的block保存为 signalBlock 属性
	_activeExecutionSignals = [[NSMutableArray alloc] init];
	_signalBlock = [signalBlock copy];
}
```

* 创建命令就是创建了一个RACCommand类的对象，并将传入的block保存为 signalBlock 属性，然后初始化了一个信号数组，留待以后接收命令信号。

**2、订阅命令发出的信号**

```
// A signal of additions to `activeExecutionSignals`.
	RACSignal *newActiveExecutionSignals = [[[[[self
		rac_valuesAndChangesForKeyPath:@keypath(self.activeExecutionSignals) options:NSKeyValueObservingOptionNew observer:nil]
		reduceEach:^(id _, NSDictionary *change) {
			NSArray *signals = change[NSKeyValueChangeNewKey];
			if (signals == nil) return [RACSignal empty];

			return [signals.rac_sequence signalWithScheduler:RACScheduler.immediateScheduler];
		}]
		concat]
		publish]
		autoconnect];

	_executionSignals = [[[newActiveExecutionSignals
		map:^(RACSignal *signal) {
			return [signal catchTo:[RACSignal empty]];
		}]
		deliverOn:RACScheduler.mainThreadScheduler]
		setNameWithFormat:@"%@ -executionSignals", self];
```

* executionSignals实际上就是最新的第一步创建命令中创建的activeExecutionSignals信号数组。
* switchToLatest实质上取信号数组的最新的信号。

**3、判断命令是否执行**
>与executionSignals类似，在创建命令时也创建了一个信号来监听当前命令是否正在执行。

```
_executing = [[[[[immediateExecuting
		deliverOn:RACScheduler.mainThreadScheduler]
		
		startWith:@NO]
		distinctUntilChanged]
		replayLast]
		setNameWithFormat:@"%@ -executing", self];
```

* executing信号实际上是在检测是否有活跃信号，replayLast能确保是最新的值。
* 判断是否执行命令就是检测是否有活跃信号。

**4、执行命令**

```
- (RACSignal *)execute:(id)input {
	BOOL enabled = [[self.immediateEnabled first] boolValue];
	if (!enabled) {
		NSError *error = [NSError errorWithDomain:RACCommandErrorDomain code:RACCommandErrorNotEnabled userInfo:@{
			NSLocalizedDescriptionKey: NSLocalizedString(@"The command is disabled and cannot be executed", nil),
			RACUnderlyingCommandErrorKey: self
		}];

		return [RACSignal error:error];
	}

	RACSignal *signal = self.signalBlock(input);
	NSCAssert(signal != nil, @"nil signal returned from signal block for value: %@", input);

	RACMulticastConnection *connection = [[signal
		subscribeOn:RACScheduler.mainThreadScheduler]
		multicast:[RACReplaySubject subject]];
	
	@weakify(self);

	[self addActiveExecutionSignal:connection.signal];
	[connection.signal subscribeError:^(NSError *error) {
		@strongify(self);
		[self removeActiveExecutionSignal:connection.signal];
	} completed:^{
		@strongify(self);
		[self removeActiveExecutionSignal:connection.signal];
	}];

	[connection connect];
	return [connection.signal setNameWithFormat:@"%@ -execute: %@", self, [input rac_description]];
}
```
* 执行命令先执行了第一步保存的signalBlock并传入接收到的参数，然后把得到的signal加入到第一步创建的 _activeExecutionSignals 信号数组中。

### 二、常用操作方法
#### 1、核心方法 bind

```
- (void)testBind {
    
    //1、创建源信号
    RACSignal *signal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        //4、发送源信号
        [subscriber sendNext:@"dianwoda"];
        [subscriber sendCompleted];
        return [RACDisposable disposableWithBlock:^{
            
        }];
    }];
    
    //2、绑定源信号，生成绑定信号
    RACSignal *bindSignal = [signal bind:^RACStreamBindBlock{
        return ^RACSignal *(id value, BOOL *stop) {
            return [RACReturnSignal return:[NSString stringWithFormat:@"%@111",value]];
        };
    }];
    
    //3、订阅绑定信号
    [bindSignal subscribeNext:^(id x) {
        NSLog(@"bind   %@",x);
    }];
}
```

**1、创建信号源**

* 创建了一个 RACDynamicSignal 类型的信号，并将传入的代码块保存起来，留待以后调用。

**2、绑定源信号，生成绑定信号**

1.  bind操作实际上直接返回了一个绑定信号信号，并将 didSubscriber 代码块 传入，保存到返回的信号内。
2.  将bind方法传入的block保存为bindinfBlock，留待以后用，并且初始化了一个信号数组。
3.  在代码块内部订阅了源信号，订阅源信号传入的block，被保存在相应订阅者中，留待源信号发送信号触发。

* bind操作实际上是直接生成绑定信号并返回，并且在生成绑定信号传入的didSubscriber block代码块中，保存了bind传入的block，初始化了信号数组，并且订阅了源信号，针对源信号发送信号的流程做了一些处理。（此时未执行，订阅才执行）

**3、订阅绑定信号**

* 订阅绑定信号就是保存了nextBlock,并且创建订阅者，实现信号的didSubscriber block代码块。

**4、发送信号**

* bind操作方法实质上就是生成新的绑定信号，利用returnSignal作为中间信号来改变源数据生成新的数据并执行新绑定信号的nextBlock代码块！

#### 2、映射方法 map

```
- (void)testMap {
    
    //1、创建信号
    RACSignal *signal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        //4、发送信号
        [subscriber sendNext:@"点我达"];
        [subscriber sendCompleted];
        return [RACDisposable disposableWithBlock:^{
            
        }];
    }];
    
    //2、映射信号
    RACSignal *mapSignal = [signal map:^id(id value) {
        return [NSString stringWithFormat:@"map %@",value];
    }];
    
    //3、订阅映射信号
    [mapSignal subscribeNext:^(id x) {
        NSLog(@"%@",x);
    }];
}
```

>与 bind 的区别就是生成绑定信号的过程不同


```
- (instancetype)flattenMap:(RACStream * (^)(id value))block {
	Class class = self.class;

	return [[self bind:^{
		return ^(id value, BOOL *stop) {
			id stream = block(value) ?: [class empty];
			NSCAssert([stream isKindOfClass:RACStream.class], @"Value returned from -flattenMap: is not a stream: %@", stream);

			return stream;
		};
	}] setNameWithFormat:@"[%@] -flattenMap:", self.name];
}
```
>在map方法里，返回的是**flattenMap**方法生成的信号，而 flattenMap 内部返回的是**bind**方法返回的信号。

* map 映射方法的实质就是 bind 方法的深度封装，实际的信号流的流程还是以 bind为核心，只是巧妙的做了一些逻辑处理而已。

## RAC与MVVM
>学以致用，来个简单的实例，来感受一下如何在MVVM中使用RactiveCocoa。
>重点在于如何在MVVM各层之间使用RAC的信号量来更方便的在各个层之间进行响应式数据交互。

**以登录界面为例**
1. 用于输入账号和密码的输入框 usernameTextField 和 passwordTextField ；
2. 一个直接登录的按钮 loginButton。

* 分析：viewModel 需要提供 view 所需的数据和命令。

>LoginViewModel.h 文件

```
@interface LoginViewModel : NSObject

@property (nonatomic, copy) NSString *username;
@property (nonatomic, copy) NSString *password;

@property (nonatomic, strong, readonly) RACSignal *validLoginSignal;
@property (nonatomic, strong, readonly) RACCommand *loginCommand;
@property (nonatomic, strong) RACSubject *successSubject;

@end
```

>LoginViewModel.m 文件


```
@implementation LoginViewModel

- (instancetype)init {
    self = [super init];
    if (self) {
        [self initialize];
    }
    return self;
}

- (void)initialize {
    
    self.validLoginSignal = [[RACSignal combineLatest:@[RACObserve(self, username), RACObserve(self, password)] reduce:^(NSString *username, NSString *password){
        return @(username.length > 0 && password.length > 0);
    }] distinctUntilChanged];
    
    self.successSubject = [RACSubject subject];
    @weakify(self)
    self.loginCommand = [[RACCommand alloc] initWithSignalBlock:^RACSignal *(id input) {
        @strongify(self)
        NSLog(@"%@",input);
        [self.successSubject sendNext:@[self.username, self.password]];
        return [RACSignal empty];
    }];

}

@end
```

* 当用户输入的用户名或密码发生变化时，判断用户名和密码的长度是否均大于 0 ，如果是则登录按钮可用，否则不可用;
* 当 loginCommand 命令执行成功时，调用 block，将用户名和密码发送出去，并进入登录成功页。

>LoginViewController.m 文件核心代码

```
- (void)bindViewModel {
    self.viewModel = [[LoginViewModel alloc] init];
    
    RAC(self.viewModel, username) = self.usernameTextField.rac_textSignal;
    RAC(self.viewModel, password) = self.passwordTextField.rac_textSignal;
    RAC(self.loginButton, enabled) = self.viewModel.validLoginSignal;
    
    @weakify(self)
    [RACObserve(self.loginButton, enabled) subscribeNext:^(id x) {
        @strongify(self)
        self.loginButton.backgroundColor = [x boolValue] ? COLOR : COLOR1;
    }];
    
    [[self.loginButton rac_signalForControlEvents:UIControlEventTouchUpInside] subscribeNext:^(id x) {
        @strongify(self)
        [self.viewModel.loginCommand execute:@"loginButton"];
    }];
    
    [self.viewModel.successSubject subscribeNext:^(id x) {
        @strongify(self)
        LoginSuccessViewController *vc = [[LoginSuccessViewController alloc] initWithNibName:@"LoginSuccessViewController" bundle:nil];
        [self.navigationController pushViewController:vc animated:YES];
    }];
}
```

* 观察 loginButton 中 enabled 属性的变化，然后设置loginButton的背景颜色;
* 将 viewModel 中的 username 和 password 属性分别与 usernameTextField 和 passwordTextField 输入框中的内容进行绑定；
* 将 loginButton 的 enabled 属性与 viewModel 的 validLoginSignal 属性进行绑定；
* 在 loginButton 按钮被点击时分别执行 loginCommand 命令。

**我们要能够分析清楚 viewModel 需要暴露给 view 的数据和命令，这些数据和命令能够代表 view 当前的状态。**

## RAC常见用法和宏

### 一、常见用法
#### 1、代替代理

>rac_signalForSelector：用于替代代理。
>原理：判断一个方法有没有调用，如果调用了就会自动发送一个信号。

* rac_signalForSelector:把调用某个对象的方法的信息转换成信号，就要调用这个方法，就会发送信号。

```
[[self rac_signalForSelector:@selector(loginBtnClick:)] subscribeNext:^(id x) {
        NSLog(@"点击登录按钮");
}];
```

#### 2、代替KVO

>rac_valuesAndChangesForKeyPath：用于监听某个对象的属性改变。
>方法调用者:就是被监听的对象。
>KeyPath:监听的属性。

* 把监听redV的center属性改变转换成信号，只要值改变就会发送信号，observer:可以传入nil。


```
[[self rac_valuesAndChangesForKeyPath:@"title" options:NSKeyValueObservingOptionNew observer:nil]  subscribeNext:^(id x) {
        NSLog(@"title == %@",x);
}];
```

#### 3、监听事件

>rac_signalForControlEvents：用于监听某个事件。

* 把按钮点击事件转换为信号，点击按钮，就会发送信号。

```
[[self.button rac_signalForControlEvents:UIControlEventTouchUpInside] subscribeNext:^(id x) {
        NSLog(@"按钮被点击了");
}];
```

#### 4、代替通知

>rac_addObserverForName:用于监听某个通知。

* 只要发出这个通知，又会转换成一个信号。

```
[[[NSNotificationCenter defaultCenter] rac_addObserverForName:UIKeyboardWillShowNotification object:nil] subscribeNext:^(id x) {
   NSLog(@"弹出键盘");
}];
```

#### 5、监听文本框文字改变
>rac_textSignal:只要文本框发出改变就会发出这个信号。

* 获取文本框文字改变的信号。

```
[self.textfield.rac_textSignal subscribeNext:^(id x) {
   NSLog(@"%@", x);
}];
```

### 二、常见宏

#### 1、RAC(TARGET, ...)

>以前为 RAC(TARGET, [KEYPATH, [NIL_VALUE]])

* 用于给某个对象的某个属性绑定。把一个对象的某个属性绑定一个信号,只要发出信号,就会把信号的内容给对象的属性赋值。
* 实现了一个属性的值跟随另一个属性值的改变而改变。

```
RAC(self.viewModel, username) = self.usernameTextField.rac_textSignal;
```
>这个宏是最常用的，RAC()总是出现在等号左边，等号右边是一个RACSignal，表示的意义是将一个对象的一个属性和一个signal绑定，signal每产生一个value（id类型），都会自动执行：

```
[TARGET setValue:value ?: NIL_VALUE forKeyPath:KEYPATH];
```

#### 2、RACObserve(TARGET, KEYPATH)

>作用是观察TARGET的KEYPATH属性，相当于KVO，产生一个RACSignal

* 用于给某个对象的某个属性绑定。
* 快速的监听某个对象的某个属性改变。
* 返回的是一个信号,对象的某个属性改变的信号。

```
@weakify(self)
[RACObserve(self.loginButton, enabled) subscribeNext:^(id x) {
    @strongify(self)
    self.loginButton.backgroundColor = [x boolValue] ? [UIColor lightGrayColor] : [UIColor orangeColor];
}];
```

#### 3、strongify(...) / weakify(...)

>他们的作用主要是在block内部管理对self的引用

* 前面加@，其实就是一个什么都没干的@autoreleasepool {}前面的那个@，为了显眼。
* 这两个宏一定成对出现，先weak再strong

```
@weakify(self); // 定义了一个__weak的self_weak_变量
[RACObserve(self, name) subscribeNext:^(NSString *name) {
    @strongify(self); // 局域定义了一个__strong的self指针指向self_weak
    self.outputLabel.text = name;
}];
```

#### 4、RACTuplePack和RACTupleUnpack

* RACTuplePack：把数据包装成RACTuple（元组类）。把包装的类型放在宏的参数里面，就会自动包装。
* RACTupleUnpack：把RACTuple（元组类）解包成对应的数据。等号的右边表示解析哪个元组。
宏的参数:表示解析成什么类型。

```
RACTuple *tuple = RACTuplePack(@1,@3);
NSLog(@"%@", tuple);
    
RACTupleUnpack(NSNumber *num1,NSNumber *num2) = tuple;
NSLog(@"%@ %@", num1, num2);
```

## 注：

1. RACDemo：链接后面加上
2. RAC的中文资源整合：https://github.com/ReactiveCocoaChina/ReactiveCocoaChineseResources
3. RAC资源帖：https://juejin.im/post/58161485128fe100558f7054
4. RAC与MVVM结合的上线开源项目(MVVMReactiveCocoa)：https://github.com/leichunfeng/MVVMReactiveCocoa






