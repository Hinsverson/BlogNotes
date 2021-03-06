# C基础总结
#C和C++

# 数组和指针
1. ::自动存储类别的变量（数组和其他类型的变量都是这样）使用前需要初始化，否则使用的值是内存地址上的现有垃圾值（未知）::
2. sizeOf计算的是运算对象的字节大小（对象是数组就是整个数组的大小），编译器特性。
3. C编译器为了效率普遍不会进行数组越界检查，高级语言一般也是在运行时去检查。
4. 变长数组必须是自动存储类别，本质只是可以使用变量值来申明大小，变长数组一旦创建大小也是不可变的。可以理解为是一种动态内存分配运行时再决议大小，确定后不变。普通数组大小是常量所以编译器就确定了大小。高级语言变长数组一般通过动态扩容实现。
5. ::多维数组a[m,n]理解为m行n列。::
6. 数组名和指针没区别，+1操作都是地址往后偏移一个存储单元（大小由类型决定）， 指针表示*(a+i)和数组表示a[i]是等价的。
7. int *a和int a[]等效，但是后者只能用于形式参数，处理数组的函数用指针作为参数。
8. 理解total += *start++是先total加上*start的值，然后start指针再自增。
9. 指针操作：
	1. 赋值：int *ptr = &a
	2. 解引用：*运算符给出指针地址上存储的值
	3. 指针加减和递增递减：ptr+4和&ptr[4]等价，ptr++等于地址向后偏移类型大小个单位长度，减法同理。
	4. 指针求差：ptr1-ptr（类型都是int）得2的意思是指针地址间隔了2个int类型的大小。
	5. 指针的递增自减等计算和数组一样不会进行越界检查。
10. ::解引用未初始化的指针是非常危险的，相当于修改了未知处的值。::
11. 对函数形式参数使用const int[]可以防止原数组被修改，编译器检查。
12. 把const指针给非const指针是不安全的，因为就能修改原const的数据了。
13. 理解const double * const ptr = a（a定义为 int a[5]）：
	1. 第一个const是不能修改ptr指向内存块的值
	2. 第二个const是不能修改ptr的指向（无法再次 ptr = &b）
14. 如果使用一个指针指向2纬数组，最好是用数组表示法，用指针表示会比较麻烦和容易出错。例如：zip[2,1] 等价于 *(*(zip+2)+1)

# 字符串
1. ::字符串字面常量属于静态存储类别，存储在数据段，整个程序生命周期内都存在只有一份。::
2. 字符串末尾用‘\0’代表。
3. 初始化数组在运行时把静态存储区的字符串拷贝到数组中，初始化指针只把字符串的地址拷贝给指针，因为是静态存储区所以用指针形式初始化字符串一般是const，不能修改指向的内容，比如const char * ptr = “this is a string”。可以想象在多维情况下字符指针数组效率比字符数组高，因为数组要拷贝。
4. ::向未初始化的字符串指针读入字符串是很危险的。::
5. 常见字符串输入输出函数的理解和区别（略）
6. 常见字符串处理函数：
	1. strlen：计算长度
	2. strcat：把第二个字符串拼接到第一个后面作为新的第一个字符串，第二个不变。
	3. strncat：比strcat多一个最大添加字符数的参数，防止拼接时的溢出。
	4. strcmp：比较返回1字符串和2字符串参数之间不同部分的ascii差值，所以0就表示是相同的字符串。
	5. strcpy：拷贝字符串到指向的位置。
	6. strncpy：比strcpy多一个最大添加字符数的参数，防止拷贝时的溢出。
7. main函数的第一个int类型参数argc指明命令行单词数量，第二个argv是指向char指针数组的指针，每个char指针指向命令行参数字符串，argv[0]指向命令名称，argv[2]指向第一个命令行参数。

