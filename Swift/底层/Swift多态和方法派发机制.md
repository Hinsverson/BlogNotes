# Swift多态和方法派发机制
#Swift/底层解析

[Swift 中的动与静 - 简书](https://www.jianshu.com/p/7fe0b4f8520d)
[深入理解 Swift 的方法派发 - 简书](https://www.jianshu.com/p/cfe7da01880d)
[深入理解 Swift 派发机制 - zzfx - 博客园](https://www.cnblogs.com/feng9exe/p/9094705.html)

[Swift5.0 的 Runtime 机制浅析 - 掘金](https://juejin.im/post/5d29fb63e51d4510aa01159d#heading-7)

函数派发就是程序判断使用哪种途径去调用一个函数的机制。

# 各种语言常见的三种函数派发方式
## Message动态派发
基于消息是最为灵活的一种派发方式, 最大限度的允许开发者在运行时修改函数的行为。以 Objective-C/ 为例，所有对象都拥有一个 isa指针，可以通过该指针找到对应的方法列表。方法列表中存储着该类实现的方法(不包括父类实现的方法)以及指向父类方法列表的指针，当消息派发时, 会沿着类的方法列表到父类的方法列表(Super指针)的顺序寻找方法实现。在Swift中，Dynamic关键字可以为方法加上OC的运行时特性。
![](Swift%E5%A4%9A%E6%80%81%E5%92%8C%E6%96%B9%E6%B3%95%E6%B4%BE%E5%8F%91%E6%9C%BA%E5%88%B6/91397DAE-34BA-48C1-BDB0-B210023152C9.png)
![](Swift%E5%A4%9A%E6%80%81%E5%92%8C%E6%96%B9%E6%B3%95%E6%B4%BE%E5%8F%91%E6%9C%BA%E5%88%B6/D0C47151-46F8-4F8B-A695-EED36A15ED46.png)
通俗的讲就是除了自身定义的方法和重写的父类方法外，还可以通过isa找到指向父类的方法列表。

## Table函数表派发
函数表是最为常见的函数派发方式。同Message Dispatch类似，所有类也会维护一个自己的函数表，不同的是所有未被重写的父类所实现的函数地址都会拷贝在这个表中，而不是由一个指向父类方法表的指针替代。

由于少了一步指针寻址步骤，在派发效率上要比基于消息的派发高效，但是在灵活性上打了折扣。

查表是一种简单, 易实现, 而且性能可预知的方式. 然而, 这种派发方式比起直接派发还是慢一点. 从字节码角度来看, 多了两次读和一次跳转, 由此带来了性能的损耗. 另一个慢的原因在于编译器可能会由于函数内执行的任务导致无法优化. (如果函数带有副作用的话)

这种基于数组的实现, 缺陷在于函数表无法拓展. 子类会在虚数函数表的最后插入新的函数, 没有位置可以让 extension 安全地插入函数. 这篇 [提案](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151207/001922.html) 很详细地描述了这么做的局限.

在Swift中, 该表被称为Witness Table
![](Swift%E5%A4%9A%E6%80%81%E5%92%8C%E6%96%B9%E6%B3%95%E6%B4%BE%E5%8F%91%E6%9C%BA%E5%88%B6/397AAB3A-EC1D-4527-8110-8D3F0C41BAC0.png)
通俗的讲就是除了自身定义的方法和重写的父类方法外，还会把没有被子类重写的父类实现的方法也会拷贝到这张表里。

## 静态派发
直接派发是效率最高的，在编译阶段就能确定调用的函数地址，但是缺乏了动态特性，也不支持继承（Swift 结构体）。

# Swift方法派发机制
[Swift 性能优化(1)——基本概念 - 掘金](https://juejin.im/post/5e47b388e51d4527214ba8b4)
Swift理论上支持了全部三种派发方式，根据具体的使用场景和关键字决定派发方式：声明的位置、 引用类型、指定派发方式、显式优化。
**final：**：final允许类里面的函数使用直接派发， 这个修饰符会让函数失去动态性。任何函数都可以使用这个修饰符，就算是extension里本来就是直接派发的函数， 这也会让Objective-C Runtime获取不到这个函数, 不会生成相应的selector。
**dynamic**：dynamic可以让类里面所有的函数使用消息机制派发，使用时必须导入Foundation包，里面包括了NSObject和Objective-C的Runtime。dynamic可以用在所有NSObject的子类和所有Swift原生类，也可以让extension中的函数能够被继承。
**@objc & @nonobjc**：@objc和@nonobjc显式地声明了一个函数能否被Objective-C Runtime捕捉到。使用@objc的典型例子就是给selector一个命名空间，让这个函数可以在运行时被调用。@nonobjc表示不让这个函数注册到Runtime中，由此禁止消息机制来派发这个函数，和final非常相似。
**final @objc**：可以同时使用final和@objc来修饰函数，这样做的结果就是，调用函数时会直接派发，但可以将函数注册到Objective-C Runtime中，来让函数可以响应perform(selector:)或者其他特性。
**@inline**：可以通过@inline来使用直接派发，但是同时使用dynamic @inline修饰时，会使用消息机制派发。

![](Swift%E5%A4%9A%E6%80%81%E5%92%8C%E6%96%B9%E6%B3%95%E6%B4%BE%E5%8F%91%E6%9C%BA%E5%88%B6/5E75BC93-BA53-4ECB-A273-D6F526FE51AB.png)
> 一个函数没有 override，即使定义在申明里，Swift 就可能会使用直接派发的方式，所以如果属性绑定了 KVO，那么属性的 getter 和 setter 方法可能会被优化成直接派发而导致 KVO 的失效，所以记得加上 dynamic 的修饰来保证有效（也是为什么swift里Selelctor需要标记为@objc的原因）。  

### 总结
协议、类声明作用域中的函数是Witness Table派发。
值类型、协议和类的extension、final、 @inline标记的函数都是静态派发。
dynamic标记后走runtime的消息发送。

**举例1：**
``` Swift
protocol MyProtocol {
    // 有申明时采用函数表派发
    // func extensionMethod()
}
struct MyStruct: MyProtocol {}

extension MyStruct {
    func extensionMethod() {
        print("In Struct")
    }
}
extension MyProtocol {
    func extensionMethod() {
        print("In Protocol")
    }
}
 
let myStruct = MyStruct()
let proto: MyProtocol = myStruct
myStruct.extensionMethod() // -> “In Struct”
proto.extensionMethod() // -> “In Protocol”
```
扩展内采用静态派发，编译时确定函数地址，所以：
myStruct执行的是In Struct
proto执行的是In Protocol。
如果在协议内把申明加上，则会走函数表派发，会产生类似的多态行为，此时proto执行的In Struct。

**举例2：**
``` Swift
protocol MyProtocol {
    func testFunc()
}

extension MyProtocol {
    func testFunc() {
        print("in extension")
    }
}

class MyClass: MyProtocol {
    func testFunc() {
        print("in class")
    }
}

let prot: MyProtocol = MyClass() //in class
prot.testFunc()
let object: MyClass = MyClass() //in class
object.testFunc()
```
都走函数表派发