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

### 2. 卡顿
#### 2.1 原因
* 复杂 UI 、图文混排的绘制量过大;
* 在主线程上做网络同步请求;
* 在主线程做大量的IO 操作;
* 运算量过大，CPU持续高占用;
* 死锁和主子线程抢锁。

#### 2.2 RunLoop监测卡顿
#### 2.2.1 原理
> **第一步**
通知 `observers`:`RunLoop` 要开始进入 `loop` 了。紧接着就进入 `loop`。

代码如下:
```
//通知 observers
if (currentMode->_observerMask & kCFRunLoopEntry )
__CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopEntry);
//进入 loop
result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
```
> **第二步**
开启一个 `do while` 来保活线程。通知 `Observers`:`RunLoop` 会触发 `Timer` 回调、`Source0` 回调，接着执行加入的 `block`。

代码如下:
```
// 通知 Observers RunLoop 会触发 Timer 回调
if (currentMode->_observerMask & kCFRunLoopBeforeTimers)
__CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeTimers);

// 通知 Observers RunLoop 会触发 Source0 回调
if (currentMode->_observerMask & kCFRunLoopBeforeSources)
__CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeSources);

// 执行 block
__CFRunLoopDoBlocks(runloop, currentMode);
```
接下来，触发 `Source0` 回调，如果有 `Source1` 是 `ready` 状态的话，就会跳转到 `handle_msg`去处理消息。代码如下:
```
if (MACH_PORT_NULL != dispatchPort ) {
    Boolean hasMsg = __CFRunLoopServiceMachPort(dispatchPort, &msg)
    if (hasMsg) goto handle_msg;
}
```

> **第三步**
回调触发后，通知 `Observers`:`RunLoop`的线程将进入休眠(`sleep`)状态。

代码如下:
```
Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);
if (!poll && (currentMode->_observerMask & kCFRunLoopBeforeWaiting)) {
    __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeWaiting);
}
```

> **第四步**
进入休眠后，会等待 `mach_port` 的消息，以再次唤醒。只有在下面四个事件出现时才会被再次唤醒:
* 基于 `port` 的 `Source` 事件
* `Timer` 时间到
* `RunLoop` 超时
* 被调用者唤醒

等待唤醒的代码如下:
```
do {
    __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort) {
        // 基于 port 的 Source 事件、调用者唤醒
        if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
            break;
        }
        // Timer 时间到、RunLoop 超时
        if (currentMode->_timerFired) {
            break;
        }
} while (1);
```

> 第五步
唤醒时通知 `Observer`:`RunLoop` 的线程刚刚被唤醒了。

代码如下:
```
if (!poll && (currentMode->_observerMask & kCFRunLoopAfterWaiting)) __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopAfterWaiting);
```

> 第六步
`RunLoop` 被唤醒后就要开始处理消息了:
* 如果是 `Timer` 时间到的话，就触发 `Timer` 的回调;
* 如果是 `dispatch` 的话，就执行 `block`;
* 如果是 `source1`事件的话，就处理这个事件。
消息执行完后，就执行加到 `loop` 里的 `block`。

代码如下:
```
handle_msg:
// 如果 Timer 时间到，就触发 Timer 回调
if (msg-is-timer) {
    __CFRunLoopDoTimers(runloop, currentMode, mach_absolute_time())
 }

// 如果 dispatch 就执行 block
else if (msg_is_dispatch) {
    __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
}

// Source1 事件的话，就处理这个事件
else {
    CFRunLoopSourceRef source1 = __CFRunLoopModeFindSourceForMachPort(runloop, currentMode, livePort); sourceHandledThisLoop = __CFRunLoopDoSource1(runloop, currentMode, source1, msg);
    if (sourceHandledThisLoop) {
        mach_msg(reply, MACH_SEND_MSG, reply);
    }
}
```

> 第七步
根据当前`RunLoop`的状态来判断是否需要走下一个`loop`。当被外部强制停止或`loop`超时时，就不继续下一个`loop`了，否则继续走下一个`loop`。

