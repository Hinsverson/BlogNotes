# 总结-OC
#iOS知识点/OC相关 

<a href='OC%E8%AF%AD%E6%B3%95.pdf'>OC语法.pdf</a>

[属性和关键字的理解](bear://x-callback-url/open-note?id=0691CA7E-DA2E-4F54-8FA0-9A3EA9F38A86-3605-000092F2EB8E1E9A)
理解属性的概念以及clang前端编译对属性做了哪些事情？
理解synthesize？
理解@dynamic？
::理解自动生成的get方法需要注意什么？::
理解atomic属性修饰符？默认基本数据类型修饰符？默认对象修饰符？
synchronized锁的原理？理解为什么它不是线程安全的？
::weak和assign的区别？::
::对于Foundation类型选择strong还是copy修饰一般依据什么？理解如果不这样可能导致什么隐形问题？::
可变对象和不可变对象的理解？
copy和mutableCopy的理解？
::Foundaation类型不可变、可变对象mutableCopy是浅拷贝还是深拷贝？::
::Foundaation类型不可变、可变对象copy是浅拷贝还是深拷贝？::
容器对象（NSArray，NAMutableArray；NSDictionary，NSMutableDictionary；NSSet集合）的内容属于深还是浅拷贝？

[Category和Extension](bear://x-callback-url/open-note?id=1683CF5F-3140-47C1-814C-EEBE1A65BBAD-3605-000092F6902DD310)
::OC Category的实现原理？Category和extension的区别？Category可以添加成员变量吗？如果要给Category添加成员变量如何实现？runtime关联对象的原理？关联对象是存储在被关联对象内存中的吗？::
::Category中有load方法吗？Category和本类的load方法是什么时候调用？调用顺序（考虑继承的情况）？::
::类的initialize方法是在什么时候调用？继承时的调用顺序？如果Category中实现了initialize会覆盖原来类的？::
::initialize和load的区别是什么？::

[Block理解](bear://x-callback-url/open-note?id=E3AD3DA5-E031-4AD2-9F9F-E9949B3D3E0D-3605-000092F685D9AA8D)
::理解Block的原理和本质是什么？::
::Block的类型有哪几种？怎样区别？存储在哪里？::
::ARC下触发栈上Block拷贝到堆上的情况有哪些（4种）？::
::MRC下Block的使用注意？::
::理解Block的捕获机制的原理，为什么要捕获和怎样捕获的？（全局变量、static全局变量、static局部变量、局部变量）？如何理解Block不允许修改栈中捕获变量的内容？::
::理解Block捕获对象和基本数据类型间的差异？::
::__block修饰符用于修饰什么类型的变量？理解捕获它的时候底层原理是怎样的？这时候在block内部是怎样访存这个对象的？::
::有没有__block修饰时对象捕获的情况有什么不同？::
::如何理解Block捕获的局部变量是浅拷贝并且const，__block捕获后Block内外部对象是一个对象？::
::理解 __weak、__strong、weak、strong？::

3种类型？捕获机制？基本类型和对象类型的差异？__block修饰底层发生了什么？

[OC对象的初始化](bear://x-callback-url/open-note?id=47460F1F-2437-4C8C-A7D5-E1890877A540-5640-00015D0642815980)

[启动和类的加载](bear://x-callback-url/open-note?id=EC04040C-2203-463C-87D7-362A2F23FA8D-5640-00037D7E87E72253)

[其他知识点](bear://x-callback-url/open-note?id=83EC7EBD-5D66-4BC1-B6FD-4A55D5E459C6-5640-0001741E2AD84C1F)

[Effective-OC](bear://x-callback-url/open-note?id=EEB55120-39A6-4906-A6BD-99A5BB23C9BC-21058-00044CD053769879)




