# Xcode报错解决

ld: symbol(s) not found for architecture x86_64
clang: error: linker command failed with exit code 1 (use -v to see invocation)

编译问题，看是否导入的库和文件，或者对应的文件删除重新加入引用

[加载Swift Framework报错：Module compiled with Swift 5.x cannot be imported by the Swift 5.3 compiler - 简书](https://www.jianshu.com/p/9aa36d7c1a6b)
一般是旧Xcode编译出来的swift库在新Xcode上跑会有这个问题，因为编译库时swift编译器版本不同导致的兼容问题（tm的不是稳定了吗），需要在旧Xcode上库build的时候设置将“Build Libraries for Distribution”选项设置为“YES”（cocoapod管理库要在pod的工程上设置），否则Swift编译器不会生成必要的.swiftinterface文件，这是将来新编译器能够加载旧库的关键。

但是又会有另外的报错说：may have used features that aren't supported by this compiler

所以最好就是：下载安装适用于您的特定Xcode版本的Xcode Toolchain。[https://swift.org/download/#releases](https://links.jianshu.com/go?to=https%3A%2F%2Fswift.org%2Fdownload%2F%23releases) ，然后打开Xcode的首选项，Components > Toolchains ，然后选择已安装的Swift工具链。现在，您可以编译并运行该应用程序。


[获取UITableview刷新完成状态 - 简书](https://www.jianshu.com/p/2069732e3038)

iOS14开始，用UITableView和UICollection的Cell时，应使用contentView做子视图的添加。

在viewDidLoad时对tableView做了RxSwift相关的数据绑定, 而RxSwift的相关Api会在绑定时调用tableView的layoutIfNeeded, 所以触发了这个警告.
[追踪一个控制台警告 — UITableViewAlertForLayoutOutsideViewHierarchy](https://juejin.cn/post/6844904111536603143)