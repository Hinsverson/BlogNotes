# Runtime数据结构
#iOS知识点/Runtime

[iOS runtime 机制解读（结合 objc4 源码）](https://juejin.cn/post/6844904069740363784#heading-2)

![](Runtime%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/8BC7BE95-0A3F-4657-A86D-529920F878E0.png)
![](Runtime%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/D8ECF3B9-80B0-444F-BB04-41489E3A67CE.png)

# objc_object（实例对象的数据结构）
``` c++
typedef struct objc_object *id;
union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;
#if defined(ISA_BITFIELD)
    struct {
        ISA_BITFIELD;  // defined in isa.h
    };
#endif
};

struct objc_object {
private:
    isa_t isa;
// public & private method...
}
```
![](Runtime%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/2408CEB8-F65A-4350-ABC6-7BF8ABD867D1.png)
**OC：**
一个objc_Object结构体，存储了包含指向类对象的isa指针、以及isa、弱引用处理、内存管理相关的操作方法。

> Objective-C 对象是由 id 类型表示的，它本质上是一个指向 objc_object 结构体的指针。  

**Swift：**
一个HeapObject结构体，存储了包含指向类信息HeapMetadata的指针和RefCounts。

# objc_class（类对象的数据结构）
``` c++
typedef struct objc_class *Class;
struct objc_class : objc_object {
    // 指向类的指针(位于 objc_object)
    // Class ISA;
    // 指向父类的指针
    Class superclass;
    // 用于缓存指针和 vtable，加速方法的调用
    cache_t cache;             // formerly cache pointer and vtable
    // 存储类的方法、属性、遵循的协议等信息的地方
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
    // class_data_bits_t 结构体的方法，用于返回class_rw_t 指针（）
    class_rw_t *data() { 
        return bits.data();
    }
    // other methods...
}
```
![](Runtime%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/6A24497E-CEF6-4948-A237-BF41C75A3E77.png)
**OC**
isa：指向元类的指针
superClass：父类对象
Cache：方法缓存
data：通过bits和掩码获取的类信息（class_rw_t）

> Objective-C 的类是由 Class 类型来表示的，它实际上是一个指向 objc_class 结构体的指针。  
> OC中的objc_class也是一个对象，它继承自objc_object，因此它也拥有了 isa 指针  

**Swift：**
StoredPointerKind：isa
Superclass：superClass
CacheData：Cache
StoredSizeData：data

# class_rw_t（类信息）
``` c++
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;
    
    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;
    
    Class firstSubclass;
    Class nextSiblingClass;
    
    char *demangledName;

#if SUPPORT_INDEXED_ISA
    uint32_t index;
#endif
    // other methods
}
```
![](Runtime%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/252F1287-7601-4168-BE3E-7A7402939CBC.png)
一个存放OC类对象信息的结构体，包含了属性、协议、方法列表等二维数组，并且是可读和可写的，可以通过修改它实现在运行时动态添加方法。方法二位数组里面又包含了method_list_t，method_list_t里包含了方法method_t的封装。

> class_rw_t类信息的获取：存储在objc_class的bits上，通过一个掩码进行&运算取出具体的类信息起始地址，再根据类型转化为对象。  

# class_ro_t（编译期的类信息）
![](Runtime%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/5E8AA60F-9369-4456-8675-FF34C679A064.png)
是class_rw_t内的成员、存储了当前类在编译期就已经确定的属性、方法以及遵循的协议等，是不可写的。在OC启动初始化时，实际是把class_ro_t里的内容进行拷贝，再和分类的内容结合起来放入class_rw_t里面。

# method_t（方法、函数的封装）
![](Runtime%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/D4392ADF-C341-4F50-8E19-0392BF77E805.png)
SEL：方法名
imp：指向函数的指针
types：方法的参数和返回值类型编码（TypeCoding技术）

> class_rw_t是可读可写的，这也是在运行时可以动态添加方法的原因。比如新增一个分类定义了很多方法，那么其实是在method_array_t里的最前面新增了一个method_list_t，这个method_list_t装着新分类中新增的方法。  

## SEL：方法选择器
```
typedef struct objc_selector *SEL;
```
方法选择器SEL，是一个指向 objc_selector 结构体的指针，也是 objc_msgSend 函数的第二个参数类型。

方法的 selector 用于表示运行时方法的名称。代码编译时，会根据方法的名字（不包括参数）生成一个唯一的整型标识（ Int 类型的地址），即 SEL。

> 一个类的方法列表中不能存在两个相同的 SEL，这也是 **Objective-C 不支持重载的原因。  
> 不同类之间可以存在相同的 SEL，因为不同类的实例对象执行相同的 selector 时，会在各自的方法列表中去寻找自己对应的 IMP。  

获取 SEL的方式有三种：
* sel_registerName函数
* Objective-C 编译器提供的@selector()方法
* NSSeletorFromString()方法

## IMP
``` C++
typedef void (*IMP)(void /* id, SEL, ... */ ); 
```
IMP 本质上就是一个函数指针，**指向方法实现的地址**。
参数说明：
	* id：指向 self 的指针（如果是实例方法，则是类实例的内存地址；如果是类方法，则是指向元类的指针）
	* SEL：方法选择器
	* …：方法的参数列表

SEL 与 IMP 的关系类似于哈希表中 key 与 value 的关系。采用这种哈希映射的方式可以加快方法的查找速度。

# Cache_t（方法缓存）
``` C++
struct cache_t {
    // 存放方法的数组
    struct bucket_t *_buckets;
    // 能存储的最多数量
    mask_t _mask;
    // 当前已存储的方法数量
    mask_t _occupied;
    // ...
}
```
![](Runtime%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/FB5841E9-3F1D-46EA-8592-67C004BE6CAE.png)
* mask：一个整数，指定分配的缓存bucket的总数。
* occupied：一个整数，指定实际占用的缓存bucket的总数。
* buckets：指向Method数据结构的散列表数组指针，表中存放bucket_t。
* bucket_t：Method的一种描述结构，由SEL和 imp函数地址组成。

> buckets数组可能包含不超过mask+1个元素。需要注意的是，指针可能是NULL，表示这个缓存bucket没有被占用，另外被占用的bucket可能是不连续的，这个数组可能会随着时间而增长。  

*特点*
1. 用于缓存最近使用的方法
2. 可增量的哈希表结构
3. 局部性原理的应用（常见方法的抽象集合）

# category_t
``` C++
struct category_t {
    // 是指类名，而不是分类名
    const char *name;
    // 要扩展的类对象，编译期间是不会定义的，而是在运行时阶段通过name对应到相应的类对象
    classref_t cls;
    // 实例方法列表
    struct method_list_t *instanceMethods;
    // 类方法列表
    struct method_list_t *classMethods;
    // 协议列表
    struct protocol_list_t *protocols;
    // 实例属性
    struct property_list_t *instanceProperties;
    // Fields below this point are not always present on disk.
    // 类（元类）属性列表
    struct property_list_t *_classProperties;
    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
};
```
category_t 表示一个指向分类的结构体的指针。
