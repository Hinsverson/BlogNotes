# Swift的内存布局
 #Swift/底层解析

# 大小计算
![](Swift%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/FD598941-5860-4E10-830B-4F68BFF1CB4F.png)
只能获取栈上连续的空间，对class不管用（获取的是指针的大小信息）。

Class类型：
``` Swift
import Foundation

class C {}
print(class_getInstanceSize(C.self)) // 16 bytes metadata for empty class (isa ptr + ref count) 实际需要的内存空间 16 bytes metadata + 成员变量

class C1 {
    var i = 0
    var i1 = 0
    var b = false
}
print(class_getInstanceSize(C1.self)) // 40 bytes
// (16 metadata + 24 ivars, 8 for i + 8 for i1 + 1 for b + 7 padding)
```
``` Swift
// 实际分配的大小，注意和class_getInstanceSize区分，iOS大小都是16的整数倍。
func heapSize(_ obj: AnyObject) -> Int {
    return malloc_size(Unmanaged.passRetained(obj).toOpaque())
}

class MyClass {
    //no properites...
}
let myObj = MyClass()
print(heapSize(myObj)) //->16

class MyBiggerClass {
    var str: String = ""
    var i: Int = 0
}
let myBiggerObj = MyBiggerClass()
print(heapSize(myBiggerObj)) //->48
```

**堆和栈的区别**
在栈上分配和释放内存的代价是很小的，因为栈是一个简单的数据结构。通过移动栈顶的指针，就可以进行内存的创建和释放。但是，栈上创建的内存是有限的，并且往往在编译期就可以确定的。

在堆上可以动态的按需分配内存，每次在堆上分配内存的时候，需要查找堆上能提供相应大小的位置，然后返回对应位置，标记指定位置大小内存被占用。
在堆上能够动态的分配所需大小的内存，但是由于每次要查找，并且要考虑到多线程之间的线程安全问题，所以性能较栈来说低很多。

# String内存本质
![](Swift%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/9ADCB1C4-2ECA-481D-9C0B-33FF5A8B1773.png)
Swift String内存大小为16各字节。
	1. 字面量字符串小于<=16个字节长度，会把字面量直接存储在变量内存中（根据实际变量的使用场景，比如临时变量就在栈上）。
	2. 字面量字符串大于>16个字节长度，会把字面量存储在常量区，再把字面量真实地址值放在变量的后8个字节中，前8个字节放字面量长度。
	3. 常量区编译完成后不可修改，如果一个string变量一开始字面量小于16个字节，而后拼接超过了16个字节，那么会把这整个字面亮放在堆区，再把字面量真实地址值放在变量的后8个字节中，前8个字节放字面量长度。

# Araay的内存本质
![](Swift%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/789135A3-9AAF-4E61-AA4F-AAF02FC63445.png)
1. 1个array变量占用内存字节数根据元素数量的不同会存在差别，会采用动态扩容的技术，当元素数量大于数组容量的一半，就会翻倍扩容数组容量。
2. array的前32个字节存储着Array相关信息（引用计数、元素数量、数组容量）
3. array虽然是用结构体实现，但实际元素数据存储是在堆空间，arrar结构存放在栈上。虽然有着值类型的行为，但是底层实现却是通过引用的方式实现（开发层面你还是可以理解为它是一个值类型就行了）。主要是相比C和C++的数组它内部做更多事情，包括写时复制等，所以表面上看起来是值类型，但是同样会有引用计数这种东西。

# 枚举
* 使用1个字节来存储成员对象（表识是第几个枚举成员）
* 使用N个字节来存储关联值（N取占用内存最大的关联值）
* 如果只有一个成员，那就不会花一个字节去标示这个成员对象。

> aligment：内存对齐的大小（8表示内存分配大小一定是8个字节的整数倍，不够的需要对齐），对齐大小和实际类型有关。  

简单枚举：用一个字节来存储成员对象，标示是第几个枚举成员。
带原始值：枚举成员对象在内存中不会把原始值存储在栈空间上，布局结构和简单枚举一致。
带关联值：枚举成员对象在内存中会把关联值存储在栈空间上，存一个int则会分配8个字节，2个则16个字节，最后会加一个用来标示枚举成员的字节。

### 简单枚举内存布局
![](Swift%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/5B3F3140-9299-4B92-BA86-0D05E5D11C6E.png)
![](Swift%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/199FA437-2387-46BF-954A-2F2A4A3700F2.png)
一个简单枚举变量占用1个字节来存储成员值，所以枚举的个数范围为0-256个（0x00-0xFF），在内存中test1存储是00，test2是01，依次类推，可以看到实际上面test3在内存中存储的就是02，用来标识第三个枚举成员。

