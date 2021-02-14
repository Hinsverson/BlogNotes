# Aspects
#iOS知识点/第三方库

[Aspects深度解析-iOS面向切面编程](https://juejin.cn/post/6844904052778598408#heading-18)
``` objc
/**
作用域：针对所有对象生效
selector: 需要hook的方法
options：是个枚举，主要定义了切面的时机（调用前、替换、调用后）
block: 需要在selector前后插入执行的代码块
error: 错误信息
*/
+ (id<AspectToken>)aspect_hookSelector:(SEL)selector
                           withOptions:(AspectOptions)options
                            usingBlock:(id)block
                                 error:(NSError **)error;
/**
作用域：针对当前对象生效
*/
- (id<AspectToken>)aspect_hookSelector:(SEL)selector
                           withOptions:(AspectOptions)options
                            usingBlock:(id)block
                                 error:(NSError **)error;


```

### 基本原理：利用了消息转发机制，通过hook第三层的转发方法forwardInvocation然后根据切面的时机来动态调用block。
1. 类A的方法m被添加切面方法aspect_hookSelector。
2. 创建一个类A的子类B，并hook住子类B的forwardInvocation:方法拦截消息转发，使forwardInvocation:的IMP指向事先准备好的ABC调度函数，block方法的执行就在ABC函数中。
3. 把类A的对象的isa指针指向B，这样就把消息的处理转发到类B上，类似KVO的机制，同时会更改class方法的IMP，把它指向类A的class方法，当外界调用class时获取的还是类A，并不知道中间类B的存在。
4. 对于方法m，类B会直接把方法m的IMP指向_objc_msgForward()方法，这样当调用方法m时就会直接走到消息转发流程，触发ABC函数，完成整个hook。

通过类似KVO的hook机制，能做到针对某一个特定的对象hook，避免了对原有类的侵入，缩小影响范围，具体的实现和注意事项如下：
	1. 如果已经存在中间类，则直接返回。
	2. 如果是类对象，则不用创建中间类，并把这个类存储在swizzledClasses集合中，标记这个类已经被hook了。
	3. 如果存在kvo的情况，那么系统已经帮我们创建好了中间类，那就直接使用
	4. 对于不存在kvo且是实例对象的，则单独创建一个继承当前类的中间类midcls，并hook它的forwardInvocation:方法，并把当前对象的isa指针指向midcls，这样就做到了hook操作只针对当前对象有效，因为其他对象的isa指针指向的还是原有类

### 哪些方法不能被hook？
1. 不允许hook retain、release、autorelease、forwardInvocation，允许hook dealloc，但是只能在dealloc执行前，主要是为了安全性。
2. 检查这个方法是否存在，不存在则不能hook。
3. Aspects对于hook的生效作用域做了区分：所有实例对象&某个具体实例对象。对于所有实例对象在整个继承链中，同一个方法只能被hook一次，这么做的目的是为了规避循环调用的问题（详情可以了解下supper关键字）

### 如何存储切面信息？
``` objc
@interface AspectsContainer : NSObject
...其他省略
@property (atomic, copy) NSArray <AspectIdentifier *>*beforeAspects; // 存储原方法调用前要执行的操作
@property (atomic, copy) NSArray <AspectIdentifier *>*insteadAspects;// 存储替换原方法的操作
@property (atomic, copy) NSArray <AspectIdentifier *>*afterAspects;// 存储原方法调用后要执行的操作
@end
```
通过关联对象，把hook前后的基本信息通过AspectIdentifier这个model封装并存储下来，在后续消息转发时取出使用。