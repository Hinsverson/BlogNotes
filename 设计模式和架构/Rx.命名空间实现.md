# Rx.命名空间实现
#iOS知识点/设计模式和架构

# 实现类似Snapkit，Rxswift的命名空间调用
https://toutiao.io/posts/r5a6k2/preview
``` swift
//代表了支持 namespace 形式的扩展。并紧接着给这个协议 extension 了默认实现。这样实现了这个协议的类型就不需要自行实现协议所约定的内容了。
public protocol NamespaceWrappable {
    associatedtype WrapperType
    var namespace: WrapperType { get }
    static var namespace: WrapperType.Type { get }
}
public extension NamespaceWrappable {
    var namespace: NamespaceWrapper<Self> {
        return NamespaceWrapper(value: self)
    }
    static var namespace: NamespaceWrapper<Self>.Type {
        return NamespaceWrapper.self
    }
}

public protocol TypeWrapperProtocol {
    associatedtype WrappedType
    var wrappedValue: WrappedType { get }
    init(value: WrappedType)
}
public struct NamespaceWrapper<T>: TypeWrapperProtocol {
    public let wrappedValue: T
    public init(value: T) {
        self.wrappedValue = value
    }
}

// 使用举例
extension String: NamespaceWrappable { }
extension TypeWrapperProtocol where WrappedType == String {
    var test: String {
        return wrappedValue
    }
}
let testStr = "foo".namespace.test
print(testStr)
```
**注意**
在对TypeWrapperProtocol这个协议做 extension 时， where 后面的WrappedType约束可以使用==或者:，两者是有区别的。如果扩展的是值类型，比如 String，Date 等，就必须使用==，如果扩展的是类，则两者都可以使用，区别是如果使用==来约束，则扩展方法只对本类生效，子类无法使用。如果想要在子类也使用扩展方法，则使用:来约束。

由于 namespace 相当于将原来的值做了封装，所以如果在写扩展方法时需要用到原来的值，就不能再使用self，而应该使用wrappedValue。
