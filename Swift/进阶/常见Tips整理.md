# 常见Tips整理
#Swift/进阶

# Array 和 ContiguousArray 区别
正因为储存策略的不同，特别是在class 或者 @objc，如果不考虑桥接到 NSArray 或者调用 Objective-C，苹果建议我们使用 ContiguousArray，会更有效率。
[Swift标准库源码阅读笔记 - Array和ContiguousArray_weixin_33763244的博客-CSDN博客](https://blog.csdn.net/weixin_33763244/article/details/88114154)


[swift 中 Self 与self - 简书](https://www.jianshu.com/p/a6bcdebd83f5)
[swift self Self 使用 - weixin_34326558的博客 - CSDN博客](https://blog.csdn.net/weixin_34326558/article/details/87724580)

# Self和self
self：指代当前实例对象
Self：指代当前类型和子类型，一般用在协议中

# .self（访问类型的一种方式）
用在某个实例后面取得这个实例本身（实例值），指向实例对象的指针。
用在类型后面取得类型本身（类型值），指向元类的指针。

# .Type 和 type(of:  AnyObject)
假设有一个类，类名叫A
```
let AMetaType: A.Type = A.self
```
A.Type：A类对象的原类类型
A.self：A类对象指向原类的类型值

可以类比下面的定义
```
let intValue: Int = 4
```
Int：变量intValue的类型
4：变量intValue的值

type(of:  AnyObject) 等同objc_getclass(self)
本质：把实例对象堆空间前8个字节取出来赋值给ptype（类对象）
作用：获取该实例运行时阶段的类型
```
var pType: p.Type = type(of: p)
```

# AnyObject 和 Any
AnyObject ：代表任何class实例对象的类型
Any ：代表任意类型（枚举、结构体、类、函数类型）

# AnyClass
只是AnyObject.Type的类型别名，代表任意的原类类型。
`typealias AnyClass = AnyObject.Type`

# isKind、is、isMember
is: （swift特有）判断是否为某种结构体或枚举等类型。对于实例对象的判断，is等同于 isKind(of: ) 的作用，即判断是否为本类或子类的类型。
isMember(of: ): 判断对象是否为某个特定类的实例（来自OC的NSObject）
isKind(of: ): 判断对象是否为某类或者其派生类的实例 （来自OC的NSObject）

# 结构体和类的区别？
类是引用类型，实例是通过引用传递
结构体是值类型，实例是通过值传递

与结构体相比，类还有如下的附加功能
1. 继承允许一个类继承另一个类的特征
2. 析构器允许一个类实例释放任何其所被分配的资源
3. 类型转换允许在运行时检查和解释一个类实例的类型
4. 引用计数允许对一个类的多次引用

# Swift 如何实现单例？
```
final public class Manager {
		static let shared = Manager()
		private init(){}
}
```
![](%E5%B8%B8%E8%A7%81Tips%E6%95%B4%E7%90%86/AC4B977C-6EFA-49C5-8CC8-988B81DF49B0.png)


# Swift如何实现DispatchOnce？
使用类型属性、或者全部变量/常量会自带lazy+DispatchOnce效果。

# Swift实现多重继承？
通过协议的方式，Swift中面向协议模拟了多继承的关系，通过协议的扩展解决了多继承中的菱形问题（A是BC的父类，D同时继承BC，BC中存在相同方法名时，不知道该调用哪个）。


