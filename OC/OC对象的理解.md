# OC对象的理解
#iOS知识点/OC相关

[iOS底层学习 - OC对象前世今生 - 掘金](https://juejin.im/post/6844904024659984391)

# OC对象：objc_object
![](OC%E5%AF%B9%E8%B1%A1%E7%9A%84%E7%90%86%E8%A7%A3/2408CEB8-F65A-4350-ABC6-7BF8ABD867D1.png)
Swift中是一个HeapObject结构体，包含了指向类信息HeapMetadata的指针和RefCounts，整体结构和OC相似。

## OC 对象大小计算
class_getInstanceSize：运行时实际需要的内存对齐后的大小（结构体基本对齐规则：分配时变量的起始地址必须是这个类型变量大小的整数倍）
malloc_size：系统实际分配的对齐大小（OC对象是16的倍数规则对齐）
sizeof：编译时直接返回类型对应的大小。

## OC对象alloc创建过程：计算并通过calloc分配内存，初始化isa并指向类对象
核心是通过class_createInstance来创建对象，主要做了下面3件事情：
1. instanceSize：计算对象内存布局对齐后实际所需的内存大小。
2. (id)calloc(1, size)：进行对象内存申请，OC对象以16字节对齐（和上面实际所需的大小可能不一致）。
3. initInstanceIsa和initIsa：初始化isa并指向类对象。

::allocWithZone：一般不是继承NSObject/NSProxy的类会重写这个方法。::
::calloc和malloc最大区别：calloc会初始化二进制为0::

## init方法理解：不做任何事情直接返回self，只是给开发者使用工厂设计模式提供一个接口
```objc
// Replaced by CF (throws an NSException)
+ (id)init {
    return (id)self;
}
```
**OC子类中if (self = [super init])为什么要这么写？**
OC是子类先出初始化父类的属性，再判断是否为空，如若为空没必要进行一系列操作了直接返回nil。
::Swift的初始化是先初始化子类属性，再从下到上初始化super，再从上到下使用继承的属性变量。::

## new理解：本质只是先alloc再init
```objc
+ (id)new {
    return [callAlloc(self, false/*checkNil*/) init];
}
```

init和initialize区别：
Initialize不是init，initialize在程序运行过程中，它会在你程序中每个类调用仅一次initialize。这个调用的时间发生在你的类接收到消息之前，但是在它的父类接收到initialize之后。
Init则是你手动初始化几次就调用几次，和普通方法一样
[oc中 load，initialize，init方法对比总结_风一样自由-CSDN博客](https://blog.csdn.net/wei371522/article/details/81207863)

## dealloc：销毁
当OC对象的引用计数变为0时，对象本身的dealloc方法会被调用，ARC下编译器会插入成员变量release、调用super dealloc的代码，所以ARC下无需显示调用父类的dealloc。整个执行顺序是逐级向上调用父类的dealloc方法，一直调到NSObject的dealloc方法。（由子到父到NSObject）。

1. NSObject的dealloc方法首先进行能否快速释放的前置判断，逻辑如下：
	1. isTaggedPointer对象直接return，此时对象是存在栈上指针变量的特定位上，无需释放手动内存，由栈进行自动管理。
	2. 如果是64位优化过的isa，那么就取出isa上特定位存储的信息，看有没有关联对象、弱引用、使用了C++、使用了sidetable等，如果都没有则可以直接使用C的free释放this，清理内存空间。
2. 如果不能快速释放，则调用runtime的object_dispose方法，这个方法内会进行如下处理：
	1. 调用C++相关析构函数。
	2. 解除所有关联对象。
	3. sidetable和弱引用置nil操作。
	4. 最后调用free方法释放内存。
 
> 由于实例变量最终的释放时机是在NSObject的dealloc方法的object_dispose方法中，所以子类本身的dealloc方法中依然可以使用实例变量。  

# ISA：64位后本质是一个Union通过位域来存储信息（类对象地址、弱引用计数等）
64位前：8个字节直接放的是class、或者Meta-class的内存地址（普通指针）
64位后：isa变成了一个共用体，用64位的内存空间存储着更多的东西，其中33位存放class、或者Meta-class的内存地址，其他位存放了一些其他信息：
![](OC%E5%AF%B9%E8%B1%A1%E7%9A%84%E7%90%86%E8%A7%A3/9E2FD2F3-5A80-472E-B004-E32297EFD521.png)
1. nonpointer：0纯指针，1不⽌是类对象地址，还包含了类信息、对象的引⽤计数等。
2. has_assoc：关联对象标志位，0没有，1存在。
3. has_cxx_dtor：该对象是否有 C++ 或者 Objc 的析构器，如果有析构函数，则需要做析构逻辑，如果没有，则可以更快的释放对象。
4. ::shiftcl（33）：arm64 架构下开启nonpointer后存储的类对象地址的值。::
5. magic：⽤于调试器判断当前对象是真的对象还是没有初始化的空间
6. ::weakly_referenced：指对象是否被⼀个 ARC 的弱引用变量指向。::
7. deallocating：标志对象是否正在释放内存
8. has_sidetable_rc：对象引⽤技术是否⼤于 10。
9. ::extra_rc：当表示该对象的引⽤计数值，实际上是引⽤计数值减 1， 例如，如果对象的引⽤计数为 10，那么 extra_rc 为 9。如果引⽤计数⼤于 10， 则需要使⽤到下⾯的 has_sidetable_rc。::

## ISA如何获取所指向的对象真实地址：&运算
通过位域&上一个mask(isa.bits & ISA_MASK )
举例：假如我想在1个字节的char上存储3个Bool类型的信息
![](OC%E5%AF%B9%E8%B1%A1%E7%9A%84%E7%90%86%E8%A7%A3/584EF368-DEEA-47A4-8EAB-CEADBD0AA5DC.png)
宏定义为移位操作，是存储的信息取值和赋值的掩码（1<<2表示00000100）

## ISA指向关系：实例对象->类对象->元类对象
![](OC%E5%AF%B9%E8%B1%A1%E7%9A%84%E7%90%86%E8%A7%A3/8B3ED982-1A45-410C-A95E-8D640DCAD08F.png)
1. 实例对象isa指向类对象
2. 类对象指isa向元类对象
3. 元类对象的isa指向元类的基类
4. 元类的基类的isa指向本身
5. 元类的基类的superClass指向类对象的元父类

::根元类的父类是根类对象意味着定义一个没有实现的类方法，如果类对象里面有同名方法，则会调用类对象里的实例方法。::

# OC类对象：objc_class（继承自objc_object）
![](OC%E5%AF%B9%E8%B1%A1%E7%9A%84%E7%90%86%E8%A7%A3/D8ECF3B9-80B0-444F-BB04-41489E3A67CE.png)

## OC类对象、元类对象的创建加载：
非懒加载的类（实现了load）在程序启动后dyld给出的load时机加载进内存空间并调用load方法，懒加载的类在第一次发消息使用时发现没有类信息时会去加载对应的类。

## 类对象组成：isa、superclss、方法cache、bits类型信息
![](OC%E5%AF%B9%E8%B1%A1%E7%9A%84%E7%90%86%E8%A7%A3/8BC7BE95-0A3F-4657-A86D-529920F878E0.png)
对应Swift中：
StoredPointerKind：isa
Superclass：superClass
CacheData：Cache
StoredSizeData：data

## Cache_t：方法缓存
![](OC%E5%AF%B9%E8%B1%A1%E7%9A%84%E7%90%86%E8%A7%A3/A5B35C20-5A96-458B-8D40-23D8C08F05CF.png)
用于缓存最近使用的方法，可增量的哈希表结构，一种局部性原理的应用，主要包含：
1. mask：一个整数，哈希表长度 - 1
2. occupied：一个整数，指定实际占用的缓存方法总数。
3. buckets：指向Method数据结构的散列表数组指针。
4. bucket_t：Method的一种描述结构，由SEL和 imp函数地址组成。

## class_rw_t运行时类信息：bits通过&mask可以获取到
![](OC%E5%AF%B9%E8%B1%A1%E7%9A%84%E7%90%86%E8%A7%A3/252F1287-7601-4168-BE3E-7A7402939CBC.png)
**主要包含：**
1. 属性二维数组
2. 协议二维数组
3. 方法二维数组
4. class_ro_t编译后的类信息

::**注意：**::
1. class_rw_t是可读和可写的，可以通过修改它实现在运行时动态添加方法。比如新增一个分类定义了很多方法，那么其实是在method_array_t里的最前面新增了一个method_list_t，这个method_list_t装着新分类中新增的方法。
2. 这里都是二位数组，方法二位数组里面又包含了method_list_t，method_list_t里才包含了方法method_t的封装。

## class_ro_t编译后的类信息：只读
![](OC%E5%AF%B9%E8%B1%A1%E7%9A%84%E7%90%86%E8%A7%A3/ACDB1E05-FCC2-4C08-8E0A-1F2F670B4158.png)
**主要包含：**
1. 成员变量列表ivar_list（存着成员变量的名字）
2. 方法列表
3. 协议列表
4. 属性列表

::**注意：**::
这里都是list一维数组，在OC运行时，实际是把class_ro_t里的方法list、协议list、属性list内容进行拷贝，再和分类的内容结合起来放入class_rw_t里的二维数组里面。

## method_t（方法、函数的封装）
![](OC%E5%AF%B9%E8%B1%A1%E7%9A%84%E7%90%86%E8%A7%A3/C2302ED9-CF18-4122-80B0-2F322DAA0225.png)
1. SEL选择器：代表方法名，是一种char*的结构
2. imp：指向函数的指针
3. types：方法的参数和返回值类型编码的字符串表示（TypeCoding技术）
