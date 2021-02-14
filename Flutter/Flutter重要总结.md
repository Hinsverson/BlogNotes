# Flutter重要总结
#flutter

# Flutter内部运行机制的整体理解
这篇文章写的很好，图文并茂理解Flutter框架层运行流程  [Flutter - Flutter internals](https://www.didierboelens.com/2019/09/flutter-internals/)
![](Flutter%E9%87%8D%E8%A6%81%E6%80%BB%E7%BB%93/5C00F097-739A-4CC2-B792-DA52C3A10F92.png)
Flutter框架通过  [Window](https://api.flutter.dev/flutter/dart-ui/Window-class.html)  与 Flutter Engine交互。Window这个抽象层暴露了一系列API，来让 Flutter框架间接地与设备进行交互。也是通过这个抽象层，当以下情况发生时，Flutter Engine会通知 Flutter框架 ：
	* 当设备发生了一些事件时，比如方向改变、设置改变、内存问题，app运行状态改变等。
	* 当屏幕上发生了一些事件时，比如手势。
	* 当平台的 channel 发送一些数据时。
	* 最主要的，当 **Flutter Engine 准备渲染新的一帧** 时。

![](Flutter%E9%87%8D%E8%A6%81%E6%80%BB%E7%BB%93/07D1A5E2-5CAE-4C06-8E51-126C5EC0E3F7.png)
1. 一些外部事件（手势、http请求等等）或者 *future*，会执行改变UI的任务。在这种情况下，一条消息（*Schedule Frame*）就会被发送给 Flutter Engine，来通知它需要刷新。
2. 在 *Flutter Engine* 准备渲染之前，它会发出一个 *Begin Frame* 请求
3. 这个 *Begin Frame* 请求会被 *Flutter框架* 拦截，然后执行 *Tickers* 相关的任务（比如动画）
4. 这些任务可能会发起重新渲染的请求（比如一个动画还没有完成，它需要在下一个阶段的 *Begin Frame* 时继续执行动画代码）
5. 接着，*Flutter Engine* 发起 *Draw Frame* 请求
6. *Draw Frame* 会被 *Flutter框架* 拦截，*Flutter框架* 会找出所有与结构、尺寸相关的更新布局的任务，然后执行。
7. 等到所有这些任务都执行完， *Flutter框架* 会继续处理与绘制相关的更新布局的任务
8. 如果UI有更新的话， *Flutter框架* 会发送新的需要被渲染的 *Scene* 给 *Flutter Engine*，*Flutter Engine* 会完成屏幕的更新
9. 然后， *Flutter框架* 会执行渲染结束之后的所有任务，以及一些与渲染无关的子任务。
10. 然后这个不断重复这个流程。


![](Flutter%E9%87%8D%E8%A6%81%E6%80%BB%E7%BB%93/F66C15D0-95DD-404A-BD9A-5BC93EF59B67.png)
被渲染到屏幕上的视觉相关的内容，相对应的类，叫做  [RenderObject](https://docs.flutter.io/flutter/rendering/RenderObject-class.html) ，它被用于：
	* 根据尺寸、位置、几何形状（统称为渲染内容）来定义一些区域
	* 在屏幕上定义能够识别手势的区域

这一系列的 *RenderObject* 形成一棵树，这棵树被叫做 *Render树* ；在树的最顶端（=root），我们会找到  [RenderView](https://docs.flutter.io/flutter/rendering/RenderView-class.html) .
*RenderView* 表示 *Render树* 的整个渲染平面，并且它自己也是一个特别的 *RenderObject*.


![](Flutter%E9%87%8D%E8%A6%81%E6%80%BB%E7%BB%93/F7DD4518-8353-40B7-A7B2-F86D90FC3393.png)
![](Flutter%E9%87%8D%E8%A6%81%E6%80%BB%E7%BB%93/4A5876CB-5212-4AE3-9108-6139FE9238C8.png)

# 启动后3颗核心树的构造
从启动创建出发，Flutter启动后创建3颗树的过程：3颗树的根，分别是RenderObjectToWidgetAdapter和RenderObjectToWidgetElement以及renderView。

1. runApp传入外widget对象，此时外widget对象已经创建，但还没有进行树的build。
2. 启动流程里创建根桥接widget对象RenderObjectToWidgetAdapter后，调用attachToRenderTree时通过createElement第一次创建了根element（RenderObjectToWidgetElement），并指定了根element.assignOwner(owner)为WidgetsBinding内创建的buildOwner，然后执行owner.buildScope()开始build，传入的回调内先进行根`element.mount(null, null)`的挂载操作。
3. 这个根element（RenderObjectToWidgetElement）的mount挂载除了调用super.mount做基本的RenderObjectElement的mount都会做的操作`widget.createRenderObject(this)`（注意此时是直接返回RenderObjectToWidgetAdapter创建时传入的RendererBinding里创建的根renderView）和`attachRenderObject(newSlot)`外，还会调用内部的_rebuild方法。
4. RenderObjectToWidgetElement的_rebuild内会触发`updateChild(_child, widget.child, null)`方法，因为此时传入的child参数_child为null，说明 newWidget（widget.child（就是外部我们写的runApp里那个外widget）） 是新插入的，需要inflateWidget为这个widget创建对应的element子节点。
5. RenderObjectToWidgetElement的inflateWidget里通过 newWidget（外widget） `newWidget.createElement()`创建它对应的 Element，并调用`newChild.mount(this, newSlot)`把创建的子element挂载在自己下面，接着根据Element类型不同，会有如下不同行为：
	1. 如果外widget是组合类的widget，这个子ComponentElement在mount第一次挂到element树上后先`_firstBuild`，再`rebuild`对活跃的、脏节点调用performRebuild进行实际的rebuild流程。
		1. ComponentElement：此时整个performRebuild的过程可以理解为调用build方法（搭建widget时写的代码）先生成对应widget节点下的所有新widget节点树，然后再把新widget树的根传入`updateChild`方法内通过`inflateWidget`拿这个新widget树的根widget节点创建其对应的element数节点，并把生成的element节点又mount挂载在当前element节点下面，递归进入5中的mount循环内。
	2. 如果外widget是render类的widget，在RenderObjectElement.mount中做的最重要的事就是拿当前element 对应的widget调用`widget.createRenderObject(this) `创建了对应的Render Object，再在attach内通过`insertRenderObjectChild`把创建的renderObject到插到render树上。（这些事情在是renderObjectElement层完成），更具体的不同操作行为会发生在子类上：
		1. SingleChildRenderObjectElement：在 super (RenderObjectElement) 的基础上，继续调用`updateChild(_child, widget.child, null)`方法处理子节点，此时_child为null，说明 newWidget（widget.child） 是新插入的（此时widget.child就是外widget的child节点），然后继续inflateWidget递归创建子element节点再mount到element树上，递归进入5中mount循环内。
		2. MultiChildRenderObjectElement：在 super (RenderObjectElement) 的基础上，对每个子节点（此时的字节点是外widget的childs节点）直接调用inflateWidget递归创建下级element节点再mount挂载到每个子element节点下面，递归进入5中mount循环内。

经过上面的mount挂载循环，最终会根据我们写的widget层级关系，生成对应的element树和render树。

# 树的更新理解
[理解Flutter UI系统 | 码农家园](https://www.codenong.com/js97ad36fbb657/)
![](Flutter%E9%87%8D%E8%A6%81%E6%80%BB%E7%BB%93/276E9B38-63F4-4E9C-AC0E-1063E8645E90.png)
下面拿setState()触发更新为例，理解具体更新过程和规则。

setState()方法的执行为_element.markNeedsBuild()，即设置对应的element为dirty，添加到dirtyElements list中。
渲染循环内WidgetsBinding.drawFrame调用buildScope，这些标脏的element将执行rebuild()，最后触发performRebuild()进行树的重建。
performRebuild内会通过state去调用到有状态的widget的build完成重建widget的工作。
当setState的时候Text 由A 变成B中的过程中，Widget树重建，上一帧的Element树从E1开始深度遍历每一个子节点，对于每一个节点，执行Element.updateChild(Element child, Widget newWidget, dynamic newSlot)；按照上述讲解的更新规则这样遍历到最后，只有E7对应的RenderObject(RenderParagraph)需要更新，调用该element所对应Widget的updateRenderObject(this，renderObject)方法进行更新操作，对其重新进行布局和绘制即可。
当然在实际开发中，如果Text 由A 变成B中是不会在顶层的Widget中执行setState更新的，因变化的只有Text，其他Widget不需要重建，考虑性能问题，会抽离出变化的组件为一个Widget,

# Flutter的渲染分析
[Flutter的Widget-Element-RenderObject](https://mp.weixin.qq.com/s?__biz=Mzg5MDAzNzkwNA==&mid=2247483782&idx=1&sn=5cca87b95f82131ed0052d935f907807&chksm=cfe3f279f8947b6f9a21b92d7c5084e9404993ccce3c80f6329c0734f037f89d1d859651616f&scene=178&cur_album_id=1566028536430247937#rd)
[Flutter核心原理之RenderObject和RenderBox - 梁飞宇 - 博客园](https://www.cnblogs.com/lxlx1798/articles/11174987.html)
![](Flutter%E9%87%8D%E8%A6%81%E6%80%BB%E7%BB%93/49FC8904-0DF3-459C-A423-C4303E04CEE4.png)

## 理解Widget、Element、RenderObject
### 三者之间不同的角色职责
* Widget只是一个个轻量的UI描述对象，它描述了UI的组织和配置信息，其中：
	* StatefulWidget和StatelessWidget主要负责组织widget。
	* RenderObjectWidget类型的widget才会在Element创建完毕被flutter framework挂载后创建RenderObject对象diff widget上的配置信息后做渲染相关的标记操作，来负责参与实际的绘制。
* Element可以理解为是一个Widget的实例，Widget描述和配置子树的样子，而Element负责实际去配置在UI骨架树中特定的位置，由framework调用amout挂载到UI树结构中组织起来，是真正保存UI树结构的对象。
* RenderObject是真正渲染的对象，其中有markNeedsLayout performLayout markNeedsPaint paint等方法。

### 三者之间的关系和区别
flutter直接渲染的是Render树，而widget树非常不稳定，而Element树充当一个桥梁，来对比老的widget和新的widget有什么样的变化。创建widget与之对应就会创建一个一模一样的Element树。Element既引用widget树同时也通知Render树哪些是变化的，哪些是没有变化的，再由渲染引擎来渲染，这样就大大的提高了效率。

Element 与 Widget 另一个区别在于，Widget 天然是不可变的（immutable），它如要更新便需要重建，如果想要把可变状态与 Widget 关联起来，可以使用 StatefulWidget，StatefulWidget 通过使用StatefulWidget.createState 方法创建 State 对象，并将之扩充到 Element 以及合并到树中。

## Flutter的UI组织和渲染源码理解
### 创建Element为挂载做准备
首先Widget类提供抽象方法createElement()，由具体子类实现具体Element对象的创建，每一次创建Widget的时候会通过自己创建一个对应的Element对象，Element内保存着对Widget的引用（在最上面的Element抽象层完成），其中：
	* StatefulElement创建时会额外调用widget.createState()方法，创建对应的私有_state管理类并自己持有，并将widget赋值给_state。
	* StatelessElement、RenderObjectElement创建时只是简单super调用，保存对widget的引用。
### mount挂载Element到UI树上
创建完Element之后，Flutter Framework会（在渲染管线的buildsCope时机）会调用Element的mount方法来将Element挂载到树中具体的位置。mount操作同时会把Element的lifecycle state状态从initial改为active（Element抽象层实现这个统一行为）。同时不同Element子类的mount会额外做不同的特殊挂载处理，主要有以下几种Element：
	* RenderElement 会调用widget.createRanderObject()方法创建RenderObject对象（只有继承RenderObjectWidget的Widget会创建RenderElement，比如Padding）并且标记_dirty = false。
	* ComponentElement（StatefulWidget和StatelessWidget对应的StatefulElement和StatelessElement都继承自ComponentElement）的mount额外实现主要就是触发rebuild，再触发performRebuild，最终调用子类Element的build方法并把自己作为context传出去（Element是实现了BuildContext接口的类）完成对UI组件树的组织，区别在于：
		* StatefulElement的build的实现为间接调用_state中的build(BuildContext context)。
		* StatelessElement的build实现为直接调用widget的build(BuildContext context)。

### 创建RenderObject打标记交给渲染引擎渲染
在widget继承链中，RenderObjectWidget子类作为需要被渲染widget的抽象类，提供`RenderObject createRenderObject(BuildContext context)`抽象方法要求子类widget实现并创建与之对应的RenderObject子类对象，为渲染做准备。比如以Padding这个widget为例，继承关系如下：
	1. Padding -> SingleChildRenderObjectWidget -> RenderObjectWidget -> Widget
	2. RenderPadding -> RenderShiftedBox -> RenderBox -> RenderObject
RenderElement挂载到树上后，拿widget.createRanderObject()方法传入自己并创建RenderObject ，RenderObject 保存对RenderElement 的引用，RenderObject创建时会拿对应widget的配置信息并保存下来并做diff，如果变化则打渲染相关操作的标记，等待下一帧绘制时做对应的Layout、paint等操作。
	1. 比如RenderPadding创建时会传入Padding配置的padding值，RenderPadding内部padding属性的set方法内会做校验，如果value和原来的相等直接return，不等则会调用markNeedsLayout，标记在下一帧绘制时需要重新布局performLayout。
	2. 同样如果是一个Opacity渲染类，最终内部diff过滤后会调用markNeedsPaint，标记在下一帧绘制时需要重新paint。

**重要点**
* 并不是所有的Widget都会被独立渲染，只有继承RenderObjectWidget的才会创建RenderObject对象。
* Widget创建Element，同时Element拿有Widget
* RenderElement挂载后传入自己并创建需要被渲染的RenderObjectWidget对应的RenderObject，RenderObject拿有RenderElement，同时RenderElement内部也拿有RenderObject。



、