![](Swift%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/47AD5608-4D92-4D63-BB15-DFA842CADECA.png)
> 这里枚举只有1个成员值，实际打印内存大小为0个字节，分配了1个字节，为什么呢？  
> 因为只有1个成员值，不需要区分是哪个，所以根本不用分配的1个字节去标识。  

### 原始值的枚举内存布局
![](Swift%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/0C19C63F-37D6-4C68-ABEB-1DBFE352D5CF.png)
> 注意：这里的Int是的原始值，这个值并不会存储到枚举变量的内存里面，为什么呢？  
>   
> 虽然Int在64位平台上占8个字节，但原始值是跟成员永远绑定在一起的，只是通过rawValue属性可以拿到这个值（本质是一个get计算属性），且定义后是不会发生变化的，即使Int实际在内存中占8个字节，但不会存储到枚举变量里，所以按照简单枚举内存布局只用1个字节就能够标识对应的枚举值（前面已经说过最多256个）。  

### 关联值的枚举内存布局
![](Swift%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/224144C3-4A02-4BEF-8288-EFF9FD6B4DAC.png)

> 注意：这里的number(int, int, Int, Int)后面的Int是枚举关联值，Swift里是把关联的值直接存储到枚举变量的内存里面，为什么呢？  
>   
> 因为关联的值是不确定的，每次创建一个枚举变量时都可能不同，所以系统为了在内存中能够无错的存储下这些值，会按照类型的实际大小（Int在64位上8个字节）给变量分配内存空间，并把值存到变量里去。  

实际在内存中第1-8个字节存储所关联元组的第一个整型数据1，所以为01。9-16个字节用来存整型数据2，所以第二个为02。由于在内存中一般采用小端对齐（低位在前，高位在后），所以存储内容从前面往后面开始。

> 一个Int在64位平台上占8个字节，需要存储test1关联元组里的3个整数，所以枚举case test1应该占用24个字节，但是打印出来实际占用25个字节，一共占用32个字节（内存对齐的原因），为什么是25个字节呢？  
>   
> 多出来的一个字节实际存储的是枚举的成员值，可以理解为一个标识。类似简单枚举在内存中的存储规则一样，第1个为00，第二个为01…，这里test1为第一个枚举值，所以存储的是00，并且在第25个字节处存储，因为前面24个字节要存储占用内存最大的关联值（test1有3个Int类型的关联值）  

### 延伸：Switch 语句底层实现
![](Swift%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/1EBB7D5B-221A-4228-87A8-FD42DE825CB5.png)
枚举用1个字节去存储成员值，用来标示成员。所以Switch在判断从跳到哪个case里，也是通过取出这个变量存储成员值的那个字节里面的成员值和case里的成员值去比较的，哪个case相等就跳到哪个case（这里是第25个字节存储成员值，上面有说到）。

# 结构体
![](Swift%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/A660093E-0EC5-4FFF-A810-76E77A2051AD.png)
结构体是值类型，Swift中结构体内存分布在栈空间。

### 举例：
```
structPerson{
   var age:Int64=0
   var sex:UInt16=0
   var address:Double=0.0
   var name:UInt8=0
}
```
开始设置size=0，alignment=1。
age是8字节，size = 8，alignment = 8, stride = 8
sex是2字节，size = 10，alignment = max(8,2) = 8，stride = 16
address是8字节，size = 24，alignment = max(8,8) = 8， stride = 24
name是1字节，size = 25，alignment = max(8,1) = 8，stride = 32

> 现代CPU每次读数据的时候为了效率，操作的原子性，都是读取一个word（32位处理器上是4个字节，64位处理器上是8个字节）  

