# iOS内存管理机制
#iOS知识点/内存相关

[iOS进阶——内存管理 - 掘金](https://juejin.im/post/5b5c51d0e51d45199358c80d)
[OC 知识：彻底理解 iOS 内存管理（MRC、ARC） - 久依 - 博客园](https://www.cnblogs.com/jiuyi/p/10404940.html)
[iOS 内存管理相关面试题 - 掘金](https://juejin.im/post/6844903821290799111)

* 临时变量在栈上，全局变量分布在全局区由系统管理。
* 分布在堆上的对象类型受到计数规则管理，有下面几种管理机制：
	* 一般情况下自己创建并持有的由ARC管理。
	* ARC下非自己创建持有的由最近的AutoreleasePool管理。
	* 和C交互的CF对象类型通过OC bridge或者Swift Unmanaged管理。
* Tagged Pointer也算一种管理机制，被Tagged Pointer指针存储的小对象数据类型受Tagged Pointer指针自身内存管理的影响。

# 系统自动管理
普通基本数据类型、函数参数、指针这些临时变量分布在栈上，离开方法作用域栈帧回退后被释放掉。
全局变量分布在全局区，属于进程，跟随进程的生命周期。

# MRC 手动对象内存管理
## 引用计数
根据引用计数是否为0来判断是否需要dealloc回收一个对象所占用的内存。

### 引用计数的存储
![](iOS%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/B52A88FA-DD24-4C30-935A-D38BAE4BF0CB.png)
OC：64位优化后的isa也可以直接存储引用计数，isa内不够存储会通过对象地址作为key，对象的引用计数为value存储在SideTable的refcents散列表中。
Swift：存储在HeapObject的InlineRefCounts的结构体中，在堆空间上占8个字节。如果不够存储也会放到SideTable里面（Swift的SideTable和OC不一样的是它不是一个全局的东西了）。

## 核心规则
* 自己生成的对象，自己持有：alloc/new/copy/mutableCopy等方法需要手动负责返回对象的释放。
* 非自己生成的对象，自己也能持有：retain持有
* 不再需要自己持有的对象时释放：realse释放
* 非自己持有的对象无法释放（或者使用autorelease）

## 操作方法
* retain：找到引用计数存储位置，将该对象的引用计数器加1。
* release：找到引用计数存储位置，将该对象的引用计数器减1。
* autorelease：不改变该对象的引用计数值，将对象添加到自动释放池中。
* retainCount：返回该对象的引用计数的值。

# ARC 自动对象内存管理
对象的内存管理规则和MRC一致通过引用技术，主要由编译器+runtime驱动完成，分析上下文代码生命周期，自动在合适的地方插入了retain和release、autorelease操作。

**注意点**
* 不能使用 retain / release / retainCount / autorelease
* 不能使用 NSAllocateObject / NSDeallocateObject
* 须遵守内存管理的方法命名规则
* 不能显式调用 dealloc
* 使用 @autoreleasepool 块替代 NSAutoreleasePool
* 不能使用区域（NSZone）
* 对象型变量不能作为 C 语言结构体（struct / union）的成员
* 显式转换 “id” 和 “void *” —— 桥接

## ARC通过所有权修饰符描述MRC内存管理操作
__strong：默认修饰符，引用计数+1。
__weak：引用计数不+1，释放时会自动置为nil。
__unsafe_unretained：同弱引用一样，但它不会置为nil。
__autoreleasing：加入到autoreleasepool中，自动释放。

1. ARC下OC中编译器会检查方法名是否以alloc/new/copy/mutableCopy开始，如果不是则会自动将返回值的对象注册到autoreleasepool由autorelase去管理，在runloop循环pop时释放。如果是则由ARC管理，会在作用域结束后插入release释放。::（注意区分平常创建的临时变量究竟是被哪种机制所管理，不同机制释放时机不同）::
2. Swift环境下当 ARC 设置弱引用为 nil 时，属性观察不会被触发。

## ARC的理解
首先ARC和GC是两码事，ARC是编译时编译器帮你插入了原本需要自己手写的内存管理代码，而非像GC一样运行时的垃圾回收系统。
[ARC到底帮我们做了哪些工作?_DCSnail-蜗牛-CSDN博客](https://blog.csdn.net/wangyanchang21/article/details/79461511)
``` objc
@interface ViewController ()
@property (nonatomic, strong) Zoo *zoo;
@end

@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    [self testForARC];
}
- (void)testForARC {
    //需要手动释放返回对象的方法
    [Zoo newZoo];               // 情景1：临时变量会在方法结束后释放，创建后马上插入release马上释放
    id temp1 = [Zoo newZoo];    // 情景2：临时变量会在方法结束后释放，先retain nil，再release生成的对象，最后效果相当于创建后马上release释放。
    self.zoo = [Zoo newZoo];    // 情景3：因为strong修饰，2者不相等，set内实现是release 旧对象，retain新对象，对象引用技术+1，离开方法不会被释放。
    
    //不需要手动释放返回对象的方法
    [Zoo createZoo];            // 情景4：会马上添加autorelease相关代码，加入到自动释放池去管理，在自动释放池释放时挂掉。
    id temp2 = [Zoo createZoo]; // 情景5：自动释放池管理
    self.zoo = [Zoo createZoo]; // 情景6：持有，引用技术+1。

    __weak id objc2 = [Zoo newZoo];               // 情景8：引用技术不变，离开方法作用域后销毁，同时对objc2指针置nil
    __unsafe_unretained id objc3 = [Zoo newZoo];  // 情景9：引用技术不变，离开方法作用域后销毁，不会对objc2指针置nil
    __autoreleasing id objc4 = [Zoo newZoo];      // 情景10：添加到自动释放池去管理，这里是被main中的那个自动释放池管理。

    //自动释放池包裹
    @autoreleasepool {
      id objc5 = [Zoo newZoo];                    // 情景11：同情景2一样，不受外部自动释放池管理，出了自动释放池作用域后被释放
      __autoreleasing id objc6 = [Zoo newZoo];    // 情景12：被这个包住的自动释放池管理，pop时销毁
      __autoreleasing id objc7 = [Zoo createZoo]; // 情景13：被这个包住的自动释放池管理，pop时销毁
    }
}
@end
```

### ARC 不会优化的情景：情景1不会释放，2能释放
``` objc
    // ARC不会为运行期的@selector添加内存管理语句
    id zoo = [Zoo performSelector:@selector(newZoo)];           // 情景1
  //id zoo = [Zoo performSelector:@selector(createZoo)]; // 情景2
    NSLog(@"instance: %@", zoo);
```
在ARC环境下，使用情景2创建的实例对象可以正常的释放，而使用情景1创建的实例对象不会自动释放，从而造成了内存泄露。这篇博客中说到了这个问题,  [多用GCD, 少用performSelector系列方法](https://blog.csdn.net/wangyanchang21/article/details/80340873#t15) 。

# AutoReleasePool管理
对非自己创建（alloc/new/copy/mutableCopy等方法）也并不持有的对象的一种内存管理方式，但要理解的是Autorelease对象本身依然尊还引用计数的管理规则。

1. AutoReleasePool是由 AutoreleasePoolPage 以双向链表的方式实现的
2. MRC下当对象调用 Autorelease 方法时或者ARC下创建的非自己持有的AutoRelease变量被AutoReleasePool包住，会将对象加入最近的AutoreleasePoolPage 的栈中（引用计数不变）。
3. AutoreleasePoolPage的pop 方法调用时会向栈中的Autorelease对象发送 release 消息（引用计数-1）。

> 这里只是发送release消息，如果当时的引用计数(reference-counted)依然不为0，则该对象依然不会被释放。  

## AutoReleasePool本质理解
使用@autoreleasepool{}来使用一个AutoreleasePool，随后编译器将其改写成下面的样子：obj也是一个对象，相当于哨兵，代表一个 autoreleasepool 的边界，与当前的AutoreleasePool对应，pop的时候用来标记终点位置。
```objc
autoreleasepool obj = objc_autoreleasePoolPush()
// 将中间对象压入栈中
objc_autoreleasePoolPop(obj)
```
![](iOS%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/FA1BDF7D-09FE-4353-AC40-3CC63FF444F1.png)
1. Push操作相当于向当前的page（hotPage）结点next指针指向的位置添加一个这次Push返回的objc对象作为哨兵对象，如果后面嵌套着AutoreleasePool则继续添加，如果向一个对象发送autorelease消息，也是将这个对象加入到栈顶next指针指向的位置。
2. 当前hotPage满了会新开一个Page继续上面的过程。
3. 调用Pop操作时会传入Push返回的objc对象地址，依次从后往前向这个哨兵对象之后添加的Autorelease对象::发送release消息::。

## AutoReleasePool和Runloop的关系
根据官方文档中对  [NSAutoreleasePool](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSAutoreleasePool_Class/index.html#//apple_ref/doc/uid/TP40003623)  的描述，在主线程的 NSRunLoop 对象的每个 event loop 开始前，系统会自动创建一个 autoreleasepool ，并在 event loop 结束时 drain ，具体是：
1. App启动后，苹果在主线程 RunLoop 里注册了两个 Observer。
2. 一个监视的事件是即将进入Loop，其回调内会调用autoreleasePoolPush() 创建自动释放池，且优先级最高，保证创建发生在其他所有回调之前。
3. 第二个 Observer监视了两个事件： 
	1. 准备进入休眠，会调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池。
	2. 即将退出Loop) ，调用 _objc_autoreleasePoolPop() 来释放自动释放池，且优先级最低，保证其释发生在其他所有回调之后。

> 所以在主线程执行的代码，非自己持有的autorelease对象会被 RunLoop 创建好的 AutoreleasePool 环绕着，所以不会出现内存泄漏，一般也不必显示创建 Pool 了（注意和main那个区分开，不是main那个autorelease）。  
> 在系统级别的其他线程中应该也是如此，比如通过 global_queue获取到的线程里也创建了autoreleasepool并监听runloop。  

## AutoReleasePool 使用场景
* 写基于命令行的的程序时，就是没有UI框架，如AppKit等Cocoa框架时。
* 写循环，循环里面包含了大量临时创建的对象。
* 自己创建的非Cocoa包装线程，比如pthread。

## Autorelease对象释放时机
::OC中在ARC环境下使用alloc、new、copy、mutableCopy的对象以及Swift init的对象都是自己生成并且持有的受到ARC管理，非以上方法创建的非自己持有的OC对象都是Autorelease的（CF那些对象不是，它们通过bridge或者Swift Unmanaged管理）。::所以他们的释放在对应最近的AutoReleasePool执行objc_autoreleasePoolPop方法时释放，常见场景中有以下2种：
1. 手动创建的AutoReleasePool里包住了autorelease对象是在离开最近的AutoReleasePool作用域时就释放。
2. 系统主线程的AutoReleasePool包起来是在runloop休眠时、退出时释放。
3. ::子线程的是在NSthread exit的时候（只要是cocoa线程都实现有autoreleasepool）::

> 在子线程你创建了 Pool 的话，产生的 Autorelease 对象就会交给 pool 去管理。如果你没有创建 Pool ，但是产生了 Autorelease 对象，就会调用 autoreleaseNoPage 方法。在这个方法中，会自动帮你创建一个 hotpage（hotPage 可以理解为当前正在使用的 AutoreleasePoolPage，如果你还是不理解，可以先看看 Autoreleasepool 的源代码，再来看这个问题 ）  

## OC main函数的AutoReleasePool理解
![](iOS%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/10F8267D-C805-4CBE-8751-33E74CE03696.png)
![](iOS%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/387830D9-8EB9-4302-9D82-9C669CEFE2DC.png)
Xcode11之前主线程的RunLoop在UIApplicationMain中的创建被包在了main中的@autoreleasepool里，但是主线程的RunLoop会自己创建自己的autoreleasepool，::也就是说平常主线程中的AutoRelease对象的释放时机和这个main中的@autoreleasepool没有关系::，只是嵌套在了里面。return即程序结束后main@autoreleasepool作用域才会结束，这意味着程序结束后main函数中@autoreleasepool包住的autorelease对象才会释放。

Xcode11之后主线程的RunLoop在UIApplicationMain中的创建没有被包住，所以可以将autorelease对象直接放在@autoreleasepool里，而不需要等到程序推出时才会被释放。

# 和C对象交互的内存管理
[深入理解Toll-Free Bridging_Leo的专栏-CSDN博客_toll-free](https://blog.csdn.net/Hello_Hwc/article/details/80094632)
* 所有权在Foundation，则不需要手动管理内存。
* 所有权在CF，需要调用CFRetain/CFRelease来管理内存。

对应Swift里的Unmanaged方案，理解上是一个东西。
![](iOS%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/96EDA7E1-7194-432C-9529-7DFF32354902.png)
1. __bridge ：单纯的OC 和 Core Foundation 类型相互转换，不移交持有权，不改变引用计数。
	1. **OC -> CF：使用__bridge相当于所有权还在Foundation，进行ARC管理，不需要手动管理**
	2.  **CF -> OC：使用__bridge相当于所有权还在CF，需要进行手动管理**
2. __bridge_retained 或 CFBridgingRetain： 用在转换 OC 到CF时，同时移交所有权，移交的时候引用技术会+1，保证在CF对象使用时不会被释放，同时意味着你需要手动管理调用CFRelease来释放这个指针。
3. __bridge_transfer 或 CFBridgingRelease：用在转换CF到 OC时，由ARC管理，所以不需要手动管理。
  
## 为什么可以互相转化？
CF内存模型里有isa指向OC类对象，所以OC转CF时，能通过CF函数调用到OC方法。
OC类对象是的实现是基于CF类，所以CF转OC时，能通过OC调用到CF类对应的方法。

* 简单说, 对于支持TFB的CF类型, 他的内存模型里都有一个isa指针,并且在创建CF对象(其实是结构体)之后, 这个对象的isa指针就被设置成对应的TFB的OC类, 这样把CF类型转成OC对象后调用方法时, 因为内存里有isa指针, 所以可以走runtime那套消息查找, 最后找到OC对象的方法, 而该方法的实现又是调用对于CF类型的函数来完成的. 相当于饶了一圈又回去了. 

* CF 转 OC对象, 对应的方法传递过程: OC方法->CF函数 
* OC对象转 CF, 对于的方法传递过程: CF函数 -> OC方法 

* 而如果没有转换的话, 方法的调用则和原来一样   
* OC对象, 方法调用: OC方法-> 内部实现是通过调用CF函数 
* CF对象, 方法调用: CF函数-> __CF对应的私有函数

```objectivec

void *p = 0;
{    
    // 转换但不移交持有权
    id obj = [[NSObject alloc]init];
    p = (__bridge void *)(obj);
}
// 崩溃，因为obj对象仍然由ARC管理，离开大括号runloop循环后已经被释放了
// 应该使用__bridge_retained，然后手动调用CFRelease(p)释放
NSLog(@"class = %@",[(__bridge id)p class]);
```

# Tagged Pointer管理机制
Tagged Pointer机制从内存和使用的角度都是一种优化，被Tagged Pointer包起来的对象内存管理受这个Tagged Pointer指针内存分布影响，在栈上就存在栈上，在对象内就存在对象内，随Tagged Pointer指针销毁而销毁，所以如果是autorelease类型的对象被包起来，这个autorelease对象也就不受原来AutoReleasePool管理了。[场景四](https://juejin.im/post/6844903736939118599#heading-5)

## Tagged Pointer原理
主要解决比如::NSNumber、NSString、NSDate等小的数据存储::，直接把数据存储在8个字节的指针内，通过标志位来区别类型，存取是直接通过位运算所以效率很高，当空间不够时才变为一个普通指针，指向堆空间来存储数据。[详细可以了解这里](https://juejin.im/post/6844904132940136462#heading-14)

1. ::objc_msgSend可以识别它，直接通过位运算取数据（意味如果这个指针作为成员变量的话，点赋值操作不会触发set）::
2. iOS上，可以通过最高（mac是最低）有效位是否为1来判断是不是Tagged Pointer指针。

## 举个例子
![](iOS%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/59B1127A-5A71-4AF6-98C5-4E6428AF6F72.png)
1. 这里上面会直接崩溃，崩溃的原因是编译器为属性name自动生成的set方法内部实现是首先判断name是不是原来的对象，不是原来的对象会release原对象，然后再把新传进来的对象根据属性修饰符（copy或者strong）调用对应操作后进行赋值，多条线程同时进行release会发生坏内存访问。::解决方案是使用atomic，在set内加锁，或者外部加锁。::
2. 下面因为字符串比较小的话，采用Tagged Pointer存储，此时存取操作直接是进行位运算而不是get、set所以不会崩溃。