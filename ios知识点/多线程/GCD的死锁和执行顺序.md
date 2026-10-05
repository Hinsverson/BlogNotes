# GCD的死锁和执行顺序

说人话，还是直接举个🌰看个问题吧，请问在主线程调用下面方法的执行顺序？

追加问题1：执行时会发生死锁吗，为什么？

追加问题2：执行时会崩溃吗，为什么？

```swift
func deadLock4() {
    print("1")
    let que = DispatchQueue.init(label: "thread")
    que.async {
        print("2")
        DispatchQueue.main.sync { 
            print("3")
            que.sync {
                print("4")
            }
        }
        print("5")
    }
    print("6")
    que.async {
        print("7")
    }
    print("8")
}
```
正确答案是输出1、2和(68)顺序不固定、3、que队列循环等待阻塞造成死锁，但并不会崩溃。

如果能准确说出执行顺序并理解为什么死锁而不崩，后面的内容可以不用看了，如果心中还有疑虑，模棱两可，这就是这篇文章的目的，往后看👇👇先理解解决执行顺序问题，再解决死锁问题。

# GCD核心
自动的根据多核CPU的使用情况，创建线程池抽象成queue来执行任务，提高程序的运行效率。

## 队列类型：决定任务怎样拿出来和放置
* 串行队列：任务只能一个一个拿出来顺序的在唯一的一条线程上执行。
* 并行队列：可以同时拿出多个任务到不同的线程并发执行。

## 任务执行方式：决定阻不阻塞当前线程，以及拿出来的任务在哪条线程上执行
sync同步：对当前线程来讲是同步执行，在当前线程中执行，并阻塞当前线程，不具备开启新线程的能力。
async异步：对当前线程来讲是同步执行，不阻塞当前线程，具备开启新线程的能力，串行队列多个异步只会开启一个线程，主队列已经存在主线程，所以异步也不会开启，并行队列并发开启线程执行，最多64条。

::main queue特殊注意点：::
1. main queue是串行队列，并且已经存在有对应的主线程，所以主队列async添加任务不会开启新线程。
2. 无论是用过sync、async向main queue提交任务都在主线程执行。

# 执行顺序问题
1. 首先是明确阻不阻塞当前线程，阻塞可以理解为当前线程没有提交后面任务和执行的能力了。
2. 其次要明确任务是在哪个线程上执行的，如果任务是分别在不同线程执行，那么顺序可能不一定。

```swift
// 输出：1 2 5 4 6 3 8（必定）
// 全部都在主线程执行
func test1() { //task
    print("1") 
    DispatchQueue.global().sync { //task1
        print("2")
        print("5")
        DispatchQueue.main.async { //task2
            Thread.sleep(forTimeInterval: 0.2)
            print("3")
        }
    }
    DispatchQueue.main.async { //task3
        print("8")
    }
    // 这里后面的所有要看成一个整体
    print("4")
    Thread.sleep(forTimeInterval: 0.2)
    print("6")
}
```
* 因为sync阻塞当前线程main提交task1，所以外部1、4、6共同组成的task要整体执行完的话依赖于task1。
* 当2和5执行之后，由于async异步提交task2，不阻塞task1的执行线程main，所以task1完成，所以125后直接46，即使sleep main，因为task是最先存在于队列中的，串行队列只能一个一个拿出来执行。
* 同样task2的提交比task3早，所以先执行task2，再执行task3。

``` swift
//在串行队列对应的那条线程执行（这里就是主线程）外部123任务先执行完，后面执行op1、op2、op3
DispatchQueue.main.async {
    for _ in 0...2 { print("op1---\(Thread.current)") }
}
for _ in 0...2 { print("1---\(Thread.current)") }
DispatchQueue.main.async {
    for _ in 0...2 { print("op2---\(Thread.current)") }
}
for _ in 0...2 { print("2---\(Thread.current)") }
DispatchQueue.main.async {
    for _ in 0...2 { print("op3---\(Thread.current)") }
}
for _ in 0...2 { print("3---\(Thread.current)") }
```
因为这里用的是async异步执行，不阻塞后续任务的提交，这里外部的1、2、3任务可以看成是一个任务块，他们添加到的是串行队列中，没有并发能力，任务都是在一条线程执行，所以最终执行顺序就是任务的添加顺序，那么最终执行情况就是在串行队列对应的那条线程执行（这里就是主线程）外部123任务先执行完，后面执行op1、op2、op3。

