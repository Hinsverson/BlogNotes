# Notification 总结


[轻松过面：一文全解iOS通知机制(经典收藏)](https://juejin.cn/post/6844904082516213768)
* 实现原理主要就是通过name和object为维度去映射存储sel和observer，然后通过performSelector向observer发送sel消息，信息的存储在center中：
	* 注册时如果使用了name，那么先通过name查找到一个map结构， map的结构内以object作为key，存取obs（有sel和observer）再做performSelector执行。
	* 注册时只用了object的话是通过object为key，从一个map中取出对应obs类型的链表完成存取和执行。
	* 注册时没有用name和object的话是直接通过一个obs类型的链表完成存取和执行。
	* 所以**添加通知**时，若指定了object参数，那么该响应者只会接收**发送通知**时object参数指定为同一实例的通知。
	* 所以**发送通知**时，若指定了object参数，并不会影响**添加通知**时没有指定object参数的响应者接收通知。
* 普通通知的发送是同步的，从center内部结构中通过name或者object取出obs后，同步遍历执行回调响应，NSNotificationQueue的通知是时机上异步的，依赖runloop时机，但并不是线程异步的。
* 在指定的NSOperationQueue中接收通知的注册方法，原理上内部还是调用了普通的注册方法，只是多了一层NotificationObserver来接收消息后转发到指定的queue上执行block。
* NSNotificationCenter接受消息时处理的线程取决于发送时的线程，要保证通知接收的线程在主线程可以指定queue或者通过其他线程通信方式转发（比如port等）
* 通知的注册后同KVO一样要移除，并且移除和添加的次数要匹配，否则崩溃。
* 通知的存储通过哈希表实现，diff一个通知的维度是name和object共同决定。
* 通知的存储过程并没有做去重操作，这也解释了为什么同一个通知注册多次则响应多次。删除时会把重复的通知全都删除掉。



# 通知存储结构
``` objc
// 根容器，NSNotificationCenter持有
typedef struct NCTbl {
  Observation		*wildcard;	/* 链表结构，保存既没有name也没有object的通知 */
  GSIMapTable		nameless;	/* 存储没有name但是有object的通知	*/
  GSIMapTable		named;		/* 存储带有name的通知，不管有没有object	*/
    ...
} NCTable;

// Observation 存储观察者和响应结构体，基本的存储单元
typedef	struct	Obs {
  id		observer;	/* 观察者，接收通知的对象	*/
  SEL		selector;	/* 响应方法		*/
  struct Obs	*next;		/* Next item in linked list.	*/
  ...
} Observation;
```

# 注册通知
### 注册接口1
``` objc
- (void) addObserver: (id)observer selector: (SEL)selector name: (NSString*)name object: (id)object {
```
1. 如果注册通知时传入了name，那么会是一个双层的存储结构
	1. 找到NCTable中的named表，这个表存储了还有name的通知
	2. 以name作为key，找到value，这个value依然是一个map
	3. map的结构是以object作为key，obs对象为value，这个obs对象的结构上面已经解释，主要存储了observer & SEL
2. 如果注册通知时只存在object
	1. 以object为key，从nameless字典中取出value，此value是个obs类型的链表
	2. 把创建的obs类型的对象o存储到链表中
3. 如果注册通知时没有name和object
	1. 直接把obs对象存放在了`Observation  *wildcard`链表结构中

存储是以name和object为维度的，即判定是不是同一个通知要从name和object区分，如果他们都相同则认为是同一个通知，后面包括查找逻辑、删除逻辑都是以这两个为维度的

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