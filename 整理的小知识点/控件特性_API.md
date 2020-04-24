# 控件特性(API)


## NSTimer
```
// 创建一个timer并把它指定到一个默认的runloop模式中，并且在 TimeInterval时间后 启动定时器
+ (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)ti target:(id)aTarget selector:(SEL)aSelector userInfo:(nullable id)userInfo repeats:(BOOL)yesOrNo;

// 默认的初始化方法，（创建定时器后，手动添加到 运行循环，并且手动触发才会启动定时器）
- (instancetype)initWithFireDate:(NSDate *)date interval:(NSTimeInterval)ti target:(id)t selector:(SEL)s userInfo:(nullable id)ui repeats:(BOOL)rep NS_DESIGNATED_INITIALIZER;

// 这是设置定时器的启动时间，常用来管理定时器的启动与停止
@property (copy) NSDate *fireDate;
      // 启动定时器
          timer.fireDate = [NSDate distantPast];    
      //停止定时器
          timer.fireDate = [NSDate distantFuture];
      // 开启
         [time setFireDate:[NSDate  distanPast]]
      // NSTimer   关闭  
        ［time  setFireDate:[NSDate  distantFunture]］
      //继续
        [timer setFireDate:[NSDate date]];

```

**NSTimer加到了RunLoop中但迟迟的不触发事件**
1. 是否添加到runloop
2. runloop是否运行
3. mode是否正确


eg:
```
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view.

    dispatch_queue_t queue = dispatch_queue_create("com.appple.cn", DISPATCH_QUEUE_CONCURRENT);
    dispatch_async(queue, ^{
        [NSTimer scheduledTimerWithTimeInterval:1.0 target:self selector:@selector(action:) userInfo:nil repeats:YES];
    });

}

- (void)action:(NSTimer *)timer
{
    NSLog(@"action:%@", timer);
}
```
**timer和runloop关系**

![timer和runloop关系](./resources/runloop与timer关系.png)

* [NSTimer的使用](https://www.jianshu.com/p/3ccdda0679c1)

## 多线程的坑点
#### 1.常驻线程
```
+ (void)networkRequestThreadEntryPoint:(id)__unused object {
  @autoreleasepool {
      // 先用 NSThread 创建了一个线程
      [[NSThread currentThread] setName:@"AFNetworking"];
      // 使用 run 方法添加 runloop
      NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
      [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
      [runLoop run];
} }
```

坑点:
> 浪费CPU 资源, 降低CPU的利用率

**既然常线程是个坑，那为什么 AFNetworking 2.0 库还要这么做呢?**
由于`AFNetworking 2.0` 使用的是 `NSURLConnection`，而`NSURLConnection`发起请求后，所在的线程需要一直存活，以等待接收 `NSURLConnectionDelegate`回调方法。但是，网络 返回的时间不确定，所以这个线程就需要一直常驻在内存中.
而且主线程还要**处理大量的UI和交互工作**, 所以得重新创建一个常住线程来满足需求;

`AFNetworking 3.0`版本时，使用苹果公司新推出的 `NSURLSession` 替换了 `NSURLConnection`，从而避免了常驻线程这个坑;

如果你需要确实需要 **保活线程一段时间** 的话，可以选择使用 `NSRunLoop` 的另外两个方法 `runUntilDate:` 和 `runMode:beforeDate`，来指定线程的保活时⻓。让线程存活时间可预期，总比让线程常驻，至少在硬件资源利用率这点上要 更加合理。
或者，你还可以使用 `CFRunLoopRef` 的 `CFRunLoopRun` 和 `CFRunLoopStop` 方法来完成 `runloop` 的开启和停止，达到将线程保活一段时间的目的。

#### 2.并发
