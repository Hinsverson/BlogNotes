# Flutter布局
#flutter

[Layout widgets  - Flutter 中文文档 - Flutter 中文资源](https://flutter.cn/docs/development/ui/widgets/layout)
[深入理解 Flutter 布局约束  - Flutter 中文文档 - Flutter 中文资源](https://flutter.cn/docs/development/ui/layout/constraints)
[深入研究Flutter布局原理](https://juejin.cn/post/6897012238318895117)
[七、 深入布局原理 · Flutter 完整开发实战详解系列](https://wizardforcel.gitbooks.io/gsyflutterbook/content/Flutter-7.html)

所有布局用的 widgets 都具有以下任一项：
	* 一个 child 属性，如果它们只包含一个子项 —— 例如 Center 和 Container
	* 一个 children 属性，如果它们包含多个子项 —— 例如 Row、Column、ListView 和 Stack
Flutter最常见的布局模式之一是垂直或水平 widgets，Row和Column
![](Flutter%E5%B8%83%E5%B1%80/B019F43E-1C9D-4DEB-B1F6-F48EE319845C.png)
Container只有child属性，一般用来描述padding、borders、margins、背景色等类UIView通用属性。

SizedBox: 两种用法：一是可用来设置两个widget之间的间距，二是可以用来限制子组件的大小。[Flutter 容器(5) - SizedBox - 荣光无限 - 博客园](https://www.cnblogs.com/leslie1943/p/13364771.html)


在Flutter的组件体系中，并非所有的Widget都会渲染到最后的页面上，整个Widget大概可以分为三类：
* **组合类**：StatelessWidget和StatefulWidget
* **代理类**：InheritedWidget
* **绘制类**：RenderObjectWidget
所有我们在屏幕上看到的UI最终几乎都会通过RenderObjectWidget实现。而RenderObjectWidget中有个createRenderObject()方法生成RenderObject对象，RenderObject实际负责实际的layout()和paint()。

RenderObject的的绘制过程分为三个阶段build()、layout()、paint()。build()方法由组合类和代理类Widget实现，layout()和paint()由RenderObject实现。

Container组件其实只是一个组合类的控件，在其中封装了多个负责绘制的原子组件。

Flutter中的layout()职能主要是计算控件自身的尺寸和位置偏移。
Flutter整个布局从最顶级的节点开始传递约束，从下开始返回测量结果的过程。

relayoutBoundary的作用：当一个控件的大小被改变时可能会影响到它的 parent，因此 parent 也需要被重新布局，那么到什么时候是个头呢？答案就是 relayoutBoundary，如果一个 RenderObject 是 relayoutBoundary，就表示它的大小变化不会再影响到 parent 的大小了，于是 parent 也就不用重新布局了。

# 布局控件的基本使用
[GitHub - yang7229693/flutter-study: Flutter Study](https://github.com/yang7229693/flutter-study)

# 约束Constraints理解
![](Flutter%E5%B8%83%E5%B1%80/12B63420-CB32-4972-BD76-4B2C1ED7888F.png)
Flutter 中的控件在屏幕上绘制渲染之前需要先进行布局（Layout）操作。其具体可分为两个线性过程：从顶部向下传递约束，从底部向上传递布局信息。

约束Constraints在Flutter中是一种布局协议，Flutter中有两大布局协议BoxConstraints和SliverConstraints。

## BoxConstraints
```dart
//BoxConstraints定义
BoxConstraints({
	this.minWidth,
	this.maxWidth,
	this.minHeight,
	this.maxHeight,
});
```
对于非滑动的控件例如Padding，Flex等一般都使用BoxConstraints盒约束，而BoxConstraints有许多工厂构造方法，根据设置的width和height以及对应的max和min主要有如下两种区分：
* tight（紧约束）：当max和min值相等时，这时传递给子类的是一个确定的宽高值。ModalBarrier采用了这种约束，它被嵌套在了Route中，使用的是expand，所以每次我们push一个新Route的时候，默认新的页面就是撑满屏幕的。
* loose（松约束）：当max和min不相等的时候，这种时候对子类的约束是一个范围，称为松约束。Scaffold组件就采用了这种布局，所以Scaffold对于子布局传递的是一个松的约束。

比如父节点传入 Min Width 为 150，Max Width 为 300 的 BoxConstraints。当子节点接受到该约束，便可以取得上图中绿色范围内的值，即宽度在 150 到 300 之间，高度大于 100，当取得具体的值之后再将取得具体的大小的值上传给父节点，从而达到父子的布局通信。
![](Flutter%E5%B8%83%E5%B1%80/40C2ED87-B5A4-4CCF-B0C5-3BF129C34FC6.png)

### Container widget被撑开占满页面的原因
创建一个`Container(color: Colors.red)`widget返回并显示，它会默认撑满整个屏幕，原因就是Container作为组合widget内部会创建许多不同的widget来完成功能指责，最后layout时对应的renderObjec层级内（一共三层分别是RenderDecoratedBox，RenderLimitedBox，RenderConstrainedBox）的performLayout方法实现里向下传递的都是紧约束，而最外层的RenderDecoratedBox的约束来自Route，是由BoxConstraints.expand工厂方法构造出来的撑开型紧约束（最后计算的size就是屏幕大小），所以最后显示的Container会撑满整个Route，详细分析如下：

1. 首先RenderDecoratedBox，RenderLimitedBox，RenderConstrainedBox这3类RenderObject都继承自RenderProxyBox，这个类混入了RenderProxyBoxMixin，里面有基本的performLayout布局方法实现：
	* 如果他有子节点的的时候他会把自己父节点的约束constraints传递给节点，然后使用子节点的尺寸作为自己的。
	* 如果不是就performResize
```dart
//RenderProxyBoxMixin
@override
void performLayout() {
  if (child != null) {
    child!.layout(constraints, parentUsesSize: true);
    size = child!.size;
  } else {
    performResize();
  }
}
```

2. 而子类们都重写了performLayout的实现，如下所示，同基本的performLayout差不多。不过对于这几个子类，还可以添加额外的_additionalConstraints约束信息（外部设置的），这个时候layout也会考虑。
```dart
//RenderConstrainedBox内
@override
void performLayout() {
  final BoxConstraints constraints = this.constraints;
  if (child != null) {
    child!.layout(_additionalConstraints.enforce(constraints), parentUsesSize: true);
    size = child!.size;
  } else {
    size = _additionalConstraints.enforce(constraints).constrain(Size.zero);
  }
}
```

3. BoxConstraints约束框有一个enforce方法，通过传入一个参考BoxConstraints，返回一个新的BoxConstraints，返回的结果它会尊重参考的约束，同时与原始约束尽可能接近。
关键就是enforce这个方法，这里他接受到的参数是来自页面Route中的紧约束（max=min=屏幕宽度)，这样无论你为自身添加的约束_additionalConstraints是多少，他都会返回一个紧约束（max=min=屏幕宽度)，所以经过几层传递之后，在最下面的RenderConstrainedBox这里还是紧约束。
```dart
//BoxConstraints内
//返回新的框约束，它尊重给定的约束，同时与原始约束尽可能接近
BoxConstraints enforce(BoxConstraints constraints) {
  return BoxConstraints(
    minWidth: minWidth.clamp(constraints.minWidth, constraints.maxWidth),
    maxWidth: maxWidth.clamp(constraints.minWidth, constraints.maxWidth),
    minHeight: minHeight.clamp(constraints.minHeight, constraints.maxHeight),
    maxHeight: maxHeight.clamp(constraints.minHeight, constraints.maxHeight),
  );
}
```

4. 即使再在container外面包一层ConstraintsBox，设置具体的宽高，也还是会被撑满页面，因为对应performLayout的实现决定了它向下传递的还是一个紧约束。
![](Flutter%E5%B8%83%E5%B1%80/E33BD938-E7E0-4469-92FE-93289544A6C7.png)

# 自定义布局控件
![](Flutter%E5%B8%83%E5%B1%80/5ce207161a7949309e8578ba587a12ef~tplv-k3u1fbpfcp-watermark.image.png)
Alignment以Rect的中间为基点，定义为0，0。可以依此建立一个坐标系，横向取值-1.0~1.0，实际就是一个百分比，根据Rect的宽度来计算百分比，从而确定横向的位置，以下是实现源码：
```dart
class CustomAlign extends SingleChildRenderObjectWidget {
  final Alignment alignment; //对齐信息

  const CustomAlign({Key key, Widget child, this.alignment = Alignment.topLeft})
      : super(key: key, child: child);

  //创建renderObject
  @override
  RenderObject createRenderObject(BuildContext context) {
    return AlignRenderBox(alignment: alignment);
  }
  //变化时更新renderObject.alignment
  @override
  void updateRenderObject(
      BuildContext context, covariant AlignRenderBox renderObject) {
    renderObject.alignment = alignment;
  }
}
```
```dart
class AlignRenderBox extends RenderBox
    with RenderObjectWithChildMixin<RenderBox> {
  Alignment alignment;

  AlignRenderBox({RenderBox child, this.alignment}) {
    this.child = child;
  }

  @override
  void performLayout() {
    ///super.performLayout(); 自定义布局不要调用super.performLayout()
    if (child == null) {
      size = Size.zero;///没有child则不占用控件
    } else {
      size = constraints.constrain(Size.infinite); //尽可能填满
      child.layout(constraints.loosen(), parentUsesSize: true); //不对child的大小进行限制，在child变化时获得通知更新布局信息
      //为child设置新的parentData数据，这里是改变偏移
      BoxParentData parentData = child.parentData as BoxParentData;
      parentData.offset = alignment.alongOffset(size - child.size as Offset); //设置偏移
    }
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    super.paint(context, offset); //需要调用super
    if (child != null) {
      //如果child不为空 绘制child 并为child
      BoxParentData parentData = child.parentData as BoxParentData;
      //结合自己的offset和为child设置的offset去画child
      context.paintChild(child, offset + parentData.offset);
    }
  }
}
```
