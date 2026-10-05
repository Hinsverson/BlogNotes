# iOS 资深开发面试知识点问答

适用于有 7 年左右开发经验、以原生 iOS 为主的面试复习。重点是能够从语言和系统机制解释实际问题，再说明方案的代价、验证方法和适用边界。

本文按现有笔记中的重点主题重新组织，共 **66 道主问题：63 道核心题，3 道岗位选读题**。每题先给可直接用于面试的回答，再补充追问、例子或易错点。优先级是结合笔记内容和资深岗位能力要求的筛选判断，不代表某家公司的题频统计。计算机基础仅保留能联系客户端实际问题的重点，Flutter 和 C++ 各保留一道综合题。

**阅读顺序：**第一轮通读一至十章；第二轮重点复习并发、UI、性能和架构；第三轮只看问题，尝试口述答案，再补上自己真实项目里的证据。第十一章按岗位选择。

**内容边界：**梳理了仓库 164 篇 Markdown 的目录和主题，重点研读与本题集相关的正文；没有逐页展开全部 PDF 课件，也不把仅存在于截图中的代码当作已验证结论。第五章为原笔记缺少的 Swift 并发补充，数据库两题也是补充内容；HTTP/3 等更新内容和关键纠错参考了官方资料。核对日期为 2026 年 10 月 3 日。Runtime 私有布局、dyld 内部调用链和编译器优化不能当作跨版本不变的 API 契约。

## 目录

