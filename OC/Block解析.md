# Block解析
#iOS知识点/OC相关

![](Block%E8%A7%A3%E6%9E%90/272A7AA2-3301-44CD-A400-4ED6C542CC50.png)

# 关键点理解
1. Block是一个对象，携带着函数执行环境以及捕获的变量。
2. Block直接访问全局性的变量，通过指针传递间接访问静态局部变量，通过捕获进行栈上局部变量的访问。基本数据类型是值拷贝，对象是指针拷贝。
3. 在Block内部不能直接修改被捕获的变量（其实是指Block不允许修改栈中的指针，但是可以拿着这个指针直接去修改东西，比如捕获局部NSMutalbeArray类型的指针可以调用addObject去往array里添加元素或者__Block捕获）。
4. Block捕获对象时和捕获基本类型的区别是捕获对象时会增加一些内存管理操作函数，使得它具备管理被捕获对象的能力（只是具备），同时Block捕获时Block结构内部存储对象的指针内存修饰符号取决于外部被捕获的对象内存修饰符。
5. 三种类型的block存储区域不同，堆上Block只能从栈上Block调用copy创建，堆上Block重复copy调用相当于增加自身引用计数。
6. ARC下栈上的block作为函数返回值、赋值给__strong指针时、Cocoa API中方法名含有usingBlock的方法参数、GCD API的方法参数时会自动调用copy转为堆空间类型的Block。
7. ARC下堆上的block同其它对象一样自身受ARC引用计数管理，但是由于它捕获对象时具备管理被捕获对象的能力，所以从栈上copy到堆上时，会对捕获的对象进行内存管理操作，这种操作是强、弱引用取决于4中Block结构内存的是什么类型的内存修饰符（相当于Swift里通过捕获列表指代替代默认的隐式捕获行为）。
8. __Block只能修饰临时变量，用来解决临时变量因为作用域和生命周期不能在Block内被修改的问题，基本原理如下：
	1. _Block修饰后会把捕获的自动变量包成一个对象，所以原来Block结构内存储捕获变量的地方现在存指向这个包装对象的强指针，包装对象内按照原捕获规则存储被捕获的变量。
	2. 这个包装对象随着Block copy时也被拷贝到堆上，解决了自动临时变量容易因作用域结束而释放从而不能修改的问题（此时Block内修改是修改堆上的复制的变量内容）。
	3. 同时当包装对象从栈上被拷贝到堆上时，会把栈上包装对象内部的__forwarding的值替换为复制到堆上的包装对象的地址，通过这种功能，无论是在Block语法中、Block语法外使用__block变量，还是__block变量配置在栈上或堆上，都可以顺利访问修改同一个变量。
	4. 如果多个block都捕获同一个__block，那么这个包装对象只复制一次。
9. 重点理解__Block修饰时，由Block来进行包装对象的内存管理操作（原来是Block直接管理被捕获的对象），由包装对象来进行被捕获对象的内存管理操作（管理规则还是按照存储的是强引用还是弱引用进行）
10. 理解无论是对基本类型还是对象使用__block修饰符，从转化后的源码来看，它们都会被转化为对应的包装对象来使用，包装对象具有引用类型数据的特性，自身受引用计数内存管理（Block管理包装对象，包装对象管理被捕获的对象）所以当多个Block捕获用__block修饰的自动变量时，这个自动变量释放的时机由多个Block共同决定。

