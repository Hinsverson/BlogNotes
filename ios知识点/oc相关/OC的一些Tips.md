# OC的一些Tips

# Effective OC
 [Effective OC干货](https://www.jianshu.com/p/9c93c7ab734d) [一篇文章拿下《Effective Objective-C 2.0》](http://www.cocoachina.com/articles/20193)

1. ==操作符比较的是指针值，也就是内存地址，想比较指针所指向的内容，在这个时候，需要通过isEqual:方法来比较。

# id类型：动态类型
``` c++
typedef struct objc_object *id;
```
一种通用的对象动态类型，它可以指向属于任何类的对象，也可以理解为万能指针，编译时不会确定真实类型。

# Class类型：objc_class别名
``` C++
typedef struct objc_class *Class;
```

# 向nil对象发消息不会崩溃因为会自动过滤nil，但是如果方法找不到会崩溃。
# [ A class] [A superclass]、object_getClass()的理解
* 实例对象class方法实现为object_getClass(self)返回类对象，类class方法实现直接返回self。
* 实例对象superclass实现为object_getClass(super)，返回父类对象，类superclass方法实现直接返回super。
* 而object_getClass(类对象)，则返回的是元类，所以object_getClass本质可以理解为取isa指向类型。

# super方法调用的理解
super调用方法的本质是转成objc_msgSendSuper向当前self发消息，只是查方法表会跳过当前类对象，直接从父类开始查方法进行调用。
```objectivec
// 创建一个Student类继承子Person类,下面代码打印出什么
NSLog(@"[self class] = %@", [self class]); //Student
NSLog(@"[super class] = %@", [super class]); //Student
NSLog(@"[self superclass] = %@", [self superclass]); //Person
NSLog(@"[super superclass] = %@", [super superclass]); //Person
```
class和superclass方法定义在NSObject里，所以不管用self 还是 super 调用class，返回的都是object_getClass(self)，或者object_getClass(super)。

# Selector的理解
SEL就是用字符串哈希来指代一个唯一的方法名。

# isKindOfClass和isMemberOfClass区别
A isKindOfClass B的本质是从A的isa开始到super链上有没有B。
A isMemberOfClass B的本质是从A的isa是否指向B。
A可以是对象，也可以是类对象。
```objectivec
BOOL re1 = [(id)[NSObject class] isKindOfClass:[NSObject class]];       // 1
BOOL re2 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];     //0
BOOL re3 = [(id)[LGPerson class] isKindOfClass:[LGPerson class]];       //0
BOOL re4 = [(id)[LGPerson class] isMemberOfClass:[LGPerson class]];     //0
NSLog(@" re1 :%hhd\n re2 :%hhd\n re3 :%hhd\n re4 :%hhd\n",re1,re2,re3,re4);
//👆上述Log的打印结果为1，0，0，0

BOOL re5 = [(id)[NSObject alloc] isKindOfClass:[NSObject class]];       // 1
BOOL re6 = [(id)[NSObject alloc] isMemberOfClass:[NSObject class]];     // 1
BOOL re7 = [(id)[LGPerson alloc] isKindOfClass:[LGPerson class]];       // 1
BOOL re8 = [(id)[LGPerson alloc] isMemberOfClass:[LGPerson class]];     // 1
NSLog(@" re5 :%hhd\n re6 :%hhd\n re7 :%hhd\n re8 :%hhd\n",re5,re6,re7,re8);
```

# 理解ISA：OC方法调用的本质就是拿ISA去找类对象
正常执行和调用，但是打印出来的是WY。
```objectivec
***********************LGPerson***************************
@interface LGPerson : NSObject
@property (nonatomic, copy)NSString *name;
//@property (nonatomic, copy)NSString *subject;
//@property (nonatomic)int age;

- (void)print;
@end

@implementation LGPerson
- (void)print{
    NSLog(@"NB %s - %@",__func__,self.name);
}
@end

**************************调用*****************************
- (void)viewDidLoad {
    [super viewDidLoad];
    
    NSString *tem = @"WY";
    id cls = [LGPerson class]; //获取到类对象指针
    void *obj= &cls; //堆对象地址=isa地址
    [(__bridge id)obj print]; //拿到isa调用    
}

```
结果为什么是WY和压栈顺序有关，从self偏移8个bety访问name访问到的是NSString这个临时变量，所以打印出来是WY。
![](OC%E7%9A%84%E4%B8%80%E4%BA%9BTips/12CBE312-3B02-4801-9954-223D5D2BD707.png)


# OC NULL、nil、Nil和Swift nil的区别
[nil / Nil / NULL / NSNull - NSHipster](https://nshipster.cn/nil/)

NULL、nil、Nil这三者对于Objective-C中值是一样的，都是(void *)0，那么为什么要区分呢？又与NSNull之间有什么区别：

NULL是宏，是对于C语言指针而使用的，表示空指针
nil是宏，是对于Objective-C中的对象而使用的，表示OC对象为空
Nil是宏，是对于Objective-C中的类而使用的，表示OC类为空
NSNull是类类型，是用于表示空的占位对象，用于填充集合元素，只有一个方法null，并且是单例的
Swift nil 是一个确定的值，标示没有值。

# OC和C参数传递问题
[OC如何实现函数参数对象的地址传递_jia12216的专栏-CSDN博客](https://blog.csdn.net/jia12216/article/details/108017695)
[说说OC参数传递的那些坑 - 简书](https://www.jianshu.com/p/d43dac503928)

OC中即使参数是指针也有地址传递和值传递2种形式，参数类型是`*NSString`这种，代表是值传递，所以你在方法内获取到的形参变量指针str的地址和外部实参的地址不是同一个，修改形参的值（存储的地址）不会影响外部实参。参数类型是`**NSString`代表是地址传递，传递后形参直接指向实参的存储单元，对形参发起的修改（存储的地址）会影响到实参。