# Swift闭包的本质
#Swift/底层解析

# 定义函数的2种方式
* func创建函数
* 闭包表达式定义函数

# 闭包的概念
![](Swift%E9%97%AD%E5%8C%85%E7%9A%84%E6%9C%AC%E8%B4%A8/5E62D319-A30B-4DAE-8567-5BA00601038E.png)

一个函数和它所捕获的变量/常量所组合成的环境才是闭包。

# 闭包捕获上层函数栈中已被回收值的原理（逃逸）：
（联想OC的__Block原理）如果闭包体内需要用到的这个变量在原来的栈空间被回收，就会调用alloc方法向堆空间内存申请一块来存储这个变量（把栈空间的临时变量的内存信息拷贝一份）。因为要动态的调用函数，延长生命周期，捕获的这个变量的生命周期，只有放在堆空间里面，栈空间系统一般会经常清理。

> 可以把闭包想象成一个类的实例对象，内存在堆空间，捕获的局部变量和常量就是对象的成员！  
> 捕获多个变量的时候，捕获的多个变量在内存中的地址并不连续，原因很简单，因为是在堆空间（malloc分配）。  

## 闭包本质示例
![](Swift%E9%97%AD%E5%8C%85%E7%9A%84%E6%9C%AC%E8%B4%A8/D710AF5C-C4F0-4421-95D5-6E122F31EB22.png)
fn1变量接受一个函数会为num创建一个堆空间
fn2变量接受一个函数也会为num创建一个堆空间

变量fn1的内存布局是一共16个字节，前8个放着plus函数的地址，后8个放着捕获变量的堆空间地址。
调用fn1(1)，实际是通过fn1里前8个字节放着的plus函数地址，找到plus函数并调用，而且除了1这个参数外，还会传入fn1里后8个字节放着的堆空间地址，这样才能访问捕获到的那些变量。
可以理解为fn1这个闭包是个指针，指向一个函数的地址的同时指向一个存放着捕获到的外部变量的堆空间。

![](Swift%E9%97%AD%E5%8C%85%E7%9A%84%E6%9C%AC%E8%B4%A8/93BD91F5-3458-4CB1-8B8A-2E7E1D699208.png)

# 闭包捕获的理解
要理解捕获都是发生在构建闭包的时候

### 隐式捕获
closure对象内部强引用着外部变量，是指针的传递，传递时是原来是常量还是常量，变量还是变量。

### 显示捕获
重新创建一个临时常量的赋值操作，赋值操作在值类型上是深拷贝会创建对象，引用类型上是浅拷贝的指针传递。

### 引用捕获里的强和弱
强弱引用的捕获针对的是分布在堆上的class对象，值类型分布在栈上无需关心内存生命周期。
weak和unowned都属于弱引用，不会导致引用计数+1，区别在于unowned在对象被释放时不会清空为nil，所以释放后继续访问的话容易造成野指针错误。

* 默认没有带捕获列表的隐式捕获行为
	* 不论被捕获的变量是值类型还是引用类型，都是使用强引用指向被捕获的变量，产生一次浅拷贝（指针的拷贝）。
* 带捕获列表就是在显式的指定闭包的捕获行为
	1. 如果被捕获的变量b是值类型[a = b]，那么相当于在赋值操作时进行了值传递，产生一次深拷贝（对象的拷贝），a和b指向不同对象。
	2. 如果被捕获的变量b是引用类型[a = b]，那么相当于在赋值操作时进行了引用传递，产生一次浅拷贝（指针的拷贝），a和b指向同一对象。

特例：下面情况捕获发生在对象内部函数里时，捕获self时，即使没有使用捕获列表显示捕获，值类型也是深拷贝。（这里有点不理解为什么？）
``` Swift
struct Demo {
    var name: String
    init(name: String) {
        self.name = name
    }
    func printName() -> () -> () {
        return {
            print(self.name)
        }
    }
}
var demoa = Demo(name: "a")
let printNamea = demoa.printName()
demoa.name = "aaa"
printNamea() // 为什么输出是a ？

var demob = Demo(name: "b")
let printNameB = { print(demob.name) }
demob.name = "bbb"
printNameB() // 为什么输出是bbb ?
```

举例：正常情况的捕获（通过有无捕获列表和值/引用类型决定深浅拷贝）
``` Swift
var a = 0
var b = 0

class SimpleClass {
    var value: Int = 0
}
var x = SimpleClass()
var y = SimpleClass()

// 隐式捕获：强引用指向上层作用域内变量
// 指针传递，closure对象内部强引用着外部变量
// 传递时常量还是常量，变量还是变量

// 显示捕获：重新创建一个临时常量的赋值操作
// 赋值操作在值类型上是深拷贝会创建对象，引用类型上是浅拷贝的指针传递

// 隐式捕获a、x，b，y：
let closure1 = {
    print(a, b) //Prints "10 10"
    print(x.value, y.value) //Prints "10 10"
}

// 显示捕获a、x，隐式捕获b，y
// a为值类型，内部aCopy通过外部a深拷贝，创建了新对象
// x为引用类型，内部xCopy通过外部x浅拷贝，还是指向同一个对象
let closure2 = { [aCopy = a, xCopy = x] in
    print(aCopy, a, b) //Prints "0 10 10"
    print(xCopy.value, x.value, y.value) //Prints "10 10 10"
}

// 显示捕获a、x，隐式捕b，y，同上面一样，只是简化的语法糖
// print里白色的a、x就是上面的aCopy，xCopy
let closure3 = { [a, x] in
    print(a, b) //Prints "0 10"
    print(x.value, y.value) //Prints "10 10"
}

a = 10
b = 10
x.value = 10
y.value = 10
closure1()
closure2()
closure3()
```

# ??运算符的本质
![](Swift%E9%97%AD%E5%8C%85%E7%9A%84%E6%9C%AC%E8%B4%A8/8C7CCDE5-CA23-42C9-94EA-790E8F4AA980.png)
![](Swift%E9%97%AD%E5%8C%85%E7%9A%84%E6%9C%AC%E8%B4%A8/6EA959FD-A3FF-4305-B29B-814F41F592B0.png)?? 运算使用了自动闭包技术

# 逃逸闭包体内self.的本质
![](Swift%E9%97%AD%E5%8C%85%E7%9A%84%E6%9C%AC%E8%B4%A8/D8CAEF00-7870-4E74-81EC-F1BC5AB44735.png)
因为是逃逸闭包，实际闭包执行可能发生在函数结束后调用，为了延长对象生命周期，所以内部必须采用强引用self.的方式访问。
