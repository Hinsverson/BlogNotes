# RxSwift核心逻辑
#iOS知识点/第三方库

[4.3 Observable & Observer 既是可监听序列也是观察者 · RxSwift 中文文档](https://beeth0ven.github.io/RxSwift-Chinese-Documentation/content/rxswift_core/observable_and_observer.html)
[RxSwift - 专题 - 简书](https://www.jianshu.com/c/de8fe3ab2397)

# 函数响应式编程思想：
1. 函数链式调用通过把函数当成参数，内部保存，同时返回自身，在恰当时候进行参数函数的调用（PromiseKit也是这种方式实现异步链式调用）。
2. 响应式编程指以关系的形式去表达逻辑而不是命令式，即A和B存在某种关系，A改变了之后，B自动响应它的变化。
3. 也是面向数据流编程思想的一种体现，把数据封装成信号流并采用观察者模式来实现监听（数据在Ob链上传递，内部匿名观察者发出信号源，A转换为B，B内部会持有A，监听B的同时B也会监听A）。

# Tips
[RxSwift异步事件追踪定位工具的研发历程 - 掘金](https://juejin.im/post/5d7246f3e51d456216553585)

# 一些原理
### Rxswift KVO原理
Rxswit的KVO实现类似Swift的KVO，通过一个中间类完成消息的接收和发送，再通过Rx响应监听的方式发出。

### RxSwift的timer原理
RxSwift的timer其实是封装的DispatchSource定时器。

### 操作符联动原理
使用某个操作符对一个Observable A进行转换的时候，这个操作符都会生成一个新的Observable B，并且在这个新的Observable B内部持有原来的那个Observable A，当有其他人订阅Observable B的时候，Observable B内部同时也会订阅Observable A以此来实现整个Observable Link的“联动”效果。

### deallocating和deallocated：Hook
https://juejin.im/post/5dd738d5518825732e666873
``` swift
override func viewDidLoad() {
    super.viewDidLoad()

    _ = rx.deallocating.subscribe(onNext: { () in
        print("准备走了-deallocating")
    })
    
    _ = rx.deallocated.subscribe(onNext: { () in
        print("已经走了-deallocated")
    })
}

deinit {
    print("走了-deinit")
}
//准备走了-deallocating
//走了-deinit
//已经走了-deallocated
```
deallocating的实现逻辑：内部先拿到NSSelectorFromString(“dealloc”)函数，然后替换了dealloc的IMP指向了一个内部函数，内部函数先调用给出去的closure再调用了原来dealloc的函数地址。

deallocated: Observable<Void>：objc_setAssociatedObject、objc_getAssociatedObject实现添加了DeallocObservable对象，这个对象的内部有个ReplaySubject在deinit的时候发出next、completed，最后返回这个ReplaySubject，也就是deallocated其实就是这个ReplaySubject。

# 核心成员分析
## Observable：序列
继承关系：`AnonymousObservable -> Producer -> Observable -> ObservableType`

### ObservableType：序列基础协议类
本身只声明了一个subscribe订阅方法，交给子类传入具体的observer（观察者）来实现具体的业务功能逻辑。

### Observable<Element>：序列抽象基类（优点类似NSObject）：
抽象实现了subscribe协议方法不做任何事情，要求子类重写实现自身具体的业务功能逻辑，并且提供了：
1. asObservable()方法：让子类们在向下不断发展变化的同时，向上具备统一性（这里就是可观察的特性），体现了接口隔离的原则。
2. 在deinit和init时处理了和内存管理相关的一些事情（Rxswift自己维护了一套引用计数逻辑）

### Producer：序列业务调度控制的基类
重写了Observable<Element>基类的subscribe协议方法把整个事件的发出和响应处理包在了一个调度环境内。但业务处理方法run依然还只是一个抽象方法，要求子类进行实际的事件发出和响应处理。

### AnonymousObservable：具体的可观察序列类
重写实现了父类Producer的run方法来完成事件的发出和响应。

## Observer：观察者（具备发出事件的能力）
### ObserverType：观察者基础协议
定义了发出事件的on方法，同时默认扩展了发出具体事件的onNext、onCompleted、onError方法。

### ObserverBase<Element>：观察者基类
遵循实现了协议ObserverType的on方法，接收event事件后交给自身的抽象方法onCore去处理，并且：
	1. 交给onCore前进行了原子操作检查。

### AnonymousObserver<Element>：具体的观察者类
重写实现了父类ObserverBase<Element>的onCore方法，接收event并传给初始化时保存的EventHandler（事件的响应处理闭包）进行响应，并且：
	1. 在deinit和init时进行了Rx内部引用计数的增减。

### AnyObserver<Element>：类型擦除的包装类
直接实现了ObserverType的on方法，内部有一个observer（其实是EventHandler）init的时候会从外部传入并初始化，调用实现的on方法其实就是调用EventHandler闭包，并且：
1. 提供了通过另一个observer进行init的方法，并保存observer的on方法（on方法和EventHandler闭包是一个类型）到observer（::这里很关键：说明通过调用AnyObserver发出事件的on方法可以调用到另一个observer的on方法上，AnyObserver在这里的作用相当于Swift 里的 Any，体现了类型擦除特性::）。
2. 提供asObserver()方法：让子类具备在向下不断发展变化的同时，向上具备统一性。

## 中间桥梁类：Sink
### Sink<Observer: ObserverType> ：
实现了Disposable协议，初始化时会保存外部传入的observer，并且定义实了一个final forwardOn方法，用来异步的发出event，调用forwardOn会调用observer的on方法。

### AnonymousObservableSink：通过run发出event并进行响应
继承自Sink<Observer: ObserverType>，并遵循了ObserverType协议表明它具备发出事件的能力，具体实现则是内部会调用父类Sink<Observer: ObserverType> 的forwardOn方法，相当于把发出事件的任务交给了父类Sink通过forwardOn发出，并且：
1. 定义了一个run方法：拿到初始化传入参数ob的SubscribeHandler（事件发出的闭包）并且创建传入一个AnyObserver<self>临时对象，执行event的发出逻辑。
::SubscribeHandler闭包内调用AnyObserver<self>的onNext等发出事件的方法 -> 调用回AnonymousObservableSink的on方法::

## Driver
就是对Observable的一层封装。在Observable的基础上：
1. 加了share(replay: 1)，来共享状态。
2. 加了observerOn(main)，将序列的处理过程（_eventHandler()）放在主线程处理。
3. 加了catchError {}，将产生的错误在Driver内部消化掉。

## Schedulers
Schedulers 是 Rx 实现多线程的核心模块，它主要用于控制任务在哪个线程或队列运行，内部主要封装了GCD和OperationQueue：
1. subscribeOn 来决定数据序列的构建函数在哪个 Scheduler 上运行
2. observeOn 来决定在哪个 Scheduler 监听这个数据序列
3. subscribeOn和observeOn都会创建一个中间层序列，所以内部也有一个订阅响应序列的流程，中间层的 sink 就是源序列的观察者。

## Disposable（可以看成构建的序列和观察者间的一种关系封装）
一个序列如果发出了 error 或者 completed 事件，那么所有内部资源都会被释放，不需要我们手动释放。
当执行销毁时，销毁的是序列和观察者之间的响应关系，不是序列和观察者对象本身。
如果是加入到disposeBag，是在disposeBag对象销毁时，依次销毁里面存储的东西。

# 核心流程
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/CEFF4772-39E3-4294-951E-2E8792139C8E.png)

## 关键点：
* 对observable调用subscribe方法一定会来到observable的run方法中进行event的发出和响应。
* observable的run实现是通过sink的run方法，sink的run会执行真正的event发出、响应。

## 准备阶段
1. Cerate静态方法创建AnonymousObservable并保存发出序列的_subscribeHandler。
2. 通过ObservableType 的 subscribe（扩展方法）做订阅时内部构建了一个AnonymousObserver观察者并在创建Disposables返回时调用asObservable().subscribe(AnonymousObserver)回到Observable的subscribe（协议方法）。
3. AnonymousObservable 的subscribe（协议方法）由父类Producer实现进行线程调度控制，并调用自身的run方法，run方法会通过subscribe（协议方法）传进来的observer创建sink。
4. 最后是通过sink串联起observer和observable间的业务逻辑调用，sink执行on方法会传入AnonymousObservable，sink同时拿到了observable和observer。

## 发出和响应流程
1. Sink.run调用时传入当前AnonymousObservable（self），并调用AnonymousObservable保存的_subscribeHandler（发出事件闭包）。
2. _subscribeHandler调用前会创建AnyObserver作为闭包参数，用来发送具体事件（onNext、onCompleted），创建AnyObserver会传入sink。
3. 接着在发出事件的闭包内调用AnyObserver的on或者发出具体的事件（onNext、onCompleted、onError）方法其实都会调用回sink的on方法。
4. Sick的on方法的调用会调用回subscribe（扩展方法）里创建的AnonymousObserver的on方法。
5. AnonymousObserver的on方法的调用会调用回AnonymousObserver创建时保存的_eventHandler回调完成事件的响应。

# 详细流程分析
## Cerate静态方法创建AnonymousObservable并保存发出序列的_subscribeHandler
ObservableType里有个create静态方法方法，调用该方法后内部会创建一个AnonymousObservable（匿名序列）, AnonymousObservable里_subscribeHandler保存了事件的发送闭包（也就是上面的onNext闭包块）
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/6D410A47-DAE8-4783-ACC8-58363C41D7BE.png)
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/8CCA1B51-2451-404E-8BC6-DC3427B5A6B3.png)