# 存储类别
1. ::首先理解存储类别的定义是如何访问和描述对象在内存中的存储时间。::
	* 访问对象通过标示符，通过作用域和链接属性来标示在程序哪些部分可以使用它。
		1. 作用域：块、函数、函数原型、文件作用域
		2. 链接属性：外部、内部、无链接，描述翻译单元内的变量可被链接的程度。
		* 具有块、函数、函数原型作用域的变量都是无链接。文件作用域可以是外部（多文件中使用）或者内部链接（当前文件翻译单元内使用），取决于存储类别说明符，比如static（这里可以和程序编译完后链接器进行符号重定位进行类比，说的是同一个东西）。
	* 描述对象可以用存储期来描述，标示对象在内存中保留了多长时间。
		1. 静态存储器： 文件作用域变量具备静态存储器期，程序运行过程一直存在。这里要⚠️理解static申明的文件作用域变量只是表明了它的链接属性是内部的。
		2. 线程存储器：从被生命到线程结束，以_Thread_local申明的对象每个线程都获得该变量的私有备份。
		3. 自动存储器：一般是块作用域内的局部变量（栈）
2. ::理解5种存储类别::
![](C%E5%9F%BA%E7%A1%80%E6%80%BB%E7%BB%93/04C48ADC-88E3-4360-AB05-E0E282FAC9F9.png)
![](C%E5%9F%BA%E7%A1%80%E6%80%BB%E7%BB%93/F81C1EE3-FCF3-4484-B555-AFFA29530117.png)
auto：分布在栈上  
register：存储在寄存器上，（只是一种“请求”不一定就能存储在寄存器上，并且因为存储在寄存器中，访问更快但没有地址）  
static：静态存储期，块作用域内无链接，文件作用域内内部链接。
自动存储期变量未赋初值都不会自动赋初值，静态存储期的会自动赋0或者空字符。
自动存储期变量每次函数调用时会赋初值，分配内存，静态存储期变量编译时只赋1次，载入程序时分配内存。
函数也有存储类别，分为外部或者static静态函数（内部访问）。  
extern：用来进行引用式声明可外部链接的变量或者函数，表示这个外部变量或者函数定义在其他文件内，需要在当前文件作用域内使用。
3. malloc：申请一块堆空间并返回首地址
4. free：释放指针所指向的堆空间，重复释放非空指针会有安全问题，free空指针是安全的。
5. ::关于动态内存分配的理解就是内存的申请和释放由程序员控制并存储在堆区。区别于静态存储期的静态变量编译时确定存储在数据段，程序结束时销毁。临时自动变量运行时进入块内确定，离开销毁，存储在栈上。::
6. ::_Atomic(c11)用来在多线程条件下申明原子类型变量，变量的存取操作通过特殊的宏函数进行，并且存取操作时其他线程不能访问，保证线程安全。::
7. 类型限定符号const：在运行时不能改变。
8. volatile防止编译器进行优化，每次都让程序从内存中去读取数据。

# 结构体和其他数据形式
1. 访问结构体可以通过.下标或者指针->形式访问
2. 结构变量名不是结构的地址，和数组不同。
3. ::和其他类型一样，函数参数值传递是深拷贝。::
4. 结构体中用指针作为成员接受字符串输入要注意安全问题，换句话说，结构体内部申明的指针并不会初始化。
5. 伸缩型数组成员结构是说结构至少包含一个成员，最后一个成员是一个未标记大小的数组，使用上先申明结构指针再用malloc的方式为结构分配动态内存空间，这样数组和结构的大小可以动态控制。但要注意：
	1. 不要用该结构进行赋值或拷贝，确实需要应该用memcpy。
	2. 不要把该结构用作数组元素或其他结构的成员。
6. ::联合类型用同一块内存空间存储不同数据类型，大小由最大的成员决定，某一刻只能存储一种成员，适用于需要存储的成员没有顺序规律的情况。::
7. 枚举的本质是用符号名称表示的整形常量，默认大小从0开始。
8. 同一作用域C的结构、联合、枚举和普通变量名称空间不同，可以允许同名但是C++不允许，因为C++这些结构和普通变量的名称空间是一样的。
9. ::typedef用来为某一类型自定义别名，它的作用对象是类型，由编译器解释。注意和#define区分开，后者的作用对象并不是类型，由预处理器解释。它的使用范围受限于定义的作用域。::
10. ::理解下面的表示：::
	1. int board[8][8]：int数组。
	2. int **ptr：指向指针的指针。
	3. int * risks[10]：一个包含10个元素的数组，成员是指向int的指针。
	4. int (* risks)[10]：一个指向数组的指针，数组成员是10个int值。
	5. int * off[3][4]：一个3x4的数组，每个元素都是指向int的指针。
	6. int (* uuf)[3][4]：一个指向3x4二维数组的指针，数组元素为int。
	7. int (* uof[3])[4]：内涵3个指针元素的数组，每个指针指向一个包含4个int类型的数组。