# 本质
[Block原理探究(上篇)-Block本质及存储域问题 - 云+社区 - 腾讯云](https://cloud.tencent.com/developer/article/1517602)
[Block原理探究(下篇)-捕获变量分析及__block原理 - 云+社区 - 腾讯云](https://cloud.tencent.com/developer/article/1517593)
[理清 Block 底层结构及其捕获行为 - 掘金](https://juejin.im/post/5bb09160f265da0adb30e30d#heading-8)

Block相当于其他语言中的闭包或者匿名函数，Block与函数区别在于，Block相当于函数加上函数执行的上下文环境(捕获外部变量下面会讲到)。
![](Block%E8%A7%A3%E6%9E%90/7C268A04-DE0D-4809-A61B-B0564EB87D6A.png)
![](Block%E8%A7%A3%E6%9E%90/D2BE37AA-F180-40BB-986C-E3A3C6491F31.png)
1. Block对应底层main_block_imll_0结构体，其中包含有isa指针，这说明Block本质上还是一个OC对象，并且是继承于NSObject的；
![](Block%E8%A7%A3%E6%9E%90/7C9F0627-99DA-4A43-9A40-5839C04A0620.png)
2. Block中待执行的代码，在底层也被封装为__main_block_func_0函数，以实现调用，说明Block还携带了函数执行的环境。
3. block的isa指向的类对象有3种，对应了不同的block类型和内存分配方式。

# Block的类型和内存分配
![](Block%E8%A7%A3%E6%9E%90/B572677C-281C-42E6-A8F2-99A1EA26EEF9.png)
![](Block%E8%A7%A3%E6%9E%90/68B6DC0F-753B-49AF-B135-8DAB73AEEFAA.png)

## ARC下Block会发生自动copy操作
ARC环境下编译器会根据情况自动将栈上的block复制到堆上通过引用计数管理，并在合适的时候release（这种情况可以理解为Swift逃逸闭包，当需要延长block生命周期，防止调用栈结束而对象被释放时）：
1. block作为函数返回值时。
2. 将block赋值给__strong指针时。
3. block作为Cocoa API中方法名含有usingBlock的方法参数时。
4. block作为GCD API的方法参数时。

## MRC环境下需要注意
MRC环境下需要使用copy修饰属性来将栈上的block复制到堆上，并且copy完后需要release操作释放堆上Block防止内存泄露。

# Block捕获机制
**总结：**
* Block直接访问全局性的变量，如全局变量、静态全局变量。
* Block间接访问静态局部变量，并使用指针传递的方式。
* Block不允许修改栈中指针的内容，捕获时发生一次拷贝，基本类型拷贝值，对象指针类型拷贝指针。

![](Block%E8%A7%A3%E6%9E%90/D1BBD84C-C950-41EC-8316-B4683D8E6896.png)
![](Block%E8%A7%A3%E6%9E%90/BF67BC54-30CA-41AE-9E4D-79A7A2BC67AA.png)
1. auto局部变量：离开作用域就销毁。
	* 若是基础类型 int val ，则__main_block_impl_0里存储的是 int val。
	* 若是对象类型指针 *obj ，则__main_block_impl_0里存储的是*obj）
2. static全局变量：本身内存分布在全局区，但是由于访问权限在外部，所以block里无法直接访问，这时候会进行指针的传递。
	* 若是基础类型 int val ，则__main_block_impl_0里存储的是 int *val。
	* 若是对象类型指针 *obj ，则__main_block_impl_0里存储的是指向指针的指针**obj）
3. 全局变量/全局静态变量：内存分布在全局区，无需捕获。

# Block对捕获对象的内存管理
当捕获的变量是对象类型时，发生捕获的时候Desc结构中多了内存相关的copy和dispose操作函数，使得block有能力对这个对象进行内存管理或者说引用，实际会不会发生取决于Block会不会发生copy操作。

::也就是说栈空上的block不会对对象强引用，堆空间的block有能力持有外部调用的对象，即对对象进行强引用或去除强引用的操作。::

1. ARC下当栈上的block复制到堆上时会调用copy，copy内部会调用_Block_object_assign函数，这个函数会根据变量的内存修饰符进行__strong、__weak、__unsafe_unretain等相关内存管理操作，block引用一次这个捕获的变量对象。
2. ARC下当堆上的 Block 被废弃时（Block自己受ARC引用计数管理），会进行dispose，根据内存修饰符执行相关内存操作，断开Block和这个捕获变量对象的引用。

# __block修饰原理
**Block修改外部变量的限制，其实是指Block不允许修改栈中指针的内容**

无论是对基本类型还是对象使用__block修饰符，从转化后的源码来看，它们都会被转化为对应的结构体实例来使用，具有引用类型数据的特性。因此__block变量随着Block被拷贝到堆上后，它们的内存管理与普通的OC对象引用计数内存管理模式完全相同。

__block只用在需要修改捕获的临时变量时，不能修饰静态变量（static） 和全局变量。

1. 无论修饰的是临时值类型还是对象类型val，Block的捕获还是遵循局部变量的捕获规则，只是会把这个变量包起来存储在一个包装对象里，Block结构内原来捕获变量的位置变成了一个同名指针，指向包装对象。
2. 包装对象里存储着真正根据捕获规则获取到的val变量，并且通过在把装对象里是通过一个`forwarding->val`指针访存值。因为包装对象内的forwarding 指针原本指向的是自己，当Block被copy到堆上时栈上的包装对象内的forwarding 会指向堆上的新的包装对象，所以对val的访存在发没发生block的copy操作时都没有问题，都能正确访问。
![](Block%E8%A7%A3%E6%9E%90/86703B11-E343-4D51-A83B-A2D0CC00CAAD.png)
3. __block变量发生捕获后，同时也会把Block外部的指针变量地址也修改为包装对象内实际存储的新变量的地址值（接口屏蔽原则，相当于__block修饰后内外部变量是同一个变量）（验证过）

# 有没有__block修饰时对象捕获的情况对比
1. 栈上的block对象的捕获永远是安全的，不会进行retain操作。
2. 没有__block修饰并且捕获局部类对象时block触发copy操作到堆上后，Block描述性结构__main_block_impl_0内部指向这个对象的指针是强还是弱根据被捕获对象用的内存管理修饰符决定，Block 被废弃时，会进行dispose，根据内存修饰符执行相关内存操作，断开Block和这个捕获变量对象的引用。
3. 有__block修饰并且捕获局部类对象时block触发copy操作到堆上后，因为__block修饰符会创建一个中间包装对象，所以此时的指向关系如下：
![](Block%E8%A7%A3%E6%9E%90/18A03513-9A6A-472F-924A-0EF86C0C6526.png)
此时Block描述性结构内部指向包装对象的这个指针永远是强指针，包装对象内部指向被捕获变量对象的这个指针是强还是弱根据被捕获对象用的内存管理修饰符决定，没有的情况默认是强引用，比如__block *person。
4. 所以ARC下当堆上lock描述性结构__main_block_impl_0被销毁时（自身引用计数为0），会先断开内部指向的包装对象（因为永远是强引用，所以引用计数-1），当包装对象被断开后（引用技术-1）如果引用计数为0也会进行释放，如果它内部是没有用内存修饰符的强指向被捕获对象时，被捕获对象也会被释放。多个Block内捕获__block对象并copy到堆上时，包装对象的释放受多个block共同决议。

## 单个Block中使用__block变量
当该Block从栈拷贝到堆上时，使用的所有__block变量也全部被从栈上拷贝到堆上。
![](Block%E8%A7%A3%E6%9E%90/1AE0FAE4-4256-49AF-B7D1-38E16F642A2F.png)

## 多个Block使用__block变量：
任何一个Block从栈上拷贝到堆上，__block变量就会一并从栈上拷贝到堆上并被该Block所持有。当剩下的Block从栈拷贝到堆上时，被拷贝的Block持有__block变量，并增加__block变量的引用计数。
![](Block%E8%A7%A3%E6%9E%90/4F37C8D1-3541-4D19-976C-0AD682C7E71A.png)


## __block修饰符理解和捕获局部auto变量的理解测试
1. 捕获的auto变量是拷贝（指针是拷贝指针，基本类型是拷贝值）oc层面不能直接修改，但是也可以在内部通过指针的形式直接修改。 ![](Block%E8%A7%A3%E6%9E%90/3ED5689D-3301-4B59-9976-DA51E25A273F.png)
2. ::__block修饰符的作用是把变量包成对象，相当于可以把内外部变量看成是同一个变量（forward指针的原因），并且内部是可修改的（copy时被复制到堆上），如果是基本数据类型原来外部的变量其实会被新变量替换掉，比如下面外部变量的地址在发生捕获后马上进行了替换改变。::
![](Block%E8%A7%A3%E6%9E%90/900BE9CD-6F41-4C49-86F2-157E87C33CDC.png)

![](Block%E8%A7%A3%E6%9E%90/F88857A1-6A1E-41E1-A2AB-DF27782A47B6.png)

# 例子理解
![](Block%E8%A7%A3%E6%9E%90/C2CCE887-1587-44B7-A188-317FB2ABBD7A.png)
ARC环境中，block作为GCD API的方法参数时会自动进行copy操作，因此block在堆空间，并且使用强引用访问person对象，因此block内部copy函数会对person进行强引用。当block执行完毕需要被销毁时，调用dispose函数释放对person对象的引用，person没有强指针指向时才会被销毁。

![](Block%E8%A7%A3%E6%9E%90/9D35C20A-DB06-4BE6-8CC1-A77B0DB840E8.png)

# __weak、__strong
https://www.jianshu.com/p/501af50cd2d9
1. strong，weak 用来修饰属性。
2. __weak, __strong 用来修饰变量。

## __weak
有时在使用block 的时候，由于self 是被强引用的，在 ARC 下，当编译器自动将代码中的block从栈拷贝到堆时，block 会强引用和持有self，而 self 恰好也强引用和持有了 block，就造成了传说中的循环引用。
此时 __weak 就出场了，在变量声明时用 __weak修饰符修饰变量 self，让 block 不强引用 self，从而破除循环。
``` objectivec
__weak __typeof(self) weakSelf = self; 
self.testBlock = ^{
       [weakSelf test];
});
```

## __strong
在上述使用 block中，虽说使用__weak，但是此处会有一个隐患，你不知道 self 什么时候会被释放，为了保证在block内不会被释放，我们添加__strong。（这里self不存在不能释放的问题，因为strongSelf是一个栈上的临时变量，被回收时self引用计数-1，完成释放）
``` objectivec
__weak __typeof(self) weakSelf = self; 
self.testBlock =  ^{
       __strong __typeof(weakSelf) strongSelf = weakSelf;
       [strongSelf test]; 
});
```

## block需要构造循环引用
你需要构造一个循环引用，以便保证引用双方都存在。比如你有一个后台的任务，希望任务执行完后，通知另外一个实例。但一定记得最后要主动断开。




