# Runloop实际应用
#iOS知识点/RunLoop

[Runloop-实际开发你想用的应用场景](https://juejin.cn/post/6889769418541252615)

1. 线程保活
2. 监控卡顿
3. 性能优化
	1. UI处理中为了保证滑动的流畅性，可以把耗时操作（下载，图片解码加载）等指定到runloop的DefaultMode里去。
	2. 监听循环，一次循环只处理一个任务。

# 自定义Runloop的应用-线程保活
![](Runloop%E5%AE%9E%E9%99%85%E5%BA%94%E7%94%A8/12E54E02-CE53-4440-B082-65719BB2DD4A.png)
![](Runloop%E5%AE%9E%E9%99%85%E5%BA%94%E7%94%A8/C5FF724F-87E1-41CF-A049-0B35FAF091BB.png)
1. 开启runloop时需要在当前线程下获取当前线程的runloop，runloop在第一次获取时由系统自动创建
2. runloop要能够一直跑圈执行的条件是当前所处mode下必须有timer、sources、observer等。
3. 通过runOnMode方法去开启runloop，但是此方法做完一次非timer任务后会自动退出runloop，所以用条件循环去控制它，在没有手动调用CFRunLoopStop方法时循环条件为true则继续跑圈并等待任务进行处理，调用CFRunLoopStop方法停止本次跑圈前将条件置为false，那么本次跑圈停止后就不会进行下一次跑圈，runloop会退出而线程也会销毁。