### AnonymousObservable 内部继承关系疏离
AnonymousObservable -> Producer -> Observable -> ObservableType

首先ObservableType协议本身声明了一个subscribe订阅方法，交给子类传入具体的observer（观察者）来实现具体的业务功能逻辑。
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/4E1490DE-1083-4207-A49A-00330EE9A268.png)
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/910DBB88-0ACF-47C8-910C-E3B62DDAEC01.png)
Observable<Element>依然是一个抽象类。

> 1.asObservable()方法的作用：  
> 让子类们在向下不断发展变化的同时，向上具备统一性（这里就是可观察的特性），体现了接口隔离的原则。  
> 2.在deinit和init时处理了和内存管理相关的一些事情（Rxswift自己维护了一套引用计数逻辑）  

子类Producer为核心业务控制类，通过实现subscribe协议方法串联起整个事件的发出和响应、以及调度环境。但具体的业务处理run放在了子类AnonymousObservable里。
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/2C2465DA-52E9-4535-9FCA-93C67A4C4F9F.png)

AnonymousObservable实现了run方法，同时在创建时保存了发出事件的闭包（_subscribeHandler）也就是下面。
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/78C38273-6C51-4CA8-B6DB-FA80C9681F0E.png)![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/F537B83D-DFCF-47E1-8181-4F92B560CD8E.png)


