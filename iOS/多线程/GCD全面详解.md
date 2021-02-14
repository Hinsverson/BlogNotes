# GCD全面详解
#iOS知识点/多线程

GCD：一套多线程解决方案
参考：[iOS 多线程：『GCD』详尽总结 - 简书](https://www.jianshu.com/p/2d57c72016c6)
参考：[GCD精讲（Swift 3&4） - Leo的专栏 - CSDN博客](https://blog.csdn.net/Hello_Hwc/article/details/54293280)
参考：[Swift 中的并发编程(第一部分：现状） | Swift 教程 - Swift 语言学习 - Swift code - SwiftGG 翻译组 - 高质量的 Swift 译文网站](https://swift.gg/2017/09/04/all-about-concurrency-in-swift-1-the-present/#multithreading_and_concurrency_primer)
[iOS - 多线程（三）：GCD - 云+社区 - 腾讯云](https://cloud.tencent.com/developer/article/1615612)
[GCD源码吐血分析(2)——dispatch_async/dispatch_sync/dispatch_once/dispatch group_无忘无往-CSDN博客_dispatch_async](https://blog.csdn.net/u013378438/article/details/81076116)


[iOS Swift GCD 开发教程 - 掘金](https://juejin.im/post/6844903588930519047#heading-38)

# GCD核心总结
自动的根据多核CPU的使用情况，创建线程池抽象成queue来执行任务，提高程序的运行效率。

## 队列类型：决定任务怎样拿出来
* 串行队列：任务只能一个一个拿出来顺序的在唯一的一条线程上执行。
* 并行队列：可以同时拿出多个任务到不同的线程并发执行。

## 任务执行方式：决定拿出来的任务在哪条线程上执行，以及阻不阻塞当前线程
sync同步：不具备开启新线程的能力，在当前上下文线程中执行，并阻塞。
async异步：具备开启新线程的能力，串行队列多个异步只会开启一个线程，主队列已经存在主线程，所以异步也不会开启，并行队列并发开启线程执行，最多64条，不阻塞。

::main queue特殊注意点：::
1. 串行队列，并且已经存在有对应的主线程，所以主队列async添加任务不会开启新线程。
2. 无论sync、async提交任务都在主线程执行。

## 常见原理
* sync 将任务 block 通过 push 到队列中，然后按照 FIFO 去执行。
* async 会把任务包装并保存，之后就会开辟相应线程去执行已保存的任务。
* semaphore 主要使用 signal 和 wait 这两个接口，底层分别调用了内核提供的方法。在调用 signal 方法后，先将 value 减一，如果大于零立刻返回，否则陷入等待。signal 方法将信号量加一，如果 value 大于零立刻返回，否则说明唤醒了某一个等待线程，此时由系统决定哪个线程的等待方法可以返回。
* group 底层也是维护了一个 value 的值，等待 group 完成实际上就是等待 value 恢复初始值。而 notify 的作用是将所有注册的回调组装成一个链表，在 async 完成时判断 value 是不是恢复初始值，如果是则调用 async 异步执行所有注册的回调。
* dispatch_once 通过一个静态变量来标记 block 是否已被执行，同时使用加锁确保只有一个线程能执行，执行完 block 后会唤醒其他所有等待的线程。

# GCD和NSOperation区别
1. GCD不支持异步操作之间的依赖关系设置，而NSOperation中的队列可以被重新设置优先级，从而实现不同操作的执行顺序调整。
2. NSOperation支持KVO，可以观察任务的执行状态。
3. GCD更接近底层，GCD在追求性能的底层操作来说，是速度最快的。
4. 从异步操作之间的事务性，顺序行，依赖关系。GCD需要自己写更多的代码来实现，而NSOperation已经内建了这些支持。
5. 如果异步操作的过程需要更多的被交互和UI呈现出来，NSOperation更好。底层代码中，任务之间不太互相依赖，而需要更高的并发能力，GCD则更有优势。

# DispatchQueue的创建
一个类似线程的概念，这里称作队列，队列是一个FIFO数据结构，意味着先提交到队列的任务会先开始执行。DispatchQueue背后是一个由系统管理的线程池。

``` swift
let label = "com.leo.demoQueue"
let qos =  DispatchQoS.default
let attributes = DispatchQueue.Attributes.concurrent
let autoreleaseFrequency = DispatchQueue.AutoreleaseFrequency.never
let queue = DispatchQueue(label: label, qos: qos, attributes: attributes, autoreleaseFrequency: autoreleaseFrequency, target: nil)
```

1. label：队列的标识符，方便调试
2. qos：用来指明队列的“重要性”，类型如下：
![](GCD%E5%85%A8%E9%9D%A2%E8%AF%A6%E8%A7%A3/626BCD5F-06AE-4EE2-A8B6-EDA6679EE185.png)
3. attributes：队列的属性，是一个结构体，遵循了协议OptionSet。意味着你可以这样传入第一个参数[.option1,.option2]
4. autoreleaseFrequency：顾名思义，自动释放频率，有些队列是会在执行完任务后自动释放的，有些比如Timer等是不会自动释放的，是需要手动释放。

# GCD队列的分类
![](GCD%E5%85%A8%E9%9D%A2%E8%AF%A6%E8%A7%A3/8CD8557C-7899-43A0-884B-F2B4F4DFDADC.png)

> 主线程队列（main）：串行队列  
> 全局队列（global）：全局的并发队列  

# GCD队列执行情况对比
![](GCD%E5%85%A8%E9%9D%A2%E8%AF%A6%E8%A7%A3/EB22169D-5A2B-43BD-B334-B36C7CAC2609.png)
![](GCD%E5%85%A8%E9%9D%A2%E8%AF%A6%E8%A7%A3/9BB51CA6-99A5-4BE4-AE9C-5C99075FFCE3.png)

*重点:*
1. 并发只会在异步执行下才会开启多线程执行。
2. ::主队列是一种特殊的串行队列，即使是在主队列中异步执行一个任务，虽然异步具备开启线程的能力，但在主队列中不会开启线程。::
3. 同步: 在当前线程中同步执行，不具备开启新线程的能力。
4. 异步：在新线程中同步执行，具备开启新线程的能力。
5. 并发队列：提交的多个任务并发（同时）分派执行
6. 串行队列，提交的多个任务，按顺序分派，一个接一个执行。

# GCD 死锁相关
[GCD的死锁](bear://x-callback-url/open-note?id=BF993B66-CC37-4FCB-B83C-33389FC0E2DA-19794-0000AE6F7B32565D)
往当前串行队列中同步添加task

# DispatchWorkItem
把任务封装成了一个对象，可以对单个任务添加配置信息。
![](GCD%E5%85%A8%E9%9D%A2%E8%AF%A6%E8%A7%A3/95B43B2C-D583-42F5-8300-A76D2A73A7A2.png)
![](GCD%E5%85%A8%E9%9D%A2%E8%AF%A6%E8%A7%A3/CAA0D375-72AB-4D92-BD24-31DB03E47258.png)

# GCD 延迟执行
![](GCD%E5%85%A8%E9%9D%A2%E8%AF%A6%E8%A7%A3/23EC40FA-4751-4975-A44D-1AF148A354CA.png)
![](GCD%E5%85%A8%E9%9D%A2%E8%AF%A6%E8%A7%A3/FDF4640A-8606-489A-A7A2-A690FFAC68CB.png)
![](GCD%E5%85%A8%E9%9D%A2%E8%AF%A6%E8%A7%A3/FC642D9C-C143-41FC-B142-5E6CD492B576.png)

# DispatchSource
Swift中DispatchSource是一个工厂类，用来实现各种source。
提供了一组接口，用来提交hander监测底层的事件，这些事件包括Mach ports，Unix descriptors，Unix signals，VFS nodes。
![](GCD%E5%85%A8%E9%9D%A2%E8%AF%A6%E8%A7%A3/F0F12715-CC1E-40D2-BBE3-AE4E8CDB4D2A.png)

## DispatchSourceProtocol
基础协议，所有的用到的DispatchSource都实现了这个协议，下面是公有方法：
![](GCD%E5%85%A8%E9%9D%A2%E8%AF%A6%E8%A7%A3/962EA483-545F-4576-A5AF-B2D1A1DA01B8.png)

## DispatchSourceTimer（GCD精准定时器）
![](GCD%E5%85%A8%E9%9D%A2%E8%AF%A6%E8%A7%A3/89C62B59-E6BE-4AAC-8EF0-9E7613260654.png)
deadline表示开始时间，leeway表示能够容忍的误差。

# DispatchGroup（队列组）
底层是基于dispatch_semaphore实现，用来管理任务的分组执行，然后监听任务都完成的事件（notify）。比如分别异步执行多个耗时任务，然后当多个耗时任务都执行完毕后再回到主线程执行任务，这时候我们可以用到 GCD 的队列组。

**队列组的2种使用方式**
1. group.enter() 和 group.leave() 成对使用，表示手动入组和出组。
2. 可以在队列执行时直接把任务加入组
que.async(group: DispatchGroup, execute: DispatchWorkItem)

``` swift
let group = DispatchGroup()

group.enter()
networkTask(label: "1", cost: 2, complete: {
    group.leave()
})
group.enter()
networkTask(label: "2", cost: 4, complete: {
    group.leave()
})

group.notify(queue: .main, execute:{
    print("All network is done")
})
```

# Dispatch_barrier（栅栏）
等到所有的之前入队的block执行完成后才开始执行。除此之外，在barrier block后面入队的所有的block，会等到到barrier block本身已经执行完成之后才继续执行。

有时候创建多个并发任务，如果在中间加入栅栏，那么加入栅栏这个任务会在前面的任务完成后执行，并且后面的任务会在栅栏任务完成后才开始执。行。如下图所示在并发队列中添加任务，执行顺序一定是：
*Task1 或 Task 2 -> Barrier任务 -> Task3*
![](GCD%E5%85%A8%E9%9D%A2%E8%AF%A6%E8%A7%A3/E036E300-1C38-4A4B-8322-3004D17AF238.png)
``` objectivec
dispatch_queue_t concurrentQueue = dispatch_queue_create(“ConcurrentQueue”, DISPATCH_QUEUE_CONCURRENT);
dispatch_async(concurrentQueue, ^{
    NSLog(@“1————“);
});
dispatch_async(concurrentQueue, ^{
    NSLog(@“2————“);
});
dispatch_barrier_async(concurrentQueue, ^{
    NSLog(@“barrier————“);
});

dispatch_async(concurrentQueue, ^{
    NSLog(@“3————“);
});
dispatch_async(concurrentQueue, ^{
    NSLog(@“4————“);
});
```

> 打印执行顺序1，2不定，3，4也不定，但是barrier一定在1和2之后，3和4一定在barrier之后。  

# DispatchSemaphore（信号量）
1. semaphore.wait() ：如果信号量<=0，当前线程就会进入休眠等待，直到信号量>0才会往下执行，如果信号量>0，就会先-1，然后往下执行
2. semaphore.signal() ：让信号量的值+1

> 原理：操作系统层面的PV原语  
> 如果设置一个线程一个semaphore，这样当你的信号量为0的时候，对其他线程是没有影响的。  
> 多个线程使用同一个semaphore的时候，信号量等于0，其他线程会处于等待状态。至于之后怎么确定执行哪个线程，这个完全是随机的，不管你的线程优先级怎样，对于操作系统来说都是随机执行一个线程。  

## 主要用途
1. 非常多的网络请求异步执行的情况下限制最大请求数量。
2. 值为1时可以用来做多线程下的同步控制，保证资源访问时的线程安全。
*实例：控制最大并发数量*
``` swift
let semaphore = DispatchSemaphore(value: 2)
let queue = DispatchQueue(label: "com.leo.concurrentQueue", qos: .default, attributes: .concurrent)

queue.async {
    semaphore.wait()
    usbTask(label: "1", cost: 2, complete: { 
        semaphore.signal()
    })
}

queue.async {
    semaphore.wait()
    usbTask(label: "2", cost: 2, complete: {
        semaphore.signal()
    })
}

queue.async {
    semaphore.wait()
    usbTask(label: "3", cost: 1, complete: {
        semaphore.signal()
    })
}
```

# dispatch_once
Q: dispatch_once  为什么可以保证只执行一次？
A: dispatch_once  封装并执行了 dispatch_once_f  函数，其内部使用原子性操作进行标记，以此来配合信号量来决定是否唤醒其他等待的线程，而信号量则用来确保同一时间只有一个线程可以执行回调。

Q: 和 synchronized的优劣分析？
A: 相比之下 dispatch_once 的性能更高，速度更快。两者分别利用来不同的方式来保证线程安全， synchronized 采用的是递归互斥锁的方式来保证线程安全，而 dispatch_once 是使用原子操作来代替锁，使用信号量来保证线程同步。

![](GCD%E5%85%A8%E9%9D%A2%E8%AF%A6%E8%A7%A3/A9F70D28-1986-4728-848A-55C8EA785D30.png)
Swift中已经被废弃，可以使用如上方式保证只执行一次。

