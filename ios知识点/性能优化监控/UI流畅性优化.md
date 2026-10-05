# UI流畅性优化
[一文读懂iOS图像显示原理与优化 - 掘金](https://juejin.im/post/6850418111976964109)
[iOS 性能优化总结 - 知乎](https://zhuanlan.zhihu.com/p/35693019)

[AsyncDisplayKit介绍（一）原理和思路. UITableView/UICollectionView的优化一直是iOS应用性… | by Jason Yu | 即刻 Engineering | Medium](https://medium.com/jike-engineering/asyncdisplaykit%E4%BB%8B%E7%BB%8D-%E4%B8%80-6b871d29e005)
1. 对于文字和图片的异步渲染操作（转化成像素点的这些操作）交由框架（ASDK）来处理，但是：
	1. CPU只适合渲染静态的元素，如文字、图片。
	2. 如果你选择使用CPU来做渲染，那么就没有理由再触发GPU的离屏渲染了，否则会同时存在两块内容相同的内存，而且CPU和GPU都会比较辛苦
2. cpu和gpu层面分别说。


# ASDK原理
核心思想就是通过异步渲染的入口，尽可能的把非必要的调整UI、绘制、图像解码等其他耗时操作全部放到子线程，再异步向主线程提交显示数据。主要思路和处理如下：
	1. 对象使用更轻量：首先通过node管理view和layer的绘制处理，并且在需要的时候才会异步地为Node加载相应的View，node的创建使用开销非常低更轻量，类似flutter中的widget思想。
	2. 预合成处理：会对符合要求（比如多个不需要交互的layer）的复杂图层做预合成处理，绘制到一张图上，CPU 避免了创建多对象的资源消耗，GPU 避免了多张 texture 合成和渲染的消耗，更少的 bitmap 也意味着更少的内存占用。
	3. 智能预加载：所有node内维护了一个interfaceState，代表在UI上的展示状态，远离屏幕Preload时节点还不可见，靠近屏幕即将显示时Display开始渲染，包括文本的光栅化以及图像解码等，Visible显示在屏幕上，这样能做到在合适的时机去处理最重要的事情，优化了cpu和gpu的使用。
	4. 自定义的一套布局系统。
	5. 向runloop注册时机，在休眠前提交执行。

# iOS图像绘制和渲染的过程
![](UI%E6%B5%81%E7%95%85%E6%80%A7%E4%BC%98%E5%8C%96/7AA22C2A-9DAD-419F-855A-95B0073D9C36.png)
![](UI%E6%B5%81%E7%95%85%E6%80%A7%E4%BC%98%E5%8C%96/v2-98077db5cb31318ec437f00762870142_1440w.jpg)
1. 首先一个视图由CPU进行对象创建、销毁、属性调整、Frame布局计算、图片编/解码，准备视图和图层的层级关系、绘制等操作（绘制里面包括查询是否有重写drawRect:或drawLayer:inContext:方法，实现自定义的额外绘制）
2. Core Animation 在主RunLoop 中注册了一个 Observer，监听了 BeforeWaiting 和 Exit 事件，对View的一些属性操作会被 CALayer 捕获，并通过 CATransaction 提交给渲染服务，渲染服务由OpenGL ES和GPU组成。
3. 渲染服务首先将图层数据交给OpenGL ES进行纹理生成和着色，生成前后帧缓存frameBuffer。
4. 然后视频控制器根据显示硬件的刷新频率（每秒60），发出一个垂直同步信号去缓存里读取渲染后的帧，显示在屏幕上。

**双帧缓存存在的问题？如何解决？**
为解决一个帧缓冲区效率问题(读取和写入都是一个无法有效的并发处理)，采用双缓冲机制，但也引入了画面撕裂问题，即当视频控制器还未读取完成时，即屏幕内容刚显示一半时，GPU 将新的一帧内容提交到帧缓冲区并把两个缓冲区进行交换后，视频控制器就会把新的一帧数据的下半段显示到屏幕上，造成画面撕裂现象。

GPU 通常有一个机制叫做垂直同步（简写也是 V-Sync），当开启垂直同步后，GPU 会等待显示器的 VSync 信号发出后，才进行新的一帧渲染和缓冲区更新，这样能解决画面撕裂现象，

iOS 设备会始终使用双缓存，并开启垂直同步。

# UI卡顿优化思路
根据屏幕显示原理视频控制器没16ms（每秒取60次）发出一个垂直信号去缓存里读取要显示的帧时，CPU 和 GPU没有完成对应画面的合成就会卡顿。

优化点思路分为2个大方向，尽可能减少CPU和GPU的资源消耗，将计算事件控制在16ms之内（按照60FPS的刷帧率，每16ms就会发出一次垂直信号）

### CPU层面
职责：比较少的ALU(运算单元)，适合少量复杂逻辑计算
1. 优化对象的使用：轻量、底层、复用
	1. 只是显示不交互用layer，coreanimation做动画。
	2. 可复用的考虑缓存思想，比如类似Cell缓存这种高频控件的创建和使用。
2. 优化布局计算：
	1. 要写重复和无用的约束。
	2. 动态改变的约束或者视图，在内存可控的情况善用active控制约束、或者hide视图避免频繁的移除和添加触发layout的刷新循环。
	3. updateConstraints内批量更新约束。
	4. 固有尺寸的label和button如果用不到应该关闭。
	5. 尽量少使用systemLayoutSizeFitting进行布局size的自动计算。
3. 优化绘制：
	1. 考虑在子线程异步的绘制（参考ASDK）。
	2. 大量文本情况使用CoreText绘制。
	3. 使用CashaperLayer+Path绘制实现GPU加速。
4. 耗时操作：思路是除非必要的提交，原则上放子线程
	1. 图片异步解码处理，按实际显示大小重绘，多图为了保证滑动性能可以考虑向runloop注册实际，每一次循环只dcode一张图片。
	2. 文本计算处理
	3. 业务逻辑上的其他耗时操作

### GPU层面
职责：比较多的ALU(运算单元)，适合大量简单的并行逻辑计算
1. 复杂的图层的合并渲染减少数量。
2. 合理使用离屏渲染。
3. 静态不会经常变化时开启光栅化做渲染缓存。

# 离屏渲染
[关于iOS离屏渲染的深入研究 - 知乎](https://zhuanlan.zhihu.com/p/72653360)
On-Screen Redering：在当前屏幕缓冲区进行渲染
Off-Screen Redering：在当前屏幕缓冲区外新开辟一个缓冲区进行渲染

### 注意点
drawRect不会触发离屏渲染，它只是在系统绘制之上进行CPU层面的额外绘制处理。

## 离屏渲染的目的
是为了解决一些复杂的图层混合问题，做额外的预处理合成。并且尽可能利用系统的资源，保证界面流畅。但如果过度的使用就会引发卡顿问题，因为需要消耗大量系统资源。

## 离屏渲染消耗性能的原因
1. 创建新的缓冲区
2. 频繁的上下文切换

## iOS 里触发离屏渲染的场景？如何优化？
### shadows（阴影） 
解决方案：设置Layer的shadowPath

### cornerRadius和maskToBounds（圆角）都打开的情况
解决方案：
1. 直接提供圆角图片。
2. 也可以直接让CPU异步绘制带圆角的图片来解决，圆角处颜色值改为透明颜色值（ASDK就是这么干）。

### allowsGroupOpacity（组不透明，默认会打开）
CALayer的 allowsGroupOpacity 属性默认开启（子 layer 在视觉上的透明度的上限是其父 layer 的 opacity ，也对应UIView的 alpha ）

解决方案：主动关闭allowsGroupOpacity属性，按产品需求自己控制layer透明度。

### 使用了layer的mask（遮罩）
解决方案：尽量少用，实在需要特殊形状的view，使用时同时打开shouldRasterize来对渲染结果进行缓存优化。

### 开启了Rasterize（光栅化）进行渲染缓存
shouldRasterize的主旨在于**降低性能损失，但总是至少会触发一次离屏渲染**，一旦被设置为true，Render Server就会强制把layer的渲染结果（包括其子layer，以及圆角、阴影、group opacity等等）保存在一块内存中，这样一来在下一帧仍然可以被复用，而不会再次触发离屏渲染。

解决方案：对于需要经常变动的内容不要开启这个属性，比如tableviewcell。

### UIBlurEffect模糊效果

# 性能测试和定位
1. 定位帧率，为了给用户流畅的感受，我们需要保持帧率在60帧左右。当遇到问题后，我们首先检查一下帧率是否保持在60帧。
2. 定位瓶颈，究竟是CPU还是GPU。我们希望占用率越少越好，一是为了流畅性，二也节省了电力。
3. 检查有没有做无必要的CPU渲染，例如有些地方我们重写了drawRect，而其实是我们不需要也不应该的，我们希望GPU负责更多的工作。
4. 检查有没有过多的离屏渲染，这会耗费GPU的资源，像前面已经分析的到的。离屏渲染会导致GPU需要不断地onScreen和offscreen进行上下文切换。我们希望有更少的离屏渲染。
5. 检查我们有无过多的Blending，GPU渲染一个不透明的图层更省资源。
6. 检查图片的格式是否为常用格式，大小是否正常。如果一个图片格式不被GPU所支持，则只能通过CPU来渲染。一般我们在iOS开发中都应该用PNG格式，之前阅读过的一些资料也有指出苹果特意为PNG格式做了渲染和压缩算法上的优化。
7. 检查是否有耗费资源多的View或效果，我们需要合理有节制的使用。
8. 最后，我们需要检查在我们View层级中是否有不正确的地方。例如有时我们不断的添加或移除View，有时就会在不经意间导致bug的发生。

### 测试工具：
Core Animation：Instruments里的图形性能问题的测试工具。
view debugging：Xcode 自带的，视图层级。
reveal：视图层级。

# 通过监控 RunLoop 来进行卡顿检测
微信开源的卡顿检测： [Matrix for iOS/macOS](https://github.com/Tencent/matrix/tree/master/matrix/matrix-iOS) 
![](UI%E6%B5%81%E7%95%85%E6%80%A7%E4%BC%98%E5%8C%96/C681DE99-F8B3-4EF6-AFB4-568B6E5CDB04.png)

## 理论分析：
对于iOS开发来说，监控卡顿就是要去找到主线程上都做了哪些事儿。我们都知道，线程的消息事件是依赖于NSRunLoop 的，所以从NSRunLoop入手，就可以知道主线程上都调用了哪些方法。我们通过监听 NSRunLoop 的状态，就能够发现调用方法是否执行时间过长，从而判断出是否会出现卡顿。

基本思路：
1. 要想监听 RunLoop，你就首先需要创建一个 CFRunLoopObserverContext 观察者，将创建好的观察者 runLoopObserver 添加到主线程 RunLoop 的 common 模式下观察。观察睡眠前的 kCFRunLoopBeforeSources 状态到唤醒后的状态 kCFRunLoopAfterWaiting间的时间间隔（因为大部分导致卡顿的的方法是在这2个状态间），超过设置的时间阈值即可判定为卡顿。代码如下：
``` objectivec
CFRunLoopObserverContext context = {0,(__bridge void*)self,NULL,NULL};
runLoopObserver = CFRunLoopObserverCreate(kCFAllocatorDefault,kCFRunLoopAllActivities,YES,0,&runLoopObserverCallBack,&context);
```
2. 开启一个子线程监控的代码如下：或者利用线程保活技术
``` objectivec
//创建子线程监控
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    //子线程开启一个持续的 loop 用来进行监控
    while (YES) {
        long semaphoreWait = dispatch_semaphore_wait(dispatchSemaphore, dispatch_time(DISPATCH_TIME_NOW, 3 * NSEC_PER_SEC));
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

## 阈值是从何而来呢？这样设置合理吗？
根据 WatchDog 机制（release环境才有，超时会被系统杀掉）来设置，WatchDog 在不同状态下设置的不同时间，如下所示：
* 启动（Launch）：20s；
* 恢复（Resume）：10s；
* 挂起（Suspend）：10s；
* 退出（Quit）：6s；
* 后台（Background）：3min（在iOS 7之前，每次申请10min； 之后改为每次申请3min，可连续申请，最多申请到10min）。

可以把启动的阈值设置为10秒，其他状态则都默认设置为3秒。总的原则就是，要小于 WatchDog的限制时间。

## 如何获取卡顿的方法堆栈信息？
**直接调用系统函数：性能消耗小但只能够获取简单的信息，无法配合dSYM定位具体哪一行**
```objectivec
// 注册NSSetUncaughtExceptionHandler(&UncaughtExceptionHandler);
void UncaughtExceptionHandler(NSException *exception) {
    NSArray *exceptionArray = [exception callStackSymbols]; //得到当前调用栈信息
    NSString *exceptionReason = [exception reason];       //非常重要，就是崩溃的原因
    NSString *exceptionName = [exception name];           //异常类型
}
```

**推荐用开源的第三方库来获取堆栈信息 [PLCrashReporter](https://opensource.plausible.coop/src/projects/PLCR/repos/plcrashreporter/browse)** 
```objectivec
// 获取数据
NSData *lagData = [[[PLCrashReporter alloc]
                                          initWithConfiguration:[[PLCrashReporterConfig alloc] initWithSignalHandlerType:PLCrashReporterSignalHandlerTypeBSD symbolicationStrategy:PLCrashReporterSymbolicationStrategyAll]] generateLiveReport];
// 转换成 PLCrashReport 对象
PLCrashReport *lagReport = [[PLCrashReport alloc] initWithData:lagData error:NULL];
// 进行字符串格式化处理
NSString *lagReportString = [PLCrashReportTextFormatter stringValueForCrashReport:lagReport withTextFormat:PLCrashReportTextFormatiOS];
//将字符串上传服务器
NSLog(@"lag happen, detail below: \n %@",lagReportString);
```

**Swift 获取调用栈信息**
[iOS获取任意线程调用栈 - 掘金](https://juejin.im/post/5d81fac66fb9a06af7126a44)
注意Swift方法名会经过一定复杂规则从新命名（比OC复杂），所以需要转换一下。