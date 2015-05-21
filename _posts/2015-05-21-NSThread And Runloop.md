---
layout: post
title: 【IOS多线程02】- NSThread And Runloop
comments: false
---

###NSThread

* [http://www.brighttj.com/ios/ios-multithreading-nsthread.html](http://www.brighttj.com/ios/ios-multithreading-nsthread.html)
* [http://daition340.github.io/blog/2014/05/18/multi-threading-programming-of-ios-part-1/](http://daition340.github.io/blog/2014/05/18/multi-threading-programming-of-ios-part-1/)
* [http://git.devzeng.com/blog/ios-multithead-of-nsthread.html](http://git.devzeng.com/blog/ios-multithead-of-nsthread.html)

具体我就不介绍了，NSTread是苹果推荐使用位于顶层的线程工具。优点是简单好用，缺点是你需要控制他的生命周期(复杂任务情况下)。

###NSThread & Run Loop
[前边](http://shuvigoss.github.io/blogs/2015/05/20/About%20Runloop.html)我们看到了，每一个Thread都有一个NSRunLoop，除了Main Thread的RunLoop是默认开启的。其他子线程都没有Run。

我们什么时候在子线程中要用到NSRunLoop呢？我想到这么一个场景，如果一个长时间在后台跑的一个任务，可以使用。那么我们设计这么一个场景吧。

假如我们要在当前应用上加入论坛功能，正常流程是看帖子、发帖子。如果在发帖子过程由于服务端，或者客户端网络问题，一直无法发送成功，但是你又不能告诉用户你发不了，这样用户体验就很差。

设计这样一个流程来完善论坛的异常情况。

发帖子-->失败--->启动一个线程尝试重发几遍--->失败--->存入本机 

启动一个线程，一直扫描在本机中未发送出去的帖子，每1分钟发一次。

这个流程还不错，那么怎么实现呢？

1、在Main Thread中使用`[NSTimer scheduledTimerWithTimeInterval:60 target:self selector:@selector(runloopSendMessage:) userInfo:nil repeats:YES];`但是这样会对主线程带来很大的压力，而且如果在selector中有线程的阻塞，会影响到主线程。pass

2、启动一个子线程 使用sleep(60)来扫描待发送的帖子，这样写肯定没问题，如果后期我还想增加一个扫描通知的功能，那么对于这个线程的改造量很大。pass

3、启动子线程，配合NSTimer进行scheduled，高大上很多。来看看怎么写呢？

```
//定义一个MessageScan类，提供start stop用于启动和停止扫描。

@interface MessageScan : NSObject

@property (nonatomic, strong) NSThread *scanThread;

+ (instancetype) shareInstance;

- (void)startScan;
- (void)stopScan;
@end

```

```
@implementation MessageScan

//单例，保证唯一
+ (instancetype) shareInstance{
  static dispatch_once_t onceToken;
  static id _instance = nil;
  dispatch_once(&onceToken, ^{
    _instance = [MessageScan new];
  });
  return _instance;
}

//保证线程的安全
- (void)startScan{
  @synchronized(self){
    //初始化扫描线程并且启动
    if (!self.scanThread) {
      self.scanThread = [[NSThread alloc] initWithTarget:self selector:@selector(startScanInThread:) object:nil];
      self.scanThread.name = @"message scan thread";
      [self.scanThread start];
    }
  }
}

- (void)startScanInThread:(id) obj{
  //执行扫描，每3秒扫描一次
  @autoreleasepool {
    [NSTimer scheduledTimerWithTimeInterval:3 target:self selector:@selector(doScan:) userInfo:nil repeats:YES];
    //启动当前线程默认的RunLoop
    CFRunLoopRun();
  }
}

- (void)doScan:(id) obj{
  //具体要做的实情
  NSLog(@"scan....");
}

- (void)stopScan{
  @synchronized(self){
    //停止线程前先停止RunLoop。
    [self performSelector:@selector(doStopScan:) onThread:self.scanThread withObject:nil waitUntilDone:YES];
    if (![self.scanThread isCancelled]) {
      [self.scanThread cancel];
    }
    self.scanThread = nil;
  }
}

- (void)doStopScan:(id) obj{
  NSLog(@"stop scan...");
  //停止RunLoop
  CFRunLoopStop(CFRunLoopGetCurrent());
}

@end

```

这样 我们只要在AppDelegate中启动MessageScan就可以了

```
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
  [[MessageScan shareInstance] startScan];
  return YES;
}
```

如果想在应用进入后台后停止，我们只要在相应地方停止掉MessageScan就可以了

```
- (void)applicationDidEnterBackground:(UIApplication *)application {
  [[MessageScan shareInstance] stopScan];
}
```

不过我建议将启动以及停止放在MessageScan中，通过NotificationCenter来控制，这样控制权在MessageScan内，修改代码时清楚明了。

```
UIKIT_EXTERN NSString *const UIApplicationDidEnterBackgroundNotification       NS_AVAILABLE_IOS(4_0);
UIKIT_EXTERN NSString *const UIApplicationWillEnterForegroundNotification      NS_AVAILABLE_IOS(4_0);
UIKIT_EXTERN NSString *const UIApplicationDidFinishLaunchingNotification;
UIKIT_EXTERN NSString *const UIApplicationDidBecomeActiveNotification;
UIKIT_EXTERN NSString *const UIApplicationWillResignActiveNotification;
UIKIT_EXTERN NSString *const UIApplicationDidReceiveMemoryWarningNotification;
UIKIT_EXTERN NSString *const UIApplicationWillTerminateNotification;
...
```

好了，完成了，是不是很帅呢？