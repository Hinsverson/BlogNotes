# 总结-UI部分
#iOS知识点/UI相关

# 基础
[[iOS文档参考]]

# 视图绘制
[视图绘制相关](bear://x-callback-url/open-note?id=367182FC-789A-4A01-98E4-39C30A23282C-3605-000092F2E4943148)
视图绘制的全流程有哪些阶段？UIView和CALayer的关系？为什么UIView的drawRect里调用UIGraphicsGetCurrentContext能拿到的当前上下文就是这次绘制的context对象？
异步绘制的原理？怎样进行异步绘制？
系统绘制的流程是怎样的？视图绘制优化方案？结合UIViewController生命周期drawRect什么时候调用？主要作用？注意点？（在drawRect里设置backgroundColor会生效吗？如果同时setNeedDisplay会生效吗？drawRect为什么会变黑？）

知道UIKit的绘制必须在当前的上下文中绘制。


# 布局
[AutoLayout](bear://x-callback-url/open-note?id=40AEADB3-F5F0-4E57-9F0E-952E451DFAFD-21058-00041E05074C2535)
自动布局原理？生命周期的理解？updateConstraints、layoutSubviews调用顺序谁前谁后，以及父子视图间这2个方法的的调用顺序？
Content Hugging Priority和Content Compression Resistance Priority的作用，举例？


# 事件传递/响应
[事件传递/响应机制](bear://x-callback-url/open-note?id=C7A8766E-84D0-44F7-9F8A-758429AA64F0-3605-000092F2CEE7C28D)
系统对于事件是如何进行捕捉的发到主线程中的（为什么实现响应要在主线程处理）？
事件的传递和响应链你是怎样理解的？
事件的传递和分发流程？hitTest内部实现逻辑？
事件传递具体有哪些应用场景？::手动实现一个增大视图触摸区域的例子？实现传递事件到点击视图之下的视图？::

# TableView
[TableView相关](bear://x-callback-url/open-note?id=1841025F-2952-431D-92DD-D2AB0A3C92D2-3605-000092F96BB20251) 
对TableView重用机制的理解？如何实现一个自定义的重用池？重用可能带来的问题，怎么解决？重用Cell的获取方式和区别？知道register注册Cell后dequeueReusableCellWithIdentifier方法不需要判空。
多线程情况下数据源同步的思路？
TableView常用方法的理解和注意点？
TableView的一般优化思路是什么？
掌握UITableViewAutomaticDimension配合AutoLayout动态高度的使用？

# Navigation和Tab
[[Navigation和Tab]]

# 其他常见问题
[UI常见问题](bear://x-callback-url/open-note?id=8670F5F0-AAB2-4C63-B47A-61BA175497ED-3605-000092F977ADBCB0)
UIViewController内的UIView生命周期，加载和移除分别做了什么？