## 通过ObservableType 的 subscribe扩展方法构建观察者。
ObservableType协议同时也扩展了一个subscribe方法。
::注意和上面的区分开来，上面是协议的声明方法，交给具体的子类去实现业务逻辑。::
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/48AED492-7358-441C-9786-3911688F8B67.png)
该方法使所有遵循ObservableType协议的类型都具备订阅功能，也就是外部ob（可观察序列）调用时用的方法。
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/EE04CB55-F076-40DB-B111-83F9D799F1B2%202.png)

1. subscribe（扩展方法）内部会创建匿名的AnonymousObserver。
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/A1DBDC5C-4AA2-4C2E-8C05-1642F8E9DE2A.png)

2. AnonymousObserver通过_eventHandler保存观察者响应Observable事件的回调_eventHandler。
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/D6C8FB96-9C30-4A64-A719-02F8A2F54ADF.png)

## Observable的subscribe（扩展方法）构建观察者返回创建Disposables时，回到Observable的subscribe（协议方法）
建Disposables时会调用可观察序列（self）的subscribe（上面说的业务协议方法）并传入创建的匿名observer，此时具体的逻辑执行就回到了可观察序列的subscribe（协议方法）里，进行业务控制处理和各类的方法调用。
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/A1DBDC5C-4AA2-4C2E-8C05-1642F8E9DE2A%202.png)

## AnonymousObservable 的subscribe（协议方法）由父类Producer实现进行线程调度控制，并调用自身的run方法创建sink进行业务逻辑调用。
1. subscribe（协议方法）的核心是run方法，Producer只作为核心业务控制类，串联起整个监听的事件响应，具体的run逻辑交给字类去实现，比如AnonymousObservable。
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/F4B23327-8F45-4B6B-A8E7-2FF34165EBA6.png)

