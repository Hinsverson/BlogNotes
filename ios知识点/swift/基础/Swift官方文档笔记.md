# Swift官方文档笔记


Double 精确度很高，至少有15位数字，而 Float 只有6位数字。选择哪个类型取决于你的代码需要处理的值的范围，在两种类型都匹配的情况下，将优先选择 Double。

当推断浮点数的类型时，Swift 总是会选择 Double 而不是 Float。

十进制浮点数也可以有一个可选的指数（exponent)，通过大写或者小写的 e 来指定；十六进制浮点数必须有一个指数，通过大写或者小写的 p 来指定。
1.25e2 表示 1.25 × 10^2，等于 125.0。
1.25e-2 表示 1.25 × 10^-2，等于 0.0125。
0xFp2 表示 15 × 2^2，等于 60.0。
0xFp-2 表示 15 × 2^-2，等于 3.75。

数值类字面量可以包括额外的格式来增强可读性。整数和浮点数都可以添加额外的零并且包含下划线，并不会影响字面量：
``` swift
let paddedDouble = 000123.456
let oneMillion = 1_000_000
let justOverOneMillion = 1_000_000.000_000_1
```

类型转换其实是一个初始化构造器。

在 Objective-C 中，nil 是一个指向不存在对象的指针。在 Swift 中，nil 不是指针——它是一个确定的值，用来表示值缺失。

fatalError(_:file:line:)不会被编译器优化，preconditionFailure(_:file:line:)使用 unchecked 模式（-Ounchecked）编译代码会被优化，assert(_:_:file:line:)生成阶段会被优化，都是是在运行时所做的检查。

Swift算术运算符（+，-，*，/，% 等）的结果会被检测并禁止值溢出，安全。

Swift 也提供恒等（===）和不恒等（!==）这两个比较符来判断两个对象是否引用同一个对象实例。


Swift每一个字符串都是由编码无关的 Unicode 字符组成，并支持访问字符的多种 Unicode 表示形式。

```swift
let sparklingHeart = "\u{1F496}" // 💖，Unicode 标量 U+1F496
```

 Swift 中的字符在一个字符串中并不一定占用相同的内存空间数量，Swift字符串通过 count 属性返回的字符数量并不总是与包含相同字符的 NSString 的 length 属性相同。NSString 的 length 属性是利用 UTF-16 表示的十六位代码单元数字，而不是 Unicode 可扩展的字符群集。

Substring适用于短期存储并共享原来的内存空间，可以使用String(SubString)来重新初始化一个String并存储在新的内存上。
![](Swift%E5%AE%98%E6%96%B9%E6%96%87%E6%A1%A3%E7%AC%94%E8%AE%B0/38DA15ED-DFC7-4FF9-83E5-0BC8F7113689.png)

访问字符的utf-16、utf-8、utf-32编码，依次使用utf8、utf16、unicodeScalars
```swift
for codeUnit in dogString.utf16 {
print("\(codeUnit) ", terminator: "")
}
print("")
// 68 111 103 8252 55357 56374
```

集合操作
```swift
let houseAnimals: Set = ["🐶", "🐱"]
let farmAnimals: Set = ["🐮", "🐔", "🐑", "🐶", "🐱"]
let cityAnimals: Set = ["🐦", "🐭"]

houseAnimals.isSubset(of: farmAnimals)
// true
farmAnimals.isSuperset(of: houseAnimals)
// true
farmAnimals.isDisjoint(with: cityAnimals)
// true
```

updateValue(_:forKey:) 方法会返回对应值类型的可选类型，如果有值存在于更新前，则这个可选值包含了旧值，否则它将会是 nil。
```swift
if let oldValue = airports.updateValue("Dublin Airport", forKey: "DUB") {
print("The old value for DUB was \(oldValue).")
}
// 输出“The old value for DUB was Dublin.”

```

使用标签（*statement label*）来标记一个循环体或者条件语句
```swift
gameLoop: while square != finalSquare {
    diceRoll += 1
    if diceRoll == 7 { diceRoll = 1 }
    switch square + diceRoll {
    case finalSquare:
        // 骰子数刚好使玩家移动到最终的方格里，游戏结束。
        break gameLoop
    case let newSquare where newSquare > finalSquare:
        // 骰子数将会使玩家的移动超出最后的方格，那么这种移动是不合法的，玩家需要重新掷骰子
        continue gameLoop
    default:
        // 合法移动，做正常的处理
        square += diceRoll
        square += board[square]
    }
}
print("Game over!")
```


全局函数是一个有名字但不会捕获任何值的闭包
嵌套函数是一个有名字并可以捕获其封闭函数域内值的闭包
闭包表达式是一个利用轻量级语法所写的可以捕获其上下文中变量或常量值的匿名闭包

Swift闭包是引用类型，这里捕获了 runningTotal 和 amount 变量的*引用*。捕获引用保证了 runningTotal 和 amount 变量在调用完 makeIncrementer 后不会消失，并且保证了在下一次执行 incrementer 函数时，runningTotal 依旧存在。
```swift
func incrementer() -> Int {
runningTotal += amount
return runningTotal
}

incrementByTen()
// 返回的值为10
incrementByTen()
// 返回的值为20
incrementByTen()
// 返回的值为30
```