## 吃🌰
``` swift
// 1和2的顺序不固定，2执行完后才执行3和4，3和4的顺序不固定
func test2() {
    DispatchQueue.global().async {
        print("1")
    }
    DispatchQueue.global().sync {
        print("2")
    }
    DispatchQueue.global().async {
        print("3")
    }
    print("4")
}
// 1、2和3的顺序不固定、3肯定在4的前面
func test3() {
    DispatchQueue.global().sync {
        print("1")
    }
    DispatchQueue.global().async {
        print("2")
    }
    DispatchQueue.global().sync {
        print("3")
    }
    print("4")
}
// 1和2执行不固定，但是3肯定在1和2的后面
func test4() {
    let que = DispatchQueue.init(label: "thread")
    que.async {
        print("1")
    }
    print("2")
    que.sync { 
        print("3")
    }
}
// 1、2和5不固定，34不打印
// 这种严格意义上来说是不是什么阻塞，只是外部task一直没执行完一直没出队
func deadLock() { //task
    print("1")
    DispatchQueue.global().async {
        print("2")
        DispatchQueue.main.sync { //主线程执行
            print("3")
        }
        print("4")
    }
    print("5")
    while true {
    }
}
```

# 死锁和崩溃的问题
死锁的发生：串行队列内任务的互相依赖等待阻塞导致死锁。
奔溃的原因：先前的串行queue被某个taskA用执行时所在线程的tid🔒住，之后还没等taskA执行完并放开queue的🔒，再向它sync提交taskB时会进行死锁检查，如果taskB执行线程tid和原来queue上的tid🔒相同就会死🔒，同时会被检测出来并崩溃。

## 源码理解：
```C++
dispatch_barrier_sync_f() {
  // 取执行线程的tid
	dispatch_tid tid = _dispatch_tid_self();
	if (unlikely(!_dispatch_queue_try_acquire_barrier_sync(dq, tid))) {
		// 等待并阻塞当前线程
		return _dispatch_sync_f_slow(); 
	}
}
_dispatch_sync_f_slow() {
	_dispatch_sync_wait();
}
_dispatch_sync_wait() {
  // 这里之后会检查死锁
  dq_state = _dispatch_sync_wait_prepare(dq);
	_dispatch_lock_is_locked_by(dq_state, tid)
}
// 异或运算：说明如果dq_state和当前执行线程的tid相等则崩溃
_dispatch_lock_is_locked_by(lock_value, tid) {
	return ((lock_value ^ tid) & DLOCK_OWNER_MASK) == 0;
}
```
说人话：如果_dispatch_queue_try_acquire_barrier_sync返回false的话，就会进入到_dispatch_sync_f_slow中等待并阻塞当前线程，或者说等待串行队列中上一个任务执行完毕。同时如果dq_state和当前执行线程的tid相等则崩溃。

从哪里拿dq_state？看下面👇👇
``` C++
_dispatch_queue_try_acquire_barrier_sync(dispatch_queue_t dq, uint32_t tid)
{
	uint64_t init  = DISPATCH_QUEUE_STATE_INIT_VALUE(dq->dq_width);
	uint64_t value = DISPATCH_QUEUE_WIDTH_FULL_BIT | DISPATCH_QUEUE_IN_BARRIER |
			_dispatch_lock_value_from_tid(tid);
	uint64_t old_state, new_state;
	return os_atomic_rmw_loop2o(dq, dq_state, old_state, new_state, acquire, {
		uint64_t role = old_state & DISPATCH_QUEUE_ROLE_MASK;
		if (old_state != (init | role)) { 
			os_atomic_rmw_loop_give_up(break);
		}
		new_state = value | role;
	});
}
```
说人话，就是取dq_state时（最好仔细看看官方源码）：
* 如果这个dq_state没被改是初始值，会设置为tid返回true， 相当于标记了当前的queue已经被（tid）这个线程lock了。
* 如果dq_state已经被修改过了, 则直接返回false。

## 深入举个简单🌰：
```swift
// 这种最常见的循环等待阻塞死锁，直接崩溃
func deadLock1() { //task tid：主线程
    DispatchQueue.main.sync { //task1 tid：主线程
        print("1")
    }
}
// main循环等待阻塞死锁，直接崩溃
//⚠️：如果用async 不会死锁崩溃
func deadLock2() { //task 主线程
    let que = DispatchQueue.init(label: "thread")
    que.sync { //task1主线程 //⚠️：如果用async
        DispatchQueue.main.sync {  //task2主线程
			  print("1") 
		  }
    }
}
```
下面函数都在主线程中调用，调用也是一个task，发生死锁崩溃原因如下：
1. 在调用函数前，main queue已经在执行task时用主线程tid lock着main queue，所以此时dq_state存着main thread 的tid。
2. deadLock1中直接在main queue sync task1时，dq_state已经被执行task时改过，所以_dispatch_queue_try_acquire_barrier_sync直接返回false，来到_dispatch_sync_f_slow里等待并阻塞当前线程（main thread）。
3. 同时触发_dispatch_lock_is_locked_by死锁检查，因为task1通过sync提交会在当前主线程中执行，所以此时_dispatch_tid_self 返回的tid为主线程tid，和dq_state中的相同，死锁并崩溃。
4. deadLock2中task1应该在主线程中执行，但是会阻塞当前线程（main thread），所以task任务依然用主线程tid lock着main queue。
5. 同时执行task1时，que会被task1用主线程tid lock着，因为它是第一次设置所以_dispatch_queue_try_acquire_barrier_sync返回true，意味着sync task1时不会走到que的死锁检查里去。