2. AnonymousObservable的run方法时会用传进来的observer和SinkDisposer创建AnonymousObservableSink（管子），管子又会调用它自己的run方法并传入当前的AnonymousObservable序列（管子是真正处理业务逻辑的地方，管子同时拿到了序列（observable）、观察者（observer）、销毁者）。
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/78C38273-6C51-4CA8-B6DB-FA80C9681F0E%202.png)

## 最后通过sink串联起observer和observable间的业务逻辑调用

1. AnonymousObservableSink.run调用AnonymousObservable保存的_subscribeHandler（发出事件闭包）
2. _subscribeHandler调用时会创建AnyObserver用来发送事件（onNext、onCompleted）。
3. 调用AnyObserver的on或者发出具体的事件（onNext、onCompleted、onError）方法其实都会调用回sink的on方法。
4. Sick的on方法的调用会调用回subscribe（扩展方法）里创建的AnonymousObserver的on方法。
5. AnonymousObserver的on方法的调用会调用回AnonymousObserver创建时保存的_eventHandler回调完成事件的响应。

### AnonymousObservableSink.run调用AnonymousObservable保存的_subscribeHandler（发出事件闭包）

AnonymousObservableSink调用run方法（Parent就是AnonymousObservable<Element>）时会调用AnonymousObservable创建时保存的_subscribeHandler（发出事件的闭包）。调用_subscribeHandler时逻辑开始走回到初始创建observable的闭包里开始进行事件发送。
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/07F41A37-A76F-46D2-A920-7811AD35DA35.png)

### _subscribeHandler调用时会创建AnyObserver用来发送事件（onNext、onCompleted）。

这里注意_subscribeHandler闭包里会传入当前的sink并创建一个AnyObserver结构体对象，也就是下面闭包里的observer对象。然后才开始调用observer的on系列方法并发出具体的on事件（onNext、onCompleted、onError）
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/0E242FBB-3943-4796-9B2D-EBB9D49403FA.png)

### 调用AnyObserver的on或者发出具体的事件（onNext、onCompleted、onError）方法其实都会调用回sink的on方法

1. AnyObserver创建时拿到sink的on闭包并保存下来，调用AnyObserver的on方法其实就是在调用sink的on方法。![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/2DAD9A70-3E38-413C-AE2F-3F5C180C93B3.png) 
2. 而AnyObserver遵循了ObserverType协议，AnyObserver的on方法也会在调用具体的事件方法（onNext、onCompleted、onError）时调用。
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/19CEF515-BE4C-4CCA-B891-D40095B8743D.png)
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/0E242FBB-3943-4796-9B2D-EBB9D49403FA%202.png)
3. 也就是这里的AnyObserver .OnNext等方法的调用会回到上面AnyObserver的on方法，前面已经说了，用AnyObserver的on方法其实就是在调用sink的on方法，所以也会走到sink的on方法里去。
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/6C0E7E4F-0884-4ADD-8CEC-E114F3373691.png)

### Sick的on方法的调用会调用回subscribe（扩展方法）里创建的AnonymousObserver的on方法。

sink的on方法里会调用forwardOn，forwardOn里调用创建sink时传入的AnonymousObserver（注意和上面发出事件的AnyObserver不是同一个东西）的on方法进行事件的响应
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/B027E900-5783-4CEB-AC97-2D74BF588F96.png)

### AnonymousObserver的on方法的调用会调用回AnonymousObserver创建时保存的_eventHandler回调完成事件的响应

1. 调用AnonymousObserver的on方法（来自父类ObserverBase）会再调用字类重写的onCore方法，onCore会调用创建AnonymousObserver时保存的_eventHandler回调。
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/07903443-498F-4BD0-A03B-EA0166C40590.png)
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/D629B3A8-5ECA-4576-ABD6-C6FE8CED96D0.png)
2. _eventHandler回调里会调用先前subscribe使（Observable扩展方法）的具体响应事件闭包，完成整个事件响应。
![](RxSwift%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91/8B3E76F4-B4F3-4A66-BDF6-0581F6302AA6.png)