令枚举遵循 CaseIterable 协议。Swift 会生成一个 allCases 属性，用于表示一个包含枚举所有成员的集合。

indirect 来表示该枚举成员可递归，一般不要用，处理起来一般要配合递归，如果数据量大，开销会很大。
```swift
enum ArithmeticExpression {
    case number(Int)
    indirect case addition(ArithmeticExpression, ArithmeticExpression)
    indirect case multiplication(ArithmeticExpression, ArithmeticExpression)
}

let five = ArithmeticExpression.number(5)
let four = ArithmeticExpression.number(4)
let sum = ArithmeticExpression.addition(five, four)
let product = ArithmeticExpression.multiplication(sum, ArithmeticExpression.number(2))
```

类比结构体多以下特性：
继承、引用计数、析构、类型转换允许在运行时检查和解释一个类实例的类型

如果一个被标记为 lazy 的属性在没有初始化时就同时被多个线程访问，则无法保证该属性只会被初始化一次。

在父类初始化方法调用之后，在子类构造器中给父类的属性赋值时，会调用父类属性的 willSet 和 didSet 观察器。
父类初始化方法调用之前，给子类的属性赋值时不会调用子类属性的观察器。

带有观察器的属性通过 in-out 方式传入函数，willSet 和 didSet 也会调用。这里采用了拷入拷出内存模式：即在函数内部使用的是参数的 copy，函数结束后，又对参数重新赋值。

Swift可以通过@propertyWrapper标记一个属性包装器，当你把一个包装器应用到一个属性上时，编译器将合成提供包装器存储空间和通过包装器访问属性的代码。（属性包装器只负责存储被包装值，所以没有合成这些代码。）不利用这个特性语法的情况下，你可以写出使用属性包装器行为的代码。
```swift
@propertyWrapper
struct TwelveOrLess {
    private var number: Int
    init() { self.number = 0 }
    var wrappedValue: Int {
        get { return number }
        set { number = min(newValue, 12) }
    }
}

struct SmallRectangle {
@TwelveOrLess var height: Int
@TwelveOrLess var width: Int
}

var rectangle = SmallRectangle()
print(rectangle.height)
// 打印 "0"

rectangle.height = 10
print(rectangle.height)
// 打印 "10"

rectangle.height = 24
print(rectangle.height)
// 打印 "12"

```

