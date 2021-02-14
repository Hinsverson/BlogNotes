# 系统Runloop常见原理
#iOS知识点/RunLoop

[iOS 从源码解析Run Loop (八)：Run Loop 与 AutoreleasePool、NSTimer、PerformSelector 系列](https://juejin.cn/post/6911946403036004366)
[iOS 从源码解析Run Loop (九)：Run Loop 与事件响应、手势识别、屏幕刷新、卡顿监测](https://juejin.cn/post/6913094534037504014)
[iOS 从源码解析Run Loop (十)：Run Loop 与GCD、FPS、CADisplayLink](https://juejin.cn/post/6914184740945952781)

# AutoreleasePool
App启动之后，苹果在主线程 RunLoop 里注册了两个 Observer，回调都是_wrapRunLoopWithAutoreleasePoolHandler()。
**1. 第一个observer，监听了一个事件**：
即将进入Loop（kCFRunLoopEntry），其回调会调用 _objc_autoreleasePoolPush()创建一个栈自动释放池，这个优先级最高，保证创建释放池在其他操作之前。
**2.第二个observer，监听了两个事件：**  
1).准备进入休眠（kCFRunLoopBeforeWaiting），此时调用 _objc_autoreleasePoolPop()和 _objc_autoreleasePoolPush()来释放旧的池并创建新的池。
2). 即将退出Loop（kCFRunLoopExit），此时调用 _objc_autoreleasePoolPop()释放自动释放池。这个 observer 的优先级最低，确保池子释放在所有回调之后。

## GCD和Runloop的关系
在GCD中，由子线程返回到主线程，这种情况下底层是一个RunLoop的Source 1 事件包装。
当调用 dispatch_async(dispatch_get_main_queue(), block) 时，libDispatch 会向主线程的 RunLoop 发送消息，RunLoop会被唤醒，并从消息中取得这个 block，并在回调 **CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE**() 里执行这个 block。但这个逻辑仅限于 dispatch 到主线程，dispatch 到其他线程仍然是由 libDispatch 处理的。

## PerformSelector afterDelay的实现原理？
实际上会创建一个 Timer 并添加到当前线程的 RunLoop 中，原理和timer类似。但是如果当前线程没有 RunLoop，则这个方法会失效。

## 事件响应的过程（结合RunLoop）
1. 首先系统会注册了一个Source1(基于mach port) 用来接收系统事件。
2. 当一个硬件事件（触摸、锁屏、摇晃等）发生后，会由底层的硬件处理框架（IOKit）捕捉后通过mach port 形式转发给需要的进程。
3. 之后触发系统注册的那个Source1 回调，并调用UIApplication对象的事件处理队列进行应用内部的事件分发。
4. 之后就是事件的响应传递过程。[事件传递/响应机制](bear://x-callback-url/open-note?id=019EBEC8-314F-4DE0-9516-41C4092EBA11-2318-00022693160DAD8D)

## 手势识别的过程（结合RunLoop）
1. 同上面事件响应的底层过程一样，当UIApplication进行手势分发时，会先取消调当前的touchesBegin、Move、End回调。随后将拿到的新手势UIGestureRecognizer标记为待处理。
2. 在主线程Runloop中系统已经注册好了一个Observer来监听Runloop即将进入休眠的状态（BeforeWaiting），并进行Observer的回调。
3. 在回调内部会获取所有刚被标记为待处理的GestureRecognizer，并执行
GestureRecognizer 自身的回调（手势识别相关代码）
5. 当UIGestureRecognizer的状态改变时(创建/销毁/状态改变)，都会触发Observer回调进行相应处理。

## UI绘制setNeedsDisplay 的原理？
UIView调用setNeedsDisplay方法后，只是相当于打上了一个标记，表示这个view需要重新绘制，真正的绘制时机是在本次主线程loop即将进入休眠时会通知observer，调用CALayer的display方法。
