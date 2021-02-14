# Runtime消息机制
#iOS知识点/Runtime

# 数据结构
![](Runtime%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6/8BC7BE95-0A3F-4657-A86D-529920F878E0.png)
class_rw_t里的列表是2维列表

# OC调用方法的本质
![](Runtime%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6/901C348E-EC75-434D-A92E-F35A19E3C586.png)
![](Runtime%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6/EA5D0A15-85FF-43E6-8100-F9134C1A354D.png)
OC方法调用，最后都会转换成C语言函数objc_msgSend，接受一个响应者（receviver）和具体的方法选择器（@selector）。

**OC消息机制就是给方法调用者（receiver）发送消息**、

**[self message]：**消息的接收者是本类对象，方法的实现从本类开始查找
**[super message]：**消息的接收者是本类对象，方法的实现从父类开始查找
**class：**方法定义在NSObject中，本质是object_getClass(self)，只和消息接受者有关系。
``` objectivec
#import "CView.h"

@implementation CView
- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        // 都打印类名，class方法获取类对象 都是CView
        NSLog(@"%@",NSStringFromClass([self class]));
        NSLog(@"%@",NSStringFromClass([super class]));
    }
    return self;
}
@end
```

# objc_msgSend执行3大阶段
消息发送阶段：从类及父类的方法缓存列表及方法列表查找方法。
动态解析阶段：如果消息发送阶段没有找到方法，则会进入动态解析阶段，负责动态的添加方法实现（只进行一次）。
消息转发阶段：如果也没有实现动态解析方法，则会进行消息转发阶段，将消息转发给可以处理消息的接受者来处理。

# 消息发送
1. 尝试在入recriver实例对象中按照方法查找流程去找对应的@selector方法（从方法缓存方法列表、方法列表、找不到就通过superClass指针到父类对象缓存和方法列表找，一直循环）
2. 找到就调用并添加到当前消息接收者方法缓存列表里。
3. 到最顶层父类都找不到就进行下一个阶段。
![](Runtime%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6/6CE783BB-D413-48B6-88A1-4C1419F78F76.png)

## 在缓存方法里查找原理（提高效率）
一个接收者对象接收到一个消息时，它会根据isa指针去查找能够响应这个消息的对象。在实际使用中，如果每次消息来时，我们都是methodLists中遍历一遍，性能势必很差。所以在每次调用过一个方法后，这个方法就会被缓存到cache列表中，下次调用的时候runtime就会优先去cache中查找，如果cache没有，才去自身methodLists中遍历查找方法，自身没有找到再根据superclass指针到父类去寻找，在父类中同样遵循这样的查找规则，找到就缓存到自己cache的散列表数组中。这样，对于那些经常用到的方法的调用，但提高了调用的效率。
![](Runtime%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6/9B4BF34C-5D23-4F24-A4F4-F5175D0A2CBA.png)
![](Runtime%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6/7FCD5200-3696-4293-8602-01F1DDDCDF6B.png)
1. buckets初始是一个4字节的哈希表，mask值为哈希表长度-1。
2. 存储时，使用SEL转换为的cache_key_t_key&mask得出一个小于散列表最大长度的值，作为从buckets数组中存取方法的索引，拿到bucket_t方法描述，然后得到对应的方法IMP内存地址。
3. 当存储容量大于哈希表容量3/4时，会进行2倍扩容。
::注意这里的key是@selector(方法名)&_mask，value是bucket_t::

### buckets 散列表数组如何处理哈希碰撞：开放地址线性探测？
如果存入方法时算出的索引已经存在（哈希碰撞）则直接将索引减1再重复判断，直到对应的索引下不存在方法为止，减到1时再从最大索引（mask）往下找。

## 如何在当前类方法列表中进行的查找？
对于已经排好序的列表，采用二分查找算法（启动runtime时会对一些类的方法进行排序）。
对于未排序的列表，采用一般遍历的方式。

## 动态方法解析
1. 首先会判断是否曾经有过解析过，有的话进入消息转发阶段。
2. 没有的话会调用resolveInstanceMethod，在里我们可以使用Runtime的class_addMethon API动态添加一些方法，并会自动保存到对象方法列表里面，重新再走消息发送流程。
![](Runtime%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6/1919EF0E-BFDE-42C4-A991-9607F8298155.png)

通过class_addMethon函数传入添加的对象、selector、方法Imp、方法编码
![](Runtime%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6/8C6B18C6-B397-4B2D-B253-2D26A402BEF0.png)

## 消息转发
![](Runtime%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6/9CDA335C-80B6-460E-9DD1-5A2606C13C49.png)
如果消息发送阶段没有找到对应的方法的话，也没有进行动态解析，或者已经解析过了，就会进入到消息转发阶段。

1. 首先会进入到forwardingTargetForSelector方法内进行快速转发，这里面可以把对应的selector（方法选择器）转发给其他类，让其他类去处理，其他类处理的时候同样会遵循objc_msgSend方法查找三大阶段的逻辑去处理。
![](Runtime%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6/BA4C9DE6-8CAD-4274-A518-296D66F90A08.png)
2. 如果forwardingTargetForSelector返回为空，则会进入慢速转发
	1. 首先进入methodSignatureForSelector，允许你返回一个方法签名（TypeCoding编类型编码），如果没有返回则没有找到方法则程序崩溃。
	2. 如果返回了方法签名后会进入forwardingInvocation方法内，把对应的方法信息包装成一个Invocation（方法调用者、方法参数、方法名），做更细的处理。比如可以将消息同时转发给任意多个对象，修改方法调用者等（相当于转发）
![](Runtime%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6/F80971AE-7214-475A-B8CE-AA59F3A46D4B.png)


