# OperationQueue
#iOS知识点/多线程

[深入浅出 iOS 并发编程](https://mp.weixin.qq.com/s/ut98-V-HU_vXz5O3CXFS2w)

**其拥有暂停、继续、终止等多个可控状态，从而可以更加灵活得适应并发编程的场景** 。

[多线程：Operation和OperationQueue - 掘金](https://juejin.im/post/6844903715892101128)
![](OperationQueue/600D904C-6AF1-404F-B0D2-B20DCF4F87B9.png)
let operation = Operation() 此时处于Pending状态
**依赖**添加完会进入Ready阶段。此时操作对象会进入**准备就绪状态**。
**优先级**设置之后会进入执行阶段。queuePriority属性决定了**进入准备就绪状态下的操作**之间的开始执行顺序。
完成之后会finished，并且有回调。
完成之后的不能取消，取消可以在其他三个时刻。

![](OperationQueue/1526E872-225D-45E5-B0C1-40BE6C80215D.png)
op2依赖于op1时，不管添加操作的顺序如何，结果都是op1先执行，op2再执行。不能相互依赖，会造成操作死锁。

![](OperationQueue/88955B18-CD13-4F59-A13F-636E43693819.png)
AB完成后执行CD

![](OperationQueue/A18180A2-65FE-4017-8486-93E4827B42F4.png)
maxConcurrentOperationCount控制最大并发数量，默认为-1并发。

![](OperationQueue/B4275A51-027B-423B-9681-3C74A8A3B6B4.png)
![](OperationQueue/608D42E4-358B-42D5-95AE-502222300173.png)
Block的第一个Block任务在主线程执行，其他在后台线程中执行。

![](OperationQueue/5C48D106-974E-496C-8585-7AEBA6FE3E8C.png)


Thread Sanitizer
