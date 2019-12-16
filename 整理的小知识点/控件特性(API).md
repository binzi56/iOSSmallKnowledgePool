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

[timer和runloop关系](https://github.com/binzi56/iOSSmallKnowledgePool/blob/master/整理的小知识点/resources/runloop与timer关系.png)

* [NSTimer的使用](https://www.jianshu.com/p/3ccdda0679c1)

## 