代码如下:
```
if (sourceHandledThisLoop && stopAfterHandle) {
      // 事件已处理完
    retVal = kCFRunLoopRunHandledSource;
} else if (timeout) {
    // 超时
    retVal = kCFRunLoopRunTimedOut;
} else if (__CFRunLoopIsStopped(runloop)) {
    // 外部调用者强制停止
    retVal = kCFRunLoopRunStopped;
} else if (__CFRunLoopModeIsEmpty(runloop, currentMode)) {
    // mode 为空，RunLoop 结束
    retVal = kCFRunLoopRunFinished;
}
```
整个`RunLoop`过程，如下所示:
![RunLoop过程图](https://github.com/binzi56/iOSSmallKnowledgePool/blob/master/整理的小知识点/resources/runloop过程图.png)

#### 2.2.2 如何监控
要想监听 RunLoop，你就首先需要创建一个 CFRunLoopObserverContext 观察者，代码如下:
```
CFRunLoopObserverContext context = {0,(__bridge void*)self,NULL,NULL};
runLoopObserver = CFRunLoopObserverCreate(kCFAllocatorDefault,
                                              kCFRunLoopAllActivities,
                                              YES,
                                              0,
                                              &runLoopObserverCallBack,
                                              &context);
```
将创建好的观察者`runLoopObserver`添加到主线程 `RunLoop` 的 `common` 模式下观察。然后，创建一个持续的子线程专⻔用来监控主线程的 `RunLoop`状态。
一旦发现进入睡眠前的 `kCFRunLoopBeforeSources` 状态，或者唤醒后的状态 `kCFRunLoopAfterWaiting`，在设置的时间阈值 内一直没有变化，即可判定为卡顿。接下来，我们就可以 `dump` 出堆栈的信息，从而进一步分析出具体是哪个方法的执行时间过⻓。
```
//创建子线程监控
dispatch_async(dispatch_get_global_queue(0, 0), ^{
	//子线程开启一个持续的 loop 用来进行监控
	while (YES) {
    long semaphoreWait = dispatch_semaphore_wait(dispatchSemaphore, dispatch_time(DISPATCH_TIME_NOW, 3*NSEC_PER_MSEC));
    if (semaphoreWait != 0) {
			if (!runLoopObserver) {
				timeoutCount = 0;
				dispatchSemaphore = 0;
				runLoopActivity = 0;
				return;
		    }
		    //BeforeSources 和 AfterWaiting 这两个状态能够检测到是否卡顿
			if (runLoopActivity == kCFRunLoopBeforeSources || runLoopActivity == kCFRunLoopAfterWaiting) {
				//将堆栈信息上报服务器的代码放到这里
			} //end activity
		}// end semaphore wait
		timeoutCount = 0;
	}// end while
});

```

其实，触发卡顿的时间阈值，我们可以根据`WatchDog`机制来设置。`WatchDog`在不同状态下设置的不同时间，如下所示:
* 启动(Launch):`20s`;
* 恢复(Resume):`10s`;
* 挂起(Suspend):`10s`;
* 退出(Quit):`6s`;
* 后台(Background):`3min`(在iOS 7之前，每次申请`10min`; 之后改为每次申请`3min`，可连续申请，最多申请到`10min`)。
通过`WatchDog`设置的时间，我认为可以把启动的阈值设置为10秒，其他状态则都默认设置为3秒。总的原则就是，要小于`WatchDog`的限制时间。

#### 2.3 获取卡顿的方法堆栈信息
* **直接调用系统函数**。
这种方法的优点在于，性能消耗小。但是，它只能够获取简单的信息，也没有 办法配合 dSYM 来获取具体是哪行代码出了问题，而且能够获取的信息类型也有限。这种方法，因为性能比较好，所以适用 于观察大盘统计卡顿情况，而不是想要找到卡顿原因的场景。
主要思路是:用 `signal` 进行错误信息的获取。

* **直接用[`PLCrashReporter`](https://opensource.plausible.coop/src/projects/PLCR/repos/plcrashreporter/browse)这个开源的第三方库来获取堆栈信息**。
这种方法的特点是，能够定位到问题代码的具体位置，而且性能消耗也不大。所以，也是我推荐的获取堆栈信息的方法。

### 3. 内存
**OOM**(`Out of Memory`)
#### 3.1 原因
