# Swift和OC混编的坑
#Swift/进阶

# 为什么慢？
2种语言本来实现上各有异同，从对象的内存布局模型，方法派发上讲…..
混编慢的主要原因就是编译时会生成大量的中间桥接对象和类型。

比如当某个Swift方法被声明为@objc关键字时，在编译时刻会生成两个函数，一个是本体函数供Swift内部调用，另外一个是跳板函数(trampoline)是供OC语言进行调用的。这个跳板函数信息会记录在OC类的运行时类结构中，跳板函数的实现会对参数的传递规则进行转换：把x0寄存器的值赋值给x20寄存器，然后把其他参数依次转化为Swift的函数参数传递规则要求，最后再执行本地函数调用。跳板函数实现伪代码如下：
``` C++
// 跳板函数的实现: 提供给objc_msgSend调用
void trampoline_foo(id self, SEL _cmd){
     asm(“mov x20, x0”);
     self->isa->vtable[0](); //这里调用本体Swift函数foo
}
```

[OC混编Swift中代理设计模式的小坑 - 简书](https://www.jianshu.com/p/d96a5a5b4a4f)
因为Swift中协议中的方法默认是必须实现的，OC中协议中的方法默认都是可选实现的。
Swift中定义的协议默认是不暴漏给OC的，如果想在OC中使用协议，需要在定义协议的时候 在前面加上@objc 告诉OC，协议加上@objc 后协议中定义的方法都变为可选实现的了。


[swift与OC相互引用的问题_鬼蟹的博客-CSDN博客](https://blog.csdn.net/qq_16379603/article/details/52156607)

Oc调用Swift结构体的问题：包起来
[oc中调用swift中的struct_wahaha13168的博客-CSDN博客](https://blog.csdn.net/wahaha13168/article/details/52438035)