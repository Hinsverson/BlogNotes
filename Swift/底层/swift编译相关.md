# swift编译相关
#Swift/底层解析 

# LLVM
![](swift%E7%BC%96%E8%AF%91%E7%9B%B8%E5%85%B3/2AA85228-18A4-4EB9-8B12-49AB82429B36.png)

![](swift%E7%BC%96%E8%AF%91%E7%9B%B8%E5%85%B3/3147AC3E-BACF-4A64-859E-7988CE47F82F.png)

![](swift%E7%BC%96%E8%AF%91%E7%9B%B8%E5%85%B3/17AB2027-FD23-4264-824A-40046516E2E0.png)

![](swift%E7%BC%96%E8%AF%91%E7%9B%B8%E5%85%B3/D038C4FB-6EBC-4610-BEDD-615FAD1346F4.png)

![](swift%E7%BC%96%E8%AF%91%E7%9B%B8%E5%85%B3/CC98FAFD-240C-43F8-AFA6-42658CD2349E.png)

# 编译过程

![](swift%E7%BC%96%E8%AF%91%E7%9B%B8%E5%85%B3/32160683-F081-4BD9-8BF6-536C43CE053C.png)

1.预处理：拷贝#include、import头文件内容，替换宏定义
2.编译：编译成中间IR代码
3.后端：编译成汇编代码
4.assembler：编译成目标代码
5.链接：链接其他动态库
6.blind-arch：生成对应架构机器码 

![](swift%E7%BC%96%E8%AF%91%E7%9B%B8%E5%85%B3/933EF1BA-F0CA-4A18-884C-B9603D71A400.png)
词法分析把代码分割成很多个token，附带行、位信息（第几行第几个字符）


![](swift%E7%BC%96%E8%AF%91%E7%9B%B8%E5%85%B3/A451FCB1-BFEB-4AA4-91F1-DB4009AED339.png)
![](swift%E7%BC%96%E8%AF%91%E7%9B%B8%E5%85%B3/301ED1F7-286B-4453-BF8A-6DD8408AAD29.png)

![](swift%E7%BC%96%E8%AF%91%E7%9B%B8%E5%85%B3/C13C33A7-A17B-4BC5-BB34-4E32B9B3524D.png)

