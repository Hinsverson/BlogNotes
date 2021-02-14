# Flutter - Widget
#flutter/核心类分析

[从源码看flutter（一）：Widget篇](https://juejin.cn/post/6844904127181357069)
[深入浅出 Flutter Framework 之 Widget](https://juejin.cn/post/6844904152905023496)


Everything’s a widget.

Widget是描述Flutter UI 的基本单元，用于配置Element的，Widget本质上是 UI 的配置信息 (附带部分业务逻辑)，通过Widget可以做到：
	* 描述 UI 的层级结构 (通过Widget嵌套)；
	* 定制 UI 的具体样式 (如：font、color等)；
	* 指导 UI 的布局过程 (如：padding、center等)；
	* …
具备如下特点：
	* 声明式 UI —— 相对于传统 Native 开发中的命令式 UI，它在开发效率显著提升、UI 可维护性明显加强。
	* 不可变性 —— Widget都是不可变的(immutable)，即其内部成员都是不可变的(final)，对于变化的部分需要通过「Stateful Widget-State」的方式实现。
	* 组合大于继承 —— Widget设计遵循组合大于继承这一优秀的设计理念，通过将多个功能相对单一的Widget组合起来便可得到功能相对复杂的Widget。

## Widget分类
![](Flutter%20-%20Widget/AC07826A-2FFA-40C8-A950-936048F28277.png)
* Component Widget —— 组合类 Widget：这类 Widget 都直接或间接继承于StatelessWidget或StatefulWidget，上一小节提到过在 Widget 设计上遵循组合大于继承的原则，通过组合功能相对单一的 Widget 可以得到功能更为复杂的 Widget。平常的业务开发主要是在开发这一类型的 Widget。
* Proxy Widget —— 代理类 Widget：Proxy Widget本身并不涉及 Widget 内部逻辑，只是为Child Widget提供一些附加的中间功能，典型的如：
	* InheritedWidget用于在「Descendant Widgets」间传递共享信息。
	* ParentDataWidget用于配置「Descendant Renderer Widget」的布局信息。
* Renderer Widge —— 渲染类 Widge：是最核心的Widget类型，会直接参与后面的Layout、Paint流程，无论是Component Widget还是Proxy Widget最终都会映射到Renderer Widget上，因为只有Renderer Widget有与之一一对应的Render Object从而可被绘制到屏幕上。

## Widget类基本分析
*  **Key key**：在同一父节点下，用作兄弟节点间的唯一标识，主要用于控制当 Widget 更新时，对应的 Element 如何处理 (是更新还是新建)。
* **Element createElement()**：每个Widget都有一个与之对应的Element，由该方法负责创建，createElement可以理解为设计模式中的工厂方法，具体的Element类型由对应的Widget子类负责创建；
* **static bool canUpdate(Widget oldWidget, Widget newWidget)**：是否可以用 new widget 修改前一帧用 old widget 生成的 Element，而不是创建新的 Element，Widget类的默认实现为：2个Widget的runtimeType与key都相等时，返回true，即可以直接更新 (key 为 null 时，认为相等)。
### StatelessWidget
无状态-组合型 Widget，通过build方法描述组合 UI 的层级结构。
* **StatelessElement createElement()**：Stateless Widget对应的 Element 为StatelessElement，一般情况下StatelessWidget子类不必重写该方法，即子类对应的 Element 也是StatelessElement。
* **Widget build(BuildContext context)**：是 Flutter UI搭建体系中的核心方法之一，以『声明式 UI』的形式描述了该组合式 Widget 的 UI 层级结构及样式信息，也是开发 Flutter 应用的主要工作『场所』，该方法在 3 种情况下被调用：
	1. Widget第一次被加入到 Widget Tree 中 (更准确地说是其对应的 Element 被加入到 Element Tree 时，即 Element 被挂载mount时)。
	2. Parent Widget修改了其配置信息；
	3. 该 Widget 依赖的Inherited Widget发生变化时。
当Parent Widget或 依赖的Inherited Widget频繁变化时，build方法也会频繁被调用，因此提升build方法的性能就显得十分重要，Flutter 官方给出了几点建议：
	* 减少不必要的中间节点，即减少 UI 的层级，如：对于「Single Child Widget」，没必要通过组合「Row」、「Column」、「Padding」、「SizedBox」等复杂的 Widget 达到某种布局的目标，或许通过简单的「Align」、「CustomSingleChildLayout」即可实现。又或者，为了实现某种复杂精细的 UI 效果，不一定要通过组合多个「Container」，再附加「Decoration」来实现，通过 「CustomPaint」自定义或许是更好的选择。
	* 尽可能使用const Widget，为 Widget 提供const构造方法，关于 const constructor 推荐  [Dart Constant Constructors](https://japhr.blogspot.com/2012/12/dart-constant-constructors.html) 看这篇文章的评论。
	* 必要时，可以将「Stateless Widget」重构成「Stateful Widget」，以便可以使用「Stateful Widget」中一些特定的优化手法，如：缓存「sub trees」的公共部分，并在改变树结构时使用GlobalKey；
	* 尽量减小 rebuilt 范围，如：某个 Widget 因使用了「Inherited Widget」，导致频繁 rebuilt，可以将真正依赖「Inherited Widget」的部分提取出来，封装成更小的独立 Widget，并尽量将该独立 Widget 推向树的叶子节点，以便减小 rebuilt 时受影响的范围。
### StatefulWidget
有状态-组合型 Widget，StatefulWidget本身还是不可变的，只是其可变状态存在于State中。
* **StatefulElement createElement()** ——「Stateful Widget」对应的 Element 为StatefulElement，一般情况下StatefulWidget子类不用重写该方法，即子类对应的Element 也是StatefulElement。
* **State createState()** —— 创建对应的 State，该方法在对应的StatefulElement构造时被调用。可以简单地理解为当「Stateful Widget」被添加到 Widget Tree 时会调用该方法，同时主意：
	* 一个 Widget 实例可以对应多个 Element 实例 (也就是同一份配置信息 (Widget) 可以在 Element Tree 上不同位置配置多个 Element 节点)，因此，createState方法在「Stateful Widget」生命周期内可能会被调用多次。
	* 配有GlobalKey的 Widget 对应的 Element 在整个 Element Tree 中只有一个实例。
### State<StatefulWidget>
![](Flutter%20-%20Widget/D01283E9-C350-4F6F-BFA5-454B078D82DA.png)
State为Element提供额外的扩展信息，以达到可变状态的widget效果，State 的生命周期大致可以分为 8 个阶段：
1. 在对应的「Stateful Element」被挂载 (mount) 到树上时，通过
StatefulElement.constructor 调用 StatefulWidget.createState 创建 State 实例，State实例也会持有对应的Element，注意：
	* State与Element的绑定关系一经确定，在整个生命周期内不会再变了 (**Element 对应的 Widget 可能会变，但对应的 State 永远不会变**)，期间，Element可以在树上移动，但是带着「State」状态移动的)。
2. StatefulElement 在挂载过程中接着会调用State.initState，子类可以重写该方法执行相关的初始化操作 (此时可以引用context、widget属性)。
3. 同样在挂载过程中会调用State.didChangeDependencies，该方法在 State 依赖的对象 (如：「Inherited Widget」) 状态发生变化时也会被调用，*子类很少需要重写该方法，*除非有非常耗时不宜在build中进行的操作，因为在依赖有变化时build方法也会被调用。
4. 此时，State 初始化已完成，其build方法此后可能会被多次调用：
	1. 在状态变化时 State 可通过setState方法来触发其子树的重建。
	2. 
5. 此时，「element tree」、「renderobject tree」、「layer tree」已构建完成，完整的 UI 应该已呈现出来。此后因为变化，「element tree」中「parent element」可能会对树上该位置的节点用新配置 (Widget) 进行重建，当新老配置 (oldWidget、newWidget)具有相同的「runtimeType」&&「key」时，framework 会用 newWidget 替换 oldWidget，并触发一系列的更新操作 (在子树上递归进行)。同时，State.didUpdateWidget方法被调用，子类重写该方法去响应 Widget 的变化；
6. 在 UI 更新过程中，任何节点都有被移除的可能，State 也会随之移除，(如上一步中「runtimeType」||「key」不相等时)。此时会调用State.deactivate方法，由于被移除的节点可能会被重新插入树中某个新的位置上（重新插入操作必须在当前帧动画结束之前），故子类重写该方法以清理与节点位置相关的信息 (如：该 State 对其他 element 的引用)、同时不应在该方法中做资源清理。
7. 当节点被重新插入树中时，State.build方法被再次调用。
8. 对于在当前帧动画结束时尚未被重新插入的节点，State.dispose方法被执行，State 生命周期随之结束，此后再调用State.setState方法将报错。子类重写该方法以释放任何占用的资源。

**setState()**：更新状态触发rebuild，需要注意以下几点：
* 在State.dispose后不能调用setState；
* 在 State 的构造方法中不能调用setState；
* setState方法的回调函数 (fn) 不能是异步的 (返回值为Future)，原因很简单，因为从流程设计上 framework 需要根据回调函数产生的新状态去刷新 UI；
* 通过setState方法之所以能更新 UI，是在其内部调用了_element.markNeedsBuild()打上了rebuild标记，会在下一次build循环内重新build。
* 若State.build方法依赖了自身状态会变化的对象，如：ChangeNotifier、Stream或其他可以被订阅的对象，需要确保在initState、didUpdateWidget、dispose等 3 方法间有正确的订阅 (subscribe) 与取消订阅 (unsubscribe) 的操作：
	* 在initState中执行 subscribe。
	* 如果关联的「Stateful Widget」与订阅有关，在didUpdateWidget中先取消旧的订阅，再执行新的订阅。
	* 在dispose中执行 unsubscribe。
* 在State.initState方法中不能调用BuildContext.dependOnInheritedWidgetOfExactType，但State.didChangeDependencies会随之执行，在该方法中可以调用。

### RenderObjectWidget
* **RenderObjectElement createElement()** ——「RenderObject Widget」对应的 Element 为RenderObjectElement。由于RenderObjectElement也是抽象类，故子类需要重写该方法，比如RenderObjectWidget的几个子类LeafRenderObjectWidget、SingleChildRenderObjectWidget、MultiChildRenderObjectWidget只是重写了createElement方法以便返回各自对应的具体的 Element 类实例。
* **RenderObject createRenderObject(BuildContext context)** —— 核心方法，创建 Render Widget 对应的 Render Object，同样子类需要重写该方法。该方法在对应的 Element 被挂载到树上时调用(Element.mount)，即在 Element 挂载过程中同步构建了「Render Tree」。
* **void updateRenderObject(BuildContext context, covariant RenderObject renderObject)** —— 核心方法，在 Widget 更新后，修改对应的 Render Object。该方法在首次 build 以及需要更新 Widget 时都会调用，主要是用当前Widget描述的信息去更新renderObject上的描述信息。
* **void didUnmountRenderObject(covariant RenderObject renderObject)** —— 对应的「Render Object」从「Render Tree」上移除时调用该方法。

### ParentDataWidget
ParentDataWidget作为 Proxy 型 Widget，其功能主要是为其他 RenderObejctWidget在布局和渲染时通过`applyParentData(RenderObject renderObject)`方法提供ParentData信息。虽然其 child widget 不一定是RenderObejctWidget 类型，但其提供的ParentData信息最终都会落地到 RenderObejctWidget 类型子孙 Widget 上。

**applyParentData(RenderObject renderObject)** —— 这个方法的触发逻辑在RenderObjectElement的attachRenderObject内，RenderObjectElement节点会从自身向上找祖先节点，在查找过程中如查到是ParentDataElement，那么会拿这个ParentDataElement节点对应的widget节点调用它的applyParentData方法并传入renderObject去为该renderObject设置ParentData信息，比如：
	* Positioned的内部实现在applyParentData方法内必要时将自己的属性赋值给了对应的RenderObject.parentData (此处是StackParentData)，并对「parent render object」调用markNeedsLayout，以便重新 layout。

### InheritedWidget
InheritedWidget 用于在树上向下传递数据。 通过BuildContext.dependOnInheritedWidgetOfExactType可以获取最近的「Inherited Widget」，需要注意的是通过这种方式获取「Inherited Widget」时，当「Inherited Widget」状态有变化时，会导致该引用方 rebuild。

**InheritedElement createElement()** ——「Inherited Widget」对应的 Element 为InheritedElement，一般情况下InheritedElement子类不用重写该方法。
**bool updateShouldNotify(covariant InheritedWidget oldWidget)** —— 在「Inherited Widget」rebuilt 时判断是否需要 rebuilt 那些依赖它的 Widget。

详细请参考状态共享

## Widget的生命周期
[译 Flutter 核心概念详解： Widget、State、Context 及 InheritedWidget](https://juejin.cn/post/6844903784187953165)

[Flutter(七)之有状态的StatefulWidget](https://juejin.cn/post/6844903951058354190#heading-10)
![](Flutter%20-%20Widget/173829C8-933A-4109-8B55-4CA9A83FD60D.png)

![](Flutter%20-%20Widget/5B90EA45-82AA-4936-B1C9-B7FFD9041BFF.png)
```dart
flutter: HomeBody build
flutter: 执行了MyCounterWidget的构造方法
flutter: 执行了MyCounterWidget的createState方法
flutter: 执行MyCounterState的构造方法
flutter: 执行MyCounterState的init方法
flutter: 执行MyCounterState的didChangeDependencies方法
flutter: 执行执行MyCounterState的build方法
// 注意：Flutter会build所有的组件两次（查了GitHub、Stack Overflow，目前没查到原因）
flutter: HomeBody build
flutter: 执行了MyCounterWidget的构造方法
flutter: 执行MyCounterState的didUpdateWidget方法
flutter: 执行执行MyCounterState的build方法
```

## 理解Widget中的Key
什么是key？在widget从频繁销毁-重建时，可以使用Key来存储保持StatefulWidget的state状态并在重建时恢复。
什么时候需要使用Key？当你需要添加、删除、重新排序处于某种状态的相同类型的小部件集合时
如何合理适当的使用 Key？
	* When: 当您想要保留 Widget 树的状态时，请使用 Key。例如: 当修改相同类型的 Widget 集合（如列表中）时
	* Where: 将 Key 设置在要指明唯一身份的 Widget 树的顶部（按层Diff）。
	* Which: 根据在该 Widget 中存储的数据类型选择使用的不同类型的Key
	* GlobalKeys 看起来有点像全局变量，不过代价昂贵，有也其他更好的方法达到 GlobalKeys 的作用，比如 InheritedWidget、Redux 或 Block Pattern。
注意不要在key中使用随机数，那样的话每次widget的变化都会引起大量的重新映射，影响性能。

### Widget 的Diff刷新原理
[(搬运含外置字幕)何时使用密钥 - Flutter小部件 101 第四集_哔哩哔哩 (゜-゜)つロ 干杯~-bilibili](https://www.bilibili.com/video/av97539447?p=1)
[理解 Flutter 中的 Key - just_yang - 博客园](https://www.cnblogs.com/jiy-for-you/p/11707366.html)
```dart
@immutable
abstract class Widget extends DiagnosticableTree {
  const Widget({ this.key });
  final Key key;
  ···
  static bool canUpdate(Widget oldWidget, Widget newWidget) {
    return oldWidget.runtimeType == newWidget.runtimeType
        && oldWidget.key == newWidget.key;
  }
}
```
* 首先widget和element一一对应形成一个widget树和element树，当widget发生移动、删除等行为时导致了widget树的结构发生变化，也就是说打破了widget树和element树上元素widget和element之间的持有映射关系。
* flutter会根据按照widget树的层级顺序层层进行diff算法diff widget元素，依据就是新旧widget运行时类型和key是否相同`oldWidget.runtimeType == newWidget.runtimeType && oldWidget.key == newWidget.key`
* 如果结果为true，则不会更正element树上element元素和这个widget元素之间的映射关系。
* 如果结果为false（即新旧widget的key不一样或者类型都不一样），则会重建Element树上element和widget之间的映射关系，重建过程就是：
	* 根据新widget 的 key 在老 Element列表里面查找，找到则会更新这个element元素在element组织树上的位置（如果还存在renderObject，也会更新对应 renderObject在render组织树上位置更新渲染），也会重新建立这个旧element与这个新widget的相互映射关系。
	* 如果找不到则会拿新的widget重新创建一个element更新到element树上并建立新element和新widget的映射关系。
	
### Key的种类和作用
Key本身是一个抽象，不过它也有一个工厂构造器，创建出来一个ValueKey，直接子类主要有：
* LocalKey：使用这个类型key的widget在老element列表中查找时只会在父widget下的旧element列表（由element树上和旧widget对应的旧element同级的所有节点组成的一张表）中查找，所以适用于具有相同父节点的widget间的比较，有以下几种子类：
	* ValueKey：以特定的值（比如一个字符串、数字）作为key时使用。
		* PageStorageKey：当你有一个滑动列表，你通过某一个 Item 跳转到了一个新的页面，当你返回之前的列表页面时，你发现滑动的距离回到了顶部。这时候，给 Sliver 一个 PageStorageKey！它将能够保持 Sliver 的滚动状态。
	* ObjectKey：使用对象来作为key，比较的是对象的不同。
	* UniqueKey：如果要确保key的唯一性，可以使用这个。
* GlobalKey：GlobalKey的内部有一个全局静态Map` Map<GlobalKey, Element>`会缓存widget对应的element，通过 GlobalKey 可以拿到持有该GlobalKey的 Widget，以及对应Element和State信息。它允许该 widget 在应用中的任何位置更改父级而不会丢失 State ，或者可以使用它们在 widget 树的完全不同的部分中访问有关另一个 Widget 的信息，比如：
	* 要在两个不同的页面上显示相同的 Widget，同时保持相同的State状态，则需要使用 GlobalKeys（相当于说这是一个全局的widget），原理就是GlobalKey类的全局静态Map _registry会缓存widget对应的element，所以widget组织结构变化进行diff刷新查找映射element的时候会通过GlobalKey获取缓存的Element，达到state一致性。
	* 要在外部访问StatefulWidget对应的state管理类做一些事情时，因为state在element内部私有外部无法访问，所以可以通过key反向拿到widget对应的element和state信息。