### ⚠️重点理解：
1. deadLock1和deadLock2的根本区别在于deadLock2里用另一个que通过sync阻塞了task 放开main queue（main queue的dq_state还是主线程tid）。
2. 比如deadLock2 中task1如果用async提交到que，那么task1不阻塞task的执行，task执行完后就放开执行时用主线程tid lock着的main queue。所以deadLock2 里task2的sync提交其实是在重新设置dq_state，此时_dispatch_queue_try_acquire_barrier_sync返回true，所以不会触发死锁检查也不会死锁。
3. 这里重新设置dq_state比较特殊的是main queue 的sync提交是在主线程执行，所以它的dq_state还是主线程tid（如果普通串行que的话就要看当前的执行线程）

判断死锁崩溃的关键点（是死锁崩溃😯）在于，先前的串行queue是否被某个taskA用执行时所在线程的tid🔒住，如果已经🔒住，那么再向它sync提交taskB时会进行死锁检查，如果taskB执行线程tid和原来的相同就死🔒并崩溃。

## 浅出举个复杂🌰
``` swift
// 输出：5、6、7和8顺序不固定、1肯定在8后面、2肯定在7后面、3、4
// ⚠️：用sync则输出：5、6、7和8顺序不固定、1肯定在8后面、2肯定在7后面，main循环等待阻塞死锁崩溃。
func deadLock3() { //task
    let que = DispatchQueue.init(label: "thread")
    DispatchQueue.main.async { //task1
        print("1")
        que.async { //task2 //⚠️：如果用sync
            print("2")
            DispatchQueue.main.sync { //task3
                print("3")
            }
            print("4")
        }
    }
    print("5")
    Thread.sleep(forTimeInterval: 1.0)
    print("6")
    que.async { //task4
        print("7")
    }
    print("8")
}
```
1. task1和task4的提交是async都不会阻塞task的执行。
2. task3向main queue sync 提交执行时，task肯定是已经完成了的，从顺序就可以看出来，3必定在1581之后（这里的理解很关键🤔，仔细体会）。
3. 上面第2点说明main queue 现在已经没有任务用tid🔒住它了，所以task3的sync提交是在重新用tid🔒住main queue，这个tid是主线程tid，不会触发死锁检查，也不存在循环等待任务的执行。
4. 对比理解⚠️中用sync的情况。

通俗一点说可以看成串行queue中taskA没有执行完，又往queue中sync丢taskB，因为是串行，任务只能1个个拿出来执行，所以taskB执行完的条件是taskA执行完（A先入），而taskB执行完taskA才算执行完并移出队列，所以出现循环等待任务，导致死锁。

## 再深入举个死锁检查的🌰
```swift
// 输出：1、2和(68)顺序不固定、3、que队列循环等待阻塞造成死锁，并不会崩溃。
// ⚠️：思考这里换成sync的情况
func deadLock4() {
    print("1")
    let que = DispatchQueue.init(label: "thread")
    que.async { //task1
        print("2")
        DispatchQueue.main.sync { //这里用sync导致目前que还是被子线程tid lock住
            print("3")
            que.sync { //task2
                print("4")
            }
        }
        print("5")
    }
    print("6")
    que.async { //⚠️：如果这里换成sync
        print("7")
    }
    print("8")
}
```
这里4578都没有打印出，比较好理解（对比浅出🌰）：因为que中的task1被main sync提交的中间任务阻塞，task1没有执行完又向que中提交task2并希望task2马上阻塞当前线程并执行，但是task1还没执行完呢，所以task2不可能执行，task2又是task1执行的一部分，所以循环等待阻塞。

但是这里并不会崩溃🤷🏿‍♀️🤷🏿‍♀️🤷🏿‍♀️，原因如下：
1. task1用子线程tid lock住que，main queue上的sync提交阻塞了task1放开que，que现在依然被task1用子线程tid lock住。
2. task2这里sync会在main上执行（sync是在当前线程执行任务），所以这里task2的sync提交是用主线程tid去和已经被lock的que做比较，不是同一个tid，所以不会被检测到死🔒，也不触发崩溃。

所以GCD死锁的根本是任务循环等待阻塞，而崩溃的根本是sync提交时的死锁检查，实际开发中应该避免这种情况的出现，因为不会崩溃但它确一直占着线程资源。

⚠️：思考如果这里换成sync的情况又是什么呢？不会死锁。
