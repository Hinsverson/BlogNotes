# Snapkit


[SnapKit 源码阅读. SnapKit 源码解读 | by Creator zkh | Medium](https://medium.com/@zkh90644/snapkit-%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB-ffe398cc7fdd)
[读 SnapKit 和 Masonry 自动布局框架源码 · 戴铭的博客 - 星光社](https://ming1016.github.io/2018/04/07/read-snapkit-and-masonry-source-code/)

![](Snapkit/058AD9FC-3BA6-4D73-86D7-CB48E1F7130C.png)

### 总结
* snp 通过 struct 进行定义，避免了循环引用，同时减少了因为增加前缀导致的不优雅。
* 基于协议和别名，对于所有对象进行统一的收拢，方便代码的实现。
* 想要做到 option 直接使用 OptionSet 即可。
* @discardableResult 可以去除 return 的值没有被调用的 warning
* 整个逻辑大概如下: snp 进行命名空间的统一收拢，然后交由 maker ，使用方通过调用 maker 链式创建对象，不断的添加对应的值。最后将所有的数值和属性交由 maker 的 description 进行描述，并通过懒加载的方式创建 constraint，最后通过便利将 description 中的 constraint 中的已经生成的系统方法约束设置到对应 view 上面。通过 maker 统一进行激活和去活。最后完成约束创建。

### 作用对象
Snapkit里约束的设置对象是ConstraintView，一个UIView和NSView的typealias对象。
对ConstraintView做了public扩展，定义了一个`snp: ConstraintViewDSL`计算属性，它是一个结构体类ConstraintViewDSL，初始化时会持有 ConstraintView，这里因为使用结构体而不会循环引用。这个类提供makeConstraints、remakeConstraints、updateConstraints、removeConstraints、设置contentHugging等能力。
``` swift
public extension ConstraintView {
    public var snp: ConstraintViewDSL {
        return ConstraintViewDSL(view: self)
    }
}
```

### 约束设置过程
首先是ConstraintViewDSL的makeConstraints，最终会创建一个ConstraintMaker对象（初始化时会统一禁用AutoresizeMask），传到平时外面写的closure内进行执行，执行的过程就是约束的设置过程，一条make.语句产生一个约束描述对象ConstraintDescription。

首先maker.center、left、right等属性是一个ConstraintMakerExtendable的计算属性，外部调用时触发构造一条ConstraintDescription并添加到内部持有的descriptions数组中进行保存，完了传入ConstraintDescription构造出ConstraintMakerExtendable对象。因为ConstraintMakerExtendable类内部又有left、top等同类型计算属性，再次调用这些属性的过程中会进行ConstraintMakerExtendable内部attributes的添加操作同时返回self（这也是外部链式调用的原因）。

> ConstraintAttributes类是一个OptionSet的移位枚举类，目的是将多个枚举选项组成一个值的枚举，ConstraintAttributes 里的 edges，size 和 center 等就是组合而成的（比如width=64、height=128，size=192）。[OptionSet和移位枚举](https://swift.gg/2016/10/25/swift-option-sets/)  

外部后续的操作都是在对ConstraintDescription这个约束描述类进行属性的赋值填充：
	* ConstraintMakerExtendable继承ConstraintMakerRelatable，提供外部调用equalTo等方法指定约束关系比，最终返回一个ConstraintMakerEditable对象。
	* ConstraintMakerEditable 继承 ConstraintMakerPriortizable，提供外部设置约束的 offset 和 inset 还有 multipliedBy 和 dividedBy 函数。ConstraintMakerPriortizable 继承 ConstraintMakerFinalizable，提供外部设置优先级，返回 ConstraintMakerFinalizable 类型的实例。
	* ConstraintMakerFinalizable持有的ConstraintDescription 的属性类是一个完整的约束描述，有了这个描述就可以做后面的处理了。里面的内容是完整的，这个类是一个描述类，用于描述一条具体的约束，包含了包括 ConstraintAttributes 在内的各种与约束有关的元素，一个 ConstraintDescription 实例，就可以提供与一种约束有关的所有内容。

### 约束的激活
构建完所有的约束des后，ConstraintMaker对象从descriptions数组中取出所有的des，然后取出所有ConstraintDescription的constraint属性，这是一个lazy属性，该约束包装类初始化时就会根据self（ConstraintDescription约束描述类）创建并持有系统的原生约束。最后对所有约束做active激活并将其添加到对应view的约束集合里（SnapKit层面通过关联对象实现存储）。

``` swift
public func makeConstraints(_ closure: (_ make: ConstraintMaker) -> Void) {
    ConstraintMaker.makeConstraints(item: self.view, closure: closure)
}

internal func makeExtendableWithAttributes(_ attributes: ConstraintAttributes) -> ConstraintMakerExtendable {
    //构建一条des描述
    let description = ConstraintDescription(item: self.item, attributes: attributes)
    //make收集des描述
    self.descriptions.append(description)
    return ConstraintMakerExtendable(description)
}

internal static func prepareConstraints(item: LayoutConstraintItem, closure: (_ make: ConstraintMaker) -> Void) -> [Constraint] {
    let maker = ConstraintMaker(item: item) //
    closure(maker) //make构建完所有的des，并收集到descriptions中
    
    //取出每一条description中的懒加载constraint，constraint初始化时创建持有系统的约束。
    var constraints: [Constraint] = []
    for description in maker.descriptions {
        guard let constraint = description.constraint else {
            continue
        }
        constraints.append(constraint)
    }
    return constraints
}

internal static func makeConstraints(item: LayoutConstraintItem, closure: (_ make: ConstraintMaker) -> Void) {
    let constraints = prepareConstraints(item: item, closure: closure)
    for constraint in constraints {
        //将所有已经创建好的约束进行激活，并将其添加到对应view的约束集合里（关联对象实现存储）。
        constraint.activateIfNeeded(updatingExisting: false)
        //布局操作完成。
    }
}
```