| 分类 | 问题范围 | 复习重点 |
| --- | --- | --- |
| [一 Objective-C 与 Runtime](#一-objective-c-与-runtime) | 01—08 | 消息派发、元类、分类、KVO、Hook |
| [二 内存管理](#二-内存管理) | 09—15 | ARC、weak、Block、释放池、所有权 |
| [三 RunLoop 与 GCD](#三-runloop-与-gcd) | 16—23 | 事件循环、死锁、同步、任务生命周期 |
| [四 Swift 语言机制](#四-swift-语言机制) | 24—30 | 值语义、派发、闭包、泛型、混编 |
| [五 Swift 并发补充](#五-swift-并发补充) | 31—35 | 隔离、重入、取消、Sendable |
| [六 UIKit 与渲染](#六-uikit-与渲染) | 36—41 | 布局、事件、动画、列表、一帧流程 |
| [七 性能与稳定性](#七-性能与稳定性) | 42—48 | 卡顿、启动、内存、崩溃、图片、指标 |
| [八 网络](#八-网络) | 49—54 | HTTPS、TCP、HTTP、缓存、重试、下载 |
| [九 架构与工程设计](#九-架构与工程设计) | 55—58 | 分层、组件化、响应式、可测试性 |
| [十 计算机基础精选](#十-计算机基础精选) | 59—63 | 编译链接、虚拟内存、汇编、索引、事务 |
| [十一 岗位选读](#十一-岗位选读) | 64—66 | C++、Metal、Flutter |

## 一 Objective-C 与 Runtime

### 01 Objective-C 消息从发送到执行经历了什么

**回答：**普通 Objective-C 消息调用通常进入 `objc_msgSend(receiver, selector, ...)`。Runtime 根据接收者找到类，先检查方法缓存，再查方法列表及父类链；得到 `IMP` 后按方法的调用约定执行。方法实现可以来自父类，不能因为对象是某个子类，就认为调用的一定是子类自己的实现。

查找失败后，依次有三类补救机会：

1. **动态方法解析**：`+resolveInstanceMethod:` 或 `+resolveClassMethod:` 中添加实现。类方法应添加到元类。
2. **快速转发**：`-forwardingTargetForSelector:` 返回另一个接收者。
3. **完整转发**：先提供 `-methodSignatureForSelector:`，再在 `-forwardInvocation:` 中处理 `NSInvocation`。

最终不能处理时，通常进入 `doesNotRecognizeSelector:` 并抛出异常。方法签名必须与真实参数、返回值匹配，否则即使“转发成功”也可能破坏调用现场。

**追问：缓存保存什么？**核心是选择子到实现地址的映射，缓存在类相关结构中，不是每个实例各有一张。通过公开 Runtime API 替换方法时，Runtime 会处理必要的缓存维护；不要手动修改私有缓存。哈希算法、扩容阈值、结构体字段是版本细节。

### 02 实例对象 类对象 元类分别存什么

**回答：**实例保存自身状态及用于找到类的信息；类对象描述实例的布局、实例方法等；元类承载类方法相关信息。类也是对象，所以类对象也有 `isa`。

```text
实例 --isa--> 类 --isa--> 元类
类 --superclass--> 父类
元类 --superclass--> 父元类
```

在 NSObject 根类体系中，根元类的 `isa` 指向自身，根元类的 `superclass` 指向根类，根类的 `superclass` 为 nil。**`isa` 表示“它是什么类的实例”，`superclass` 表示继承关系，不能混用。**

`object_getClass(obj)` 读取 Runtime 所见的类；`[obj class]` 是可被重写的方法。KVO 等机制下，两者的结果可能不同。现代 `isa` 还可能编码额外标志，不能直接把其所有位当成裸类地址。

**追问：`isKindOfClass:` 和 `isMemberOfClass:` 呢？**对普通实例，前者检查类及父类链，后者要求直接属于指定类。接收者换成类对象后，应从元类关系分析，不能机械套用实例结论。例如 NSObject 默认实现下，`[NSObject isKindOfClass:[NSObject class]]` 为 YES，而 `isMemberOfClass:` 为 NO；原因是元类继承链能走到 NSObject 类，但元类本身不等于 NSObject 类。

### 03 self 和 super 的区别是什么

**回答：**`super` 不会创建父类对象，也不改变消息接收者。它只是让方法查找从**当前方法词法所属类的父类**开始，接收者仍然是 `self`。这比“从对象真实类型的父类开始”更准确，尤其是在多层继承中。

```objc
@implementation Child
- (void)printClass {
    NSLog(@"%@", [self class]);
    NSLog(@"%@", [super class]);
}
@end
```

假设 Child 继承 NSObject、没有重写 `class`，且接收者正好是 Child 实例，两次都输出 Child。第二次只是从父类找到 `-class` 的实现，执行时收到的仍是 Child 实例。

**追问：在父类实现里再调用 `[self otherMethod]` 呢？**这是新的普通消息发送，会按接收者的动态类型查找，因此可以调用到子类重写的方法。`super` 不会让后续所有消息都固定走父类。

### 04 Category 和 Extension 有什么区别 关联对象如何实现

**回答：**Extension 是编译期组成类声明的一部分，常用于隐藏属性、成员变量和内部方法。Category 可以给已有类增加方法、协议声明和属性声明，其方法信息在运行时附加到类或元类。

Category 声明属性不会自动生成实例变量和访问器。它也不能直接改变已有实例的内存布局。关联对象是把额外状态保存在 Runtime 管理的外部关联关系中，通过宿主对象和唯一 key 查找，因此不等于给实例内存“塞进了一个成员变量”。

key 常用具有稳定唯一地址的静态变量；policy 决定 retain、copy 等所有权行为。`OBJC_ASSOCIATION_ASSIGN` **不是自动置 nil 的 weak**，需要弱语义时可用一个内部 weak 属性的包装对象。

**追问：分类同名方法会怎样？**可能遮蔽原实现或另一分类实现，但跨分类冲突的最终选择不应作为业务契约。关联对象也会形成循环引用，宿主析构时清理关联关系并不能解决“宿主因循环引用而根本不析构”。

### 05 load 和 initialize 的区别是什么

**回答：**`+load` 与镜像加载、Runtime 装载类和分类有关；启动时加载的镜像通常在 `main` 前执行它。Runtime 直接调度各自实现，父类先于子类，类先于其分类。不要依赖互不相关的类或分类之间的具体顺序。后续动态加载的镜像也可能有 `+load`，所以不是所有 `+load` 都一定发生在进程启动阶段。

`+initialize` 在类第一次使用前按需触发，先保证父类初始化；它遵循消息派发规则。子类没有实现时，可能执行继承来的父类实现，但此时 `self` 是子类，所以同一份父类实现可能被不同类的初始化调用。

```objc
+ (void)initialize {
    if (self == [MyClass class]) {
        // 只初始化 MyClass 自己的状态
    }
}
```

**工程取舍：**避免在 `+load` 中做磁盘、网络、大量注册或对象构造；`+initialize` 也不能当成无代价的懒加载容器，它可能把首次使用卡在初始化上，复杂的跨线程互等还会造成死锁。业务初始化更适合显式、有依赖和可测量的任务入口。

### 06 KVC 如何查找属性 与 Swift KeyPath 有什么区别

**回答：**KVC 用字符串 key 间接访问对象状态。基本 setter 查找优先尝试 `set<Key>:`、`_set<Key>:`；没有访问器且 `accessInstanceVariablesDirectly` 为 YES 时，依次尝试 `_<key>`、`_is<Key>`、`<key>`、`is<Key>`。失败调用 undefined-key 处理，默认抛异常。

getter 优先尝试 `get<Key>`、`<key>`、`is<Key>`、`_<key>`，还支持集合访问器规则，之后才回退查实例变量。给非对象标量设置 nil 会进入 `setNilValueForKey:`，默认也是异常。面试重点是说明访问器优先、直接 ivar 回退和错误边界，不应漏掉集合规则后声称只有一种查找路径。[Apple KVC 查找规则](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueCoding/SearchImplementation.html)

**追问：Swift 的 `\User.name` 是不是 KVC？**不是。Swift KeyPath 是带类型信息的属性访问路径，可用于结构体、泛型和编译期检查；Objective-C KVC 依赖字符串及 KVC 合规对象。`#keyPath` 与原生 `KeyPath` 也不是同一种机制。

### 07 KVO 的原理是什么 哪些修改不会触发通知

**回答：**传统 NSObject 自动 KVO 通常采用动态子类和 isa 替换：观察对象的实际类变为通知子类，拦截受观察属性的 setter，在修改前后执行变更通知；`class` 方法通常仍报告原始类。面试应说明这是经典实现机制，不把生成类名等私有细节当作 API 保证。

直接 `_value = x` 一般绕过自动 setter 通知。KVC 的默认设值路径具有自动 KVO 支持，即使回退到 ivar，也不能等同于源码里裸写 ivar；手动通知和关闭自动通知的配置仍需单独考虑。集合内部变化应使用 KVC 的可变集合代理、合规集合访问器或手动通知。[Apple KVO 合规规则](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueObserving/Articles/KVOCompliance.html)

**追问：Swift 中怎么观察？**传统 KVO 常用 `NSObject` 子类的 `@objc dynamic` 属性及 `NSKeyValueObservation` token。token 管理观察关系的有效期，不是负责销毁动态生成的类。回调一般发生在修改值的线程，更新 UI 要回到主线程。新式 Swift Observation 与 KVO 是不同机制，不能把 `@Observable` 理解成 KVO 子类化。

**易错点：**是否收到变更前回调与观察选项有关，不是每次 setter 都给普通观察者各发一次“前、后”回调；旧式 KVO 的移除要求也不能脱离 API 形式和系统版本一概而论。

### 08 Method Swizzling 适合做什么 风险在哪里

**回答：**Swizzling 通过更换方法与实现的对应关系拦截调用，适合受控的埋点、兼容修复或调试基础设施。实施前要确认目标方法实际归属、签名一致、交换只执行一次，以及原实现调用链不会丢失。

如果目标方法继承自父类，直接交换取得的 `Method` 可能影响父类及其他子类。应先明确是否要在目标类添加本地实现，再决定替换或交换。类方法操作元类；多个 SDK 同时 Hook 时，安装顺序和各方是否正确调用前一实现会影响结果。

**追问：为什么不做全局防崩溃兜底？**统一吞掉越界、未识别方法等异常可能把明确故障变成静默的数据错误，还会让现场消失。更合理的是在明确边界做校验和有监控的降级，修复根因。不能承诺 Hook 后所有调用都能拦截：缓存的 IMP、直接调用、Swift 优化后的调用路径均可能绕过它。

本章原笔记：[Runtime 消息机制](</Users/bytedance/Desktop/BlogNotes/iOS/Runtime/Runtime消息机制.md>)、[isa 相关](</Users/bytedance/Desktop/BlogNotes/iOS/Runtime/isa相关.md>)、[Category 和 Extension](</Users/bytedance/Desktop/BlogNotes/OC/Category和Extension.md>)、[KVC](</Users/bytedance/Desktop/BlogNotes/iOS/系统特性/KVC的本质.md>)、[KVO](</Users/bytedance/Desktop/BlogNotes/iOS/系统特性/KVO本质.md>)。

## 二 内存管理

### 09 ARC 做了什么 为什么仍会内存泄漏

**回答：**ARC 根据所有权规则自动安排引用计数相关操作，并配合 Runtime 执行；它不是定期扫描对象图的垃圾回收器。强引用表达所有权，最后一个强引用释放后，对象才能结束生命周期。两个对象互相强持有时，即使业务已经不需要它们，引用计数仍不归零。

要从完整持有链定位泄漏，例如 `VC → ViewModel → callback → VC`，或单例、缓存、通知 token、重复定时器长期持有页面。`deinit/dealloc` 不调用既可能是循环引用，也可能是预期外的长生命周期 owner。

**追问：离开大括号时一定释放吗？**不能只看源码作用域。最后使用点、编译优化、其他强引用及 autorelease 都影响实际释放时机；需要强制延长 Swift 对象有效期时，用 `withExtendedLifetime` 等明确表达。

**纠错：**“非 alloc/new/copy/mutableCopy 返回的对象一定入 autorelease pool”不成立。它们主要表达返回值所有权约定，编译器和 Runtime 可以省略配对操作。[Clang ARC 规范](https://clang.llvm.org/docs/AutomaticReferenceCounting.html)

### 10 weak 如何自动置 nil 与 assign unowned 有何区别

**回答：**Objective-C weak 的 Runtime 管理会记录对象对应的弱引用存储位置，并协调赋值、读取和对象析构；对象销毁时清除这些弱引用。因此 weak 不拥有对象，也不会在对象释放后继续暴露悬空对象指针。

`assign` 适合标量。对对象使用 assign 或 `__unsafe_unretained`，不会自动清零，可能留下悬空指针。Swift `weak` 常为可选引用；`unowned` 不保活对象，要求使用时对象仍然有效，安全的 `unowned` 在违反生命周期约定时会触发运行时错误，不能把它与 `unowned(unsafe)` 混为一谈。

**追问：weak 是线程安全的吗？**weak 读取与对象最终释放之间有必要协调，但这不保证对象内部状态、多个属性或“读后再做某事”的业务逻辑线程安全。需要稳定使用时，把 weak 读取成一次局部强引用；还要为共享状态单独建立同步规则。

### 11 strong copy atomic 各自解决什么问题

**回答：**`strong` 保持对象生命周期，`copy` 根据对象的拷贝契约取得快照，`atomic` 主要保证合成访问器的单次存取完整性。三者解决的不是同一个问题。

字符串等希望对外保持不变的属性常用 copy，避免传入 `NSMutableString` 后外部修改影响内部状态。可变容器属性不能机械用 copy，因为得到的可能是不可变对象；应明确使用 strong 持有可变对象，还是由接口显式创建独立的可变副本。

**追问：容器 copy 是深拷贝吗？**普通容器拷贝通常不递归复制内部对象。新数组和旧数组可以共享同一个元素引用；深拷贝要定义对象图、共享关系和循环引用如何处理。

**追问：atomic 能保证 `count++` 吗？**不能。读、加一、写回是复合操作；两个 atomic 属性之间的一致性也不受保护。必须把整段不变量维护放到同一同步边界中。`@synchronized` 在统一使用同一个锁对象时能保护临界区，但不能保护绕过这把锁的访问。

### 12 Block 捕获了什么 __block 为什么能修改外部变量

**回答：**Block 可以理解为“函数入口 + 捕获上下文”。普通自动局部变量通常按创建 Block 时的值捕获；对象变量捕获的是对象引用，因此不能重新给捕获变量赋值，但仍可以修改它指向的可变对象。全局变量和静态存储期变量不需要通过复制局部值来延长生命周期。

`__block` 把局部变量改为可共享的 byref 存储。Block 逃逸并需要堆存储时，byref 存储也可迁移，通过 forwarding 机制让 Block 内外继续访问同一份变量。这个辅助存储不应简单说成普通 NSObject 对象。

```objc
int a = 1;
__block int b = 1;
void (^f)(void) = ^{ NSLog(@"%d %d", a, b); };
a = 2;
b = 2;
f(); // 1 2
```

**追问：三类 Block？**传统实现可区分全局、栈、堆 Block，捕获及逃逸决定是否需要持久上下文。ARC 会对需要保活的 Block 进行适当管理；API 是否复制参数应看契约，不能仅凭名称中带 `usingBlock` 就推断所有权。

### 13 Block 循环引用如何判断 weak strong dance 有什么用

**回答：**只有存在强引用闭环才是循环引用。`self` 拥有 Block，Block 又强捕获 `self` 是典型闭环；交给队列的短期 Block 强捕获 self 可能只是延长生命周期，并不自动构成永久泄漏。

```objc
__weak typeof(self) weakSelf = self;
self.completion = ^{
    __strong typeof(weakSelf) strongSelf = weakSelf;
    if (!strongSelf) return;
    [strongSelf finish];
};
```

weak 打断长期闭环；Block 执行时的局部 strong 让这次执行需要的对象保持有效。不要在 Block 外先创建 strongSelf 再捕获它，那仍是强捕获。

**追问：加 `__block` 就能解决吗？**ARC 下不能，`__block id` 默认仍具有强所有权。可以使用弱引用，或在可靠的任务终止路径清掉回调。只在“成功回调”里清理不够，失败、取消和永不回调都必须有生命周期设计。

### 14 AutoreleasePool 何时释放对象 为什么子线程也可能需要

**回答：**autorelease 相当于登记一次未来的 release；退出对应 pool 边界时执行这些 release，只有没有其他强所有权时对象才真正销毁。嵌套池只处理其边界内登记的对象，它不会主动清除外部容器里的强引用。

主线程 UIKit 事件循环有自动释放池管理，因此许多临时对象会在事件循环相关边界回收。但 **GCD 工作线程不是靠“每条线程都跑着 RunLoop”来回收**；队列的 autorelease 策略和任务执行边界需要单独理解。

```objc
for (NSURL *url in urls) {
    @autoreleasepool {
        // 创建、处理一批 Objective-C 临时对象
        process(url);
    }
}
```

**追问：为什么加了池内存还涨？**被集合、缓存、闭包强持有的结果仍不会释放；还可能是 C 内存、像素缓冲、映射文件或其他资源增长。释放池主要控制临时对象峰值，不能当作通用“清内存”按钮。

### 15 CF 桥接和 Tagged Pointer 有什么面试重点

**回答：**CF 与 Objective-C 交互首先要回答谁拥有对象。`__bridge` 只转换表示，不新增或移交所有权；`__bridge_retained` / `CFBridgingRetain` 取得供 CF 一侧负责平衡的所有权；`__bridge_transfer` / `CFBridgingRelease` 将已有的 CF 所有权交给 ARC。只有支持相应桥接的类型才能按文档转换。

Swift `Unmanaged` 也要求区分已有的 retained / unretained 返回约定，不能靠试到“不崩溃”为止选择 `takeRetainedValue`。Create/Copy 规则与 API 注解是判断依据。

**追问：Tagged Pointer 是什么？**某些小对象把值及类型信息编码进指针表示，避免普通堆对象分配。是否采用这种表示由平台、类型和值决定，业务不应依赖固定的高低位或字符串长度阈值。

**纠错：**Tagged Pointer 不会绕过属性 setter；`obj.name = value` 仍然遵循属性调用语义。小字符串并发赋值暂时不崩，不代表共享属性已经线程安全。

本章原笔记：[属性和关键字](</Users/bytedance/Desktop/BlogNotes/OC/属性和关键字的理解.md>)、[Block 解析](</Users/bytedance/Desktop/BlogNotes/OC/Block解析.md>)、[iOS 内存管理机制](</Users/bytedance/Desktop/BlogNotes/iOS/内存管理/iOS内存管理机制.md>)、[常见内存问题](</Users/bytedance/Desktop/BlogNotes/iOS/内存管理/常见内存问题.md>)。

## 三 RunLoop 与 GCD

### 16 RunLoop 是什么 为什么它不会一直占 CPU

**回答：**RunLoop 是线程处理事件的循环机制，管理输入源、定时器和状态观察者。可将过程概括为：处理就绪事件，准备休眠，等待内核事件或超时唤醒，再继续处理。空闲时线程可以阻塞等待，不是 while 空转，因此无需持续消耗 CPU。

主线程由框架维护事件循环；其他线程可以按需获取并运行自己的 RunLoop，但“存在线程”不等于“它已运行 RunLoop”。GCD 队列也不是 RunLoop。

**追问：Source0、Source1、Observer？**Source0 是应用主动标记待处理的源，需要配合唤醒；Source1 与端口等事件驱动来源关联。Timer 在合适时机处理，Observer 观察 Entry、BeforeTimers、BeforeSources、BeforeWaiting、AfterWaiting、Exit 等阶段。

**易错点：**Observer 是观察者，不是让循环保持运行的输入源。仅添加 Observer 不足以维持一个空 RunLoop；需要有效的输入源或定时器等运行条件。[Apple Run Loops](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html)

### 17 为什么滑动时 Timer 不触发 怎样实现可靠计时

**回答：**RunLoop 每次运行在一个具体 mode 中。Timer 若只注册在 default mode，滚动跟踪期间切换 mode 后，它就可能暂不处理。将 Timer 添加到 common modes，意味着让它参与标记为 common 的多个具体 mode；common 本身不是实际运行的 mode。

这只能解决 mode 不匹配，不能解决主线程忙、系统调度和合并唤醒造成的延迟。DispatchSourceTimer 不依赖 RunLoop，但也会受执行队列阻塞、调度和 leeway 影响，不是实时定时器。

**追问：倒计时怎么写？**保存时间基准，每次回调根据“现在到截止时刻的差”重算剩余时间，而不是每回调一次就减一。测量经过时长优先单调时钟；活动结束时间若来自服务端，应按业务校准后的绝对时间计算。动画刷新使用 CADisplayLink，并依据时间戳推进状态，不能假设永远 60 Hz。

### 18 什么时候需要常驻线程 怎样正确停止

**回答：**只有确实需要固定线程亲和性、端口事件处理或依赖 RunLoop 的 API 时，才考虑专门线程。普通后台任务用队列或 Swift 并发即可；串行队列只保证执行顺序，不保证每次是同一条线程。

常驻线程需要在自己的入口设置 autorelease pool、注册有效事件源，再运行循环。外部提交工作时应通过线程支持的事件源或调度机制传入，保证执行发生在该线程。

**追问：`CFRunLoopStop` 就够了吗？**它停止当前 RunLoop 运行调用，不会强杀线程。如果外层还有 `while` 或重复 `run`，需要同时设置停止条件，并使循环及时被唤醒和退出，随后撤销源、计时器并释放资源。`Thread.cancel()` 也是协作式标记，入口不检查就不会自动终止。

### 19 sync async 串行 并发到底是什么关系

**回答：**两组概念分别描述“调用者何时继续”和“队列允许怎样执行任务”。`sync` 等待提交的工作完成；`async` 提交后返回。串行队列保证任务不重叠执行，并发队列允许多个任务重叠，具体是否并行受资源和调度影响。

| 组合 | 行为 |
| --- | --- |
| 串行 + async | 调用者不等待，该队列任务依次执行 |
| 串行 + sync | 调用者等待，仍受串行约束 |
| 并发 + async | 允许重叠执行，完成次序不保证 |
| 并发 + sync | 当前调用者等待；其他提交者仍可能有并行工作 |

普通串行队列不绑定固定线程；`async` 不承诺创建新线程；`sync` 可以优化为在当前线程执行，但不能把它作为所有队列的线程保证。主队列的任务在主线程执行。[Apple DispatchQueue](https://developer.apple.com/documentation/dispatch/dispatchqueue/)

**追问：异步串行是不是“只开一条线程”？**它的契约是同一时刻只执行一个工作项，不是承诺整个生命周期只有同一个工作线程。队列用于组织任务，线程是执行资源。

### 20 怎样判断 GCD 死锁 主线程和主队列有何区别

**回答：**画等待关系，检查是否形成无法解除的环。典型情况是串行队列正在执行任务 A，A 同步提交 B 到同一个队列；B 必须等 A 完成才可执行，而 A 又等待 B 完成。

```swift
let queue = DispatchQueue(label: "serial")
queue.async {
    print("A")
    queue.sync { print("B") }
    print("C")
}
```

上面的 B、C 无法正常到达；libdispatch 可能检测到错误并直接终止，不应保证一定“只卡住不崩”。主线程同步派发到主队列也是典型错误。

**追问：当前不在主线程就安全？**仍可能有跨队列环。例如主线程等待后台任务，后台任务再同步等待主队列。锁、semaphore、group.wait 也可能参与等待环。

工程上优先异步依赖，避免持锁回调外部代码，规定统一锁顺序。判断队列身份可使用 queue-specific 数据或 dispatch precondition；判断线程身份不能替代队列隔离判断。

### 21 锁 信号量 barrier 应如何选择

**回答：**先确定要保护的状态和不变量，再选同步手段。短小同步临界区可用 NSLock、合适的底层锁或串行隔离；确实存在同线程重入才考虑递归锁。信号量适合数量有限的资源准入，不应该为了“异步转同步”在主线程等待网络。

读多写少可考虑读写锁或私有并发队列加 barrier，但不一定比简单锁快。barrier 只对同一私有并发队列上的工作建立独占边界；放到全局队列不具备这种 barrier 语义。所有访问必须经过同一套保护，异步 barrier 写入返回也不等于写操作已完成。[Apple barrier 说明](https://developer.apple.com/documentation/dispatch/dispatch_barrier_async)

**追问：自旋锁为什么不推荐直接用 OSSpinLock？**等待者忙等消耗 CPU，优先级反转还可能让持锁线程迟迟得不到执行机会。不要背一张“某锁永远最快”的排行榜，应评估竞争程度、临界区长度、调度和能耗。

**追问：锁内能 `await` 吗？**不应把线程锁跨异步挂起持有，恢复可能发生在不同线程，也可能阻碍其他任务推进。异步场景应使用与隔离和任务模型相容的设计。

### 22 DispatchGroup 和 OperationQueue 能解决什么

**回答：**Group 汇合一组工作的完成事件；OperationQueue 进一步提供操作对象、依赖关系、并发上限和可观察状态。简单汇合可以用 Group，复杂任务依赖和取消状态管理可考虑 Operation；现代异步代码也可以用 TaskGroup。

**常见坑：**把“发起网络请求”的 Block 放入 group，并不意味着 group 会等网络回调。Block 结束时该工作项已完成，真实异步生命周期需要 `enter/leave` 配对。失败、取消、超时路径也必须恰好 leave 一次，UI 汇合优先 `notify`，避免主线程 `wait`。

**追问：Operation 取消、暂停会马上停吗？**取消是协作式的，要检查 `isCancelled` 并取消底层工作；暂停队列阻止新的操作开始，不会冻结正在执行的操作。自定义异步 Operation 必须正确维护 `isExecuting/isFinished` 及 KVO 状态，依赖只有在前置操作完成后才能继续。依赖 A 完成不等于 A 成功，错误传播要自行定义。

### 23 条件变量和线程安全容器有哪些容易漏掉的点

**回答：**条件变量用于等待共享状态满足条件，必须和保护该状态的锁配合。wait 会原子地释放锁并进入等待，返回前重新取得锁；唤醒只表示应重新检查，不代表条件一定满足，因此用 `while` 而不是 `if`。

```text
lock()
while buffer.isEmpty && !stopped:
    condition.wait()   // 释放锁等待，返回时重新持锁
if stopped:
    unlock()
    return
item = buffer.removeFirst()
unlock()
```

**追问：每个字典方法都加锁就安全吗？**单次方法安全不保证组合事务安全。`if cache[key] == nil { cache[key] = make() }` 的检查与写入仍会竞争。应提供一个完整操作，或维护同 key 的 in-flight 任务实现请求合并。避免锁内执行外部闭包或耗时构造；可以先登记占位状态，再解锁执行，最后提交结果并唤醒等待者。

本章原笔记：[RunLoop 总结](</Users/bytedance/Desktop/BlogNotes/iOS/Runloop/Runloop 总结.md>)、[GCD 全面详解](</Users/bytedance/Desktop/BlogNotes/iOS/多线程/GCD全面详解.md>)、[GCD 的死锁](</Users/bytedance/Desktop/BlogNotes/iOS/多线程/GCD的死锁.md>)、[同步和锁](</Users/bytedance/Desktop/BlogNotes/iOS/多线程/iOS多线程同步、锁和文件读写方案.md>)、[OperationQueue](</Users/bytedance/Desktop/BlogNotes/iOS/多线程/OperationQueue.md>)。

## 四 Swift 语言机制

### 24 struct 和 class 怎么选 COW 如何保证值语义

**回答：**值类型的赋值和传递表达独立的值，引用类型的赋值可以让多个变量共享同一个对象身份。建模时先问“共享身份是否有意义”：页面协调器、连接、可共享服务常适合 class；配置、快照、坐标、展示模型常适合 struct。

这不是堆栈划分。值可能位于栈、对象内部、闭包捕获存储或被优化到寄存器；值类型也可以包含指向堆存储的引用。结构体包含 class 属性时，复制结构体不会递归复制该对象。

Array 等集合使用写时复制：多个值可暂时共享底层存储，在需要修改且存储不唯一时复制，以维持外部可见的值语义。自定义 COW 常用私有引用存储，并在写入口用 `isKnownUniquelyReferenced` 判断是否需复制。

**追问：COW 是线程安全机制吗？**不是。多个线程并发修改同一个变量仍需要同步。若 COW 容器把内部可变 class 暴露出去，调用方可以绕开写入口直接修改它，此时也不能保证深层值语义。

### 25 Swift 方法有哪些派发方式 协议扩展为什么容易踩坑

**回答：**可从直接调用、类虚表、协议见证表和 Objective-C 消息派发来理解。类的可重写方法通常可经 vtable；协议要求可经 witness table；已知具体类型或编译器能确定目标时，可以直接派发或去虚拟化。不要把 vtable 和 witness table 统称为同一张表。

```swift
protocol P { }
extension P {
    func name() -> String { "P" }
}
struct S: P {
    func name() -> String { "S" }
}
let concrete = S()
let erased: any P = concrete
print(concrete.name()) // S
print(erased.name())   // P
```

这里 `name` 没有声明为协议要求，经协议类型调用时选择扩展方法。把 `func name() -> String` 放入 P 的要求后，S 的实现才参与该要求的协议派发。

**追问：`@objc`、`dynamic`、`final`？**`@objc` 暴露 Objective-C 入口，不表示所有 Swift 调用都强制走消息发送；与 Objective-C 动态派发相关的场景通常用 `@objc dynamic`；`final` 禁止继承或重写，帮助编译器确定调用目标。是否实际内联仍由优化决定，不能用“加一个关键字必然更快”代替测量。

### 26 Swift 闭包默认捕获和捕获列表有什么区别

**回答：**闭包携带执行所需的环境。对可变局部变量的隐式捕获可以共享其存储，因此之后修改变量，闭包可能看到更新后的值。捕获列表则在闭包创建时求值并建立捕获项；值类型形成当时的值，class 仍可能只是同一对象的另一个引用。

```swift
func captureExample() {
    var number = 1
    let live = { print(number) }
    let snapshot = { [number] in print(number) }
    number = 2
    live()     // 2
    snapshot() // 1
}
```

`@escaping` 表示函数返回后闭包仍可能被调用，不等于“必然异步”，也不等于“必须强持有 self”。显式 self 有助于看见捕获关系，实际所有权可通过 `[weak self]` 等控制。

**追问：为什么 struct 的实例方法返回闭包后看不到外部变量新值？**方法中的 self 是本次调用的值；闭包捕获该 self，不是自动追踪调用处那个可变变量的后续赋值。需要区分“捕获一个变量的共享存储”和“捕获一次方法调用中的值”。

### 27 泛型 some 和 any 有什么区别

**回答：**泛型以类型参数表达“调用方选择的具体类型”，编译器保留类型关系，可以特化；`some P` 隐藏一个由实现选择的固定具体类型，调用方不知道名字，但类型身份及关系仍然保留；`any P` 是存在类型，可容纳不同遵循 P 的具体类型，适合运行时异构存储。

```swift
func process<T: P>(_ value: T) { /* 保留 T 的类型关系 */ }
func makeValue() -> some P { S() } // 对该实现是固定具体类型
var values: [any P] = [S()]       // 可容纳不同的 P 实现
```

**追问：`any` 为什么可能慢？**可能涉及存在容器、间接访问、见证表及大值装箱，也可能因类型信息减少而难以优化；不是每个 any 都必然堆分配。泛型也不保证永远零成本，跨模块、未特化代码和代码体积增长都要考虑。

设计 API 时优先表达真实需求：需要异构集合就用存在类型，需要保持入参与返回值类型关系就用泛型。不要为了消灭 `any` 把整个架构变成难以维护的泛型链。

### 28 inout lazy 和 static let 分别有哪些陷阱

**回答：**`inout` 的语义是读入、修改、写回，编译器可在符合语义时优化为地址传递。因此计算属性也可作为 inout 实参，通过 getter/setter 实现效果；不能把它简单定义为 C 指针。独占访问规则会限制对同一存储的重叠读写。[Swift inout 语义](https://docs.swift.org/latest/documentation/the-swift-programming-language/declarations/)

实例 `lazy var` 推迟到首次使用初始化，但不保证多线程同时首次访问只初始化一次。对于 struct，读取 lazy 属性需要修改自身存储，所以不可变 struct 值有相应访问限制；class 引用为 let 则不代表它内部属性不可变。

类型存储属性的初始化有线程安全保证，常见单例写法是 `static let shared = Service()`。**这个保证只覆盖初始化，不覆盖 Service 之后的可变状态。**

**追问：内存对齐怎么看？**`MemoryLayout.size` 是值本身布局所需大小，`stride` 是连续存放同类型元素的步长，`alignment` 是对齐要求。对 class 类型测到的是引用表示的布局，不是对象及其引用成员的总内存占用。

### 29 Swift 集合和 String 有哪些性能与正确性问题

**回答：**Array 连续存储适合索引与遍历，尾部追加通常是摊还 O(1)，中间插入、删除或反复 `removeFirst()` 通常要移动元素。队列可用头索引或环形缓冲，避免把整段 BFS 写成 O(n²)。Dictionary/Set 查找通常平均 O(1)，依赖哈希分布，不能保证最坏情况也是常数。

`Hashable` 必须满足相等值具有相同哈希语义，参与 key 比较和哈希的状态不应在入表后被外部偷偷改变。结构体 key 的独立复制与包含共享引用的 key 要分清。

String 按扩展字形簇提供 Character 语义，字符不等于字节，不能拿 UTF-8 偏移直接当 String.Index。ASCII 限定题可用 utf8 加速，但要明确输入前提；通用文本按字符或合适编码处理。

**追问：小切片为什么可能占着大内存？**Substring、ArraySlice 可能共享原始存储。长期保存小片段时，可按需要创建独立 String/Array；ArraySlice 的索引也不保证从 0 开始。

### 30 Swift 与 Objective-C 混编要注意哪些边界

**回答：**先区分可表示性、所有权和派发。不是所有 Swift 类型都能直接导出给 Objective-C，例如复杂泛型、纯 Swift 协议等需要边界适配。Swift 调用 OC 会受到头文件 nullability、轻量泛型和错误注解质量影响，注解不准确可能把错误推迟成运行时崩溃。

跨语言调用可能有桥接 thunk、集合或字符串表示转换、装箱和额外引用计数操作，但不能说“每次桥接都完整复制”。热点路径应尽量保持稳定的数据表示，避免大集合反复来回桥接，并用 Instruments 定位真实成本。

**追问：加 `@objc` 后协议方法就自动可选吗？**不会。可选协议要求需要显式 `@objc optional`；Objective-C 协议方法默认也是 required。`@objc` 不会把所有 Swift 方法自动变成可 Hook 的消息派发。

**工程例子：**业务模型保持 Swift 类型，在 SDK 边界集中转换；错误转成稳定的领域错误；对历史 OC nullable 返回值显式处理，避免上层普遍强制解包。

本章原笔记：[Swift 派发](</Users/bytedance/Desktop/BlogNotes/Swift/底层/Swift多态和方法派发机制.md>)、[闭包](</Users/bytedance/Desktop/BlogNotes/Swift/底层/Swift闭包的本质.md>)、[写时复制](</Users/bytedance/Desktop/BlogNotes/Swift/进阶/Swift写时复制.md>)、[属性](</Users/bytedance/Desktop/BlogNotes/Swift/底层/Swift各种属性的本质.md>)、[内存布局](</Users/bytedance/Desktop/BlogNotes/Swift/底层/Swift的内存布局.md>)、[混编](</Users/bytedance/Desktop/BlogNotes/Swift/进阶/Swift和OC混编的坑.md>)。

## 五 Swift 并发补充

这部分用于补齐旧笔记，不要求背提案编号，重点是能解释隔离边界和真实竞态。

### 31 async await 是不是自动切到后台线程

**回答：**不是。`async` 表示函数可以挂起，`await` 标出可能挂起的位置；挂起时可以让出执行资源，但不等于启动新线程，也不保证每次都会实际挂起。在哪执行取决于 actor 隔离、执行器和函数声明等条件。

`@MainActor` 用于隔离 UI 状态，不能在其上执行长时间同步计算后以为包进 Task 就不会卡主线程。连续两个 `await` 通常表达顺序依赖；独立任务需要 `async let` 或任务组等方式组织并发。

**版本边界：**不能笼统背“nonisolated async 一律切后台”。Swift 6.2 引入 `nonisolated(nonsending)` 及 `@concurrent`，并通过 `NonisolatedNonsendingByDefault` 设置控制相关默认行为。判断旧项目时必须看工具链和构建配置。[SE-0461](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md)

**面试落点：**UI 状态明确 MainActor，重计算移到合适执行环境；异步等待本身不需要用阻塞线程来实现。

### 32 actor 能彻底避免竞态吗 什么是重入

**回答：**actor 保护其隔离状态免受同时读写的数据竞争，但异步方法跨越 `await` 时，其他任务可以进入 actor，导致先前判断失效。这是重入带来的业务逻辑竞态，actor 不会自动提供跨挂起点的事务。

```swift
actor Inventory {
    var stock = 1

    // 示意代码，存在逻辑竞态
    func buy(pay: @Sendable () async -> Bool) async -> Bool {
        guard stock > 0 else { return false }
        guard await pay() else { return false }
        stock -= 1
        return true
    }
}
```

两个调用都可能在支付前看到 stock 为 1，随后分别扣减。可在同步隔离片段内先预占库存，再异步支付，失败时回滚；还要定义超时、取消和重复回调的补偿规则。仅把检查移到支付后，可能变成已扣款却无库存。

**追问：actor 会严格 FIFO 吗？**不要依赖多个独立任务的进入顺序。需要业务顺序时，显式维护序号、状态机或任务链。actor 也不保证拥有专属线程。[Swift Actors 提案评审](https://forums.swift.org/t/accepted-with-modification-se-0306-actors/47662)

### 33 Task async let TaskGroup 和 detached 怎么选

**回答：**固定数量的独立子任务可用 `async let`，动态数量的并发任务可用 TaskGroup。它们属于结构化并发，子任务生命周期受作用域约束，作用域退出前要完成相应收尾；结构化子任务能接收父任务取消，但仍须协作响应。

`Task {}` 创建非结构化任务，在相应上下文中可以继承 actor 隔离、优先级和 task-local 值；它的生命周期不会因为创建它的函数返回就自动结束。`Task.detached` 不继承调用处的 actor 隔离和 task-local 值，不应当作默认后台入口。

**追问：批量 1000 张图片全部 addTask 可以吗？**不宜无界创建昂贵工作。设置并发窗口，先加入有限任务，每完成一个再补一个；根据内存和解码成本选择上限。TaskGroup 按完成顺序产出结果，想保持输入顺序应随结果携带索引。

**易错点：**在 throwing group 中，错误只有被观察并向外传播时才会触发相应退出和取消流程，不要假设任意子任务一失败，其他工作立即强制停止。[Swift 结构化并发](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)

### 34 Sendable 表示什么 Swift 6 并发检查怎样理解

**回答：**Sendable 表达值能够安全跨隔离域传递或共享的语义。不可变且成员满足条件的值类型容易符合要求；包含不受保护的共享可变引用时，即使外面套 struct，也不能据此断言安全。actor 可以通过隔离来保护状态。

`@Sendable` 约束闭包跨域使用的安全性及捕获；`@unchecked Sendable` 是开发者承诺自己维持安全，不会自动生成锁。应写清锁保护哪些字段、所有访问是否都遵守，以及是否把内部可变对象泄露出去了。

**追问：编译通过就没有并发 bug 了吗？**严格检查主要防数据竞争，不保证业务顺序、不保证 actor 跨 await 的检查仍有效，也不能兜住 unchecked、unsafe 和所有跨语言边界。迁移应先界定 UI、服务、缓存的状态归属，再修复跨域传值，不应到处加 unchecked 消掉诊断。

较新的隔离分析也可允许编译器证明安全的非 Sendable 值转移，所以“跨域的任何值都必须声明 Sendable”过于绝对。[Sendable](https://docs.swift.org/latest/documentation/swift/sendable/)、[Swift 数据竞争安全](https://www.swift.org/migration/documentation/swift-6-concurrency-migration-guide/dataracesafety/)

### 35 任务取消和 continuation 有哪些必须保证的事

**回答：**取消是协作式信号。耗时循环应检查 `Task.isCancelled` 或 `Task.checkCancellation()`；桥接旧请求时还要取消底层工作。取消后的结果也可能迟到，因此更新 UI 前仍需确认任务身份或请求版本。

把回调 API 转为 async 时，checked continuation 必须在全部有效终止路径上**恰好恢复一次**。成功与失败、取消与回调可能竞争；要用同步保护的状态机认领“完成权”。Checked 版本有诊断能力，但不能自动补齐遗漏的恢复或替你解决竞争。[Swift Continuation 规范](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0300-continuation.md)

**追问：能否用 semaphore 等待异步结果？**不应在协作执行器上阻塞等待同一体系里的任务，这可能让有限执行线程全部被占住，导致无法推进。应使用异步组合和挂起。

**页面任务的生命周期：**由谁创建、何时取消、取消是否传到底层、回调是否仍可写状态，要分别回答。尤其是无限 AsyncSequence：若先把 weak self 提升为强引用，再进入永不结束的循环，就可能长期保活页面，仅在 deinit 中取消会形成生命周期困局。

## 六 UIKit 与渲染

### 36 setNeedsLayout layoutIfNeeded setNeedsDisplay 有何区别

**回答：**布局决定几何位置和尺寸，绘制生成视觉内容，它们是不同阶段。`setNeedsLayout()` 标记布局失效，系统可以合并后续布局；`layoutIfNeeded()` 在需要时立即完成相关子树的布局。`setNeedsDisplay()` 标记需要重绘，不保证立刻调用 `draw`，也不保证“下一轮 RunLoop 必然只调用一次”。

改 Auto Layout 约束后，需要求解约束并布局；约束动画常在更新约束前布局到旧状态，再在动画块里对合适的共同祖先调用 `layoutIfNeeded()`。不要直接调用 `layoutSubviews` 或 `draw` 来模拟系统流程。

**追问：Hugging 和 Compression Resistance？**前者抵抗被拉大，后者抵抗被压小。两个 label 水平排布时，剩余空间优先给 hugging 较低的一方；空间不足时优先压缩 compression resistance 较低的一方。约束冲突与缺约束分别代表无可满足解和解不唯一，修复方法不同。

**纠错：**`layoutIfNeeded` 不等于立刻 `drawRect`。应在布局生命周期处理几何、在绘制入口只做绘制，避免布局中重复无条件改约束造成循环。

### 37 点击事件怎样找到 View 手势和响应链是什么关系

**回答：**命中测试从窗口沿视图树寻找适合接收触摸的最深层视图。标准 UIView 逻辑先检查交互、隐藏、透明度和 `pointInside`，再按逆序检查子视图，进行坐标转换；命中子视图就返回，否则可能返回自身。

响应链描述事件后续可交由谁处理，与“从父到子找命中 View”不是同一个方向或概念。手势识别器观察关联触摸并维护状态机，可通过失败依赖、同时识别策略以及 touches 的延迟、取消配置影响 View 接收事件，不能简化为“永远由手势先消费完”。

**追问：怎样扩大按钮点击范围？**可重写 `point(inside:with:)` 扩大有效区域，但若祖先先拒绝命中，按钮自己的扩区逻辑根本不会被访问。超出父 bounds 的可见内容也不意味着默认一定能点到，要处理父级命中规则。

**追问：透明蒙层怎样只拦截按钮？**可以在 hitTest 结果为蒙层自身时返回 nil，让下方视图参与命中；如果命中其按钮则保留结果。不要一律返回 nil，导致整个子树失去交互。

### 38 UIView 与 CALayer 如何分工 动画结束为什么会跳回

**回答：**UIView 负责交互、布局、视图生命周期等 UIKit 行为，CALayer 负责可合成的视觉属性与内容。UIView 的 backing layer 通常由 View 管理其代理关系，不应随意换掉该代理来实现异步绘制。

Core Animation 中，model layer 保存业务设置的目标状态，presentation layer 反映当前呈现中的插值状态。给 layer 添加显式动画，通常不会自动把 model 值修改为动画终值，所以动画移除后可能回到原状态。

**正确做法：**在合适事务中更新 model 最终值，并添加从当前 presentation 状态到目标状态的动画；需要时禁用更新 model 引起的隐式动画。用 `fillMode` 加“不移除动画”只维持视觉效果，不等于 model 已正确更新。

**追问：动画中的点击或中断怎么处理？**需要视觉位置时参考 presentation 状态并正确转换坐标；中断后从当前呈现值继续，避免从旧 model 起点跳变。直接操作独立 layer 时也要留意隐式动画，与 UIView 管理的动画行为区分。

### 39 列表复用为什么出现图片错位 怎样保证数据一致

**回答：**Cell 是可复用的容器，不是某条数据的永久身份。旧请求完成时，Cell 可能已经展示另一条数据，因此不能只捕获 Cell 或旧 indexPath 就设置图片。

```text
+ 配置 Cell 时记录 stableID 和本次请求 token
+ 回收或重新配置时取消旧订阅，重置可见状态
+ 回调时校验 stableID 与 token 都仍匹配
+ 只在主线程提交 UI 更新
```

取消只是减少浪费，**身份校验才是防迟到结果污染的最后一道约束**。`prepareForReuse` 不是唯一设置状态的地方，每次 configure 都应把该显示和该隐藏的状态完整赋值。

**追问：异步 diff 怎么防新旧快照覆盖？**后台基于不可变快照计算，结果带版本号；主线程提交前确认基础版本仍有效，否则丢弃或重算。不要在后台直接改 UI 正在读取的可变数组。

动态高度缓存应包含数据身份、内容版本、可用宽度、字体或内容大小类别等失效条件，不能只按 indexPath 缓存。预取、取消和局部刷新也必须跟数据身份关联。

### 40 ViewController 生命周期与容器有哪些重点

**回答：**`loadView` 负责建立根 View，`viewDidLoad` 适合一次性视图设置；出现、消失回调表达展示变化，不等于对象创建和销毁。页面多次出现，不应重复注册通知、创建永不结束的任务或重复添加子视图。

自定义容器除了 `addSubview`，还需正确建立父子 VC 关系：添加时 `addChild`，放入子 View、布局，再 `didMove(toParent:)`；移除时先 `willMove(toParent: nil)`，移除 View，再 `removeFromParent()`。否则外观、旋转、safe area 等行为可能异常。

**追问：`viewDidDisappear` 就能释放所有业务资源吗？**不一定，可能是临时覆盖、切换 Tab 或可取消的转场。应按资源需求区分“不可见时暂停”和“业务真正结束时销毁”，对交互式转场取消也要考虑状态恢复。

**追问：SwiftUI 的刷新是否也等于对象重建？**这是独立专题；声明式 View 描述的重复计算与状态身份不同。现有笔记主要是 UIKit，若岗位强调 SwiftUI，需额外准备状态所有权、identity 和 Observation，不宜用 VC 生命周期生搬硬套。

### 41 从修改界面到显示一帧 中间发生了什么

**回答：**App 侧处理事件、更新状态、计算布局，必要时绘制内容，随后提交 Core Animation 事务；渲染服务准备并执行合成，GPU 产生可显示结果，显示系统在对应时机呈现。各阶段形成流水线，不能把整个流程说成“CPU 画完所有像素后 GPU 只是贴上去”。

在 60 Hz 下刷新周期约 16.67 ms，120 Hz 下约 8.33 ms，但这不是 App 主线程可以独占的完整预算。屏幕可变刷新率、流水线和呈现 deadline 都影响实际约束。

**追问：异步绘制为什么有用？**将可安全后台执行的文本排版、图片处理和独立位图绘制移出主线程，再在主线程设置结果。必须使用不可变输入快照，处理任务取消和结果过期，且不能在后台任意操作 UIKit 视图树。

**易错点：**Core Animation 的具体内部渲染后端、缓冲数量不是业务 API 保证；旧笔记里的“固定 OpenGL ES、固定双缓冲、固定 60 FPS”不能直接套到所有现代设备。[Apple 渲染循环与卡顿](https://developer.apple.com/la/videos/play/tech-talks/10855/)

本章原笔记：[Auto Layout](</Users/bytedance/Desktop/BlogNotes/iOS/UI相关/AutoLayout.md>)、[事件和手势](</Users/bytedance/Desktop/BlogNotes/iOS/UI相关/事件传递响应、手势机制.md>)、[TableView](</Users/bytedance/Desktop/BlogNotes/iOS/UI相关/TableView相关.md>)、[绘制](</Users/bytedance/Desktop/BlogNotes/iOS/UI相关/iOS绘制相关.md>)、[UI 常见问题](</Users/bytedance/Desktop/BlogNotes/iOS/UI相关/UI常见问题记录.md>)。

## 七 性能与稳定性

### 42 一个列表滑动卡顿 你如何定位和优化

**回答：**先固定可复现场景、设备档位、系统、数据量与构建配置，明确是交互无响应、App 提交超时还是渲染阶段 hitch，再用相应工具取证。不能上来就把所有工作派到后台或全部改手写 frame。

App 侧重点看主线程长任务、图片解码、重复布局、文本计算、同步 I/O、锁等待和瞬时对象分配；渲染侧看复杂阴影、mask、模糊、过度混合、大纹理及不必要的中间渲染。Time Profiler 用于定位 CPU 调用栈；线程状态分析帮助区分“在算”和“在等”；Animation Hitches 等工具帮助区分提交与渲染瓶颈。

**追问：离屏渲染怎么优化？**它是先渲染到中间目标再合成，会增加 pass、带宽和资源开销。阴影可提供正确的 `shadowPath`；减少不必要 mask、模糊和大面积混合；静态复杂内容可评估栅格缓存，但频繁变化会导致缓存重建。圆角是否触发额外 pass 与具体组合和实现有关，不是设置 cornerRadius 就一定昂贵。[Apple 离屏渲染诊断](https://developer.apple.com/videos/play/tech-talks/10857/?time=1)

**面试结尾应说明：**优化哪条热点、代价是什么，在相同条件下 hitch、交互延迟、内存和耗电如何变化。平均 FPS 不能完整代表用户感知。

### 43 启动优化如何拆解 为什么只看 main 之前不够

**回答：**先定义启动终点：首帧、首屏内容展示和首次可交互不是同一个指标。区分新进程启动与进程仍存活的前台恢复，并记录缓存、首次安装、登录态和页面恢复条件，避免把不同场景混为一组。

启动关键路径大体包括镜像映射和必要的链接修复、运行时准备、初始化器，以及 main 之后的应用与场景初始化、首屏数据和视图构造。现代 dyld 有预构建数据和多种优化，旧版 `Load → Rebase → Bind` 可作为概念理解，不应当成每版系统逐步执行的固定日志。

**方案选择：**减少非必要启动依赖和静态初始化；把任务分成首屏必需、首屏后、按需三类；建立依赖关系，删除重复任务，控制并发，避免后台任务争抢主线程所需 CPU/I/O。启动时必须完成的数据迁移则应优化本身，不能简单推迟到用户点击时卡住。

**追问：二进制重排为什么可能有效？**让启动使用的代码更集中，提高页面局部性，可能减少相关缺页和读取成本。但收益受采样覆盖、代码布局、缓存、系统及设备影响，需要同场景 A/B 验证，不能保证固定比例收益。首帧后的长任务也要监控，避免“指标快了、用户仍点不动”。[Apple 启动优化](https://developer.apple.com/documentation/xcode/reducing-your-app-s-launch-time)

### 44 内存一直上涨 怎样区分泄漏 缓存和正常峰值

**回答：**先看增长是否随相同操作重复累积、离开页面后是否回落、是否达到稳定平台，再拆分对象分配、图片和图形资源、映射、缓存及其他 VM 区域。仅看一条内存曲线无法判断循环引用。

具体做法是反复进入退出同一页面，用 Memory Graph 查持有路径，用 Allocations 的时间区间或代际标记定位持续增加的分配，再核对缓存淘汰和任务取消。Leaks 能发现部分不可达分配，但发现不了所有“仍被合法引用、业务却已无用”的对象。

**追问：对象释放了，进程内存为什么没立刻降？**分配器可能保留页供后续复用，缓存和碎片也会影响观测值；虚拟地址大小、驻留内存和系统计费的 footprint 不是同一概念。比较时要统一指标。

**优化顺序：**修复无用持有，限制缓存成本和解码并发，按展示尺寸降采样，分批处理临时对象，再评估数据结构和内存局部性。不要依赖一个固定的“iOS 超过多少 MB 一定被杀”阈值，也不能保证收到内存警告后才发生系统终止。

### 45 卡顿 崩溃 OOM 应怎样分别监控

**回答：**卡顿可结合主线程忙状态持续时间、响应探针和多次堆栈采样；监控 RunLoop 时要排除正常睡眠和应用挂起。不能把 `BeforeWaiting → AfterWaiting` 的睡眠时间当作主线程忙，也不能用主线程 RunLoop 指标覆盖所有 GPU/render hitch。

崩溃应分类处理：Objective-C 异常、无效内存访问、Swift trap、断言等。符号化需要与发布二进制 UUID 对应的 dSYM，结合异常类型、故障线程、其他线程、版本和业务上下文定位。坏内存访问可用 Zombies、Address Sanitizer 等辅助复现；数据竞争使用 Thread Sanitizer，但工具未发现不等于不存在。

**追问：signal handler 里上传日志行不行？**不行。崩溃现场可能已经持锁或内存损坏，不能随意调用 malloc、Objective-C、复杂日志和网络。应使用经过验证的崩溃采集方案，在受限安全路径保存最小现场，下次启动再处理。

**追问：OOM、Watchdog 都能捕获吗？**不能依赖进程内 handler 捕获系统强制终止。结合系统诊断、MetricKit、退出原因和内存趋势建立证据；“上次没正常退出”也可能是用户强退等，不能直接归为 OOM。Watchdog 条件随场景和系统变化，不应背固定秒数作为产品契约。

### 46 怎样设计一个图片加载系统

**回答：**把链路拆成请求身份、缓存查询、下载、解码变换、显示提交，各层分别定义取消、并发和成本。基本流程为内存缓存命中就复用，否则查磁盘，再请求网络；解码与缩放放在适当后台环境，最终在主线程校验 Cell 身份后显示。

缓存 key 不应只包含 URL；按业务加入目标像素尺寸、缩放裁剪方式、图片版本、相关变换和必要的用户隔离信息。相同资源的在途请求可以合并，多个订阅者共享底层下载；一个订阅者取消不应误取消其他仍需要的订阅。

**追问：1 MB 图片为什么占几十 MB 内存？**磁盘文件是压缩数据。普通 RGBA8 位图约为 `bytesPerRow × height`，粗估为像素宽 × 像素高 × 4；4000 × 3000 约 45.8 MiB，还没计中间缓冲。小 View 显示大图时，用 ImageIO 按目标像素大小降采样，通常比先完整解码再缩小更能降低峰值。

内存缓存按成本控制，磁盘缓存限制容量和期限；限制解码并发、预取距离，处理内存警告、动态图帧缓存、损坏文件和失败退避。`NSCache` 的淘汰细节不是严格 LRU 契约；需要可预测 LRU 时，用哈希表加双向链表实现平均 O(1) 查找和移动，并明确同步边界。

### 47 包体积和耗电优化应抓哪些重点

**回答：**包体积先区分下载大小、安装占用和运行内存，再用实际构建产物分析二进制、资源和重复依赖占比。优先删除无用或重复资源、限制不必要的语言与变体、优化图片和音视频编码、检查 dead stripping 与依赖引入。动态库改静态库不天然等于体积更小或启动更快，需要评估重复链接、共享方式和构建产物。

Swift 的泛型特化、内联等可能提升热点性能，也可能增加代码体积；不能只追求单项指标。资源瘦身还应验证画质、解码成本和展示尺寸，不是统一转成某种格式。

**追问：耗电优化从哪看？**CPU、GPU、网络唤醒、定位、定时器和后台活动。合并请求、批量 I/O、合理 leeway、避免轮询、按生命周期停掉定时器和高频传感器，比把任务全部标成高 QoS 更有效。验证应固定网络、亮度、温度、时长等条件，考虑温升后的降频，而不是只看一次短跑 CPU 曲线。

### 48 面试中怎样把一次优化讲得有说服力

**回答：**按照“问题与影响 → 可复现证据 → 根因 → 方案比较 → 实施与回退 → 收益及副作用”说明。每一项要能回答数据从哪里来、口径是什么、为什么是这个改动带来的。

例如列表优化，应说明卡在解码还是布局，用哪段 trace 定位；为什么选择降采样而不是仅加缓存；有没有内存、画质、命中率和首屏回退；最后在相同设备和数据下对比，再用线上分组验证。不要把“功能上线”“平均值变好”和“证明因果”当成同一个结论。

**追问：实验开关怎么设计？**关键路径要可控，关闭后能回到原有行为；检查默认值、配置缓存、持久化和已启动异步任务的状态。启动早期取不到远端配置时，要明确使用哪个快照。新旧路径共享缓存或数据结构时，还要考虑兼容与污染，不能只在入口写一个 if。

**准备自己的案例时至少填清：**用户影响、规模与分母、基线及分位数、你的具体责任、关键证据、备选方案、验证范围和仍未覆盖的风险。没有真实数据时就说明限制，不要把本文的示例当成亲历项目。

本章原笔记：[卡顿与离屏渲染](</Users/bytedance/Desktop/BlogNotes/iOS/性能优化和监控/卡顿优化和离屏渲染.md>)、[启动](</Users/bytedance/Desktop/BlogNotes/iOS/性能优化和监控/启动和优化.md>)、[图片加载与缓存](</Users/bytedance/Desktop/BlogNotes/iOS/性能优化和监控/图片加载优化和缓存设计.md>)、[崩溃监控](</Users/bytedance/Desktop/BlogNotes/iOS/性能优化和监控/崩溃监控.md>)、[安装包瘦身](</Users/bytedance/Desktop/BlogNotes/iOS/性能优化和监控/安装包瘦身.md>)。

## 八 网络

本章只保留客户端面试重点，不展开各协议报文位、路由算法和完整密码学推导。

### 49 一个 HTTPS 请求有哪些阶段 慢请求怎样定位

**回答：**先检查业务和 HTTP 缓存，必要时完成 DNS 解析、连接建立与 TLS 握手，然后发送请求、等待服务端首字节、接收响应，最后解析和更新 UI。连接复用或缓存命中时会跳过部分步骤；HTTP/3 使用 QUIC，不能全部按 TCP 流程理解。

诊断时用 URLSessionTaskMetrics 等分解 DNS、连接、TLS、请求、首字节和下载耗时，同时记录缓存与连接复用、重定向、网络类型、响应大小和错误类别。总耗时高可能是排队、慢服务端、大响应或客户端解析，不应统一归因于“网络差”。

**追问：HTTPS 如何安全？**TLS 提供认证、机密性和完整性。常见证书认证握手通过证书链与域名校验验证服务端身份，再协商会话密钥，用对称加密保护数据。TLS 1.3 的典型握手不能背成“客户端生成密钥，再用 RSA 公钥加密发给服务器”；0-RTT 早期数据有重放风险。[TLS 1.3](https://www.rfc-editor.org/rfc/rfc8446.html)

受信任的调试代理可分别建立两段 TLS 连接，因此 HTTPS 不等于无法抓包。证书或公钥 pinning 要考虑轮换、备用和故障回退，不能通过关闭信任校验来处理线上证书错误。

### 50 TCP 为什么可靠 三次握手 粘包分别考什么

**回答：**TCP 提供有序字节流，通过序号、确认、重传和校验等处理丢失、重复与乱序；流量控制保护接收方处理能力，拥塞控制适应网络承载能力。可靠传输不等于对端业务已经处理成功。

三次握手用于双方建立连接状态、交换并确认初始序号等信息，也使服务端能确认客户端收到了自己的握手信息。关闭时两方向分别结束发送，因此常见四个报文，也可能合并 ACK/FIN；TIME_WAIT 用于处理末次 ACK 丢失后的重传及旧报文影响，不能理解为“等一会就证明服务端一定正常”。

**追问：什么是粘包？**TCP 没有业务消息边界，一次 write 与一次 read 没有一一对应关系。协议用固定长度、分隔符或长度字段划分消息，接收端累积缓冲并处理半包、多包，同时验证长度上限。

UDP 保留数据报边界，但不自行保证可靠和有序。QUIC 在 UDP 上实现自己的可靠性等能力，所以“UDP 上层协议必然不可靠”是错误推论。

### 51 HTTP 1 1 HTTP 2 HTTP 3 的关键区别是什么

**回答：**HTTP/1.1 默认持久连接，但常见使用方式下连接内请求容易排队，客户端常用多连接改善。HTTP/2 通过二进制分帧、多路复用和头部压缩，在一条连接上承载多条流，缓解 HTTP 层面的队头阻塞。

HTTP/2 仍基于 TCP，底层丢包可能阻塞同连接中其他流的数据交付。HTTP/3 把 HTTP 映射到 QUIC 流，降低因一条流丢失数据而阻塞其他流的影响，但仍共享网络带宽与拥塞控制，也可能有应用或压缩依赖造成等待。[HTTP/3 标准](https://www.rfc-editor.org/rfc/rfc9114.html)

**追问：是不是升级 H3 就一定更快？**不是，收益与丢包、RTT、建连复用、服务端和网络支持等有关，还要有协议降级路径。客户端通常通过系统网络栈与服务端协商，验证应按实际使用协议分组，而不是只看配置开关已打开。

### 52 HTTP 缓存 强缓存 协商缓存如何工作

**回答：**通常所说的“强缓存”是响应仍新鲜时直接复用；新鲜度常由 Cache-Control 的 max-age 等决定。需要验证时，客户端携带 ETag 对应的 `If-None-Match` 或 Last-Modified 对应的 `If-Modified-Since`，资源未变可返回 304，并结合缓存正文使用。

`no-cache` 允许存储，但重用前需要成功验证；`no-store` 要求不存储该请求/响应的相应缓存内容。`private` 限制共享缓存；`Vary` 决定哪些请求字段参与响应匹配，不能把相同 URL 就当作相同内容。[HTTP 缓存标准](https://www.rfc-editor.org/rfc/rfc9111.html)

**追问：业务缓存和 URLCache 有何区别？**URLCache 遵循请求和响应的 HTTP 缓存策略；业务缓存可存模型、合并多接口并定义离线策略，但需自己管理版本、账号隔离、失效和数据一致性。退出账号时尤其不能让旧用户数据通过共享 key 被新用户命中。

GET 的 safe 表示语义上不请求改变服务器状态，不是“信息加密”；POST 也不是天然保密，并非绝对不能缓存。是否自动重试应依据幂等语义和业务协议，而不是请求体放在哪。

### 53 如何处理重试 重复请求 Token 刷新和结果乱序

**回答：**先区分可重试的瞬时故障、业务失败和主动取消。合理重试有上限、退避和随机抖动，并遵守服务端给出的限流提示。非幂等操作需要幂等键或服务端去重，否则“超时重发”可能造成重复创建或扣款。

Token 刷新可使用 single-flight：同一账号只允许一个刷新任务在途，其他请求等待其结果；失败统一结束等待，防止每个 401 都发刷新请求。账号切换时让旧一代请求失效，刷新接口自身不能陷入无限刷新链。

**追问：搜索请求 A 后发出 B，A 最后回来怎么办？**取消 A 并给请求或查询分配递增版本，只提交与当前版本一致的结果。debounce 控制触发频率，取消节省资源，版本校验防乱序，这三者解决不同问题。

网络层应统一编码、认证、可观测性、错误分类和传输生命周期；“失败后要不要清空页面”“显示旧数据还是空态”属于业务策略。Alamofire/Moya 提供封装便利，但不会自动替业务决定这些语义。

### 54 大文件下载 断点续传 后台传输有哪些坑

**回答：**避免把整个文件读到内存，优先使用下载到文件的任务或流式处理。下载完成后按 API 生命周期及时移动临时文件，再做大小、校验和等验证，通过原子替换提交最终文件。

断点续传需要校验服务端是否真正返回对应范围：发送 Range 后，206 与 Content-Range 应匹配；若返回 200 全量内容，不能直接追加到旧文件。使用 ETag、If-Range 等确认资源未变化；遇到 416 要重新核对范围和本地状态，而非无限重试。`resumeData` 是系统维护的恢复信息，可能不可用，需要降级重下。

**追问：后台 Session 能无限保活吗？**不能。它让符合条件的传输交给系统管理，不表示 App 可以无限执行任意代码；用户强退、系统调度、上传来源等都有约束。重新进入进程后需要按标识恢复会话和正确处理委托回调。

取消后仍要允许迟到回调被安全忽略。下载状态、进度元数据、临时文件和最终文件应有明确的一致性规则，进程中断后能恢复或清理。

本章原笔记：[网络重点](</Users/bytedance/Desktop/BlogNotes/计算机网络/面试准备/总结-网络重点.md>)、[网络基础](</Users/bytedance/Desktop/BlogNotes/计算机网络/面试准备/网络基础总结.md>)、[URL Loading System](</Users/bytedance/Desktop/BlogNotes/iOS/网络/iOS 官方 URL Loading System 解析.md>)、[网络经验处理](</Users/bytedance/Desktop/BlogNotes/iOS/网络/iOS网络经验性处理.md>)、[Alamofire 与 Moya](</Users/bytedance/Desktop/BlogNotes/iOS/网络/Alamofire和Moya解析.md>)。

## 九 架构与工程设计

### 55 MVC MVVM 应怎样选择 怎样避免 ViewModel 变成巨型类

**回答：**MVC、MVVM 是职责组织方式，核心目标是让变化集中、依赖清晰、逻辑可测试。MVC 不必然导致巨型 VC，MVVM 也不要求 RxSwift 或双向绑定。用闭包、delegate、Combine、Rx 或其他状态机制都可以表达界面与状态的关系。

View 负责显示和输入，ViewModel 负责展示状态和交互转换；可复用业务规则放在领域服务或 use case，数据访问通过 repository 或 service 抽象。路由、持久化和网络细节不应无边界堆进 ViewModel。

**追问：页面状态怎样定义？**显式描述 loading、content、empty、error 及“有旧内容同时刷新”等合法组合，明确事件如何迁移状态。避免多个 Bool 任意组合，出现 loading 和 error 同时为真却不知道显示什么。

架构好坏要看需求改动范围、测试成本、错误定位和团队认知成本，而不是文件数量。简单页面不必套完整复杂框架，但依赖方向和异步生命周期仍要清晰。

### 56 组件化怎样真正降低耦合 Router 和协议服务怎么选

**回答：**组件化首先是依赖治理。明确基础能力、业务组件和产品组装层，限制跨模块直接访问内部实现，避免依赖环。拆成多个 Pod 或 Package 只是载体，不代表耦合已经下降。

页面导航可以用 Router 表达目标和参数；业务能力更适合类型明确的协议服务，通过依赖注入在组装层绑定实现。协议尽量由需要能力的一方或独立契约模块定义，避免为了调用一个功能就依赖整个业务实现。

**追问：URL Router 与协议注册各有什么代价？**URL 便于跨端和外部入口，但字符串参数缺少静态检查，要有校验、版本和找不到路由时的降级；协议更利于编译检查和替身测试，但也需要治理协议粒度、注册时机和可选能力。

不要把所有业务细节搬进全局 Mediator，使其成为新的巨型依赖中心。多产品场景应让产品层提供 adapter 或配置，基础层不直接知道“当前是哪一个 App”。可用依赖图、禁止反向依赖检查和模块契约测试持续维持边界。

### 57 RxSwift 或响应式编程到底解决了什么

**回答：**它用事件流组合异步和状态变化。订阅建立源到观察者的传播关系，操作符可以形成新的序列，处理转换、过滤、合并和切换。不是“写成链式就自动异步”，也不是“Rx 自己另造一套对象内存回收”。

需要分清冷流与热流：冷流订阅可能各自发起工作，热流订阅的是共享事件来源；重复订阅冷的网络流可能触发多次请求。共享策略要明确 replay 数量、连接生命周期、错误后行为以及是否允许重复副作用。

**追问：subscribe on 与 observe on？**前者控制订阅和相关源启动、退订动作的调度位置，后者影响其后的观察回调调度；源自身异步回调的线程仍受源实现影响，不能靠 subscribe(on:) 保证所有事件都在该线程。[RxSwift ObservableType](https://docs.rxswift.org/protocols/observabletype)

**追问：DisposeBag 就没有泄漏了？**若 `self → bag → subscription → closure → self` 构成闭环，bag 本身也不会正常析构。应处理捕获和订阅生命周期。搜索可用 debounce、去重和切换到最新流组合，但底层请求是否实际取消仍取决于封装是否实现取消。

### 58 怎样设计可测试的异步模块 第三方库如何选型

**回答：**先让副作用可替换：网络传输、持久化、时钟、随机数和调度策略通过明确边界注入；业务状态迁移尽量保持确定性。测试应验证用户可见行为和关键不变量，而不是把实现逐行复写成断言。

例如请求模块至少验证成功、失败、取消、超时、重复回调、乱序完成、账号切换；缓存模块验证失效、容量淘汰、同 key 请求合并和错误后重新请求；分页模块验证刷新与加载更多同时发生时，旧页不会混入新数据。

**追问：接口越多越好？**不是。为真实替换点、跨模块契约和复杂副作用建抽象即可，过多只有一个实现且不稳定的协议也会增加维护成本。

第三方库评估功能是否必要、维护活跃度、API 稳定性、平台支持、性能、包体积、依赖和许可证；最好在适当边界包一层自己的业务接口，保留替换空间。讲源码时说明“哪一版、哪条关键链、如何取消与释放、哪种边界会失败”，比背类名更能体现掌握程度。

本章原笔记：[iOS 应用架构](</Users/bytedance/Desktop/BlogNotes/设计模式和架构/iOS应用架构.md>)、[项目组件化](</Users/bytedance/Desktop/BlogNotes/设计模式和架构/项目组件化.md>)、[设计模式](</Users/bytedance/Desktop/BlogNotes/设计模式和架构/设计模式.md>)、[RxSwift 核心逻辑](</Users/bytedance/Desktop/BlogNotes/iOS/第三方库/RxSwift核心逻辑.md>)。

## 十 计算机基础精选

只要求能解释与 iOS 调试、启动和本地存储相关的概念。数据库两题根据官方资料补充，原仓库该主题只有参考资料索引。

### 59 从源代码到 App 经历什么 静态库和动态库有何区别

**回答：**以 C/Objective-C 为例，经过预处理、编译到中间表示及机器代码、生成目标文件，再由链接器解析符号与重定位并生成产物。Swift 前端还涉及类型检查、SIL 等处理，然后进入后续代码生成。资源处理、签名和 App 打包属于相关但不同的构建步骤。

静态库通常是目标文件的归档，链接时把需要的目标代码合入最终镜像；动态库作为独立镜像，在装载或运行时处理其依赖与符号。framework 是打包形式，可以承载静态或动态二进制，不能看到 `.framework` 就判定动态链接。

**追问：为什么静态库里的 Category 有时没加载？**归档成员按符号需求提取时，仅包含分类的成员可能没有被拉入，`-ObjC` 等链接选项会影响 Objective-C 内容的装入。应先检查产物和 Link Map，再选择合适选项；盲目 `-all_load` 可能增加体积或引入重复符号。

Undefined symbols 先查符号实现是否参与当前目标、架构和链接；Duplicate symbols 查重复编译或重复依赖。dSYM 保存调试符号映射，线上符号化必须匹配实际二进制 UUID。

### 60 dyld ASLR fishhook 与 Swizzling 的关系是什么

**回答：**dyld 负责动态镜像装载与链接相关工作；ASLR 使镜像装载地址随机化，相关引用需正确修复。系统共享缓存和现代 fixup 机制优化装载成本，因此旧版源码中的某个函数不代表所有系统都相同。

Swizzling 改变 Objective-C 方法到 IMP 的关系；fishhook 类方案修改 Mach-O 中导入符号的间接引用槽，使部分经这些槽调用的 C 函数转到自定义实现。它不是直接修改所有函数体，也不能保证拦截静态链接、内联、直接调用或已缓存函数指针等路径。

**追问：为什么这是面试重点？**它串起编译链接、Runtime 与 APM 的边界。说明 Hook 原理时，应能指出修改了哪层间接关系、哪些调用会经过该关系、什么情况会绕过，以及多方 Hook、保护页或平台变化带来的限制。

### 61 进程 线程 虚拟内存与汇编只需掌握哪些点

**回答：**进程提供资源和地址空间隔离，同一进程内的线程共享许多资源，各自有栈、寄存器和执行上下文。并发是任务推进可以交错，并行是同一时刻实际执行；线程更多不必然更快，还会增加上下文切换、栈空间和资源竞争。

虚拟地址通过映射对应物理页或文件等后备存储。访问未驻留页可能触发缺页处理，缺页不必然是错误，也不必然每次都读磁盘；非法访问则可能导致异常。顺序访问与局部性可减少缓存失效及相关页面访问成本，启动代码重排也与此有关。

**追问：堆与栈？**栈适合调用上下文和部分局部存储，堆支持更灵活的生命周期；语言值类型与引用类型不能机械对应栈与堆。内存碎片、页对齐和对象实际分配量也可能使统计值高于字段大小之和。

**汇编最低目标：**能区分“地址”和“地址处的值”，看懂 load/store、分支、调用和返回。常见 ARM64 调用约定用 x0—x7 传部分参数，SP 是栈指针、x29 常作帧指针、x30 保存返回地址；普通 Objective-C 方法调用中常可从 x0/x1 看 receiver/selector，但具体还受 ABI、优化和现场影响。笔记里的 x86 `eax/ebp/call` 不能原样当作 ARM64 规则，排查时结合反汇编和符号化调用栈。

### 62 数据库索引为什么快 怎样判断索引是否有效

**回答：**索引维护可搜索的有序结构，以额外空间和写入维护成本换取更少的数据扫描。很多数据库索引用 B 树家族结构，适合页式存储；不要把所有引擎都说成同一种 B+ 树实现，SQLite 的表和索引组织也有自己的细节。

联合索引要结合查询的过滤、排序和取列设计，理解左侧列通常怎样参与查找。等值前缀后接范围或排序字段是常见模式，但具体计划由优化器决定，不能只背“左前缀”就断言任意 SQL 一定走或不走索引。

```sql
CREATE INDEX idx_message_conversation_time
ON message(conversation_id, created_at);

SELECT id, created_at
FROM message
WHERE conversation_id = ? AND created_at < ?
ORDER BY created_at DESC
LIMIT 50;
```

这类会话消息分页可利用会话过滤和时间范围。若时间可能相同，稳定分页还需加入唯一 id 作为次序和游标的一部分，并相应调整索引。用 `EXPLAIN QUERY PLAN`、实际数据量和耗时验证；少量数据测试、返回超多行或排序开销都可能掩盖问题。[SQLite 查询规划](https://www.sqlite.org/queryplanner.html)

### 63 事务 ACID 和 SQLite WAL 怎样联系客户端实际工作

**回答：**事务把一组修改作为逻辑单元。ACID 分别是原子性、一致性、隔离性和持久性；一致性需要约束与正确业务逻辑共同维护，不能理解成数据库会自动验证所有业务规则。

本地批量写入可放进事务，减少重复提交开销并避免部分更新；迁移应具有版本记录、失败回滚及再次启动时的恢复策略。不要在长事务中等待网络或做大量无关计算，它会延长锁和快照持有时间。

**追问：WAL 解决什么？**SQLite WAL 模式把修改先追加到日志，读者按自己的快照读取，通常允许读写并行，但同一时刻仍只有一个写者。Checkpoint 把日志内容回写主库；长时间读事务可能阻碍进度，导致 WAL 增长。开启 WAL 不等于不会遇到 SQLITE_BUSY，也不等于支持任意数量写者同时提交。[SQLite WAL](https://www.sqlite.org/wal.html)

客户端应定义连接与队列的使用规则、短事务、忙等待或重试策略，主线程避免耗时数据库操作。SQL 参数用绑定方式，不拼接不可信字符串；Core Data 若被使用，还必须遵守 context 自身的队列隔离，不能因为底层 SQLite 可并发就跨队列传递托管对象。

本章原笔记：[编译链接](</Users/bytedance/Desktop/BlogNotes/操作系统/编译链接/编译链接过程.md>)、[程序员的自我修养笔记](</Users/bytedance/Desktop/BlogNotes/操作系统/编译链接/程序员的自我修养笔记.md>)、[操作系统重点](</Users/bytedance/Desktop/BlogNotes/操作系统/操作系统基础/总结-操作系统重点.md>)、[汇编基础](</Users/bytedance/Desktop/BlogNotes/汇编/汇编基础.md>)、[数据库参考资料](</Users/bytedance/Desktop/BlogNotes/数据库/数据库参考资料.md>)。

## 十一 岗位选读

以下每题只需先掌握整体机制。没有实际项目经历时，明确说学习过或理解原理，不必扩展到模板元编程、完整渲染器或 Flutter 引擎细节。

### 64 C++ 面试只抓哪些核心点

**回答：**对 iOS 混编岗位，优先掌握 RAII、所有权和多态。RAII 让资源跟随对象生命周期管理，构造时取得资源、析构时释放，可用于内存、锁和文件。`unique_ptr` 表达独占所有权，可移动而不可普通复制；`shared_ptr` 表达共享所有权；`weak_ptr` 不增加强计数，使用前通过 lock 尝试取得有效共享所有权。

`shared_ptr` 互相持有仍会形成循环；引用计数控制块的并发安全不等于对象内部数据线程安全。不要从同一裸指针重复构造相互独立的 shared_ptr 控制块。

**追问：为什么基类析构函数常是 virtual？**当允许经基类指针删除派生类对象时，需要虚析构来正确析构完整对象；不是所有 C++ 类都必须有虚析构。`new/delete` 包含对象构造析构语义，`malloc/free` 主要处理原始存储，配对不能混用。Objective-C++ `.mm` 边界要把所有权、错误和线程约定写清楚。

原笔记：[C++ 重点](</Users/bytedance/Desktop/BlogNotes/C和C++/牛客C++重点.md>)、[C++ 学习总结](</Users/bytedance/Desktop/BlogNotes/C和C++/C++学习总结.md>)。

### 65 Metal 和图形处理需要知道到什么程度

**回答：**没有图形专项经历，掌握 CPU 提交工作、GPU 执行图形或计算任务的分工即可。传统 Metal 命令模型中，device 创建资源，command queue 组织提交，command buffer 承载一批命令，encoder 编码渲染或计算，pipeline 描述着色程序和相关状态，buffer/texture 提供数据；编码结束后提交，并安排 drawable 呈现。[Apple Metal 命令结构](https://developer.apple.com/documentation/Metal/setting-up-a-command-structure)

**追问：为什么提交结束不等于可以复用所有内存？**GPU 异步执行，CPU 写入 GPU 仍在使用的资源可能造成错误。需要资源生命周期和同步设计，例如多份 in-flight 资源、完成回调或适用的同步机制；共享内存也不等于没有访问协调问题。

图像链路优先关注像素格式、尺寸、颜色空间、坐标转换和拷贝次数；视频还涉及 YUV 到 RGB 及色彩矩阵。OpenGL ES 笔记可用于理解管线历史概念，面试不必背整套 API。Metal 4 等新接口另有差异，回答传统模型时应明确范围。

原笔记：[渲染重要总结](</Users/bytedance/Desktop/BlogNotes/图像处理/渲染重要总结.md>)、[Metal 总结](</Users/bytedance/Desktop/BlogNotes/图像处理/Metal总结.md>)。

### 66 Flutter 只抓三棵树 状态与异步模型够不够

**回答：**非 Flutter 专项岗位，先掌握这几个重点即可。Widget 是不可变 UI 配置，Element 维护挂载位置、身份和连接关系，RenderObject 处理布局与绘制等渲染工作。Widget 可以频繁创建，Element/RenderObject 则在匹配条件满足时复用，不是每次 setState 都重建全部底层对象。

setState 的回调同步修改状态并标记对应 Element 需要重建；后续根据变更决定布局和绘制。Key 参与身份匹配，列表重排时应使用稳定业务身份，避免状态跟着错误位置复用。BuildContext 本质上是对 Element 位置相关能力的接口；异步返回后访问 UI 前要确认仍然 mounted。[Flutter 架构](https://docs.flutter.dev/resources/architectural-overview)

**追问：async 会自动到后台吗？**Dart 一个 isolate 中通过事件循环处理任务，async/await 不会自动把 CPU 重活变成并行；重计算可评估其他 isolate，同时考虑数据传递成本。平台通道用于 Dart 与原生通信，要定义序列化、线程和取消语义，不能当成零成本的直接函数调用。

原笔记：[Flutter 重要总结](</Users/bytedance/Desktop/BlogNotes/Flutter/Flutter重要总结.md>)、[Dart 单线程与异步](</Users/bytedance/Desktop/BlogNotes/Flutter/Dart/Dart的单线程和异步.md>)、[Native 交互](</Users/bytedance/Desktop/BlogNotes/Flutter/Native交互.md>)。

## 旧笔记易错结论速查

这张表适合复习结束后自测。它只列本次题集涉及的重点纠错，不代表原仓库每一条内容都已逐项校订。

| 容易记错的说法 | 面试中更准确的说法 |
| --- | --- |
| super 把消息发给父类对象 | 接收者仍是 self，只改变查找起点 |
| Category 给实例添加了成员变量 | 关联对象存于外部关系，不改变实例布局 |
| 给对象用关联 assign 就是 weak | assign 不提供自动置 nil 的弱引用语义 |
| atomic 保证对象线程安全 | 单次访问器完整性不等于复合操作和状态一致性 |
| ARC 下 `__block` 能解除循环引用 | byref 存储默认仍可强持有对象 |
| 非 new 方法返回值都一定进释放池 | 所有权约定不保证实际 autorelease 操作 |
| Tagged Pointer 不走属性 setter | 属性语义不因值的表示方式改变 |
| 串行队列就是一条固定线程 | 它保证串行执行，不保证线程身份 |
| GCD Timer 绝对精准 | 仍受调度、队列阻塞和容忍误差影响 |
| Observer 能让空 RunLoop 常驻 | 观察者本身不是维持循环的输入源 |
| 值类型在栈上，复制就是递归深拷贝 | 值语义与物理存储位置、引用成员需分开 |
| Swift 类和协议都通过 witness table | 类虚表与协议见证表是不同机制 |
| `@objc` 协议里的方法自动 optional | optional 必须显式声明 |
| actor 方法整体不可被打断 | 跨 await 可以重入，要保护业务不变量 |
| `layoutIfNeeded` 会立即执行 drawRect | 布局与绘制是不同失效和更新阶段 |
| 圆角必然离屏，iOS 永远 60 FPS | 需结合图层组合、设备、系统和工具验证 |
| no-cache 表示禁止缓存 | 通常表示重用前验证，no-store 才限制存储 |
| 超时代表服务器没有处理请求 | 可能处理成功但响应丢失，重试需幂等设计 |
| WAL 允许多个写事务同时提交 | 读写并行不等于多写者并行 |

## 复习完成的判断标准

每个核心模块至少做到三件事：能用一两分钟回答主问题；能解释一个反例或边界；能说明如何在实际工程中验证。遇到性能或架构题，再补充方案代价和回退方式。

建议用四条真实项目线串起这些问题：一次卡顿或启动优化，一次内存或崩溃定位，一次复杂异步状态问题，一次组件边界或架构演进。按自己的经历填证据，不必追求每个名词都背得很深。

这份材料覆盖现有笔记里的知识问答主线，不替代算法编码练习。若时间有限，优先练熟数组与哈希、链表、二叉树遍历、二分、滑动窗口和 LRU，并能说清复杂度及边界条件；SwiftUI 则按目标岗位要求另行补齐。
