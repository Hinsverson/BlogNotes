# Swift里的指针
#Swift/进阶

[Swift里的指针](bear://x-callback-url/open-note?id=63AF3D94-39D9-40C5-A822-E8C756145F82-3605-000092F92FB14E25)
Swift里有那几种类型的指针？有什么区别？
为什么指针是unsafe的？
如何获取实例对象的地址？
Unmanaged passRetained的作用？Unmanaged passUnretained的作用？使用时如何选择？

# 指针的类型
[Swift三部曲（一）：指针的使用 - 掘金](https://juejin.im/post/5d0dde2cf265da1baa1e7d50)
https://www.raywenderlich.com/7181017-unsafe-swift-using-pointers-and-interacting-with-c#toc-anchor-009
https://www.jianshu.com/p/178f161011ca

[Swift内存赋值探索二： 指针在Swift中的使用 - 简书](https://www.jianshu.com/p/c2a1d758b90e)

[译Unsafe Swift - 指针与C交互 - 掘金](https://juejin.im/post/5a58703651882573473db316)

[关于Swift中的指针的那些事 - 掘金](https://juejin.im/post/5c2cc325518825480635dd39)

[在 Swift 里安全管理指针_老司机技术周报-CSDN博客](https://blog.csdn.net/sinat_35969632/article/details/108396403)

![](Swift%E9%87%8C%E7%9A%84%E6%8C%87%E9%92%88/A0199315-9C0C-40D6-B49D-FE1F459D2B6B.png)
主要有2种（指定类型的和没指定类型的），每一种又分为可变的或不可变的。

# 获取变量指针
![](Swift%E9%87%8C%E7%9A%84%E6%8C%87%E9%92%88/047421B7-6AC8-40DB-801A-171E82EFC104.png)
withUnsafeMutablePointer、withUnsafePointer方法，返回指向传入对象的指针并带上对应类型。

UnsafeRawPointer($0) 
$0为UnsafePointer，表示把带类型的指针转成不带类型的指针。

带类型的指针.pointee相当于取出指针所指向的内容。 
不带类型的指针.storeBytes相当于存储修改内容，.load相当于访问内容。

# 获取指向堆空间指针
![](Swift%E9%87%8C%E7%9A%84%E6%8C%87%E9%92%88/235E6D08-6EC0-4786-A9FA-53525CB70390.png)
ptr指针此时应该在栈空间，存储着一个指向person实例对象堆空间地址的指针。
UnsafeRawPointer(bitPattern: )接收一个地址值，ptr.load就是取出ptr指针存储的那个指向person实例对象堆空间地址的指针，这是一个堆空间指针。

另一种方式：Unmanaged.passUnretained($0).toOpaque()

# 创建指针
![](Swift%E9%87%8C%E7%9A%84%E6%8C%87%E9%92%88/0B62174C-2568-47B7-8FA2-3281E1DFA0D8.png)
这里的storeBytes和load后面的of和by都是偏移量，相当于存储/访问指针地址后几个字节的数据。

# 指针转换
![](Swift%E9%87%8C%E7%9A%84%E6%8C%87%E9%92%88/CC6ED542-9C73-4473-B3ED-D4EE0890A7DB.png)

# 和CF框架的内存交互
[Swift三部曲（一）：指针的使用 - 掘金](https://juejin.im/post/6844903872905871367#heading-16)
[Swifter - Swift 必备 tips](https://swifter.tips)

Cocoa 框架中的大部分 NS 开头的类其实在 CF 中都有对应的类型存在，可以说 NS 只是对 CF 在更高层面的一个封装，比如 NSURL 和它在 CF 中的 CFURLRef 内存结构其实是同样的，而 NSString 则对应着 CFStringRef。

在 Objective-C 中 ARC 负责的只是 NSObject 的自动引用计数，因此对于 CF 对象无法进行内存管理，我们在把对象在 NS 和 CF 之间进行转换时需要向编译器说明是否需要以及怎样转移内存的管理权（详细了解Toll-Free Bridging技术）。

同时在Objective-C 中对于 CF 系的 API，如果 API 的名字中含有 Create，Copy 或者 Retain 的话，在使用完成后我们需要调用对应的 CFRelease 方法来进行释放。

但在Swift中因为只支持ARC管理，官方在系统提供的标准API里，在合适的地方加上了像 CF_RETURNS_RETAINED 和 CF_RETURNS_NOT_RETAINED 这样的标注，所以在Swift里不再需要再调用对应的 CFRelease 方法来进行手动释放了，同时在OC中CF对象一般以Ref结尾，比如CFStringRef，而为了不致于迷惑性，Swift使用typealias进行了处理，CFStringRef直接变为CFString，达到了一种貌似ARC也能自动管理CF对象的效果。

但是有一点例外，那就是对于非系统的 CF API (比如你自己写的或者是第三方的)，因为并没有强制机制要求它们一定遵照 Cocoa 的命名规范，所以贸然进行自动内存管理是不可行的。如果你没有明确地使用上面的标注来指明内存管理的方式的话，将这些返回 CF 对象的 API 导入 Swift 时，它们的类型会被对对应为 Unmanaged<T>，通过Unmanaged来管理，它封装了一些内存操作方法便于我们去管理这些不被ARC管理的对象内存。

## Unmanaged<T>（托管对象）：
Unmanaged 主要扮演CF和Swift对象交互时内存的中间管理者角色。
一个 Unmanaged 实例封装有一个 CF类型 T，它在相应范围内持有对该 T 对象的引用。

Unmanaged 将由 CF 传递过来的CF对象进行托管，通过Unmanaged提供的方法标定它是否接受引用计数的分配，以便实现类似内存自动释放的效果，而无需我们通过指针手动去管理这些CF对象的内存。

### 向C函数中传递Swift对象
* passRetained：增加被管理Swift对象的引用计数，然后转化为C指针，效果类似OC的__bridge_retained，移交了管理权，意味着后续需要。
* passUnretained：不增加被管理Swift对象的引用计数，然后为C的指针，效果等于OC中的OC到CF的__bridge，没有移交管理权，该Swift对象仍由ARC管理。

### 从托管对象中获取Swift对象
* takeUnretainedValue：当明确的知道获得对象时并不会获得该对象的所有权时使用，意味着你不需要负责对象的释放，被托管对象的引用计数不变，比如Foundation内一些Get函数，效果等同OC中的CF到OC的__bridge。
* takeRetainedValue：当明确的知道获得对象时同时会获得该对象的所有权时使用，意味着你需要需要负责对象的释放，所以使用这个方法后被托管对象的引用计数会自动-1，比如CF框架内一些Create或Copy函数 [参考Ownership Policy](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/Concepts/Ownership.html) 。效果等同于OC中的__bridge_transfer。

> CF框架内一些Create或Copy函数，当调用者不再使用对象时候，在Swift代码中不需要调用CFRelease函数放弃对象所有权，这是因为Swift仅支持ARC内存管理，这一点和OC对象同CF对象交互时内存管理略Toll-Free Bridging 的处有不同。  

### 使用场景举例🌰
1. 使用托管的方式向C的API或者回调函数传递一个Swift对象，并转化为 UnsafeMutableRawPointer（C指针），具体使用passRetained还是passUnretained取决于当前这个user对象的生命周期是否超过编译器：
```swift
Unmanaged<Person>.passRetained(user).toOpaque() 
Unmanaged<Person>.passUnretained(user).toOpaque()
```
如果在C的API或者回调函数使用的时候，我们能保证这个user对象一直存活（比如全局持有），那么就使用passUnretained的方式不移交管理权。
如果这个对象可能被销毁，那么就需要用passRetained增加它的引用计数，保证在C里使用时不会被释放（比如创建的是一个临时对象，但需要传递给C的回调）。

2. 对应的反过来从C回调函数或API里UnsafeMutableRawPointer指针取出一个Swift对象。
```swift
Unmanaged<Person>.fromOpaque(rawPtr).takeUnretainedValue()
Unmanaged<Person>.fromOpaque(rawPtr).takeRetainedValue()
```
如果参数rawPtr被retain过（比如使用passRetained传进来），那么就需要使用takeRetainedValue()的方式取出这个swift对象，做对应引用计数的-1。

# 其他
* Swift数组名等于指向第一个元素的不可变指针，而不是地址值。
* &数组名等于指向第一个元素的可变指针

