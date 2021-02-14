# 总结-Swift
#swift/总结

[Swift官方文档笔记](bear://x-callback-url/open-note?id=28D8C9A4-4F2D-4299-9A24-F4A5DF8F1220-415-0000405FC7B7BACB)

# 基础+Swift进阶笔记整理
[About Swift — The Swift Programming Language (Swift 5.1)](https://docs.swift.org/swift-book/)[Swift 教程 - SwiftGG](https://swiftgg.gitbook.io/swift/swift-jiao-cheng) 

## 数组
[2.1数组](https://github.com/Liaoworking/Advanced-Swift/blob/master/%E7%AC%AC%E4%BA%8C%E7%AB%A0%EF%BC%9A%E5%86%85%E5%BB%BA%E9%9B%86%E5%90%88%E7%B1%BB%E5%9E%8B/2.1%20%E6%95%B0%E7%BB%84.md) 
::Swift的数组和NSMutableArray不一样在于前者是struct实现的值类型结构，采用写时复制保证不必要的内存copy浪费。::
Swift不提倡在数组直接使用下标索引，理解dropFirst、dropLast、enumerated、firstIndex、map、filter、flatMap、compactMap、reduce、forEach常见使用和功能。
::forEach的return相当于for-in中用continue跳过这次闭包的执行，不会离开大括号作用域，并且闭包里不能用break和continue）::
数组和String的slice切片操作都是返回一个Slice类型，指向同一块内存区域，并不会产生内存的复制。

## 字典
[2.2字典](https://github.com/Liaoworking/Advanced-Swift/blob/master/%E7%AC%AC%E4%BA%8C%E7%AB%A0%EF%BC%9A%E5%86%85%E5%BB%BA%E9%9B%86%E5%90%88%E7%B1%BB%E5%9E%8B/2.2%20%E5%AD%97%E5%85%B8.md) 
updateValue(_:forKey:)更新元素，返回的是可选类型的旧值
merge(_:uniquingKeysWith:)闭包内是当key重复时的处理
::当用值类型作为dic的key时，如果改变了key那么可能导致哈希值和/或相等特性往往也会发生改变，导致错误的dic存储和使用。::

## 集合
[2.3 set 2.4 Range](https://github.com/Liaoworking/Advanced-Swift/blob/master/%E7%AC%AC%E4%BA%8C%E7%AB%A0%EF%BC%9A%E5%86%85%E5%BB%BA%E9%9B%86%E5%90%88%E7%B1%BB%E5%9E%8B/2.3%20set%202.4%20Range%20.md) 
集合可以理解为没有value的字典，元素同样需要遵循hashable协议。
集合代数运算来源于SetAlgebra协议，Foundation还有IndexSet、CharacterSet也实现了该协议。
::indexSet这种数据结构比Set更高效，因为它只会在内部存储首尾2个范围。::
Range和ClosedRange满足Comparable协议，CountableRange和CountableClosedRange满足Strideable协议，并且后者是可迭代的。
Collection协议里下标操作是接受RangeExpression协议这个类型的参数，上下界缺失会自动补firstIndex或endIndex，比如[..<]。

## sequence
[3.1 序列](https://github.com/Liaoworking/Advanced-Swift/blob/master/%E7%AC%AC%E4%B8%89%E7%AB%A0%EF%BC%9A%E9%9B%86%E5%90%88%E7%B1%BB%E5%9E%8B%E5%8D%8F%E8%AE%AE/3.1%20%E5%BA%8F%E5%88%97.md) 
通过sequence协议自定义一个序列的一般过程？（先创建迭代器），::掌握下面AnyIterator、AnySequence的使用方式：::
``` swift
/// 通过引用语义的特性写斐波那契
func fibsIterator() -> AnyIterator<Any> {
    var startNum = (0, 1)
    return AnyIterator {
        let nextNum = startNum.0
            startNum = (startNum.1 , startNum.0 + startNum.1)
        return nextNum
    }
}
let fibsSequence = AnySequence(fibsIterator)
Array(fibsSequence.prefix(10)) // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]”
```
::知道自定义Iterator时有值类型和引用类型的差别，AnyIterator采用类型消除，将原来的迭代器包装到内部的对象中，它是引用语意的。::
Iterator的生成是延迟的，只在需要的时候生成，Iterator(10)只会生成前10个元素。
序列Sequence可以是无限的，而集合Collection是有限的。
::Sequence是流动的，不稳定的，也就是说并不保证多次遍历情况一样，因为它不关心遵循协议的类型是否会在迭代后将序列元素销毁，所以它没有first属性，而Collection集合是能保证并且有的。::
迭代器Iterator可以看成即将返回的元素组成的不稳定序列sequence，也就是说可以单纯的将Iterator申明为Sequence，因为Sequence提供了一个默认makeIterator实现，对于满足协议的迭代器类型将返回迭代器本身。

## Collection
[3.2集合类型](https://github.com/Liaoworking/Advanced-Swift/blob/master/%E7%AC%AC%E4%B8%89%E7%AB%A0%EF%BC%9A%E9%9B%86%E5%90%88%E7%B1%BB%E5%9E%8B%E5%8D%8F%E8%AE%AE/3.2%E9%9B%86%E5%90%88%E7%B1%BB%E5%9E%8B.md)  [3.3索引](https://github.com/Liaoworking/Advanced-Swift/blob/master/%E7%AC%AC%E4%B8%89%E7%AB%A0%EF%BC%9A%E9%9B%86%E5%90%88%E7%B1%BB%E5%9E%8B%E5%8D%8F%E8%AE%AE/3.3%E7%B4%A2%E5%BC%95.md)  [3.4切片](https://github.com/Liaoworking/Advanced-Swift/blob/master/%E7%AC%AC%E4%B8%89%E7%AB%A0%EF%BC%9A%E9%9B%86%E5%90%88%E7%B1%BB%E5%9E%8B%E5%8D%8F%E8%AE%AE/3.4%E5%88%87%E7%89%87.md) [3.5专门的集合类型](https://github.com/Liaoworking/Advanced-Swift/blob/master/%E7%AC%AC%E4%B8%89%E7%AB%A0%EF%BC%9A%E9%9B%86%E5%90%88%E7%B1%BB%E5%9E%8B%E5%8D%8F%E8%AE%AE/3.5%E4%B8%93%E9%97%A8%E7%9A%84%E9%9B%86%E5%90%88%E7%B1%BB%E5%9E%8B.md) 
Collection遵循了Sequence协议，并且是稳定的，所以有count属性。
swift数组可以当做栈用，append入栈，popLast出栈。也可以当作队列用，push入队和remove(at:0)出队。

## Option
[4.1—4.3 序列-魔法数问题-可选值概览](https://github.com/Liaoworking/Advanced-Swift/blob/master/%E7%AC%AC%E5%9B%9B%E7%AB%A0%EF%BC%9A%E5%8F%AF%E9%80%89%E5%80%BC/4.1_3%20%E5%BA%8F%E5%88%97_%E9%AD%94%E6%B3%95%E6%95%B0%E9%97%AE%E9%A2%98_%E5%8F%AF%E9%80%89%E5%80%BC%E6%A6%82%E8%A7%88.md)  [4.4 强制解包的时机](https://github.com/Liaoworking/Advanced-Swift/blob/master/%E7%AC%AC%E5%9B%9B%E7%AB%A0%EF%BC%9A%E5%8F%AF%E9%80%89%E5%80%BC/4.4%20%E5%BC%BA%E5%88%B6%E8%A7%A3%E5%8C%85%E7%9A%84%E6%97%B6%E6%9C%BA.md)   [4.5 多灾多难的隐式可选值](https://github.com/Liaoworking/Advanced-Swift/blob/master/%E7%AC%AC%E5%9B%9B%E7%AB%A0%EF%BC%9A%E5%8F%AF%E9%80%89%E5%80%BC/4.5%20%E5%A4%9A%E7%81%BE%E5%A4%9A%E9%9A%BE%E7%9A%84%E9%9A%90%E5%BC%8F%E5%8F%AF%E9%80%89%E5%80%BC.md) 
option的本质是枚举。
::OC中对nil对象发消息不会崩溃，因为运行时进行了处理，相当于什么都不做但如果是野指针则直接崩溃。::
知道Swift一般不使用强解包，用map和filter或者可选链进行过滤。
::OC和swift混编时，注意OC函数返回类型会隐式可选值，在swift里使用时要手动用？。::
```swift
NSString *string = [Person alloc]init].name;
/// 在swift中去调用时  我们这样写，编译器不会报错 但是当saySomething为nil的时候就会崩溃
Person().name.count ❌
/// 我们需要自己在name属性后主动添加?防止因为隐式可选值引起的崩溃。
Person().name?.count ✅
```

## 结构体和类
[5.1 值类型—5.2 可变性](https://github.com/Liaoworking/Advanced-Swift/blob/master/%E7%AC%AC%E4%BA%94%E7%AB%A0%EF%BC%9A%E7%BB%93%E6%9E%84%E4%BD%93%E5%92%8C%E7%B1%BB/5.1_2%E5%80%BC%E7%B1%BB%E5%9E%8B_%E5%8F%AF%E5%8F%98%E6%80%A7.md) [5.3 结构体](https://github.com/Liaoworking/Advanced-Swift/blob/master/%E7%AC%AC%E4%BA%94%E7%AB%A0%EF%BC%9A%E7%BB%93%E6%9E%84%E4%BD%93%E5%92%8C%E7%B1%BB/5.3%20%E7%BB%93%E6%9E%84%E4%BD%93.md)  [5.4 写时复制](https://github.com/Liaoworking/Advanced-Swift/blob/master/%E7%AC%AC%E4%BA%94%E7%AB%A0%EF%BC%9A%E7%BB%93%E6%9E%84%E4%BD%93%E5%92%8C%E7%B1%BB/5.4%20%E5%86%99%E6%97%B6%E5%A4%8D%E5%88%B6.md) [5.5_6 闭包和可变性_内存](https://github.com/Liaoworking/Advanced-Swift/blob/master/%E7%AC%AC%E4%BA%94%E7%AB%A0%EF%BC%9A%E7%BB%93%E6%9E%84%E4%BD%93%E5%92%8C%E7%B1%BB/5.5_6%20%E9%97%AD%E5%8C%85%E5%92%8C%E5%8F%AF%E5%8F%98%E6%80%A7_%E5%86%85%E5%AD%98.md) [5.7_8 闭包和内存](https://github.com/Liaoworking/Advanced-Swift/blob/master/%E7%AC%AC%E4%BA%94%E7%AB%A0%EF%BC%9A%E7%BB%93%E6%9E%84%E4%BD%93%E5%92%8C%E7%B1%BB/5.7_8%20%E9%97%AD%E5%8C%85%E5%92%8C%E5%86%85%E5%AD%98.md) 
::讨论struct和class的区别可以从内存分布、继承性、值引用语意、语法等几个层次进行讨论。::
理解可变性，不要在Swift里for-in遍历NSMutableArray同时在内部删除元素，这样会崩溃，使用inout参数时也应该注意内存访问一致性的原则。
Swift中isKnownUniquelyReferenced方法判断是否是唯一的引用，常常用在自定义写时复制的结构上（set修改时检查唯一引用，不满足则创建新对象包住修改后的新值并返回）。
::字典、集合这类值语意的结构相比数组通过下标的方式访问值类型元素会发生写时复制，而数组不会发生写时复制，因为Swift数组下标的实现是进行特殊处理过的，底层是通过[地址器](https://github.com/apple/swift/blob/master/docs/proposals/Accessors.rst)直接访存。::
值类型的属性被赋值时, 它的didSet方法会被调用，同样改变数组中元素的属性时数组的didSet也会被调用。
::mutating方法的本质是隐式的 self 被标记为了 inout 而已，sort()和sorted()的区别在于前者是原地排序，不会产生一个新对象。::
::Swift结构体默认也是分布在堆上，但是一般非逃逸情况下会进行编译器优化最终分布在栈上，逃逸会破坏这种优化，因为需要捕获的变量在栈帧之外依然存在。::

## 函数
::知道swift中的排序算法是基于内省算法（introsort），实质是快排和堆排的混合，当集合很小时会转化为插入排序，避免不必要的性能消耗。::

## 字符串
  [7.1 不再固定宽度](https://github.com/Liaoworking/Advanced-Swift/blob/master/%E7%AC%AC%E4%B8%83%E7%AB%A0%EF%BC%9A%E5%AD%97%E7%AC%A6%E4%B8%B2/7.1%20%E4%B8%8D%E5%86%8D%E5%9B%BA%E5%AE%9A%E5%AE%BD%E5%BA%A6.md)   [7.3 简单的正则表达式匹配器。 7.4 ExpressibleByStringLiteral](https://github.com/Liaoworking/Advanced-Swift/blob/master/%E7%AC%AC%E4%B8%83%E7%AB%A0%EF%BC%9A%E5%AD%97%E7%AC%A6%E4%B8%B2/7.3%20%E7%AE%80%E5%8D%95%E7%9A%84%E6%AD%A3%E5%88%99%E8%A1%A8%E8%BE%BE%E5%BC%8F%E5%8C%B9%E9%85%8D%E5%99%A8%E3%80%82%20%207.4%20ExpressibleByStringLiteral)   [7.7 CustomStringConvertible 和 CustomDebugStringConvertible](https://github.com/Liaoworking/Advanced-Swift/blob/master/%E7%AC%AC%E4%B8%83%E7%AB%A0%EF%BC%9A%E5%AD%97%E7%AC%A6%E4%B8%B2/7.7%20CustomStringConvertible%20%E5%92%8C%20CustomDebugStringConvertible.md) 
Unicode拥有可变长度的特性，原因是不同语言的字符存放字节数不同，若都统一长度，效率太低，这也是为什么String 不支持通过下标操作符Str[i]来替换字符。

## 错误处理
::理解defer是在返回语句之后但是真正返回之前获得执行[#19: Defer · objc.io](https://www.objc.io/quiz/19/)::
```swift
var counter = 5

func increment() -> Int {
defer { counter += 1 }
return counter
}

counter = increment() //5
```

## 泛型
> ::理解Swift范型的底层实现原理？::  
> Swift范型重载和C++类似也是在编译时就确定了，范型的目标是一次编译动态派发，但是会导致运行时性能的降低，使用范型特化优化可以解决这个问题。  
>   
> 泛型定义和调用在同一个文件中时，泛型特化才能工作，也可以开启-whole-module-optimization 进行全局优化。  

# 探究底层本质
[Swift各种属性的本质](bear://x-callback-url/open-note?id=6BB48820-F3D7-454E-8885-62C85ACCA619-3605-000092F913ECACAE)
Swift里let和var的理解？
Swift里计算型属性的本质是什么？占多少个字节？是存储在当前对象里的吗？可以用let修饰吗？
枚举的原始值rawValue的本质是什么？占几个字节？它在内存中是存储在枚举里吗？
::lazy属性可以用let修饰吗？lazy属性是线程安全的吗？::
观察型属性在初始化的时候会触发吗？定义的时候给定默认值会触发吗？
::Swift里inout修饰的函数参数传递本质的理解？::
::inout的参数能传递计算属性吗？传递计算属性的底层原理copy on copy out是什么？::
::inout参数传递观察型属性会触发观察的willset和didset方法吗？底层原理又是什么？为什么这样设计？::
枚举可以定义存储属性吗？枚举可以定义类型存储属性吗？
::类型存储属性和lazy一样是延迟加载吗？如果一样那它是线程安全的吗？为什么？::

[Swift闭包的本质](bear://x-callback-url/open-note?id=C48543E5-74D5-44ED-80B4-2EED1E398A2F-3605-000092F8DCBF81DE)
Swift闭包的理解，本质是什么？理解闭包表达式和闭包？
::Swift闭包值捕获的原理是什么？捕获到的值存储在哪里？捕获多个值时它们在内存中是连续存储的吗？::一个捕获到int值的闭包在内存中占几个字节？
::swift闭包的捕获发生在什么时候？对默认隐式捕获的理解？对比显示捕获的理解？引用捕获里的强引用的理解？weak和unowned的区别？Struct成员方法内部捕获self时的捕获行为是什么（这里暂时有点疑问🤔️）？::
理解DispatchQueue.async闭包体内为什么要强制加self.访问成员变量？对逃逸闭包的理解？
Swift里的??运算符的作用？本质是什么？

[Swift的内存布局](bear://x-callback-url/open-note?id=8BB91D1D-0E8B-45E7-B8B4-B8370BE9F0A0-3605-000092F145B6DB60)
理解内存布局中堆和栈的区别？
::Swift一个String类型占多少个字节？String类型变量的字面量在内存中是怎样存储的，字面量长度小于16个字节是怎样存储的？大于16个字节又是怎样存储的？如果一个string变量一开始字面量小于16个字节，而后拼接超过了16个字节？::
如何计算一个Swift数组在内存中的大小？数组实际存储在栈空间还是堆空间？
Swift可选类型的本质？
简单枚举的内存布局？占几个字节？
带关联值的布局是怎样的？占几个字节？
带原始值的布局是怎样的？占几个字节？
::只有一个成员的简单枚举占几个字节？::
Swift中结构体的内存布局方式？分布在哪里？
::Swift中类的内存布局？分布在哪里？理解HeapObject结构？理解HeapMetadata？Swift和OC类对象的实现上有什么相似点？::
Swift中的协议的内存布局方式？Existential Container 的规则是什么？::遵循协议的对象属性存储在什么地方？VWT是什么？PWT又是什么？::
Swift函数范型多态的原理？它是用Existential Container实现的吗？::如果不是是如何实现的？::
Any类型的内存布局？

[Swift 方法派发](bear://x-callback-url/open-note?id=6EED9C8B-6FE6-4714-AB4A-5AB4D8E0D433-3605-000092F8B81588A1)
理解Swift多态？Swift支持哪些方法派发方式？3种常见派发方式有什么优缺点？::协议、类声明作用域中的函数走的什么派发？值类型、协议和类的extension、final、 @inline标记的函数走的什么派发？dynamic标记的函数走的什么派发？::@ objc和@nonobjc理解以及为什么Selector函数需要加这个标记？为什么官方建议使用结构体+协议的组合而不使用class类型？

[[Swift Runtime 分析]]

[[swift编译相关]]

# 特性和优化
[Swift性能优化](bear://x-callback-url/open-note?id=36EA4D11-F26E-4C19-BB1D-3324AEF6B5BB-3605-000092F95F2A348F)
如何优化Swift性能可以从哪些方向入手，怎么优化？

# 其他
[Swift和OC混编的坑](bear://x-callback-url/open-note?id=4FF692B5-36DB-4226-BA44-F9994EFBAB99-1795-00033F6DB9C0DB78)

[Swift里的指针](bear://x-callback-url/open-note?id=63AF3D94-39D9-40C5-A822-E8C756145F82-3605-000092F92FB14E25)
Swift里有那几种类型的指针？有什么区别？
为什么指针是unsafe的？
如何获取实例对象的地址？
Unmanaged passRetained的作用？Unmanaged passUnretained的作用？使用时如何选择？

[[常见Tips整理]] 
Self和self的区别？.self的理解？
.type和type(of: AnyObject)的区别和理解？
AnyObject、Anyclass、Any的区别和理解？
::isKind、is、isMember的区别和理解？::

[[Swift写时复制]]