11. void (*pf) (char *)：函数指针，该函数返回值为void，参数为char *，调用形式为(*pf) (“aaaaa”)，等价为pf(“aaaaa”)，可以理解为函数名整体就是一个指针。 函数名代表函数地址。

# C预处理器
1. C编译器翻译程序的第一步
	1. 源代码字符映射到源字符集（字符扩展让C更国际化）
	2. 替换反斜杠，把多个物理行变成一个逻辑行，因为预处理器只能按一个逻辑行处理。
	3. 把文本划分为预处理记号序列、空白序列、注释序列，用一个空白字符替换每一条注释，空白序列。
	4. 开始预处理阶段。
2. C的#define PX printf() 由3部分组成，一但预处理器在程序中找到宏示例，就会进行宏展开用替换体替换宏，预处理器不求值不做计算只是进行替换：
	1. 第一个#define为预处理指令
	2. PX为宏
	3. 替换体
3. 宏定义中的#运算符可以用来表示宏参数。
4. 宏定义中的##运算符可以用来表示宏替换部分。
5. …和__VA_ARGS用来表示多个参数的变参宏。
6. 牢牢记住宏的本质只是替换不进行运算，处理的是字符串而不是实际的值，宏的调用比函数快，适用于多次需要调用的情况，但是看起来很恶心🤢。
7. ::C的#include 预处理命令会把后面文件的内容拷贝到当前文件中替换该指令，该指令有2种形式：::
	1. ::第一种#include <stdio.h>：告诉预处理器在标准系统目录下查找文件::
	2. ::第二种#include “stdio.h”：告诉预处理器首先在当前目录找，没有再去标准系统目录下面找。::
	3. 第三种#include “/usr/biff/stdio.h”：在指定目录下找
8. includ拷贝的内容并不是最终内容，这里最终是按需拷贝，所以include大型文件不一定增加程序大小。
9. C的#pragma指令用于向编译器发出指令，进行条件编译。
10. C的#undef用于取消一个宏定义，无论该名称是否已经被定义过。
11. C的#ifdef表示如果已经定义
12. C的#ifndef表示如果未定义，通常用于防止文件被多次包含：
```
#ifndef A
		#define A
      // 文件内容XXXXXX
#endif
```
13. C11中新增_Generic范型表达式，一般结合宏定义来使用。
14. atexit()函数接受一个函数指针，注册清理函数，在exit执行前自动调用。
15. memcpy和memmove的区别在于前者带有restrict关键字（唯一访问），即memcpy假设2者之间没有重叠，memmove类似于先拷贝到临时缓冲区再拷贝到最终目的地，不会出现重叠问题。

# malloc和free的底层理解
https://azhao.net/index.php/archives/81/
1. 操作系统内核维护了一个向堆空间的某个地址的ptr，从堆start位置开始用来记录已经映射好的虚拟内存空间，通过系统调用函数sbrk()用来控制这个ptr的移动。相当于向系统进货，“批发”了一块较大的内存空间，然后“零售”给程序用，因为频繁的系统调用影响性能，当不够时才发起申请。
2. malloc() 和 free() 所做的工作主要是对已有内存块的分拆和合并，结构上一般有很多不同的实现，常见的是双向链表（性能不是很高），每一个chunk节点内存块里存储着前后块的指向信息、是否被使用、大小、以及实际返回的负载内存开始地址。
3. malloc时找标记为可用的空闲chunk（闲置），如果有能符合大小的就将标记改为使用，然后将剩余的分割为可用的chunk。free时根据指针和大小又重新合并前后空闲的chunk为一个大chunk。
4. 当然，实际上还可以根据需要分配的大小划分维护多个这种链表，解决查找效率的问题，或者其他高级数据结构去优化分配时的效率，实际上也是多种方式混合实现的。



