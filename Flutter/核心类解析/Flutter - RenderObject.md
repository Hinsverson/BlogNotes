# Flutter - RenderObject
#flutter/核心类分析


[14.3：RenderObject与RenderBox · 《Flutter实战》](https://book.flutterchina.club/chapter14/render_object.html)

RenderObject的主要职责是Layout和绘制

RenderObject就是渲染树中的一个节点，通过parentData插槽（slot）向子节点存储一些和子元素相关的数据。如在Stack布局中，RenderStack就会将子元素的偏移数据存储在子元素的parentData中（具体可以查看Positioned实现）。

RenderObject类本身实现了一套基础的layout和绘制协议，但是并没有定义子
节点模型（如一个节点可以有几个子节点，没有子节点？一个？两个？或者更多？）。
坐标系统（如子节点定位是在笛卡尔坐标中还是极坐标？）
具体的布局协议（是通过宽高还是通过constraint和size?，或者是否由父节点在子节点布局之前或之后设置子节点的大小和位置等）。

Flutter提供了一个RenderBox类，它继承自`RenderObject，布局坐标系统采用笛卡尔坐标系，除非遇到要自定义布局模型或坐标系统的情况。一般使用这个就好。


# RenderObject主要方法介绍
[Flutter核心原理之RenderObject和RenderBox - 梁飞宇 - 博客园](https://www.cnblogs.com/lxlx1798/articles/11174987.html)

### 重要理解总结
* 通过重写performLayout()或者performResize()方法，我们确认此RenderObject的大小信息。
* 通过在performLayout()方法中为child调用layout，来传递布局约束信息。
* paint方法里我们可以进行一些绘制，如果有child，还需要为child进行绘制。绘制顺序不同也会影响实际显示效果，诸如Decoration的实现，前景色和背景色的效果主要还是绘制顺序的影响。
* 想要改变child显示的位置，我们需要给child设置一定的绘制Offset，Offset是相对于此Widget而言的，通常来说，Offset存储在child的parentData里，调用paintChild绘制时，需要取出parentData中的Offset结合上文传递的Offset进行绘制。

RenderObject 作为一个抽象类。每个节点需要实现它才能进行实际渲染。扩展 RenderOject 的两个最重要的类是RenderBox 和 RenderSliver。这两个类分别是应用了 Box 协议和 Sliver 协议这两种布局协议的所有渲染对象的父类，其还扩展了数十个和其他几个处理特定场景的类，并实现了渲染过程的细节，如 RenderShiftedBox 和 RenderStack 等等。

### setupParentData：设置父节点向子节点提供的通信数据

`void setupParentData(covariant RenderObject child)`
重写此方法，可以在child被加入进来之前为他设置一个ParenData，定义不同的ParenData可以存储一些我们需要的信息，满足不同的需求。比如RenderAligningShiftedBox这个renderObject会重写这个方法并为child的parentData设置一个Offset。

当layout结束后，每个节点的位置（相对于父节点的偏移）就已经确定了，RenderObject就可以根据位置信息来进行最终的绘制。但是在layout过程中，节点的位置信息怎么保存？对于大多数RenderBox子类来说如果子类只有一个子节点，那么子节点偏移一般都是Offset.zero ，如果有多个子节点，则每个子节点的偏移就可能不同。而子节点在父节点的偏移数据正是通过RenderObject的parentData属性来保存的。在RenderBox中，其parentData属性默认是一个BoxParentData对象，该属性只能通过父节点的setupParentData()方法来设置：

一定要注意，RenderObject的parentData 只能通过父元素设置.ParentData并不仅仅可以用来存储偏移信息，通常所有和子节点特定的数据都可以存储到子节点的ParentData中，如ContainerBox的ParentData就保存了指向兄弟节点的previousSibling和nextSibling，Element.visitChildren()方法也正是通过它们来实现对子节点的遍历。再比如KeepAlive Widget，它使用KeepAliveParentDataMixin（继承自ParentData） 来保存子节的keepAlive状态。

### markNeedsLayout：标记节点需要重新布局，下一桢刷新
```dart
void markNeedsLayout() {
  ...
  assert(_relayoutBoundary != null);
  if (_relayoutBoundary != this) {
    markParentNeedsLayout();
  } else {
    _needsLayout = true;
    if (owner != null) {
      ...
      owner._nodesNeedingLayout.add(this);
      owner.requestVisualUpdate();
    }
  }
}
```
此方法会对布局信息进行标记，渲染管道会在下一次渲染循环对此对象的布局信息进行更新。

当一个控件的大小被改变时可能会影响到它的 parent，因此 parent 也需要被重新布局，那么到什么时候是个头呢？答案就是 relayoutBoundary，

如果一个 RenderObject 自身是 relayoutBoundary，就表示它的大小变化不会再影响到 parent 的大小了，于是 parent 也就不用重新布局了。否则会一直继续向祖先节点查找，一直向上查找到是 relayoutBoundary 的 RenderObject为止，然后再将其标记为 dirty ，祖先节点layout后，自身也会被layout完成布局。

### layout：决议自己和子节点的布局约束
```dart
void layout(Constraints constraints, { bool parentUsesSize = false }){
  ...
  RenderObject? relayoutBoundary;
  if (!parentUsesSize || sizedByParent || constraints.isTight || parent is! RenderObject) {
    relayoutBoundary = this;
  } else {
    relayoutBoundary = (parent as RenderObject)._relayoutBoundary;
  }
  _constraints = constraints;
  _relayoutBoundary = relayoutBoundary;
  if (sizedByParent) {
    ...
    performResize();
    ...
  }
  performLayout();
  ...
}
```
Flutter中的layout主要负责完成Widget的Constraints布局约束的配置。Constraints会影响RenderObject展示的区域。

在布局阶段，父节点会调用子节点的layout()方法，这个方法会将Constraints传递给child，用来描述child可以使用的宽高约束，child需要遵循这个约束。如果需要child的宽高来调整大小信息，则需要传递parentUsesSize为true，这样当child布局信息发生改变之后会通知此RenderObject来更新布局信息（通过_relayoutBoundary）。
如果RenderObject定义为容器，那么他需要为容纳的所有RenderObject调用此方法。

这个方法数据串起流程的模版方法，不应该被重写，具体的布局自己和子节点的行为由子类重写performLayout或者performResize完成。

### relayoutBoundary：relayout的刷新范围标记
relayoutBoundary整体理解：在layout时被设置，用来表示当前render节点的布局更新会影响到的祖先节点范围，所以markNeedsLayout当前节点时同样会mark影响到的祖先节点，以便下一个渲染循环时更新祖先节点的布局，更新更新祖先节点的布局时又会向下完成自身的布局更新。

RenderBox子类要定制布局算法不应该重写layout()方法，因为对于任何RenderBox的子类来说，它的layout流程基本是相同的，不同之处只在具体的布局算法，而具体的布局算法子类应该通过重写performResize() 和 performLayout()两个方法来实现，他们会在layout()中被调用。

### sizedByParent：表示size是否由parent唯一确定
这个get方法默认返回的是false，这样返回的话表明这个RenderObject的大小信息不会受到父容器的影响，这样我们可以在performLayout方法中为这个RenderObject设置大小信息。
如果重写为true，表示父节点传来的constraints是决定当前节点尺寸的唯一参考，那么size应该在performResize()中通过constraints去设置，而**不用**在performLayout()中再参考子节点的size。当然，这只是一种约定，但建议遵守。

### performResize：当sizedByParent为true时，在这里进行重新布局
`void performResize()`
重写此方法以用来更新Widget的大小信息。如果想要依赖父容器的大小来设置大小，则需要sizedByParent返回true。父容器完成布局工作后，这个方法会被调用，可以在这里拿到父容器传递的布局约束信息。这个方法只有sizedByParent返回true的时候才会被调用。

### performLayout：当sizedByParent为false时，在这里进行重新布局
`void performLayout()`
通常情况下我们会重写此方法来为此RenderObject设置大小信息，如果有child，还需要在这个方法里为child调用layout方法，完成child的布局。

### paintBounds：可以拿到此RenderObject应该绘制的区域
`Rect get paintBounds`
通过这个方法拿到此RenderObject应该绘制的区域。如果返回为null，那么这个RenderObject将会被完整绘制。这个Rect也是showOnScreen方法显示Rect所使用的。

### paint：自定义绘制的方法
`void paint(PaintingContext context, Offset offset)`
重写此方法，通过调用PaintingContext.canvas ，我们可以拿到Canvas对象来进行一些绘制。通常情况下，我们需要把此canvas的起点平移到offset，如果不进行移动，canvas的默认起点将会是屏幕的左上角。

如果此RenderObject是一个容器类型，那么可以在这里调用PaintingContext.paintChild来绘制指定的child，paintChild方法需要传递一个child和Offset，根据传入的Offset不同，会影响到child在屏幕中显示位置。

相对iOS的drawRect方法，Flutter中的paint方法里进行绘制的时候，需要根据Offset进行偏移绘制，这个Offset定义了此RenderObject应该绘制的起点信息。
Offset类中重写了操作符&，我们之前也确定了此RenderObject的Size，通过Offset & Size我们可以拿到Rect，这个区域就是此RenderObject应该绘制的区域。（虽然通过offset & size可以拿到一个rect，但是我们实际绘制中是可以不在这个区域内进行的，你可以在屏幕的任何区域进行绘制，但是最好还是遵守此区域的设置，不要绘制出边界，除非你有特殊的需求。）
```dart
Rect operator &(Size other) => Rect.fromLTWH(dx, dy, other.width, other.height);
```

### isRepaintBoundary：Layer的绘制标记

不是每个**RenderObject**都具有**Layer**的，因为这受**isRepaintBoundary**的影响。

决定这个RenderObject重绘时是否独立于其父元素，如果该属性值为true ，则生成layer独立绘制，反之从父节点中获取 Layer ，与父节点一起绘制。所以很明显，正确使用isRepaintBoundary属性可以提高绘制效率，避免不必要的重绘，原理如下：

当调用 markNeedsPaint() 方法时，会从当前 RenderObject 开始一直向父节点查找，直到找到 一个isRepaintBoundary 为 true的RenderObject 时，才会触发重绘，这样便可以实现局部重绘。当有RenderObject 绘制的很频繁或很复杂时，可以通过RepaintBoundary Widget来指定isRepaintBoundary 为 true，这样在绘制时仅会重绘自身而无需重绘它的 parent，如此便可提高性能。