# 类
[swift之内存布局 - 简书](https://www.jianshu.com/p/d341974404a7)
``` C++
struct HeapObject {
  /// This is always a valid pointer to a metadata object.
  HeapMetadata const *metadata;
  
  // 展开后是RefCounts<InlineRefCountBits> refCounts;
  SWIFT_HEAPOBJECT_NON_OBJC_MEMBERS;

  HeapObject() = default;

  // Initialize a HeapObject header as appropriate for a newly-allocated object.
  constexpr HeapObject(HeapMetadata const *newMetadata) 
    : metadata(newMetadata)
    , refCounts(InlineRefCounts::Initialized)
  { }
  
  // Initialize a HeapObject header for an immortal object
  constexpr HeapObject(HeapMetadata const *newMetadata,
                       InlineRefCounts::Immortal_t immortal)
  : metadata(newMetadata)
  , refCounts(InlineRefCounts::Immortal)
  { }

};
```
上面SWIFT_HEAPOBJECT_NON_OBJC_MEMBERS是一个宏定义，展开后：
``` C++
template <typename RefCountBits>
class RefCounts {
  std::atomic<RefCountBits> refCounts;
  // ...
};
```
RefCounts 是一个线程安全的 wrapper，模板参数 RefCountBits 指定了真实的内部类型，在 Swift ABI 里总共有两种：Swift4 之后InlineRefCounts是没有weak和无主引用时存储强引用的，SideTableRefCounts是有weak存储指向SideTable指针的。
``` C++
typedef RefCounts<InlineRefCountBits> InlineRefCounts;
typedef RefCounts<SideTableRefCountBits> SideTableRefCounts;
```

![](Swift%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/C6BAB5FB-98C1-4FC0-8FB6-C2200DFCBC05.png)
类对象分配在堆空间上，Swift实现上用一个HeapObject的结构体来描述分配在堆上的东西（是objc_object的一种扩展类），前8个字节存放HeapMetadata，这个对象有着与OC isa_t 类似的作用，就是用来描述对象类型的（等价于 type(of:) 取得的结果）。后8个字节存放引用计数RefCounts。

HeapMetadata：元类信息，可通过TargetAnyClassMetadata初始化，TargetAnyClassMetadata类似于objc_class。
![](Swift%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/3639677F-587D-4072-9C71-85DC8159ED31.png)

### 举例
![](Swift%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/E128F0F3-9615-4DA2-A5CA-9BCF703BE943.png)
* 由于Point是值类型，栈空间内存里存储的是实际变量的值，这个值变量的地址为ox10000， 2个Int成员，所以每一个分配8个字节，一共占用16个字节。
* Size是引用类型（指针类型），size的本质是指针变量，栈空间内存里存储的是这个指针指向的实际类对象在堆空间的地址，栈空间里的这个地址ox10010是这个指针变量的地址，1个指针变量在64位里占8个字节。 
* 指向类对象会占用32个字节，首先指向元类的类型信息描述会占8个字节（存储的也是一个地址，指向类型信息描述的那块内存），对象引用计数会占用8个字节，剩下才是成员变量2个Int，占16个字节（这里方法列表地址在指向元类的类型信息里面，Swift方法列表存储在全局区）。

### Swift和OC类对象内存布局的理解
1. 很大程度上参考了OC，进行了类似数据结构实现。
2. HeapObject对应objc_Object
3. HeapMetadata模版包的TargetAnyClassMetadata对应objc_Class
4. Swift中InlineRefCounts相当于OC中优化后的isa。

# 协议
以结构体遵循协议为例：Drawable是一个带有draw方法的协议。
![](Swift%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/E59279A2-5F4D-4C66-8AFD-B59D4C5DAB94.png)
![](Swift%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/1DD89A53-E71A-4F32-BF1B-3C580E5A8157.png)
![](Swift%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/45FD0BD7-46FD-47A4-BC78-76B3420F0DD3.png)

### 总结：
![](Swift%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/C02026CC-413F-4B64-BC88-B05B602483E2.png)
1. 由于实现协议的类型可能不同，内存空间、初始化方法等都不相同，所以Swift引入了一个容器的概念来对遵循协议的对象进行内存布局。
2. 其中3个固定word（一个8bety）大小的缓冲区（valueBuffer）存放对象属性，第4个word放VWT地址，第5个word放PWT地址。
3. 如果3个word24个字节不够存储属性的话，缓冲区会存储指向Heap空间的地l址，然后去Heap存储属性。
VWT：遵守了协议的实例初始化，拷贝，内存消减和销毁的方法表，结构存储在堆上。
PWT：协议的动态方法派发表，存储在堆上。
![](Swift%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/59BDC320-998D-47CD-A62C-EC4AAB97D3A0.png)

# 范型多态
![](Swift%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/86ECF492-A668-4444-8EDE-4E3D87390955.png)
![](Swift%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/9365A2E3-554A-4913-8921-5DED295F6BEF.png)
范型并不采用Existential Container，但是原理类似。
* VWT和PWT作为隐形参数，传递到范型方法里。
* 临时变量仍然按照ValueBuffer的逻辑存储 - 分配3个word，如果存储数据大小超过3个word，则在堆上开辟内存存储。

## Any类型
和protocol类似，只是没有了PWT。
``` C++
struct OpaqueExistentialContainer{
 void*fixedSizeBuffer[3];//数据区
 Metadata*type;//数据区的类型
};
```

