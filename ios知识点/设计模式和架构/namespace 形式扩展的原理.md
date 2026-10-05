# namespace 形式扩展的原理

[swift kf_xxx 转向了kf.xxx风格 - 简书](https://www.jianshu.com/p/1383fc662533?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)

# 实现类似Snapkit，Rxswift的命名空间调用
1. 定义一个抽象protocol类NamespaceWrappable声明一个前缀变量WrapperType（比如rx），然后extension默认扩展NamespaceWrappable的前缀变量，把当前self对象作为NamespaceWrapper泛型类包裹的具体对象传入，构造出NamespaceWrapper实例。这样之后对任意类做NamespaceWrappable遵循实现后，就能通过前缀变量构造访问到NamespaceWrapper<self>类来进行一些处理。
2. 一般还会对NamespaceWrapper<T>这个泛型类做一层TypeWrapperProtocol抽象，这样可以针对T做类型约束，使得不同的包装类型T在NamespaceWrapper处理上具备不同的行为。

https://toutiao.io/posts/r5a6k2/preview
``` swift
//1.命名空间形式的扩展抽象类NamespaceWrappable
public protocol NamespaceWrappable {
    associatedtype WrapperType
    var hk: WrapperType { get }
    static var hk: WrapperType.Type { get }
}
//对NamespaceWrappable做默认扩展，调用前缀hk后就返回这个包装对象
public extension NamespaceWrappable {
    var hk: NamespaceWrapper<Self> {
        return NamespaceWrapper(value: self)
    }
    static var hk: NamespaceWrapper<Self>.Type {
        return NamespaceWrapper.self
    }
}

//平常写的范型类，这里只不过用TypeWrapperProtocol再抽象了一层
//这样通过.namespace访问时，能根据不同类做约束
public struct NamespaceWrapper<T>: TypeWrapperProtocol {
    public let wrappedValue: T
    public init(value: T) {
        self.wrappedValue = value
    }
}
public protocol TypeWrapperProtocol {
    associatedtype WrappedType
    var wrappedValue: WrappedType { get }
    init(value: WrappedType)
}

//对某个类做hk空间前缀能力的扩展
extension String: NamespaceWrappable { }
//约束通过hk访问到的这个类的一些行为
extension TypeWrapperProtocol where WrappedType == String {
    var test: String {
        return wrappedValue
    }
}
let testStr = "foo".hk.test
print(testStr)
```
1. 在对TypeWrapperProtocol这个协议做 extension 时， where 后面的WrappedType约束可以使用==或者:，两者是有区别的。如果扩展的是值类型，比如 String，Date 等，就必须使用==，如果扩展的是类，则两者都可以使用，区别是如果使用==来约束，则扩展方法只对本类生效，子类无法使用。如果想要在子类也使用扩展方法，则使用:来约束。
2. 由于 namespace 相当于将原来的值做了封装，所以如果在写扩展方法时需要用到原来的值，就不能再使用self，而应该使用wrappedValue。
3. 下面kf的实现也是这样，只不过简单一点，少了TypeWrapperProtocol
``` swift
// MARK: 申明了泛型类Kingfisher 实现了一个简单构造器,final修饰不可继承，不做任何实际的操作
public final class Kingfisher<Base> {
    public let base: Base
    public init(_ base: Base) {
        self.base = base
    }
}
/**
 A type that has Kingfisher extensions.
 */
// MARK: 这段代码定义了一个协议，然而协议是不支持泛型的，只能用assocaitedtype这个关键字来声明一个类型。
public protocol KingfisherCompatible {
    associatedtype CompatibleType
    var kf: CompatibleType { get }
}
//MARK: 实现里面这个属性get方法里面返回的就是Kingfisher<Base>这个类
public extension KingfisherCompatible {
    var kf: Kingfisher<Self> {
        get { return Kingfisher(self) }
    }
}
extension UIImageView: KingfisherCompatible {}
```


4. 类似的Alamofire中如下
``` swift
/// Type that acts as a generic extension point for all `AlamofireExtended` types.
public struct AlamofireExtension<ExtendedType> {
    /// Stores the type or meta-type of any extended type.
    public private(set) var type: ExtendedType

    /// Create an instance from the provided value.
    ///
    /// - Parameter type: Instance being extended.
    public init(_ type: ExtendedType) {
        self.type = type
    }
}

/// Protocol describing the `af` extension points for Alamofire extended types.
public protocol AlamofireExtended {
    /// Type being extended.
    associatedtype ExtendedType

    /// Static Alamofire extension point.
    static var af: AlamofireExtension<ExtendedType>.Type { get set }
    /// Instance Alamofire extension point.
    var af: AlamofireExtension<ExtendedType> { get set }
}

extension AlamofireExtended {
    /// Static Alamofire extension point.
    public static var af: AlamofireExtension<Self>.Type {
        get { AlamofireExtension<Self>.self }
        set {}
    }

    /// Instance Alamofire extension point.
    public var af: AlamofireExtension<Self> {
        get { AlamofireExtension(self) }
        set {}
    }
}
```