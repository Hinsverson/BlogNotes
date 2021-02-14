# Category和Extension
#iOS知识点/OC相关

[iOS底层原理总结 - Category的本质 - 简书](https://www.jianshu.com/p/fa66c8be42a2)

![](Category%E5%92%8CExtension/951C921B-5F73-4447-82E2-4543BFEA46D2.png)
# Category：分类
## Category加载和实现的原理
1. 编译时会把分类替换为一个Category_t结构，里面描述了分类的信息，比如实例、类方法列表，协议列表，属性列表等（里面还含有一个指向原类对象的cls指针）。
![](Category%E5%92%8CExtension/5D26E149-518C-4316-91BD-FDFF1D7CE300.png)
2. App启动后运行时runtime在dyld 给出的mapped时机回调里，会通过Category_t里的cls指针找到原类objc_class对象。
3. 把objc_class类对象的data进行位&运算，取出类信息class_rw_t（存放着类对象运行时的方法，属性和协议等数据以及编译时的class_ro_t）
4. 通过attachList函数，将所有的Category里的实例、类方法、属性、协议列表等信息添加到class_rw_t里对应的二位数组里进行合并。
5. 合并的过程可以理解为将分类方法的列表追加到本来的列表二位数组前面去，所以分类方法在消息查找时会优先调用。

::分类重写本类的方法时，本质只是优先调用，并不会覆盖本类的方法::

::class_ro_t：::
存储的是编译完成，没有合并Category时的方法、属性、协议列表信息。
::class_rw_t：::
存储的是运行时，合并了Category时的方法、属性、协议列表信息。

## Category作用：主要是分解类
1. 分解体积庞大的类文件
2. 声明私有方法
3. 把Framework的私有方法公开

# Extension：扩展
1. 在编译时决议，属于类的一部分，在编译器和头文件的@interface和实现文件里的@implement一起形成了一个完整的类，直接编译进class_ro_t里。
2. 只以声明的形式存在，多数器情况下寄生与宿主类的.m中，伴随着类的产生而产生，也随着类的消失而消失。
3. Extension一般用来隐藏类的私有消息，你必须有一个类的源码才能添加一个类的Extension，所以对于系统一些类，如NSString，就无法添加类扩展。
4. 如果以文件单独存在的extension，并没有import进相关文件，编译时是不会链接的，因为底层编译器会自动过滤未使用的类，方法等。

## Extension作用：隐藏类的私有信息
1. 声明私有属性
2. 声明私有方法
3. 声明私有成员变量

# OC Category 和 Extension的区别
::1. 加载处理时机::：Category启动后运行时合并进class_rw_t，Extension编译时编译进class_ro_t。
::2. 是否能添加属性和成员变量：::extension和category都可以添加属性，但是category的属性不会生成成员变量和getter、setter方法的实现，因为成员变量具体的值属于对象的内存模型，编译后无法修改。（如果确实需要添加成员变量，可以用runtime关联对象动态添加，本质也只是把成员变量值存储在全局的关联表里， 而不是对象的内存模型中）

# load和initialize
## load：当类（Class）或者类别（Category）加入Runtime中时（就是被引用的时候），实现该方法，可以在加载时做一些类特有的操作。

1. 当父类和子类都实现load函数时，父类的load方法执行顺序要优先于子类
2. ::当子类未实现load方法时，不会去触发调用父类中的load方法::
3. 类中的load方法执行顺序要优先于类别(Category)
4. 当有多个类别(Category)都实现了load方法,这几个load方法都会执行,但执行顺序不确定(其执行顺序与类别在Compile Sources中出现的顺序一致)
5. 当然当有多个不同的类的时候,每个类load 执行顺序与其在Compile Sources出现的顺序一致

## initialize：调用所有链接到目标文件的framework中的初始化方法
1. 父类的initialize方法会比子类先执行。
2. 当子类未实现initialize方法时会调用父类initialize方法，子类实现initialize方法时，会覆盖父类initialize方法。
3. 当有多个Category都实现了initialize方法，会覆盖类中的方法，只执行一个(会执行Compile Sources 列表中最后一个Category 的initialize方法)

## 区别对比
::调用时机和方式上：::
1. load在App启动后，dyld给出的load时机里，runtime加载类和分类时递归地沿着类的继承链，如果类和分类有load方法实现的话会拿到load的函数地址并进行调用，所以如果子类未实现load方法时，不会触发调用继承自父类load方法（父类自己实现的load方法在加载自己时还是会调用的）。
2. initialize是基于msgSend在类第一次接收消息时调用，如果子类没有实现initialize，会默认调用父类的实现，所以父类的initialize可能会被多次调用。
::调用顺序上：::
1. load是从父到子调用类的load，再按编译顺序调用分类的load。
2. initialize是从父到子调用，因为基于msgSend，如果分类重写initialize方法会覆盖类本身的initialize方。

# 关联对象原理
![](Category%E5%92%8CExtension/A92B3944-7561-4E68-80CE-384C14E70DD1.png)
![](Category%E5%92%8CExtension/72928496-F4CA-4FCD-979A-53FB1139CA10.png)
1. ::通过维护了一个全局的map用来存放其对应关联属性，所以关联对象并不是存储在被关联对象本身内存中，::而是存储在全局的统一的一个AssociationsManager中，如果设置关联对象为nil，就相当于是移除关联对象。
2. 一个实例对象就对应一个ObjectAssociationMap，而ObjectAssociationMap中存储着多个此实例对象的关联对象的key以及ObjcAssociation，ObjcAssociation中存储着关联对象的value和policy策略。

## 关联对象表结构
1. 第一级表里key是对象地址。
2. 第二级表的key是APi传进来来的key，value就是policy修饰符号和value的封装结构ObjcAssociation。

## 参考修饰
![](Category%E5%92%8CExtension/EFB79974-C0D7-454F-9525-39876791169F.png)
系统通过管理一个全局哈希表，通过对象指针地址和传递的固定参数地址来获取关联对象。根据setter传入的参数协议，来管理对象的生命周期。

## 如何移除关联对象？
### 移除一个object的某个key的关联对象：
调用objc_setAssociatedObject设置关联对象value为nil。objc_setAssociatedObject函数会调用_object_set_associative_reference函数，并在该函数中判断传进来的value是否为nil，是的话会调用erase(j)擦除函数，将j变量擦除，j即为ObjectAssociationMap对象里的一对【key: key value: ObjcAssociation（_policy、_value）】。

### 移除一个object的所有关联对象：
调用函数objc_removeAssociatedObjects。 objc_removeAssociatedObjects函数会调用_object_remove_assocations函数，并在该函数中调用对象的erase(i)擦除函数，将i变量擦除。i即为AssociationsHashMap对象中的一对【key: object value: ObjectAssociationMap】。

### object被释放的时候需要手动将关联对象置空么？

不需要。因为当NSObject dealloc时，disposed时，会通过置nil的方式移除关联对象。
