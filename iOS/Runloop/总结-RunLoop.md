# 总结-RunLoop
#iOS知识点/RunLoop 

<a href='RunLoop.pdf'>RunLoop.pdf</a> 

# 重点掌握
## 基础总结
[Runloop全面总结](bear://x-callback-url/open-note?id=1A3EC8FE-7AC9-44A9-B68B-30F8C796CD87-3605-000092F84EEC2E5C)
什么是RunLoop？对RunLoop的理解？
RunLoop的启动方式有哪些？有什么区别？
RunLoop的退出方式有哪些？有什么区别？
RunLoop和线程有什么关系？保存在哪里？线程间Port如何通信？
RunLoop有哪几种mode？对常见mode的理解？定时器滑动失效的原因？怎样处理？::处理的生效原理？为什么timer不精准？::如果实现精准的定时？
RunLoop的事件源有哪些？特点是什么？::source1和source0的区别？::
::RunLoop的监听状态有哪些？::怎样监听？
::RunLoop的内部循环逻辑是怎样的？::
RunLoop休眠的理解？处于RunLoop唤醒的方式有哪些？

## 系统内Runloop的常见问题
[系统内Runloop的常见问题](bear://x-callback-url/open-note?id=5C5C85B6-4659-421B-829B-5B54C09D3CDB-3605-000092F3F62B1B67)
RunLoop和AutoreleasePool的关系？autoreleasePool在什么时候被释放？
 GCD和Runloop的关系？
PerformSelector afterDelay的实现原理？
事件响应的过程（结合RunLoop）？
手势识别的过程（结合RunLoop）？
UI绘制setNeedsDisplay 的原理？

## Runloop的应用
[自定义Runloop的应用-线程保活](bear://x-callback-url/open-note?id=8BBD61E4-47F3-481B-9A35-028981945C25-3605-000092F624970050)
如何实现一个常驻线程？

# 其他常见问题
[UIApplication和RunLoop](bear://x-callback-url/open-note?id=232C8FE4-461E-4E9F-9164-151B3BE2F6BC-3605-000092F97E877552)