![](swift%E7%BC%96%E8%AF%91%E7%9B%B8%E5%85%B3/BFB7B7C8-7C9F-4439-8DD8-5D93427AA153.png)
 [https://llvm-tutorial-cn.readthedocs.io/en/latest/index.html](https://llvm-tutorial-cn.readthedocs.io/en/latest/index.html) 


[深入浅出iOS编译 - 掘金](https://juejin.im/post/5c22eaf1f265da611b5863b2)

编译器分为前端和后端

前端负责词法分析，语法分析，生成中间代码。
后端以中间代码作为输入，进行行架构无关的代码优化，接着针对不同架构生成不同的机器码。

# iOS 编译流程
 [https://swift.org/compiler-stdlib/#standard-library-design](https://swift.org/compiler-stdlib/#standard-library-design) 
![](swift%E7%BC%96%E8%AF%91%E7%9B%B8%E5%85%B3/7E60DDB1-3695-4144-9D4F-4F63DE0EA3F3.png)
![](swift%E7%BC%96%E8%AF%91%E7%9B%B8%E5%85%B3/692FEF4E-0E6A-4B5C-9C08-A3BAA613C235.png)
Objective C/C/C++使用的编译器前端是 [clang](https://clang.llvm.org/docs/index.html) ，swift是 [swift](https://swift.org/compiler-stdlib/#compiler-architecture) ，后端都是 [LLVM](https://llvm.org/) 。

**编译：生成目标文件的过程**
1. 前端预处理：进行头文件引入，宏替换，注释处理，条件编译ifdef等操
2. 前端进行词法和语法分析：生成 AST 。AST 是抽象语法树，结构上比代码更精简，遍历起来更快，所以使用 AST 能够更快速地进行静态检查，同时还能更快地生成 IR（中间表示）。**AST是开发者编写clang插件主要交互的数据结构**，clang也提供很多API去读取AST。更多细节： [Introduction to the Clang AST](https://clang.llvm.org/docs/IntroductionToTheClangAST.html) 。
3. LLVM 后端IR代码优化：优化会调用相应的Pass进行处理。Pass由多个节点组成，都是 [Pass](http://llvm.org/doxygen/classllvm_1_1Pass.html) 类的子类，每个节点负责做特定的优化，更多细节： [Writing an LLVM Pass](https://llvm.org/docs/WritingAnLLVMPass.html) 。
4. LLVM后端生成汇编代码：LLVM对IR进行优化后，会针对不同架构生成不同的目标代码，最后以汇编代码的格式输出arm 64、x86等。
5. 汇编器生成机器码：汇编器以汇编代码作为输入，将汇编代码转换为机器代码，最后输出目标.o文件。

**链接）：内置链接器（lld）将项目中的多个可执行文件（代码和静态库）合并成一个Mach-O文件的过程。**
1. 符号解析。
	1. 查找目标文件里没有定义的undefined 变量。
	2. 扫描项目中的不同目标文件，将所有符号定义和引用地址收集起来，并放到全局符号表中。
	3. 计算合并长度及位置，生成同类型的段进行合并。
2. 地址重定位。
> 链接器在整理函数的调用关系时，会以main函数为源头，跟随每个引用，并将其标记为live。跟随完成后，那些未被标记live的函数，就是无用函数。然后，链接器可以通过打开 Dead code stripping 开关，来开启自动去除无用代码的功能。并且，这个开关是默认开启的。  

**动态链接：iOS dyld在程序启动时进行动态库的链接**
启动后加载 Mach-O 文件，根据 Mach-O 文件里 undefined 的动态库符号，dlopen 递归加载用到的动态库，dlsym 得到符号地址完成调用绑定。
系统还会设置一个共享缓存库，在App进程的地址空间映射这些共享缓库，优化App启动速度的作用。

**动态链接：iOS dyld在程序运行时时进行动态库的链接**
通过动态链接器提供的 API dlopen 和 dlsym 来加载。这种方式，不允许上线 App Store 。

## 实际编译顺序
![](swift%E7%BC%96%E8%AF%91%E7%9B%B8%E5%85%B3/A87E72B1-8F5F-4E4C-8E3C-E7F7DEB2D010.png)
创建Product.app的文件夹
把Entitlements.plist写入到DerivedData里，处理打包的时候需要的信息（比如application-identifier）。
创建一些辅助文件，比如各种.hmap，这是headermap文件，具体作用下文会讲解。
执行CocoaPods的编译前脚本：检查Manifest.lock文件。
编译.m文件，生成.o文件。
链接动态库，o文件，生成一个mach o格式的可执行文件。
编译assets，编译storyboard，链接storyboard
拷贝动态库Logger.framework，并且对其签名
执行CocoaPods编译后脚本：拷贝CocoaPods Target生成的Framework
对Demo.App签名，并验证（validate）
生成Product.app

# iOS 编译热更新
**Flutter 是怎么实现实时编译**
点击 reload 时去查看自上次编译以后改动过的代码，重新编译涉及到的代码库，还包括主库，以及主库的相关联库。
所有这些重新编译过的库都会转换成内核文件发到 Dart VM 里，Dart VM 会重新加载新的内核文件。
加载后会让 Flutter framework 触发所有的Widgets 和 Render Objects 进行重建、重布局、重绘。

参考实现：
https://github.com/johnno1962/InjectionIII 

原理和思路：
![](swift%E7%BC%96%E8%AF%91%E7%9B%B8%E5%85%B3/8E74077E-B50A-4DB9-B3CF-E2F03651727B.png)
1. Injection 会监听源代码文件的变化，如果文件被改动了，Injection Server 就会执行 rebuildClass 重新进行编译、打包成动态库，也就是 .dylib 文件。
2. Server 会在后台发送和监听 Socket 消息，编译、打包成动态库后使用 writeSting 方法通过 Socket 通知运行的 App。
3. Client 接收到消息后会调用 inject(tmpfile: String) 方法，运行时通过dlopen进行类的动态替换。

# SwiftC 常用指令
![](swift%E7%BC%96%E8%AF%91%E7%9B%B8%E5%85%B3/96FB9215-7B6D-473E-BC7F-A22819D885B1.png)
[Swift编译流程 & Swift类 - 简书](https://www.jianshu.com/p/e917bf0e8a7d)