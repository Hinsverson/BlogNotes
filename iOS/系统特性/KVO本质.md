# KVO本质
#iOS知识点/系统特性

参考：[iOS 底层探索 - KVO - 掘金](https://juejin.im/post/6844904067794206733#heading-13)

[iOS《Key-Value Observing Programming Guide》官方文档](https://juejin.cn/post/6916448958575280135#heading-25)

# KVO本质
1. KVO的本质是生成中间派生类重写set方法，并通过isa_swizing把对象的isa指向这个中间派生类，中间派生类superclass指针指向原来的类对象，实现监听通知，set方法内实现如下：
	1. 先调用willChange方法通知监听器对象值即将改变（触发observerValueForKeyPath）
	2. 然后再调用父类的setAge方法真的去改变值
	3. 最后didchange再通知监听器对象值已经发生了改变（触发observerValueForKeyPath）
2. 中间类除了 set方法外，还会重写如下方法：
	1. class：对中间类进行伪装，返回原来的类对象。
	2. dealloc：进行派生类自身资源的释放。
	3. isKVOA：标识是否是在观察状态的一个标志位。
3. removeObserver的时候会换回观察对象的isa指向，但并不会马上删除中间类（之后可能会再次KVO）。
4. automaticallyNotifiesObserversForKey实现自动或者手动触发
5. 可以通过keyPathsForValuesAffectingValueForKey结合willChange、didchange方法来手动实现1对多、或者重属关系的KVO监听。

# KVO 理解
1. KVO是属性观察者模式的一种实现，其实也是一种代理模式，效率是比delegate差的。
2. KVO基于走不到set方法的修改是不会触发的，比如：
	1. 观察数组时addObject不会触发，需要使用mutableArrayValueForKey
	2. 直接用->指针修改访问成员变量也是不能触发的。
3. isa指针的值不一定反映实例的实际类，不要依靠isa指针来确定类成员，相反使用该class方法确定对象实例的类。
4. 可以手动触发，重写automaticallyNotifiesObserversForKey方法，通过判断key来进行自由的手动和自动的选择，返回false后然后重写对应属性的set方法，如下：
```objc
- (void)setName:(NSString *)name{
    [self willChangeValueForKey:@"name"];
    _name = name;
    [self didChangeValueForKey:@"name"];
}
```

# 基础使用：
```objc
- (void)addObserver:(NSObject *)observer
         forKeyPath:(NSString *)keyPath
            options:(NSKeyValueObservingOptions)options
            context:(void *)context
```
observer：KVO 通知的对象
keyPath：被观察者的属性的名称
options：枚举类型，主要是观察的属性的变化类型
context：上下文，主要是传递给代理进行判断使用

一旦对某个对象上的属性注册了观察者，可以选择在收到属性值变化后取消注册，也可以在观察者声明周期结束之前(比如：dealloc 方法) 取消注册，如果忘记调用取消注册方法，那么一旦观察者被销毁后，KVO 机制会给一个不存在的对象发送变化回调消息导致**野指针**错误。

重复移除也会崩溃。

# 使用KVO的对象取/赋值原理
![](KVO%E6%9C%AC%E8%B4%A8/76D477F5-CA9F-4648-8D72-E2F88934535B.png)
1. OC中本身实例对象设/取值的原理是通过实例对象的isa指针找到类对象里的get、set方法来完成。
2. 添加了KVO监听后，基于Runtime会在运行时动态生成一个新的派生类对象，并且把实例对象的isa指针指向它，它的superClass指针指向原来的类对象，它的isa指向新的派生类的原类。
3. 之后实例对象的设/取值还是通过isa，但是去这个派生类对象中查找get、set方法完成设/取值。

# Swift 4.0以后的KVO原理
``` swift
@objcMembers class Test: NSObject {
    dynamic var field = "field"
}

var observer: NSKeyValueObservation!
    override func viewDidLoad() {
        super.viewDidLoad()
        let test = Test()
        observer = test.observe(\.field, options: [.new, .initial]) { (object, change) in
            print(change)
        }
        test.field = "change"
    }
```
和OC中变化的点在于，通过一个NSKeyValueObservation来管理runtime生成的派生类NSKVONotifying_Test的生命周期。
NSKeyValueObservation内有一个object属性指向观察者test和callback回调闭包，以及path代指被观察的属性。
当被观察属性发生改变时。 NSKeyValueObservation 作为了观察者和消息转发者，接收通知和通知 test 的属性发生改变，从而调用闭包内的具体操作。


# RXSwift KVO的理解
[RxSwift(十一) KVO基本使用及实现原理探究 - 掘金](https://juejin.im/post/5d72fe6ae51d453b1f37eb90)
RxSwift的KVO底层实现其实是通过中介者模式，外部调用和响应遵循整个Rxswift 核心流程，但是观察者内部会创建一个OC内部类使用OC KVO的方式完成运行时的动态观察，这个内部类作为消息的接受者和转发者（同Swift 4.0之后原生的KVO比较像）把消息转发给观察者，然后调用外部的闭包完成响应。

# KVO注意事项
[关于KVO的那些事 之 KVO安全用法封装 - 简书](https://www.jianshu.com/p/f8bb89aad2df)

* 在以下三种情况下会无情Crash：
	1. 监听者dealloc时，监听关系还存在。当监听值发生变化时，会给监听者的野指针发送消息，报野指针Crash。（猜测底层是保存了unsafe_unretained指向监听者的指针）；
	2. 被监听者dealloc时，监听关系还存在。在监听者内存free掉后，直接会报监听者还存在监听关系而Crash；
	3. 移除监听次数大于添加监听次数。报出多次移除的错误；
* 封装一个稳定安全的KVO，主要就是实现自动移除通知并保证移除的次数和添加的次数相匹配：
	1. 向下hook观察者的dealloc时机做移除，引入管理对象存储添加和移除的映射匹配。
	2. 向上封装，中间代理思想，维护和管理KVO的添加和移除。