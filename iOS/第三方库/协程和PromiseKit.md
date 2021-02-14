# 协程和PromiseKit
#iOS知识点/第三方库

# 重要概念理解
### 协程(coroutine)
一种设计模式的体现，把异步的纵向回调处理变成了横向的同步处理，提高异步编程的可读性，但也不是非必须的。

核心思想就是在传统进程、线程的维度下，把线程的处理再细分，完成一连串的异步任务。线程相对独立，有自己的上下文，切换受系统控制，而协程也相对独立，有自己的上下文，但是其切换由自己控制，由当前协程切换到其他协程由当前协程来控制。尤其在一些单线程环境（比如Flutter），通过协程思想进行异步编程非常的必要，它的主要运行流程如下
	1. 协程A开始执行
	2. 协程A执行到一半，暂停执行，执行的权利转交给协程B。
	3. 一段时间后B交还执行权
	4. 协程A重得执行权，继续执行

### Promise
Promise就是协程思想的一种具体体现，可以把它看成是一个协程对象，封装了协程切换的控制逻辑，等同于Flutter里的future对象。

### Async/Sync
通过系统编译器支持，在函数的声明和返回时通过关键字标记，由编译器自动插入一些协程切换逻辑，协程的处理对外不可见，比Promise更高级的一种协程思想的封装。

# PromiseKit
一个promise承诺有2种处理状态，fullfill兑现或者rejectd拒绝。
处理后这个承诺对象都会标记为已解决（resolve)，被销毁，如果既不兑现，也不拒绝，则这个承诺会一直有效（即未解决）。

[Swift - 异步编程库PromiseKit使用详解1（安装配置、基本用法）](https://www.hangge.com/blog/cache/detail_2231.html)
[PromiseKit GitHub](https://github.com/mxcl/PromiseKit)
[深入解析 Promises　輕鬆控制 Parallel Programming (平行程式設計)](https://www.appcoda.com.tw/promises/)
[Swift 中实现 Promise 模式 - 简书](https://www.jianshu.com/p/7268aa4e6b5b)

### 基本使用
[PromiseKit 的常见模式 - 掘金](https://juejin.im/post/5f0fc5aaf265da22fc256247#heading-12)

### PromiseKit如何保证一系列block顺序执行的呢？
把外部传入的thenBlock等保存起来了，保存到一个数组中，handlers.append(to)，当自己的任务执行完去执行存在数组的任务，添加的时候时用栅栏函数同步添加的，保证了任务的顺序执行。

### 闭包中返回值promise 如何与 上一个函数中的promise联系起来的？
rv.pipe(to: rp.box.seal)的处理

### catch 错误捕获函数，为什么promise连中无论哪一环节报错，都能走到catch中去呢？
catch定义在CatchMixin协议中，promise是实现了这个协议的。当promise链中任何一环执行reject(error)时，顺着promise链一直往下走，每一个都是执行rejected(error)这个，直到catch。

### when函数的原理
通过两个Int值，维护表示已经执行完的promise任务数和总共的任务数。再参与进后面的链式调用。
