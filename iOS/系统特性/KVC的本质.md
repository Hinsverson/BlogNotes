# KVC的本质
#iOS知识点/系统特性

[iOS底层-KVC使用实践以及实现原理 - 简书](https://www.jianshu.com/p/fbd1e7c93fd0)
[iOS《Key-Value Coding Programming Guide》官方文档](https://juejin.cn/post/6915290445069156366)

# KVC 的概念：
由NSKeyValueCoding非正式协议启用的一种机制，对象采用这种机制来提供对其属性的间接访问，核心是runtime提供的支持。

![](KVC%E7%9A%84%E6%9C%AC%E8%B4%A8/7448DD7F-1E70-47C3-8801-470E76FC1A72.png)
setVaule forkeyPath  和 setVaule forkey 的区别：
在于前者更强大，可以通过类似key1.key2路径访问当前key对象的成员对象，并对其赋值。

# KVC设值原理
![](KVC%E7%9A%84%E6%9C%AC%E8%B4%A8/FEA67360-DFE2-4626-BA1F-29C8ADDBD084.png)
1. 先按顺序找setKey、_setKey方法，如果有的话调用并直接赋值
2. 没有找到的话，会查看accessInstanceVariableDirecty的返回值判断是否允许直接访问成员变量（默认为YES）。
3. 不允许直接访问的话会抛出异常
4. 允许的话会按顺序寻找_key、_is_Key、key、isKey成员变量
5. 找到的话直接赋值，找不到则抛出异常

## KVC修改属性时如果该属性被KVO观察的话会触发KVO？如果找到的是成员变量并赋值会触发KVO吗？
::都会，因为KVC也是运行时发生的，此时修改的属性调用的就是派生类中的set实现，会触发KVO。::

即使是直接找到成员变量并赋值修改，也会触发，因为实际实现上在找到成员变量并赋值时会调用willChangeValueForKey()、didChangeValueForKey()方法，通知监听对象。而KVO里set方法的内部调用本质也是如此，所以即使没有set方法，KVC修改成员变量还是会触发KVO（相当于手动触发）。

# KVC取值原理
![](KVC%E7%9A%84%E6%9C%AC%E8%B4%A8/785B5ED4-8008-4CD9-831B-1AD93E8626DF.png)
1. 按先后顺序搜索getKey:、key、isKey、_isKey方法，若某一个方法被实现，取到的即是方法返回的值。
2. 若没找到，则会查看accessInstanceVariablesDirectly返回值判断是否允许取成员变量的值（默认为YES）。
3. 不允许直接访问的话会抛出异常。
4. 允许的话会按顺序寻找_key、_is_Key、key、isKey成员变量
5. 找到的话直接取值，找不到则抛出异常。

# Swift KVC原理
[KeyPath在Swift中的妙用 - 掘金](https://juejin.im/post/5bf4055af265da616b105331)
![](KVC%E7%9A%84%E6%9C%AC%E8%B4%A8/7DF54D72-81F6-4D53-8371-4451DC13EC90.png)
Swift的KVC实际是通过KeyPath语法糖实现，本质是Swift会为每个实例对象创建一个下标方法，并通过`[keyPath: …]`在下标方法的get或者set内得到或者修改成员变量的值。

> Swift下标参考：[下标 - SwiftGG](https://swiftgg.gitbook.io/swift/swift-jiao-cheng/12_subscripts)   
> 下标可以定义在类、结构体和枚举中，是访问集合、列表或序列中元素的快捷方式。可以使用下标的索引，设置和获取值，而不需要再调用对应的存取方法。举例来说，用下标访问一个 Array 实例中的元素可以写作 someArray[index]，访问 Dictionary 实例中的元素可以写作 someDictionary[key]。  
>   
>  一个类型可以定义多个下标，通过不同索引类型进行重载。下标不限于一维，你可以定义具有多个入参的下标满足自定义类型的需求。  

## Swift中KeyPath的理解
1. KeyPath: 提供对属性的只读访问
2. WritableKeyPath: 提供对具有价值语义的可变属性提供可读可写的访问
3. ReferenceWritableKeyPath: 只能对引用类型使用(比如一个类的实例), 对任意可变属性提供可读可写的访问
```
class Dog {
    var name: String
    init(name: String) {
        self.name = name
    }
}
//ketPathDogName类型为ReferenceWritableKeyPath<Dog, String>
let dog = Dog(name: “xiaobai”)
let ketPathDogName = \Dog.name
```
Keypaths可作为对象进行拼接操作，并且类型时可以推断出来的
```
let authorKeyPath = \Book.primaryAuthor
type(of: authorKeyPath)
// You can omit the type name if the compiler can infer it
let nameKeyPath = authorKeyPath.appending(path: \.name)
book[keyPath: nameKeyPath]
```
Keypaths内部可进行下标操作
```
// Now possible in Swift 4.0.3
book[keyPath: \Book.authors[0].name]
```
下标可以像路径一样往下取
```
book[keyPath: \Book.title]
// Key paths can drill down multiple levels
// They also work for computed properties
book[keyPath: \Book.primaryAuthor.name]
```

