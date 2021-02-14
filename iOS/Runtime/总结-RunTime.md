# 总结-RunTime
#iOS知识点/Runtime 

[iOS Runtime - 和风细羽 - 博客园](https://www.cnblogs.com/dins/p/ios-runtime.html)

<a href='Runtime.pdf'>Runtime.pdf</a>

# 重点掌握
## Runtime相关数据结构
[Runtime数据结构](bear://x-callback-url/open-note?id=BE71F2A5-F4CB-497E-8051-A9B7B4277E44-3605-000092F8710838EF)
runtime基础数据结构有哪些？对应的关系？
实例对象数据结构？
类对象的数据结构？
运行时的类相关信息存放在哪？是怎样获取的？
编译期的类相关信息存放在哪？和运行期的类相关信息有什么关系？
::方法描述method_t是一个怎样的结构？::
::方法缓存chche_t是一个怎样的结构？有哪些特点？::

## isa指针的理解
[isa相关](bear://x-callback-url/open-note?id=82929659-E3CF-44A9-A4F0-43C9A7E2480D-3605-000092F784A6376F)
你对OC中isa的理解？
isa的指向关系？根元类指向哪里，意味着什么？
实例对象、类对象和元类对象的联系和区别有哪些？
::ARM64位之后isa优化原理？什么是共用体？怎样判断共用体的大小？isa中结构体的位域有什么用吗？::
::isa取指向地址的原理？::

## Runtime消息机制
[Runtime消息机制](bear://x-callback-url/open-note?id=2140DBCF-FFE2-4E11-8F92-DCB235750DCB-3605-000092F88B73AE6A)
OC消息调用的本质是什么？
OC动态方法派发的过程？
消息发送阶段的过程是怎样的？
消息发送阶段缓存查找的过程？::缓存查找的原理？查找过程中如何处理哈希碰撞？::
::消息发送阶在当前类对象中的查找是怎样的？::
动态方法解析阶段的过程？怎样动态添加方法实现？
::消息转发阶段的过程是怎样的？有哪些应用场景？::

## Runtime的实际应用
[常见API](bear://x-callback-url/open-note?id=C4044633-5DE2-479C-82BD-3B7355D18AE2-3605-000092EDD3B035ED) [Runtime 应用](bear://x-callback-url/open-note?id=75DE44F1-57C0-4B75-9236-0C2C9F931D9D-3605-000092F856C2D82D)
runtime场景API有了解吗？
平常有用过runtime？一般来干什么？怎样实现？

## Swift 中的Runtime理解
[Swift runtime](bear://x-callback-url/open-note?id=FA3BD993-2CAE-4D84-9D23-9BE0DE8F5C34-1795-00033DBBA9FB40E5)
你对Swift中Runtime的理解？


[[Runtime初始化和类的加载]]


