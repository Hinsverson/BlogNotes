# PerformSelector详解
#iOS知识点/多线程

1. 同步执行的几个方法底层基于runtime，直接发送objc_msgSend消息，在当前线程上执行，与开不开线程的runloop无关。
2. 异步延迟执行的几个方法依赖于runloop的timer，调用时会向mode里添加一个timer源，所以需要注意当线程没有开启runloop时，这几个方法的异步延迟执行会失效，同时timer源被消耗后会释放（开启的runloop可能再次推出），延迟执行的方法在执行前可以取消。
3. 线程间通信的几个方法底层基于runloop的port，同样要注意当线程开启runloop后才会执行，port加入后是永远留在runloo的mode中的，且inBackground会开启子线程去执行selector，waitUntildone为真会阻塞当前线程，等待selector执行完成。

> 配合runloop时要注意，若想开启某线程的Runloop，必须具有timer、source、observer任一事件才能触发开启。  

参考：
[关于 performSelector 的一些小探讨 - 掘金](https://juejin.im/post/6844903775816122375#heading-1)
[关于performSelector看我就够了 - 掘金](https://juejin.im/post/6844903838550327304)

# 同步执行消息
![](PerformSelector%E8%AF%A6%E8%A7%A3/6F4462FF-4675-4248-A4E9-5878DF8C0AC0.png)
![](PerformSelector%E8%AF%A6%E8%A7%A3/DBEB9D96-37F4-4609-A25E-941726EA26F7.png)
这三个方法，均为同步执行，与线程无关，主线程和子线程中均可调用成功，不需要开启Runloop。

原理：基于runtime的消息机制，在需要动态的去调用方法的时候去使用，相当于直接向receiver发送objc_msgSend消息。

# 异步延迟执行消息
![](PerformSelector%E8%AF%A6%E8%A7%A3/A032973E-C808-4F98-862D-B06F0A206E35.png)
这两个方法为异步执行，即使delay传参为0，仍为异步执行。
只能在runloop开启的线程中执行，在子线程中不会调到aSelector方法。
在方法未到执行时间之前，可以进行方法的取消。

原理：基于Runloop timer的延迟实现，这个方法调用后，在当前runloop里设置了一个timer，来触发这个方法执行，所以当前线程没有开启Runloop 的话，对应方法也就不会执行。

**延迟执行例子**
![](PerformSelector%E8%AF%A6%E8%A7%A3/84D4289A-69C1-41A8-8033-C4760B6BC161.png)
上面代码中不会打印2，因为performSelectorAfterDelay底层是用定时器实现，相当于向runloop里添加一个timer事件等到下一次runloop循环过来时唤醒runloop去处理。而子线程默认没有开启runloop，所以不会执行。

注释部分由于在主线程中，默认是开启runloop的所以会打印，但是顺序是1、3、2，这是由于runloop要等待下一次被唤醒后才去执行timer事件，前面需要先处理完上一次的点击事件（异步）。

**取消例子**
![](PerformSelector%E8%AF%A6%E8%A7%A3/A3A4D0AA-56E8-4DDA-8851-E8ED765A3F1B.png)

# 线程间通信
![](PerformSelector%E8%AF%A6%E8%A7%A3/EA70F8CC-5414-4239-B54F-EE8EF3F3B25E.png)
1. 前面2个方法：向主线程发消息执行selector
2. 中间2个方法：向任意线程发消息执行selector
3. waitUntildone：如果设置为true，表示执行完selector后，当前线程才继续往下执行（相当于会阻塞当前线程）如果设置为false，当前线程接着继续往下执行，不等待selector的执行结束（相当于不会阻塞当前线程）
4. inBackground方法会开启新的线程在后台执行selector方法

原理：performSelectorOnThread方法底层基于线程间的port通信实现。

注意：apple不允许程序员在主线程以外的线程中对ui进行操作，可以调用performSelectorOnMainThread函数在主线程中完成UI的更新。




