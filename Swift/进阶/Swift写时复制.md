# Swift写时复制
#Swift/进阶

Swift有值类型和引用类型
值类型在被赋值或被传递给函数时是会在内存中进行拷贝的。
引用类型拷贝的是指向内存的指针。

# 什么是写时复制
Swift中如果你具有较大的值类型数据，像Array，Dictionary 和 Set 这样的集合类型。每一次参数传递都进行一次拷贝的话，会严重浪费内存，降低性能。所以只有当它被改变的时候，才会在内存中进行复制。
改变的时候会通过isKnownUniquelyReferenced方法判断当前引用是否唯一，如果是则不会复制，直接修改内存中的原数据，如果不是则会复制。这就是swift的写时复制。

> struct，enum，以及 tuple 都是值类型。  
> Int， Double，Float，String，Array，Dictionary，Set 是用结构体实现。  
> Array，Dictionary，Set默认实现了写时复制技术。  

![](Swift%E5%86%99%E6%97%B6%E5%A4%8D%E5%88%B6/46C4FD58-72C4-4468-8F21-526B1AA46F11.png)

# 为自定义值类型实现时复制技术
```swift
final class Ref<T> {
    var value: T
    init(value: T) {
        self.value = value
    }
}

struct Box<T> {
    private var ref: Ref<T>
    init(value: T) {
        ref = Ref(value: value)
    }

    var value: T {
        get { return ref.value }
        set {
            guard isKnownUniquelyReferenced(&ref) else {
                ref = Ref(value: newValue)
                return
            }
            ref.value = newValue
        }
    }
}

let user = User()

let box = Box(value: user)
var box2 = box                  // box2 shares instance of box.ref

box2.value.identifier = 2       // Creates new object for box2.ref
```

# Swift类对象的深度复制（使用Codable协议）
## 类对象的深度复制原理
使用codable协议进行编解码获得全新值相同的实例对象，并且和复制之前的实例互不影响。
![](Swift%E5%86%99%E6%97%B6%E5%A4%8D%E5%88%B6/B222C354-F867-4F8C-A29A-9515C5BEA2EC.png)
![](Swift%E5%86%99%E6%97%B6%E5%A4%8D%E5%88%B6/EBE119BB-A2D2-4E05-BB7F-01B4D8BAB108.png)


