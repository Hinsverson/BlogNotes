# Notification 总结
#iOS知识点/系统特性


[轻松过面：一文全解iOS通知机制(经典收藏)](https://juejin.cn/post/6844904082516213768)
* 实现原理主要就是通过name和object为维度去映射sel和observer，然后通过performSelector向observer发送sel消息。
* 普通通知的发送是同步的，同步遍历取出回调响应，NSNotificationQueue的通知是时机上异步的，依赖runloop时机，但并不是线程异步的。
* NSNotificationCenter接受消息时处理的线程取决于发送时的线程，要保证通知接收的线程在主线程可以指定queue或者通过其他线程通信方式转发（比如port等）
* 通知的注册后同KVO一样要移除，并且移除和添加的次数要匹配，否则崩溃。
* 通知的存储通过哈希表实现，diff一个通知的维度是name和object共同决定。

# 注册通知
### 注册接口1
``` objc
- (void) addObserver: (id)observer selector: (SEL)selector name: (NSString*)name object: (id)object {
```

### 注册接口2
``` objc
// 这个api使用频率较低，怎么实现在指定队列回调block的，值得研究
- (id) addObserverForName: (NSString *)name 
                   object: (id)object 
                    queue: (NSOperationQueue *)queue 
               usingBlock: (GSNotificationBlock)block
{
	// 创建一个临时观察者
	GSNotificationObserver *observer = 
		[[GSNotificationObserver alloc] initWithQueue: queue block: block];
	// 调用了接口1的注册方法
	[self addObserver: observer 
	         selector: @selector(didReceiveNotification:) 
	             name: name 
	           object: object];

	return observer;
}
```
依赖于接口1，只是多了一层代理观察者GSNotificationObserver，设计上通过这个中间对象接受通知消息，然后把消息再转发到指定队列上去执行。
	1. 创建一个GSNotificationObserver类型的对象observer，并把queue和block保存下来
	2. 调用接口1进行通知的注册
	3. 接收到通知时会响应observer的didReceiveNotification:方法，然后在didReceiveNotification:中把block抛给指定的queue去执行

# 发送通知
``` objc
// 发送通知
- (void) postNotificationName: (NSString*)name
		       object: (id)object
		     userInfo: (NSDictionary*)info
```
发送主要做了三件事
	1. 通过name & object 查找到所有的obs对象(保存了observer和sel)，放到数组中。
	2. 通过performSelector：逐一调用sel，这是个同步操作。
	3. 释放notification对象。


# NSNotificationQueue
1. 依赖runloop，所以如果在其他子线程使用NSNotificationQueue，需要开启runloop
2. 最终还是通过NSNotificationCenter进行发送通知，所以这个角度讲它还是同步的
3. 所谓异步，指的是非实时发送而是在合适的时机发送，并没有开启异步线程

# 主线程响应通知
异步线程发送通知则响应函数也是在异步线程，如果执行UI刷新相关的话就会出问题，那么如何保证在主线程响应通知呢？
其实也是比较常见的问题了，基本上解决方式如下几种：
1. 使用addObserverForName: object: queue: usingBlock方法注册通知，指定在mainqueue上响应block
2. 在主线程注册一个machPort，它是用来做线程通信的，当在异步线程收到通知，然后给machPort发送消息。