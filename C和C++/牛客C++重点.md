# 牛客C++重点
#C和C++

# const作用
1. 修饰函数时：
当const在函数名前面的时候修饰的是函数返回值。
当const在函数名后面表示是常成员函数，该函数不能修改对象内的任何成员，只能发生读操作，不能发生写操作。 （等同于Swift Struct 函数内不能修改成员的值，要修改需要加mutaing，本质是C++函数隐式传参时this的类型是一个指向const类型对象的const指针）
2. 修饰变量：不可修改

# Static作用
1. 修饰全局、局部变量时
	1. 从内存角度看作为一种存储类别，标示变量存储在静态存储区，程序运行期间只有一份，未初始化会自动初始化清0（bss）。
	2. 作用域角度上看全局static变量从定义之处开始，到文件结尾，而局部static只能在当前作用域内访问。
2. 修饰全局函数时，默认情况下都是extern的，但是只在当前文件当中可见，不能被其他文件引用，::一般cpp文件内使用static修饰局部函数::
> 不要在cpp内声明非static的全局函数，如果你要在多个cpp中复用该函数，就把它的声明提到头文件里去，否则cpp内部声明需加上static修饰，否则链接的时候找不到这个函数。  
3. 修饰类的成员、函数。::在静态成员函数中没有隐式this指针，不能直接引用类中说明的非静态成员，但是可以引用类中说明的静态成员。::

# 智能指针
1. 类似高级语言里的2种强弱指针：
	1. shared_ptr：类似强引用，对象被shared_ptr指向一次引用计数+1，shared_ptr销毁时引用计数-1，为0时对象自动销毁。
	2. weak_ptr：类似弱引用，引用计数不会+1
2. 特殊的指针unique_ptr：强引用，但是能保证同一时间只有1个指针指向对象，unique_ptr销毁时，指向对象也就自动释放了。

::智能原理就是通过模版把指针包起来，在自己析构的时候delete指针指向的对象，然后重写指针访问符实现外部直接访问该对象）::

# 类型转换
1. const_cast：用于将const变量转为非const。
2. static_cast：一般用于非const转const，缺乏对多态类型的安全检测。
3. dynamic_cast：一般用于动态类型转换，所以会有安全检测（非法的对于指针返回NULL）。
4. reinterpret_cast：这种转换就是存储的二进制数据直接赋值（将int转指针），比较底层，没有安全检查。

::C语言的强制类型转换是直接赋值，没有安全检测::

# 指针和引用的区别？
高级语言的引用可以看成是一种安全的指针弱化，指向后不能再瞎指向别处。

# 为什么析构函数必须是虚函数？为什么C++默认的析构函数不是虚函数？
可能被继承的父类需要把析构函数设置为虚函数，这样可以保证多态指向时可以正确调用子类析构函数释放子类内存空间。

C++默认的析构函数不是虚函数是因为虚函数需要额外的虚函数表和虚表指针，占用额外的内存，对于不会被继承的类来说浪费内存。

# 析构函数的理解？
当对象结束其生命周期执行。
编译系统会自动生成一个缺省的析构函数。
构造从父类到子类，析构从子类到父类。

# 静态函数和虚函数的区别？
静态函数编译时候就确定了调用地址，虚函数在运行的时候动态绑定。

# C++函数重载、重写的区别？
重载的本质是编译器内部通过不同的函数名表示一个函数，并且存储在不同的内存地址中。
重写是对虚函数的重新定义。

# strcpy和strlen？
strcpy是字符串拷贝函数，从src逐字节拷贝到dest，直到遇到’\0’结束，因为没有指定长度，可能会导致拷贝越界。安全版本是strncpy函数。
strlen函数是计算字符串长度的函数，返回从开始到’\0’之间的字符个数。

# 写个函数在main函数执行前先运行？
```c++
__attribute((constructor))void before()
{
    printf("before main\n");
}
```

# new/delete与malloc/free的区别是什么？
都是堆空间的申请和释放操作。
new/delete方式创建销毁类型对象会调用构造函数和析构函数。
malloc/free来源于C，创建类型对象不会调用构造函数和析构函数。

# C++多态原理？
**vurtual和C++虚函数、多态的综合理解：**
1. vurtual修饰符的作用就是延迟绑定，运行时决议，vurtual意味着对象的内存布局里前4个bety存储着一个指向需表地址的指针，虚表里记录调用函数的地址完成调用。
2. 父类中有虚函数，则子类内存布局的里也有指向父类虚函数表的指针，多个继承就有多个，顺序按照继承的顺序放置。如果子类也有虚函数那么在放置完父类虚表指针后也会放置指向子类自己虚表的指针在内存布局中。
3. 如果子类去重写了父类的某个虚函数，会替换掉父类虚表中对应的函数地址，此时用父类指针指向子类对象就会发生多态。

![](%E7%89%9B%E5%AE%A2C++%E9%87%8D%E7%82%B9/311436_1552470920741_7D40CEF3951A10F626301148E06D89DA.png)

一句话，申明为虚函数使得函数的派发同过虚函数表派发，也使得该类的子类具备多态行为。

# C语言是怎么进行函数调用的？
[函数的过程调用细节](bear://x-callback-url/open-note?id=4979AA4E-AE9B-4BBE-840F-E58447906B8D-3605-000092EF272FFA3A)
参数压榨顺序从右到左

# vector和list的区别
1. vector底层实现是数组；list是双向链表。
2. vector支持随机访问，list不支持。
3. vector是顺序内存，list不是。
4. vector在中间节点进行插入删除会导致内存拷贝，list不会。
5. vector一次性分配好内存，不够时才进行2倍扩容；list每次插入新节点会进行内存申请。
6. vector随机访问性能好，插入删除性能差；list随机访问性能差，插入删除性能好。

# 访问权限
public：公开（struct默认）
protected：子类可以访问
private：私有（class默认）

# struct和class的区别
C++Struct和Calss除了默认访问权限，其他没有区别，struct的存在是为了兼容C。

# C++使用空指针调用函数的情况？
[具体细节查看](https://blog.csdn.net/chenzrcd/article/details/60472616)
1. NULL对象指针可以调用成员函数。
2. 通过对象调用成员函数，对象的指针会被传入函数中，指针名称为this
3. NULL对象指针调用成员函数时，只要不访问此对象的成员变量，则程序常运行。
4. NULL对象指针调用成员函数时，一旦访问此对象的成员变量，则程序崩溃。
> OC中不会，消息直接过滤，swift直接非可选会直接崩。  
