# Runloop 总结
#iOS知识点/RunLoop

[译iOS RunLoops - 掘金](https://juejin.im/post/59cc46e65188256fbd43a216)官方翻译
[iOS底层原理总结 - RunLoop - 简书](https://www.jianshu.com/p/de752066d0ad)
[苹果开发中文网站深入理解RunLoop - CocoaChina_让移动开发更简单](http://www.cocoachina.com/ios/20150601/11970.html)
[基于runloop的线程保活、销毁与通信 - 简书](https://www.jianshu.com/p/4d5b6fc33519)
[RunLoop NSMachPort 详解 - jeffasd的专栏 - CSDN博客](https://blog.csdn.net/jeffasd/article/details/52027733)
 [https://blog.ibireme.com/2015/05/18/runloop/](https://blog.ibireme.com/2015/05/18/runloop/) 
https://blog.ibireme.com/2015/05/18/runloop/
[将Runloop的理解都写到这里 - 掘金](https://juejin.im/post/5eb81420e51d454dd15f0364#heading-5)

![](Runloop%20%E6%80%BB%E7%BB%93/9D81D993-0935-46D9-AA22-C4750FFFF9E8.png)
Run Loop 是处理iOS中各种事件的一种循环抽象机制，实际上就是一个对象，这个对象管理了其需要处理的事件和消息，不断的检测并通过和它绑定的线程来循环处响应的传入事件，具备以下特点：
	1. 有消息处理时，唤醒处理消息
	2. 没有消息时休眠，避免资源占用

具体的理解上可以把 RunLoop 看成一个函数，其内部是一个 do-while 循环。当你调用 CFRunLoopRun() 时，线程就会一直停留在这个循环里，去执行加入到 RunLoop 里的 Source0，Timer 的回调，以及 Observer 回调，以及用于线程或进程间通讯的 Source1，当所有的都处理完之后，结束一次循环，进入休眠状态，休眠的时候等待 Timer 注册的时间点或者 Source1 唤醒 RunLoop（也可以手动唤醒）。

# RunLoop的运行逻辑
``` swift
function startLoop(){
	initialize()
	do{
		//检测并等待消息或事件
		var message = get_next_message()
		//线程处理消息或事件
		process_message(message)
	} while (message!=quit)
}
```
交互性的应用与命令行程序的区别，就是 RunLoop 。使用命令行程序， 先配参数开起，执行命令，就结束了。交互性的应用等待用户输入，给用户反馈，然后一直等。很多长周期的进程中，都有这种机制。

# RunLoop启动方式
iOS除主线程外，默认子线程不会开启runloop，runloop运行方法一共5种：包括NSRunloop的3种（基于CFRunloop），CFRunloop的两种。

## NSRunLoop启动：
1. run
```
-(void)run; 无条件运行
```
在DefaultRunLoopMode模式下重复调用runMode:beforeDate:方法，无法主动停止（死循环），即使调用CFRunLoopStop方法也只是停止某一次的运行。
2. runUntilDate
```objc
-(void)runUntilDate:(NSDate *)limitDate;  有一个超时时间限制
```
在DefaultRunLoopMode模式下重复调用runMode:beforeDate:方法，在超时时间到达之前，runloop会一直运行。
3. runMode:beforeDate
```objc
//有一个超时时间限制，而且设置运行模式
(BOOL)runMode:(NSString *)mode beforeDate:(NSDate *)limitDate;
```
可以指定运行的mode，在时间到达后或者处理完了事件后自动退出。并且可以被CFRunLoopStop方法主动停止。

## CFRunLoop启动
1. run
```objc
void CFRunLoopRun();
```
运行在默认的kCFRunLoopDefaultMode模式下，直到使用CFRunLoopStop停止这个RunLoop。
2. runInMode
``` objc
SInt32 CFRunLoopRunInMode (mode, seconds, returnAfterSourceHandled);
```
可以指定运行的mode和运行时间，returnAfterSourceHandled为Bool值，表示是否在处理事件后让Run Loop退出返回（NSRunloop的第三种开启runloop的方法基于这个方法实现，并且returnAfterSourceHandled为真）

# RunLoop的退出
1. runloop当前对应的mode中没有事件源（Timer、Source、Observer）。
2. 启动时设置一个超时时间。
3. 只要CFRunloop运行起来就可以用：CFRunLoopStop去停止本次。

> NSRunLoop的run方法之所以无法主动停止，是因为内部会重复执行runMode方法，即使调用CFRunLoopStop方法也只是停止某一次的运行。  
>   
> NSRunLoop的runMode方法开启运行后也能使用CFRunLoopStop去停止（因为它基于CFRunloop的runInMode方法实现）。  

# RunLoop和线程的关系：
1. RunLoop和线程是一一对应的关系，每条线程有唯一1个和它对应的Runloop对象。
2. 主线程的Runloop是默认已经获取并创建了，子线程默认没有开启Runloop。
3. Runloop的创建发生在第一次获取时，销毁是在线程结束时。
4. Runloop存储在一个全局的Dic里，线程作为存储的key，本身作为Value。

## 线程间通过Port通信
在线程内创建自己的port，然后加入到runloop的mode中，设置相应的接受消息代理，最后调用发送消息函数sendBeforeDate可以在线程中互相发送消息。

# RunLoop 组成
![](Runloop%20%E6%80%BB%E7%BB%93/7E74E039-FDC0-4E9B-B002-7C94A639C97B.png)
1. 每个Runloop可以有多个Mode，每个 Mode 又包含若干个 Source、Timer、Observer等事件源。
2. 某一时刻只能选择一种运行模式，处理mode内的事件。
3. 如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。
::4. 一个事件源可以被同时加入多个 mode，但重复加入同一个 mode 时是不会有效果的。::
::5. 如果一个 mode 中一个事件源都没有，则 RunLoop 会直接退出。::

## 常见RunloopMode
RunloopMode是Runloop的运行模式，是一组事件源的抽象集合，通过在不同的模式下集中处理不同的事件，主要作用是区分指定事件在运行循环中的优先级，一般常用的就是：
1. commonModes：默认包含default和Tracking的一种mode集合。
2. defaultMode：默认mode
3. trackingRunLoopMode：界面跟踪mode，保证滑动时不受其他mode影响

> commonModes是一组mode集合，并不是一个mode。你可以创建并添加自己的mode，然后加到这个集合里面去。这样把runloop的mode设置为commonModes后，对应的事件就能在多个mode下也会执行。  

# Runloop事件源
1. Source1 ：由RunLoop和内核管理，Mach port驱动，如CFMachPort、CFMessagePort。source1包含了一个 mach_port 和一个回调（函数指针），被用于通过内核和其他线程相互发送消息。这种 Source 能主动唤醒 RunLoop 的线程。
2. Source0 ：source0是App内部事件，由App自己管理的，像UIEvent、CFSocket都是source0。source0并不能主动触发事件，当一个source0事件准备处理时，要先调用 CFRunLoopSourceSignal(source)，将这个 Source 标记为待处理。然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 RunLoop，让其处理这个事件。框架已经帮我们做好了这些调用，比如网络请求的回调、滑动触摸的回调，我们不需要自己处理。
3. Timer：定时器事件、performSelector afterDelay就是基于timer实现。
4. Observer ：监听器事件，用于监听RunLoop的状态，每个 Observer 都包含了一个回调（函数指针），当 RunLoop 的状态发生变化时，观察者就能通过回调接受到这个变化：
![](Runloop%20%E6%80%BB%E7%BB%93/EAA256E6-6525-45DA-8495-B78B9DD09ADD.png)

## RunloopMode和定时器相关问题
**问题1：为什么定时器timer在滑动的时候会失效？怎样防止？**
因为默认timer在defaultmode中，在滑动时runtime会将mode切换为UITrackingRunLoopMode，导致事件回调无法执行。解决办法是把它加入到上层的commonModes中。

**问题2：定时器commonMode生效原理：**
runloop会为加入的modeitem（timer）打上一个common标记属性，而kCFRunLoopDefaultMode和UITrackingRunLoopMode都默认已经被标记为Common属性。随后modeitem被runloop自动更新到所有具有Common标记的 Mode 里去，所以无论在滑动还是平常状态都能发送回调。

**问题3：为什么timer定时器不精确？**
由于timer是基于runloop实现的，而runloop事件循环可能存在的延迟。原因在于当一个Timer注册到RunLoop后，原本会为其重复的时间点注册好事件（如果每1秒执行一次）但是每次的loop循环所花费的时间都可能不一样，假如某一次循环事件处理时间过长，超过了本来应该触发timer回调的时间，那么这一次的回调就可能会被放弃（取决于timer的容错率），这样就会导致timer的延迟执行。

**问题4：如何实现精准定时器？**
要实现准时的定时器，推荐使用GCD定时器，因为它不依赖于runloop，和系统内核世间直接挂钩。

# RunLoop 循环逻辑
![](Runloop%20%E6%80%BB%E7%BB%93/0B40377F-3975-45E3-963C-2427F755AFB9.png)
1. 首先找到对应的Mode，事件源为空的话直接退出，不为空则进入循环。
2. 先会通知Observer即将进入Loop循环了，然后再通知observer即将处理各种事件源item（timer、source0）后处理对应的事件。
3. 完成处理后会通知Observer即将进入休眠并等待事件源唤醒（timer、source1）
4. 得到事件后会通知Observer刚被唤醒，并再次回到第二阶段处理事件。
5. 直到mode里的item为空（被移除），或者被强制停止，则发送通知Observer并退出。

![](Runloop%20%E6%80%BB%E7%BB%93/02F23891-DF47-46A3-AEB6-E432C72857A9.png)
![](Runloop%20%E6%80%BB%E7%BB%93/376DDF11-B9F6-463A-977C-F7659ECC8BC3.png)

# Runloop休眠的理解
和普通的死循环阻塞休眠不一样，死循环是一直在做操作而不执行后面的逻辑，runloop里的休眠是真的什么也不做，只等待消息来唤醒它，::它基于内核操作系统相关的技术，当需要休眠时切换到内核态，达到这种真的什么也不做的效果，不占用CPU资源，和一般的线程阻塞是不同的。::

RunLoop 的核心就是一个 mach_msg() ，RunLoop 调用这个函数去接收消息，如果没有别人发送 port 消息过来，内核会将线程置于等待状态。
![](Runloop%20%E6%80%BB%E7%BB%93/A261D0EC-27A3-4EAB-A938-B9A01B92A365.png)
休眠时Runloop唤醒的3方式
	1. source1
	2. timer事件回调 
	3. runloop超时
	4. 外部主动唤醒

# MachPort
在 Mach 中，所有的东西都是通过自己的对象实现的，进程、线程和虚拟内存都被称为 “对象”，和其他架构不同， Mach 的对象间不能直接调用，只能通过消息传递的方式实现对象间的通信。“消息”（mach msg）是 Mach 中最基础的概念，消息在两个端口 (mach port) 之间传递，这就是 Mach 的 IPC (进程间通信) 的核心。

消息的发送和接收统一使用 mach_msg 函数，而 mach_msg 的本质是调用了 mach_msg_trap，这相当于一个系统调用，会触发内核态与用户态的切换。

### 基本使用
就是线程和进程通信的一种手段，使用方式就是将 NSMachPort 对象添加到一个线程所对应的 run loop 中，并给 NSMachPort 对象设置相应的代理。在其他线程中调用该 MachPort 对象发消息时会在 MachPort 所关联的线程中执行相关的代理方法。详细参考[iOS 从源码解析Run Loop (五)：NSPort、TSD 相关内容解析](https://juejin.cn/post/6909815807555928077#heading-6)

iOS的中通直在主线程中添加观察者，但是如果在子线程中发出通知，那么就是在该子线程中处理通知所关联的方法。[iOS开发之线程间的MachPort通信与子线程中的Notification转发 - 青玉伏案 - 博客园](https://www.cnblogs.com/ludashi/p/7460907.html)

