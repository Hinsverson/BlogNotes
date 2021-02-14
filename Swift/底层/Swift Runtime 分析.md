# Swift Runtime 分析
#Swift/底层解析

[Swift5.0 的 Runtime 机制浅析 - 掘金](https://juejin.im/post/5d29fb63e51d4510aa01159d#heading-10)
[Object-C与Swift的RunTime运行机制对比 - 掘金](https://juejin.im/post/5e9eb13ff265da47e90d1a33)
[Swift Runtime ？ - 云+社区 - 腾讯云](https://cloud.tencent.com/developer/article/1037800)

Swfit中的对象方法调用机制加快了程序的运行速度，同时减少了程序包体积的大小。但是从另外一个层面来看当编译链接优化功能开启时反而又会出现包体积增大的情况。Swift在编译链接期间采用的是空间换时间的优化策略，是以提高运行速度为主要优化考虑点。

Swift中的对象方法调用，在Debug模式下和Release模式下，也就是说在Release编译链接优化选项开启的情况下二者的实现差异巨大。


### Swift类对象的创建和销毁
Swift中可以定义两种类，一种是从NSObject或者派生类派生的类，一类是从系统Swift基类SwiftObject派生的类，SwiftObject是一个隐藏的基类。

Swift类对象的内存布局和OC类对象的内存布局相似，最开始部分都有一个isa成员变量指向类的描述信息。Swift类的描述信息结构swift_class继承自OC类的描述信息objc_class，但是并没有完全使用里面定义的属性，对于方法的调用则主要是使用其中扩展了一个所谓的虚函数表的区域。

一个swift类在编译后，编译器会生成`模块名.类名.__allocating_init(类名,初始化参数)`*的初始化函数和`模块名.类名.__deallocating_deinit(对象)`的析构销毁函数。实现如下：
``` c++
//假设定义了一个CA类。
class CA {
   init(_ a:Int){}
}
//编译生成的对象内存分配创建和初始化函数代码
CA * XXX.CA.__allocating_init(swift_class  classCA,  int a)
{
    CA *obj = swift_allocObject(classCA);  //1.分配内存。
    obj->init(a);  //2.调用初始化函数。
}
//编译时还会生成对象的析构和内存销毁函数代码
XXX.CA.__deallocating_deinit(CA *obj)
{
   obj->deinit()  //调用析构函数
   swift_deallocClassInstance(obj);  //销毁对象分配的内存。
}
```

对于一个创建后的对象通过ARC进行生命周期的管理，初次创建时引用计数被设置为1，每次进行对象赋值操作都会调用swift_retain函数来增加引用计数，而每次对象不再被访问时都会调用swift_release函数来减少引用计数。swift_release函数内部会在当引用计数变为0后就会调用编译时为每个类生成的析构和销毁函数。这些内存管理操作相关函数保存在swift类描述信息结构体swift_class中。
``` c++
////////Swift源代码
let obj1:CA = CA(20);
let obj2 = obj1

///////C伪代码
CA *obj1 = XXX.CA. __allocating_init(classCA, 20);
CA *obj2 = obj1;
swift_retain(obj1);
swift_release(obj1);
swift_release(obj2); 
```

### Swift类的对象方法调用：编译链接优化选项开关在关闭的时候
1. OC类的派生类并且重写了基类的方法：objc_msgSend
2. extension中定义的方法：调用总是会在编译时就决定，方法调用指令中的函数地址将会以硬编码的形式存在，这就是在Swift中派生类无法重写一个基类中extension定义的方法的原因了。因为extension中的方法调用是硬编码完成，无法支持多态！
3. 类中定义的常规方法：和C++中的虚函数表派发相似，swift_class的第0x50个字节的偏移处有类的虚函数表，每个虚表条目中保存着一个常规方法的函数地址指针。每一个对象方法调用的源代码在编译时就会转化为从虚表中取对应偏移位置的函数地址来实现间接的函数调用。
``` C++
////////Swift源代码

//类定义
class MyUIView:UIView {
    open func foo(){}   //常规方法
    override func layoutSubviews() {}  //重写OC方法
}
//类的extension定义
extension CA {
   open func extfoo(){}
}

func main(){
  let obj = MyUIView()
  obj.layoutSubviews()   //调用OC类重写的方法
  //objc_msgSend(obj, @selector(layoutSubviews);

  obj.foo()   //常规的swift方法采用间接调用实现
  /* Swift方法调用时对象参数被放到x20寄存器中
  asm("mov x20, obj");
  obj->isa->vtable[0](); //objc的isa找class结构再取函数表派发
  */

  objc.extfoo()   //硬编码调用
  /* 直接硬编码调用，而不是间接调用实现
  extfoo();
  */
}
```

### Swift类中成员变量的访问
Swift类在编译链接时确定成员变量在对象的偏移位置，然后会为每个定义的成员变量都生成一对get/set方法并保存到虚函数表中，所有对对象成员变量的访问都会转化为通过虚函数表来执行get/set相对应的方法。

OC类在编译链接时确定成员变量在对象的偏移位置。然后会为所有成员变量，生成一张变量表信息，变量表的每个条目记录着每个成员变量在对象内存中的偏移量。每个OC类的get和set两个属性方法的实现中再根据偏移量来读取或设置对象的成员变量数据。

### Swift结构体中的方法
不支持多态：因为结构体的内存结构中并没有isa数据成员保存结构体的信息。
不支持派生、override：所有方法调用都是在编译时硬编码来实现的

### 类的方法以及全局函数
Swift类中定义的类方法和全局函数一样，因为不存在对象作为参数，因此在调用此类函数时也不会存在将对象保存到x20寄存器中这么一说。
类方法和全局函数就像C语言的普通函数一样被实现和定义，所有对类方法和全局函数的调用都是在编译链接时刻硬编码为函数地址调用来处理的。

**OC调用Swift类中的方法**
当某个Swift方法被声明为@objc关键字时，在编译时刻会生成两个函数（有可能会被静态优化，如果想完全按照OC派发方法，则需要加dynamic）：
	1. 一个是本体函数供Swift内部调用
	2. 另外一个是跳板函数(trampoline)是供OC语言进行调用的。
跳板函数信息会记录在OC类的运行时类结构中，跳板函数的实现会对参数的传递规则进行转换：把x0寄存器的值赋值给x20寄存器，然后把其他参数依次转化为Swift的函数参数传递规则要求，最后再执行本地函数调用。
``` C++
////////Swift源代码
//Swift类定义
class MyUIView:UIView {
  @objc    
  open func foo(){}
}

func main() {
  let obj = MyUIView()
  obj.foo() //obj->isa->vtable[0]();
}

//////// OC源代码
#import "工程-Swift.h"
void main() {
  MyUIView *obj = [MyUIView new];
  [obj foo]; //objc_msgSend(obj, @selector(foo));
  /*
  OC语言对foo的调用还是用objc_msgSend来执行调用。 
  因为objc_msgSend最终会找到methods中的方法结构并调用trampoline_foo   
  而trampoline_foo内部则直接调用foo来实现真实的调用。
   */
}

//////// 编译后的信息存储
void foo(){} 
//跳板函数的实现 
void trampoline_foo(id self, SEL _cmd){ 
	asm("mov x20, x0"); 
	self->isa->vtable[0](); //这里调用本体函数foo 
}
//类的描述信息构建，这些都是在编译代码时就明确了并且保存在数据段中。 
struct swift_class classMyUIView; 
classMyUIView.methods[0] = {"foo", &trampoline_foo}; classMyUIView.vtable[0] = {&foo};
```

### Swift类方法的运行时替换实现的可行性
要实现这种机制有三个难点需要解决：
1. swift中虽然可以将方法函数名称赋值给某个变量，但是这个变量的值并非是类方法函数的真实地址，而是一个包装函数的地址。
2. Swift中的类方法调用和参数传递的ABI规则和其他语言不一致，在OC类的对象方法中，对象是作为方法函数的第一个参数传递的，机器指令层面以arm64体系结构为例，对象是保存在x0寄存器作为参数进行传递。在Swift的对象方法中这个规则变为对象不再作为第一个参数传递了，而是统一改为通过寄存器x20来进行传递。这个规则不会针对普通的Swift函数，所以想将一个普通的函数来替换类定义的对象方法实现时就几乎变得不太可能了，当然可以通过为类定义一个extension方法，将这个extension方法函数的指针来替换掉虚函数表中类的某个原始方法的函数指针地址，这样能够解决对象作为参数传递的寄存器的问题。但仍然需要面临两个问题：
	1. 一是如何获取得到extension中的方法函数的地址，
	2. 二是在替换完成后如何能在合适的时机调用原始的方法。
3. Swift语言将不再支持内嵌汇编代码了，所以我们很难在Swift中通过汇编来写一些跳板程序了。

### 编译链接优化开启后的Swift方法定义和调用
1. 弱化了通过虚函数表来进行间接方法调用的实现，而是大量的改用了一些内联的方式来处理方法函数调用，好处就是运行时更快，坏处就是会增大包体积。
2. 就是对多态的支持，也可能不是通过虚函数来处理了，而是通过类型判断采用条件语句来实现方法的调用，避免了大量的函数表派发的跳转指令，某种意义上对比函数表实现的多态好处就是同时加快了程序运行速度，也会删除了程序中那些永远不会调用的代码从而减少程序包的体积。


[Swift Runtime - 类和对象](https://juejin.cn/post/6844903876215193608)对比理解OC和Swift对象的差异

**SwiftObject**
在Swift中class如果没有显式继承其他的类，都被隐式继承SwiftObject。SwiftObject实现了NSObject协议的所有方法和一部分NSObject类的方法。主要是重写了一部分方法，将方法实现改为Swift相关方法。比如retain，release方法改为了使用swift runtime进行引用计数管理:
``` C++
- (id)retain {
  auto SELF = reinterpret_cast<HeapObject *>(self);
  swift_retain(SELF);
  return self;
}
- (void)release {
  auto SELF = reinterpret_cast<HeapObject *>(self);
  swift_release(SELF);
}
```
没有实现 resolveInstanceMethod，forwardingTargetForSelector等方法，这些方法可以在找不到特定方法时可以进行动态处理，应该是不想提供纯 Swift类在这块的能力。

**HeapObject**
``` C++
struct HeapObject {
  HeapMetadata const *metadata;
  InlineRefCounts refCounts;
};
/*
Objc对象结构 {
    isa_t,
    实例变量
}
Swift对象结构 {
    metadata,
    refCounts, 
    实例变量
}
*/
```

**ClassMetadata**
``` C++
struct objc_object {
    Class isa;
}
struct objc_class: objc_object {
    Class superclass;
    cache_t cache;           
    class_data_bits_t bits;
}
struct swift_class_t: objc_class {
    uint32_t flags;//类标示
    uint32_t instanceAddressOffset;
    uint32_t instanceSize;//对象实例大小
    uint16_t instanceAlignMask;//
    uint16_t reserved;// 保留字段
    uint32_t classSize;// 类对象的大小
    uint32_t classAddressOffset;// 
    void *description;//类描述
};
```
Swift和Objective-C的类元数据是共用的，Swift类元数据只是Objective-C的基础上增加了一些字段（源代码中也有一些地方直接使用 reinterpret_cast进行相互转换）
泛型类不会生成类元数据__objc_class结构，不过会生成roData，class如果没有显式继承某个类，都被隐式继承SwiftObject。。


**swift_allocObject**：创建一个堆上的Swift对象。
``` C++
static HeapObject *_swift_allocObject_(HeapMetadata const *metadata,
                                       size_t requiredSize,
                                       size_t requiredAlignmentMask) {
  auto object = reinterpret_cast<HeapObject *>(
      swift_slowAlloc(requiredSize, requiredAlignmentMask));
  // NOTE: this relies on the C++17 guaranteed semantics of no null-pointer
  // check on the placement new allocator which we have observed on Windows,
  // Linux, and macOS.
  new (object) HeapObject(metadata);//创建一个新对象，
  return object;
}
```
**swift_initStackObject**：在栈上创建一个对象。没有引用计数消耗，也不用malloc内存。

**swift_deallocClassInstance**：销毁对象，在对象dealloc时调用。
``` C++
void swift::swift_deallocClassInstance(HeapObject *object,
                                       size_t allocatedSize,
                                       size_t allocatedAlignMask) {
#if SWIFT_OBJC_INTEROP
  objc_destructInstance((id)object); //释放Objc runtime的弱引用和关联对象释放处理。
#endif
  swift_deallocObject(object, allocatedSize, allocatedAlignMask); //调用free回收内存。
}
```


**Swift泛型类**

``` C++
//定义
class GenericClass<T> {
}
GenericClass<Int>()

//实现
MetadataResponse response = swift_getGenericMetadata();
ClassMetadata *classMetadata = swift_allocateGenericClassMetadata();
swift_initClassMetadata2(classMetadata);
HeapObject *object = swift_allocObject(objcClass);

```
根据泛型类型作为参数，调用swift_getGenericMetadata方法获取类对象缓存，存在则直接返回，没有则swift_allocateGenericClassMetadata方法生成类对象缓存（**每一个不同的泛型类型都会创建一个新的ClassMetadata，之后保存到缓存中复用**）。

* swift_allocateGenericClassMetadata：
	1. 创建一个新的ClassMetadata结构。
	2. 初始化objc_class和swift_class_t相关的属性, 同时设置isa和roData。
* swift_initClassMetadataImpl：
	1. 设置Superclass，如果没有指明父类，会被设置为SwiftObject。
	2. 初始化Vtable。
	3. 设置class_ro_t的InstanceStart和InstanceSize字段，遍历ivars修改每个ivar的offset。
	4. 将该类注册到objc runtime。

# Swift反射机制
所谓反射就是可以动态获取类型、成员信息，在运行时可以调用方法、属性等行为的特性。在使用OC开发时很少强调其反射概念，因为OC的Runtime要比其他语言中的反射强大的多。不过在Swift中并不提倡使用Runtime，而是像其他语言一样使用反射(Reflect)，即使目前Swift中的反射功能还比较弱，只能访问获取类型、成员信息。

## Swift 实现Json解析的2套方案
1. 反射
[Swift - 使用反射将自定义对象数据序列化成JSON数据](http://www.hangge.com/blog/cache/detail_983.html)
2. codable（encode&decode）
