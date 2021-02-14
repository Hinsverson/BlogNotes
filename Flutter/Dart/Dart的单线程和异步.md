# Dart的单线程和异步
#flutter

[彻底搞定Dart的异步](https://mp.weixin.qq.com/s?__biz=Mzg5MDAzNzkwNA==&mid=2247483686&idx=1&sn=031c056995c75a4c9ca43eacc881f6b6&chksm=cfe3f2d9f8947bcf1808c582f08f571edc2679d1bb494938dbd441a904f06f3d56b64062c7db&scene=178&cur_album_id=1566028536430247937#rd)

如果在多核CPU中，单线程是不是就没有充分利用CPU呢？
单线程是如何来处理网络通信、IO操作它们返回的结果呢？

# 理解Dart的单线程异步模型原理
首先理解操作系统中的阻塞式调用和非阻塞式调用的概念。
	* 阻塞式调用： 调用结果返回之前，当前线程会被挂起，调用线程只有在得到调用结果之后才会继续执行。
	* 非阻塞式调用：调用执行之后，当前线程不会停止执行，只需要过一段时间来检查一下有没有结果返回即可。
Dart单线程异步模型就是基于非阻塞式调用，单线程异步模型中主要就是在维护着一个事件循环（Event Loop），比如点击事件、IO事件、网络事件时，它们就会被加入到eventLoop中。然后不断的从事件队列（Event Queue）中取出事件，并执行其对应需要执行的代码块，直到事件队列清空位置。同时在Dart中还存在另一个队列：微任务队列（Microtask Queue），微任务队列的优先级要高于事件队列。具体的任务入队情况如下：
	* 事件队列中的主要是所有的外部事件任务，如IO、计时器、点击、以及绘制事件等。
	* 微任务队列中的微任务通常来源于Dart内部，并且微任务非常少。这是因为如果微任务非常多，就会造成事件队列排不上队，会阻塞任务队列的执行。（实际开发中可以通过dart中async下的scheduleMicrotask来手动创建一个微任务。）

# Dart的异步API基本使用
通过Future对象把耗时操作包起来，使用上类似Promise的思想，区别于Promise的三种状态，Future对象只有未完成状态（uncompleted）和完成状态（completed）。

也可以使用async关键字标记函数，返回一个Future对象，函数内使用wait关键字实现写法上同步使用异步返回的结果。

# 理解Dart的单线程异步代码执行顺序
Dart中，main函数中的代码执行后。首先，会按照先进先出的顺序，执行微任务队列（Microtask Queue）中的所有任务，微任务队列执行完为空后，会按照先进先出的顺序，执行事件队列（Event Queue）中的所有任务。

Future代码中通常有2部分函数执行体，分别遵循以下规则加入任务队列：
	1. Future构造函数传入的函数体放在事件队列中
	2. then的函数体（catchError等同看待）要分成三种情况：
		* 情况一：Future没有执行完成（有任务需要执行），那么then会直接被添加到Future的函数执行体后；
		* 情况二：如果Future执行完后就then，该then的函数体被放到如微任务队列，当前Future执行完后执行微任务队列；
		* 情况三：如果Future是链式调用，意味着then未执行完，下一个then不会执行；
```dart
// future_1加入到eventqueue中，紧随其后then_1被加入到eventqueue中
Future(() => print("future_1")).then((_) => print("then_1"));
// Future没有函数执行体，then_2被加入到microtaskqueue中
Future(() => null).then((_) => print("then_2"));
// future_3、then_3_a、then_3_b依次加入到eventqueue中
Future(() => print("future_3")).then((_) => print("then_3_a")).then((_) => print("then_3_b"));
```

``` dart
import "dart:async";

main(List<String> args) {
  print("main start");

  Future(() => print("task1"));

  final future = Future(() => null);

  Future(() => print("task2")).then((_) {
    print("task3");
    scheduleMicrotask(() => print('task4'));
  }).then((_) => print("task5"));

  future.then((_) => print("task6"));
  scheduleMicrotask(() => print('task7'));

  Future(() => print('task8'))
    .then((_) => Future(() => print('task9')))
    .then((_) => print('task10'));

  print("main end");
}
/* 执行顺序：
main start
main end
task7
task1
task6
task2
task3
task5
task4
task8
task9
task10
*/
```
1. main函数先执行，所以main start和main end先执行，没有任何问题；
2. main函数执行过程中，会将一些任务分别加入到EventQueue和MicrotaskQueue中；
3. task7通过scheduleMicrotask函数调用，所以它被最早加入到MicrotaskQueue，会被先执行；
4. 然后开始执行EventQueue，task1被添加到EventQueue中被执行；
5. 通过final future = Future(() => null);创建的future的then被添加到微任务中，微任务直接被优先执行，所以会执行task6；
6. 一次在EventQueue中添加task2、task3、task5被执行；
7. task3的打印执行完后，调用scheduleMicrotask，那么在执行完这次的EventQueue后会执行，所以在task5后执行task4（注意：scheduleMicrotask的调用是作为task3的一部分代码，所以task4是要在task5之后执行的）
8. task8、task9、task10一次添加到EventQueue被执行；

# Dart对于多核CPU的利用
Dart中一条线程和对应可以访问的内存空间以及需要运行的事件循环整个context被称为一个Isolate。
Flutter中就有一个Root Isolate，负责运行Flutter的代码，比如UI渲染、用户交互等等。
Isolate 之间不共享任何资源，只能依靠消息机制通信，因此也就没有资源抢占问题。

在开发中，我们有非常多耗时的计算，可以自己创建Isolate，在独立的Isolate中完成想要的计算操作达到充分利用多核CPU并行处理的目的。
