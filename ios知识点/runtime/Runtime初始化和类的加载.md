# Runtime初始化和类的加载

[iOS 从 _objc_init 分析类的加载流程 - 掘金](https://juejin.im/post/6844904081983537166#heading-23)

# OC类的加载
image（镜像）：表示一个二进制文件(可执行文件或 so 动态链接库文件)，里面是被编译过的符号、代码等，
libobjc：runtime的动态连接库。
![](Runtime%E5%88%9D%E5%A7%8B%E5%8C%96%E5%92%8C%E7%B1%BB%E7%9A%84%E5%8A%A0%E8%BD%BD/99223DBA-3EB3-4402-9AEA-5CF6E6C3528D.png)

## 应用加载过程：
![](Runtime%E5%88%9D%E5%A7%8B%E5%8C%96%E5%92%8C%E7%B1%BB%E7%9A%84%E5%8A%A0%E8%BD%BD/2AB7D657-334B-45CB-9D23-4E00B4452252.png)
Load dylibs -> Rebase -> Bind -> ObjC -> Initializers

1. 系统调用 exec() 为我们的应用映射到新的虚拟地址空间，创建app进程。
2. 接着系统调用创建并加载dyld到app进程中，然后进行dyld初始化和自举（通过macho内存的地址加偏移直接调用的方式进行dyld层面的初始化和一些链接处理） 
3. dyld开始通过ImageLoader加载磁盘上的mach-o文件并扫描Header和load commands信息，递归地加载依赖的动态链接库（先处理主可执行文件，再递归处理库），并进行 rebase 指针调整和 bind 符号绑定等linker操作。
	1. Rebase（内）重定位修正内部(指向当前mach-o文件)的指针指向（因为现在操作系统基本为了安全使用ASLR使得程序入口地址随机化）。
	2. Bind（外）根据字符串匹配的方式查找符号表修正外部指针指向，确定依赖的系统动态库里的函数地址。
4. 同时dyld还会初始化libSystem（系统核心动态库），最后会进行_objc_init的初始化工作，objc_init 中会注册dyld给出来的回调监听（当dyld工作过程中objc 镜像被映射（mapped）到内存、加载（laod）和卸载（unmapped）的时候，注册的回调函数就会被调用），runtime会在回调里进行相关处理，主要是类的加载，分类信息的合并。
5. 然后进行attribute((constructor)) 修饰的C++初始化函数的调用、创建静态全局变量等（先处理库再主可执行文件）。
6. 递归操作完成后，dyld 返回主程序的入口函数，开始进入主程序的 main 函数。

## runtime接手后类的加载
1. 项目环境变量初始化：Xcode项目中配置的环境变量初始化。
2. staticinit：进行objc环境实现层面的初始化（通过mac-o文件header内记录的这些函数的起地址+偏移计算得到最终每个函数的执行地址后直接调用）。
3. 异常初始化：objc层向系统注册系统层面的异常回调（最终通过NSSetUncaughtExceptionHandler，foundation层向objc注册，并把异常抛给我们，实现异常的一些监听处理）。
4. 向dyld注册mapped、load、remove等时机的回调
	1. mapped：dyld已经将可执行资源加载到了内存中。
		1. SLE处理：以SEL名为key，SEL为value注册到SEL表中，并会修正SEL，还会进行唯一性检查。
		2. 类注册：将类名name为key，初始地址cls为value，存储到类表中。
		3. 协议注册：以协议名为key，protocol_t为value，存储到协议表中。
		4. OC类数据的非懒加载（类和分类任何一种实现了load）：通过attachCategories头插合并分类和类中的属性、协议、方法列表（会根据SEL地址进行排序，后序msgsend才能进行二分查找）。
	2. load：主要是直接拿到函数地址，调用load方法。
		1. 获取所有非懒加载的类，沿着superclass，从父类-子开始load调用。
		2. 获取所有非懒加载的分类，调用load。
	3. unmap：处理被dyld进行unmap后的image镜像，先卸载后移除。

非懒加载类在类的内部实现了 load 方法，类的加载就会提前到程序启动时。
而懒加载类没有实现 load 方法，在使用的第一次发消息发现没有加载时才会加载（alloc）。

# objc 初始化过程分析
``` c++
void _objc_init(void)
{
    static bool initialized = false;
    if (initialized) return;
    initialized = true;
    
    // fixme defer initialization until an objc-using image is found?
    environ_init();
    tls_init();
    static_init();
    runtime_init();
    exception_init();
#if __OBJC2__
    cache_t::init();
#endif
    _imp_implementationWithBlock_init();

    _dyld_objc_notify_register(&map_images, load_images, unmap_image);

#if __OBJC2__
    didCallDyldNotifyRegister = true;
#endif
}
```
1. **environ_init**：Xcode项目中配置的各种环境变量初始化，环境变量一般以OBJC开头，后面又分为PRINT（打印）、DEBUG（调试）、DISABLE（禁用）这几类，环境变量设置，会帮助我们更快速的处理一些问题，比如：
	* **OBJC_PRINT_LOAD_METHODS**可打印项目中所有的load方法
	* **OBJC_DISABLE_NONPOINTER_ISA**可关闭isa优化。
2. **tls_init**：初始化线程池。
3. **static_init**：dyld研究的时候能看到，它也会进行静态调用（通过mac-o文件header内记录的这些函数的起地址+偏移计算得到最终每个函数的执行地址后直接调用）image内的C++构造函数，但是因为dyld调用的时机比libc调起_objc_init的时机晚，所以需要提前进行objc实现层的C++构造函数调用完成objc实现层相关类的初始化。
4. **runtime_init**：初始化类、分类的容器。
5. **exception_init**：objc层向系统注册系统层面的异常回调，系统方法在执行的过程中，出现异常触发中断，操作系统就会报出异常。这个方法的实现`old_terminate = std::set_terminate(&_objc_terminate)`就是完成了回调注册，_objc_terminate的内部实现如下：
	1. 如果是一个active的exception，则继续检查是否是objc层的错误。
	2. 如果是objc层的错误则调用foundation层注册的异常回调uncaught_handler处理（objc层暴露给了上层一个`objc_setUncaughtExceptionHandler`接口方法来注册回调，foundation层如果没有调用这个方法向objc层注册的话，objc层本身也会有一个默认的_objc_default_uncaught_exception_handler回调）
	3. foundation通过NSSetUncaughtExceptionHandler方法把回调又抛给了上层的我们，所以我可以可以捕获这些异常，进行一些异常、崩溃的调试。
6. **_imp_implementationWithBlock_init**：对imp的Block标记进行初始化
7. **_dyld_objc_notify_register**：向dyld注册回调，之后等待dyld通知objc 镜像image被处理后的mapped、load、remove状态，进入map_images、load_images、unmap_image做后续的处理。

### Dyld向objc给出的时机：
1. 在recursiveInitialization方法中调用`bool hasInitializers = this->doInitialization(context);`这个方法是来判断image是否已加载
2. doInitialization这个方法会调用`doModInitFunctions(context)`这个方法就会进入libSystem框架里调用`libSystem_initializer`方法，最后就会调用`_objc_init`方法
3. _objc_init会调用`_dyld_objc_notify_register`将map_images、load_images、unmap_image传入dyld方法`registerObjCNotifiers`。
4. 在`registerObjCNotifiers`方法中，我们把`_dyld_objc_notify_register`传入的map_images赋值给`sNotifyObjCMapped`，将load_images赋值给`sNotifyObjCInit`，将unmap_image赋值给sNotifyObjCUnmapped。
5. 在`registerObjCNotifiers`方法中，我们将传参复制后就开始调用`notifyBatchPartial()`。
6. notifyBatchPartial方法中会调用`(*sNotifyObjCMapped)(objcImageCount, paths, mhs)；`触发map_images方法。
7. dyld的recursiveInitialization方法在调用完`bool hasInitializers = this->doInitialization(context)`方法后，会调用notifySingle()方法
8. 在notifySingle()中会调用`(*sNotifyObjCInit)(image->getRealPath(), image->machHeader());`上面我们将load_images赋值给了sNotifyObjCInit，所以此时就会触发load_images方法。
9. sNotifyObjCUnmapped会在removeImage方法里触发，字面理解就是删除Image（映射的镜像文件）。

# map_images
map_image内处理被dyld进行mapped映射后（重定位完）的image镜像
，首先计算image内class数量hCount方便在后续处理中对class进行递归处理，之后进入_read_images开始下面的处理。
``` c++
//_read_images内的处理
ts.log(“IMAGE TIMES: first time tasks”);

ts.log("IMAGE TIMES: fix up selector references");

readClass(cls, headerIsBundle, headerIsPreoptimized);
ts.log("IMAGE TIMES: discover classes");

if (!noClassesRemapped()) { 
	_getObjc2ClassRefs(hi, &count);
	remapClassRef(&classrefs[i]);
}
ts.log("IMAGE TIMES: remap classes");

#if SUPPORT_FIXUP 
	_getObjc2MessageRefs(hi, &count);
	fixupMessageRef(refs+i);
	ts.log("IMAGE TIMES: fix up objc_msgSend_fixup");
#endif

_getObjc2ProtocolList(hi, &count);
readProtocol(protolist[i], cls, protocol_map, 
                         isPreoptimized, isBundle);
ts.log("IMAGE TIMES: discover protocols");

_getObjc2ProtocolRefs(hi, &count);
remapProtocolRef(&protolist[i]);
ts.log("IMAGE TIMES: fix up @protocol references");

load_categories_nolock(hi);
ts.log("IMAGE TIMES: discover categories");

remapClass(classlist[i]);
addClassTableEntry(cls);
if (cls->isSwiftStable() && cls->swiftMetadataInitializer()) {
	_objc_fatal("Swift class %s with a metadata initializer, is not allowed to be non-lazy")
}
realizeClassWithoutSwift(cls, nil);
ts.log("IMAGE TIMES: realize non-lazy classes");

if resolvedFutureClasses {
	realizeClassWithoutSwift(cls, nil);
}
ts.log("IMAGE TIMES: realize future classes");
```

1. `ts.log(“IMAGE TIMES: first time tasks”);` 第一次进入时：
	1. non-pointer开关处理：一些特殊条件（image包含swiftcode，macOS上app太老）下关闭isa的non-pointer优化处理。
	2. TaggedPointer开关处理：如果OBJC_DISABLE_TAGGED_POINTERS配置为true则关闭TaggedPointer的优化处理。
	3. 创建保存类的哈希表：通过NXCreateMapTable创建存储类的gdb_objc_realized_classes哈希表，并会自动根据hCount做扩容（扩容因子4/3，同时只要是不在共享缓存的类，所有实现或者非实现的都会在这个表里面）。
2. `ts.log("IMAGE TIMES: fix up selector references");`进行SEL的注册和修正：
	1. 遍历取出编译后的所有SEL，再注册到namedSelectors表中。
	2. 注册后会再修正原SEL（处理预编译时SEL的处理混乱问题）的地址为新完成注册的SEL地址。
3. `ts.log("IMAGE TIMES: discover classes");`进行类和类名的关联并注册到类表中，如果是FutureClasses还会进行它内存空间的初始化（只是先开辟了一块空间而已）🤔️。
	1. 遍历取出编译后的所有类cls，进行readClass操作，经过readClass后，po cls就打印出类名而不再是地址，类对应的首地址就是原cls存储的地址，即这一步完成了oc类的内存起始布局，readClass读取到内存的仅仅只有地址+名称，类的data数据并没有加载出来，其中还会：
		1. addNamedClass：将类的名字name跟地址cls进行关联存储到gdb_objc_realized_classes表中（_read_images第一次进入时创建的表）。 
		2. addClassTableEntry：将这个类添加到类表allocatedClasses中（objc_init里创建，存放所有的实例化的类）
	2. 🤔️如果readClass 返回后的newcls和cls不同，则初始化所有FutureClasses的类需要的内存空间（resolvedFutureClasses指向位置为起点，class大小为偏移单位）
4. `ts.log("IMAGE TIMES: remap classes");`当noClassesRemapped为false时，如果懒加载的类中还有未进行映射的类会进行重映射（一般不会进入）。
5. `ts.log("IMAGE TIMES: fix up objc_msgSend_fixup");`修复旧的objc_msgSend。
6. `ts.log("IMAGE TIMES: fix up @protocol references");`遍历取出编译后的protocal，进行redProtocal读取操作，将协议注册到Protocol表中。
7. `ts.log("IMAGE TIMES: fix up @protocol references");`修复协议的引用。
8. `ts.log("IMAGE TIMES: discover categories");`处理categories（一种特殊情况，暂不研究）
9. `ts.log("IMAGE TIMES: realize non-lazy classes");`进行非懒加载类的数据初始化：
	1. 遍历取出编译后的非懒加载的类（实现了load方法），重映射获得正确地址后调用addClassTableEntry把cls添加到类表allocatedClasses中（表中已有，已经在discover classes阶段插入过则不会插入）
	2. 再通过`realizeClassWithoutSwift(cls, nil)`对cls进行初始化加载（加载出除cls地址、name之外的data数据）。
10. `ts.log("IMAGE TIMES: realize future classes");`🤔️对resolvedFutureClasses的类进行realize，也会通过`realizeClassWithoutSwift(cls, nil)`进行初始化加载。

> 🤔️的地方暂时没确切理解到FutureClasses的处理，可能是一种特殊情况，暂时不做深入研究。  

# realizeClassWithoutSwift：oc类数据的加载
因为类有很多代码，很多方法排序和临时变量，如果都放在main函数前加载，会导致加载时间很长，如果类从来没有被调用，那他不需要提前加载而提高性能，所以类的加载存在懒/非懒加载2种形式：
1. 懒加载在map_images内触发：主类和分类任何一种实现了load
2. 非懒加载在第一次消息发送时：主类和分类都没有实现load

``` c++
//static Class realizeClassWithoutSwift(Class cls, Class previously) 方法主要处理如下：
...
auto ro = (const class_ro_t *)cls->data();
auto isMeta = ro->flags & RO_META;
if (ro->flags & RO_FUTURE) {
    // This was a future class. rw data is already allocated.
    rw = cls->data();
    ro = cls->data()->ro();
    ASSERT(!isMeta);
    cls->changeInfo(RW_REALIZED|RW_REALIZING, RW_FUTURE);
} else {
    // Normal class. Allocate writeable class data.
    rw = objc::zalloc<class_rw_t>();
    rw->set_ro(ro);
    rw->flags = RW_REALIZED|RW_REALIZING|isMeta;
    cls->setData(rw);
}
...
supercls = realizeClassWithoutSwift(remapClass(cls->getSuperclass()), nil);
metacls = realizeClassWithoutSwift(remapClass(cls->ISA()), nil);
...
// Update superclass and metaclass in case of remapping
cls->setSuperclass(supercls);
cls->initClassIsa(metacls);
...
// Connect this class to its superclass's subclass lists
if (supercls) {
    addSubclass(supercls, cls);
} else {
    addRootClass(cls);
}
// Attach categories
methodizeClass(cls, previously);
```
1. ro-rw数据拷贝：先将cls的data数据读取出来，并转换为class_ro_t *，复制一份拷贝到rw。
2. 递归处理父类和元类的realize：通过`cls->ISA()`和`cls->getSuperclass()`方法，拿到父类和元类，开始递归的realize处理，并返回supercls和metacls。
3. 递归处理完后更新父类和元类的指向：通过`cls->setSuperclass(supercls)`和`cls->initClassIsa(metacls)`更新realize完成后的父类和元类。
4. 把cls加入到父类的subclass列表中：双向链表指向关系。
5. 通过attachCategories，合并categories内的信息到rw中：
	1. 合并method_list_t（合并前会根据SEL地址对方法进行排序，所以msgsend查找时才能进行二分查找，提高查找效率）
	2. 合并property_list_t
	3. 合并protocol_list_t
	4. 合并分类的内容

>  _category_t是个结构体，里面存在名字，cls，对象方法列表，类方法列表，协议，属性，之所以分类有两个列表是因为分类是没有元分类的，分类的方法是在运行时通过attachToClass插入到class的。  

# load_images
objc image镜像被被dyld进行mapped映射处理后进行镜像内class的load处理，主要是调用load方法。
1. 如果didInitialAttachCategories为false，分类还没有init则会进行。loadAllCategories操作（属于特殊情况，不研究）
2. 健壮性检查如果一个类没有load方法，则直接返回。
3. 如果有load则`prepare_load_method(mh)`先获取所有的+load方法，再：
	1. `_getObjc2NonlazyClassList()`获取所有非懒加载的类，遍历这些类将其+load方法添加到loadable_classes数组中进行保存，其中会开始递归的根据`cls.superclass`找父类，处理父类的load方法。
	2. `_getObjc2NonlazyCategoryList()`获取所有非懒加载的分类，遍历这些分类将其+load方法添加到loadable_categories数组中进行保存，其中同一个类的分类的load添加顺序按照image内（编译时）的顺序。
4. `call_load_methods()`：通过函数指针直接调用获取到的所有load方法，在while循环的时候是先遍历执行loadable_classes表中类和父类的+load方法，后遍历loadable_categories表中分类的+load方法。

**总结**
1. Load调用顺序：第一个维度先父再子，第二个维度先本类再分类（多个分类 load 的顺序根据编译顺序）
2. Load调用方式：直接拿到函数指针进行调用而非msgsend。
3. 子类未实现load方法时，不会触发调用继承自父类load方法（父类自己实现的load方法在加载自己时还是会调用的）

# unmap_image
处理被dyld进行unmap后的image镜像
1. 先卸载需要unmap的image数据。
2. 再移除需要unmap的image。

**总结：**
第一次进来
判断是否使用non-pointer对isa进行优化
对TaggedPointer的优化处理
创建保存类的哈希表
注册修正sel
获取所有类，读取类并将类存储到3创建的表中
修复需要重映射的类
获取并修正协议
对非懒加载类进行处理
对懒加载类进行处理

```
TARGETS -> debug-objc -> Build Settings -> Enable Hardened Runtime = NO
```