全局的常量或变量都是延迟lazy计算的，跟  [延时加载存储属性](applewebdata://07ADC17B-305B-41BA-B39C-412958A13E79/swift/swift-jiao-cheng/10_properties#lazy-stored-properties)  相似，不同的地方在于，全局的常量或变量不需要标记 lazy 修饰符。

存储型类型属性是延迟初始化的，在第一次被访问的时候才会被初始化，同时是线程安全的，并且不需要对其使用 lazy 修饰符。

class 和 static的区别是它允许子类重写。

结构体构造器嵌套使用。
```swift
struct Rect {
    var origin = Point()
    var size = Size()
    init() {}

    init(origin: Point, size: Size) {
        self.origin = origin
        self.size = size
    }

    init(center: Point, size: Size) {
        let originX = center.x - (size.width / 2)
        let originY = center.y - (size.height / 2)
        self.init(origin: Point(x: originX, y: originY), size: size)
    }
}
```

每一个类都必须至少拥有一个指定构造器，指定构造器像一个个“漏斗”放在构造过程发生的地方，让构造过程沿着父类链继续往上进行。

指定构造器必须总是*向上*代理，便利构造器必须总是*横向*代理
![](Swift%E5%AE%98%E6%96%B9%E6%96%87%E6%A1%A3%E7%AC%94%E8%AE%B0/2DD2C258-5D5B-420A-81A3-EAE329262BC6.png)
指定构造器必须调用其直接父类的的指定构造器。
便利构造器必须调用*同*类中定义的其它构造器。
便利构造器最后必须调用指定构造器。

Swift的构造可以分为2个过程：
1. 指定构造器将确保所有子类的属性都有值，然后它将调用父类的指定构造器，并沿着继承链一直往上完成父类的构造过程直到没有父类，一旦父类中所有属性都有了初始值，实例的内存被认为是完全初始化，阶段 1 完成。。
2. 父类开始有机会进一步自定义实例，层层往下直到子类也有机会进一步自定义实例，子类指定构造器完成调用，则整个过程结束。

Swift默认不会继承父类构造器，在一些特定条件会。
Swift析构函数在实例释放发生前被自动调用的，类继承了父类的析构器，并且在子类析构器实现的最后，父类的析构器会被自动调用。

defer 语句调用发生在return之后，离开之前。

*类型检查操作符*（is）来检查一个实例是否属于特定子类型
Any 可以表示任何类型，包括函数类型、可选值。
AnyObject 可以表示任何类类型的实例。

Swift 中的扩展可以：
* 添加计算型实例属性和计算型类属性
* 定义实例方法和类方法
* 提供新的构造器
* 定义下标
* 定义和使用新的嵌套类型
* 使已经存在的类型遵循（conform）一个协议

required 修饰符可以确保所有子类也必须提供此构造器实现，从而也能遵循协议。

为协议扩展添加限制条件
```swift
extension Collection where Element: Equatable {
    func allEqual() -> Bool {
        for element in self {
            if element != self.first {
                return false
            }
        }
        return true
    }
}
```

范型类型约束语法
```swift
func someFunction<T: SomeClass, U: SomeProtocol>(someT: T, someU: U) {
    // 这里是泛型函数的函数体部分
}
```

协议的关联类型，并且可以添加约束。
```swift
protocol SuffixableContainer: Container {
    associatedtype Suffix: SuffixableContainer where Suffix.Item == Item
    func suffix(_ size: Int) -> Suffix
}
```

some用来修饰不透明的类型，理解上可以对比范型理解，比如范型函数的返回值类型由调用者传入的参数决定，而不同名类型的返回类型由函数体内决定。
``` swift
func makeTrapezoid() -> some Shape {
    let top = Triangle(size: 2)
    let middle = Square(size: 2)
    let bottom = FlippedShape(shape: top)
    let trapezoid = JoinedShape(
        top: top,
        bottom: JoinedShape(top: middle, bottom: bottom)
    )
    return trapezoid
}
let trapezoid = makeTrapezoid()
print(trapezoid.draw())
```

协议类型更具灵活性，底层类型可以存储更多样的值，而不透明类型对这些底层类型有更强的限定，比如。
```swift
func protoFlip<T: Shape>(_ shape: T) -> Shape {
    return FlippedShape(shape: shape)
}
func protoFlip<T: Shape>(_ shape: T) -> Shape {
    if shape is Square {
        return shape
    }
    return FlippedShape(shape: shape)
}
```

理解单线程下，使用inout参数可能会导致内存访问冲突的问题。
![](Swift%E5%AE%98%E6%96%B9%E6%96%87%E6%A1%A3%E7%AC%94%E8%AE%B0/11BBD147-3865-4CDC-8BE9-551F6D213153.png)
```swift
var stepSize = 1
func increment(_ number: inout Int) {
    number += stepSize
}
increment(&stepSize)
// 错误：stepSize 访问冲突
```

open、public、internal、fileprivate、 private访问权限由高到低。

模块指的是独立的代码单元，框架或应用程序会作为一个独立的模块来构建和发布。在 Swift 中，一个模块可以使用 import关键字导入另外一个模块。

Open 和public 一般用来指定框架的外部接口，但是Open 只能作用于类和类的成员，它和 public 的区别主要在于 open 限定的类和成员能够在模块外能被继承和重写。
internal是默认的访问级别。

一个类型的访问级别也会影响到类型/成员/（属性、方法、构造器、下标）的默认访问级别。如果你将类型指定为 private或者 fileprivate级别，那么该类型的所有成员的默认访问级别也会变成 private或者 fileprivate级别。如果你将类型指定为 internal或 public（或者不明确指定访问级别，而使用默认的 internal），那么该类型的所有成员的默认访问级别将是 internal

单元测试 target 时。默认情况下只有 open或 public级别的实体才可以被其他模块访问。然而，如果在导入应用程序模块的语句前使用 @testable特性，然后在允许测试的编译设置（Build Options -> Enable Testability）下编译这个应用程序模块，单元测试目标就可以访问应用程序模块中所有内部级别的实体。

元组的访问级别将由元组中访问级别最严格的类型来决定。例如，如果你构建了一个包含两种不同类型的元组，其中一个类型为 internal，另一个类型为 private，那么这个元组的访问级别为 private。

无符号数采用原码编码，左右移动是逻辑位移，右移除2，左移乘2。
有符号数采用补码编码，左右移动是算术位移，采用补码编码的优点是能达到同无符号数一样的右移除2，左移乘2的效果。同时在运算中可以直接按照标准二进制处理，只需要丢弃超过位数的益出。

Swift溢出运算符一般使用常见是，当你希望让系统在数值溢出的时候采取截断处理，而非报错时，溢出加法 &+、溢出减法 &-、溢出乘法 &*
```swift
var unsignedOverflow = UInt8.max
// unsignedOverflow 等于 UInt8 所能容纳的最大整数 255
unsignedOverflow = unsignedOverflow &+ 1
// 此时 unsignedOverflow 等于 0
```
![](Swift%E5%AE%98%E6%96%B9%E6%96%87%E6%A1%A3%E7%AC%94%E8%AE%B0/7E51F6E0-066C-4241-A309-B5A1A92226B2.png)

``` swift
var signedOverflow = Int8.min
// signedOverflow 等于 Int8 所能容纳的最小整数 -128
signedOverflow = signedOverflow &- 1
// 此时 signedOverflow 等于 127
```
![](Swift%E5%AE%98%E6%96%B9%E6%96%87%E6%A1%A3%E7%AC%94%E8%AE%B0/64EE4947-B892-4EEF-8DD3-7EB5E6ECC77A.png)
