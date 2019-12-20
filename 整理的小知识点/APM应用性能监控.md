# APM应用性能监控

### 1. 崩溃
#### 1.1 应用崩溃的原因
* **数组越界**:在取数据索引时越界，App会发生崩溃。还有一种情况，就是给数组添加了`nil`会崩溃。
* **多线程问题**:在子线程中进行`UI`更新可能会发生崩溃。多个线程进行数据的读取操作，因为处理时机不一致，比如有一个 线程在置空数据的同时另一个线程在读取这个数据，可能会出现崩溃情况。
* **主线程无响应**:如果主线程超过系统规定的时间无响应，就会被`Watchdog`杀掉。这时，崩溃问题对应的异常编码是 `0x8badf00d`。
* **野指针**:指针指向一个已删除的对象访问内存区域时，会出现野指针崩溃。野指针问题是需要我们重点关注的，因为它是导致App崩溃的最常⻅，也是最难定位的一种情况。

![部分崩溃情况收集](https://github.com/binzi56/iOSSmallKnowledgePool/blob/master/整理的小知识点/resources/部分崩溃情况收集.png)

#### 1.2 如何收集崩溃
**信号可捕获的崩溃日志收集**
1. 提交`AppStore`时选上“`Upload your app’s symbols to receive symbolicated reports from Apple`”，以后你就可以直接在`Xcode`的`Archive`里看到符号化后的崩溃日志了
2. [PLCrashReporter](https://www.plcrashreporter.org/)(需要上传服务端);数据不敏感则用[Fabric](https://get.fabric.io/)或[Bugly](https://bugly.qq.com/v2/);

信号的种类有很多，但是都可以 通过注册`signalHandler`来捕获到。其实现代码，如下所示:
```
void registerSignalHandler(void) {
  signal(SIGSEGV, handleSignalException);
  signal(SIGFPE, handleSignalException);
  signal(SIGBUS, handleSignalException);
  signal(SIGPIPE, handleSignalException);
  signal(SIGHUP, handleSignalException);
  signal(SIGINT, handleSignalException);
  signal(SIGQUIT, handleSignalException);
  signal(SIGABRT, handleSignalException);
  signal(SIGILL, handleSignalException);
}

void handleSignalException(int signal) {
NSMutableString *crashString = [[NSMutableString alloc]init]; void* callstack[128];
int i, frames = backtrace(callstack, 128);
char** traceChar = backtrace_symbols(callstack, frames);

for (i = 0; i <frames; ++i) {
	[crashString appendFormat:@"%s\n", traceChar[i]];
	}
	NSLog(crashString);
}
```

**信号捕获不到的崩溃信息收集**

首先了解一下iOS`后台保活`的5种方式:
* 使用 `Background Mode`方式的话，App Store在审核时会提高对App 的要求。通常情况下，只有那些地图、音乐播放、 VoIP 类的 App 才能通过审核。
* `Background Fetch`方式的唤醒时间不稳定，而且用户可以在系统里设置关闭这种方式，导致它的使用场景很少。
* `Silent Push` 是推送的一种，会在后台唤起 App 30秒。它的优先级很低，会调用 application:didReceiveRemoteNotifiacation:fetchCompletionHandler: 这个 delegate，和普通的 remote push notification 推送调用的 delegate 是一样的。
* `PushKit` 后台唤醒 App 后能够保活30秒。它主要用于提升 VoIP 应用的体验。
* `Background Task` 方式，是使用最多的。App 退后台后，默认都会使用这种方式。

```
//Background Task方式的使用方法
- (void)applicationDidEnterBackground:(UIApplication *)application {
self.backgroundTaskIdentifier = [application beginBackgroundTaskWithExpirationHandler:^( void) {
        [self yourTask];
    }];
}
```
yourTask 任务最多执行3分钟，3分钟内 yourTask 运行完成，你的 App 就会挂起。 如果 yourTask 在3分钟之 内没有执行完的话，系统会强制杀掉进程，从而造成崩溃，这就是为什么App退后台容易出现崩溃的原因。

#### 1.3 解决崩溃方法
我们采集到的崩溃日志，主要包含的信息为:
* **进程信息**:崩溃进程的相关信息，比如崩溃报告唯一标识符、唯一键值、设备标识;
* **基本信息**:崩溃发生的日期、iOS 版本;
* **异常信息**:异常类型、异常编码、异常的线程;
* **线程回溯**:崩溃时的方法调用栈。

完整的崩溃日志里，除了线程方法调用栈还有异常编码。你可以在维基百科上，查看[完整的异常编码](https://en.wikipedia.org/wiki/Hexspeak)。这里列出了44种异常 编码，但常⻅的就是如下三种:
* `0x8badf00d`，表示 App 在一定时间内无响应而被`watchdog`杀掉的情况。
* `0xdeadfa11`，表示App被用户强制退出。
* `0xc00010ff`，表示App因为运行造成设备温度太高而被杀掉。
