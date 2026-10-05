# OC和C++混编



[swift c++ 混编](https://juejin.cn/post/6844904022344728590)
1. 通过module把C++代码封装成模块导入。
2. OC++再做桥接。

特别注意OC和C++对象的内存管理，OC里的bridge使用。

[ARC 下 C++/OC 混编计数器的问题_chaoyuan899的专栏-CSDN博客](https://blog.csdn.net/chaoyuan899/article/details/72346775)

[混编ObjectiveC++ | 折腾范儿の味精](https://awhisper.github.io/2016/05/01/%E6%B7%B7%E7%BC%96ObjectiveC/)
* h 需要保持各自的数据结构，保证纯正的C++ 或 OC。
* 将需要混编引用的.m 或者 .cpp文件的后缀修改为 .mm ，告诉编译器 这个文件可以进行混编 — ObjectiveC++。
* Objective-C 和 C++ 都是向下兼容 C的，可以通过灵活的使用 void *指针 作为纽带在两者之间传递对象。
* Objective-c 可以通过工程的Build Settings/Apple Clang - Language - Objective-C/ Objective-C Automatic Reference Counting 来配置模式内存管理为 ARC（特定文件配置标示：-objc-arc） 或则 MRC（特定文件配置标示：-fno-objc-arc），但是 C++ 的内存管理是谁创建谁释放的原则；故在一个项目如果要使用多种语言，尽量的将两种语言分开封装，各自内部维护内存；


在 OC 中调用 C++ 代码时，需要将 OC 代码所在的 .m 文件后缀名修改为 .mm。
在 OC 的 .mm 文件中调用 C 代码，需要将 C 代码所在的文件后缀名(通常为 .c)修改为 .mm，或者建一个中间的OC类调用C代码即可。
有时候可能需要在 Build Settings -> Other Link Flags 添加 -lstdc++。
甚至可能需要导入C++系统库libstdc++.tbd

作者：KODIE
链接：https://www.jianshu.com/p/e905610c8e78
來源：简书
简书著作权归作者所有，任何形式的转载都请联系作者获得授权并注明出处。

oc可直接调用c

c++调用c，c这文件在c和c++中都能用
``` c
#ifdef __cplusplus
extern "C" {
    // 如果被C++性质的文件包含了，则需要这样声明这个函数，这样他会从C性质的文件中寻找
#endif
#ifdef __cplusplus
    void func1(); //如果是被C性质文件包含，则直接声明，不执行ifdef的内容
}
#endif
```

c调用c++的话，需要把c++的函数
```c
extern "C" void func2() { // 现在这就是个C性质的函数了，可以在C性质文件中使用了。
    func1(); // 这是C性质函数
    printf("C++");
}
```