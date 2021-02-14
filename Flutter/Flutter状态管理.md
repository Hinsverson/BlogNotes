# Flutter状态管理
#flutter

[Flutter 系统教程](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg5MDAzNzkwNA==&action=getalbum&album_id=1566028536430247937&scene=173&from_msgid=2247483765&from_itemidx=1&count=3#wechat_redirect)

[Flutter | 状态管理指南篇——Provider](https://juejin.cn/post/6844903864852807694#heading-26)
[简单的应用状态管理  - Flutter 中文文档 - Flutter 中文资源](https://flutter.cn/docs/development/data-and-backend/state-mgmt/simple)
[Flutter状态State管理](https://mp.weixin.qq.com/s?__biz=Mzg5MDAzNzkwNA==&mid=2247483789&idx=1&sn=c552cb4aba354ce3509d39b402e4c7c2&chksm=cfe3f272f8947b6402e9ba08af81ae05ee87176e81b0e06b025bf651e7f1434823b34017d2d2&scene=178&cur_album_id=1566028536430247937#rd)
[Flutter状态管理 - 初探与总结](https://juejin.cn/post/6844903842992095240#heading-1)

 [声明式的编程思维](https://flutter.cn/docs/development/data-and-backend/state-mgmt/declarative)  里  [短时 (ephemeral) 与应用 (app) 状态](https://flutter.cn/docs/development/data-and-backend/state-mgmt/ephemeral-vs-app)  之间的区别主要就是当前的state变量是在控件内部使用，还是需要共享出去外部使用，比如你维护一个TabController内的index，那么index属于一个短时state，直接作为变量封装在TabController内部就好。

在 Flutter 中，每次当 widget 内容发生改变的时候，需要明白的是你就需要构造一个新的widget，调用 MyCart(contents)（构造函数），而不是命令编程里MyCart.updateWith(somethingNew)（调用方法去更新）。
```dart
// BAD: DO NOT DO THIS
void myTapHandler() {
  var cartWidget = somehowGetMyCartWidget();
  cartWidget.updateWith(item);
}
```
但因为整个Flutter widget刷新机制的限制，你只能通过父类的 build 方法来构建新 widget，如果你想修改到contents，就需要去调用 MyCart 的父类甚至更高一级的类build。

响应state数据变化的处理，不要尝试通过从widget创建时传入响应回调，然后再build时刷新这种方式，虽然可以这样做，但可能需要大量传递回调函数，难于维护。Flutter原声的机制是使用InheritedWidget、InheritedNotifier、InheritedModel来为其子孙节点提供数据和服务。

# InheritedWidget使用
InheritedWidget可以用于在Widget树中向下共享数据。
1. 构造可观察的数据源widget：InheritedWidget抽象类提供`bool updateShouldNotify(covariant ShareDataWidget oldWidget)`方法，由子类实现发生数据变化的diff逻辑，这里的ShareDataWidget就充当了响应式编程里可观察数据源的角色。
```dart
class HYCounterWidget extends InheritedWidget {
  final int counter;  // 共享的数据
	HYCounterWidget({@required this.data, Widget child}) : super(child: child);
  //数据变化时的diff判断逻辑
	@override
  bool updateShouldNotify(HYCounterWidget oldWidget) {
    return oldWidget.counter != counter; //data不同时通知出去
  }
}
```

2. 获取监听到的数据并显示到展示性widget上：BuildContext抽象类提供了dependOnInheritedWidgetOfExactType方法（Element实现了它），可以获取到传入的ctx下widget树中上级中的InheritedWidget（也就是ShareDataWidget），然后可以拿到这个对象，从而访问数据。这里分别使用StatefulWidget和StatelessWidget来实现数据的获取并展示。
```dart
class HYCounterWidget extends InheritedWidget {
  ...
  //新增一个获取变化后ShareDataWidget对象的类方法
  static HYDataWidget of(BuildContext context) {
    return context.dependOnInheritedWidgetOfExactType();
  }
	...
}
```
```dart
class HYShowData01 extends StatefulWidget {
  @override
  _HYShowData01State createState() => _HYShowData01State();
}
class _HYShowData01State extends State<HYShowData01> {
  @override
  Widget build(BuildContext context) {
	  //通过类方法拿到变化的数据实例
    int counter = HYCounterWidget.of(context).counter;
    return Card(
      color: Colors.red,
      child: Text("当前计数: $counter", style: TextStyle(fontSize: 30),),
    );
  }
}
class HYShowData02 extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    //通过类方法拿到变化的数据实例
    int counter = HYCounterWidget.of(context).counter;
    return Container(
      color: Colors.blue,
      child: Text("当前计数: $counter", style: TextStyle(fontSize: 30),),
    );
  }
}
```

3. 使用数据源widget和展示性widget：HYHomePage内部把HYDataWidget设为HYShowData01和HYShowData02的后代Widget，点击button时修改数据，刷新界面，过程如下：
	* HYHomePage内setState修改_counter来到build重构widget
	* HYCounterWidget重构时传入新的_counter。
	* HYCounterWidget内部diff发现_counter不等，数据改变。
	* HYShowData01和HYShowData02重构时通过`HYCounterWidget.of(context)`方法拿到包含新数据的HYCounterWidget取出数据被展示。
```dart
class HYHomePage extends StatefulWidget {
  @override
  _HYHomePageState createState() => _HYHomePageState();
}
class _HYHomePageState extends State<HYHomePage> {
  int _counter = 100;
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("InheritedWidget"),
      ),
      body: HYCounterWidget( //数据源widget
        counter: _counter,
        child: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: <Widget>[
              HYShowData01(), //展示性widget
              HYShowData02() //展示性widget
            ],
          ),
        ),
      ),
      floatingActionButton: FloatingActionButton(
        child: Icon(Icons.add),
        onPressed: () {
          setState(() {
            _counter++;
          });
        },
      ),
    );
  }
}
```

## InheritedWidget数据共享原理
每次build widget树时，从根widget开始一直创建到展示性widget，展示性widget build时使用dependOnInheritedWidgetOfExactType完成Element树上父InheritedElement节点和当前Element节点的关联，之后返回InheritedWidget，最后访问InheritedWidget内的数据并展示，具体实现逻辑如下：
	1. 根据类型T从当前Element的祖先InheritedElement记录表中取出InheritedElement，调用dependOnInheritedElement。
	2. dependOnInheritedElement内把取出的InheritedElement添加到了当前Element的InheritedElement依赖集合_dependencies中，也把当前Element传给了InheritedElement，完成了InheritedElement和当前Element的关联。
	3. 返回ancestor.widget也就是取到的InheritedElement对象对应的InheritedWidget对象。
```dart
@override
T dependOnInheritedWidgetOfExactType<T extends InheritedWidget>({Object aspect}) {
  //_inheritedWidgets是一个Map<Type, InheritedElement>
  final InheritedElement ancestor = _inheritedWidgets == null ? null : _inheritedWidgets[T];
  if (ancestor != null) {
    return dependOnInheritedElement(ancestor, aspect: aspect) as T;
  }
  _hadUnsatisfiedDependencies = true;
  return null;
}

@override
InheritedWidget dependOnInheritedElement(InheritedElement ancestor, { Object aspect }) {
  _dependencies ??= HashSet<InheritedElement>();
  //把取出的InheritedElement添加到了当前Element的_dependencies中（_dependencies是Set<InheritedElement>）
  _dependencies.add(ancestor);
  //把当前Element传给了InheritedElement(后面widget更新流程里会用到)
  ancestor.updateDependencies(this, aspect);
  return ancestor.widget; //返回widget
}
```

## InheritedWidget状态共享通知原理
在外部widget中点击按钮，会调用State的setState方法，引发重绘：
	1. WidgetsBinding.drawFrame
	2. BuildOwner.buildScope
	3. 根Element.rebuild
根rebuild后会调用到子InheritedElement的update方法（由父类ProxyElement实现），主要有2步动作：
1. 第一步首先调用InheritedElement自身的updated方法，updated内部调用updateShouldNotify检查（数据变化的diff判断）是否需要通知关联的子Element。 如果需要则通知`super.updated(oldWidget)`将找到之前InheritedElement关联依赖的子Element，调用子Element的didChangeDependencies方法把widget的变化通知出去。
	* 对于StatefulElement类型，重写了didChangeDependencies方法，并把内部的_didChangeDependencies先改为true表示依赖已更新。
```dart
//ProxyElement内update方法实现
@override
void update(ProxyWidget newWidget) {
  final ProxyWidget oldWidget = widget;
  super.update(newWidget);
  updated(oldWidget);//1
  _dirty = true;
  rebuild();//2
}
//InheritedElement内updated方法实现
@override
void updated(InheritedWidget oldWidget) {
  //此刻的widget为上面super.update(newWidget)更新后的新widget;
  if (widget.updateShouldNotify(oldWidget))
    super.updated(oldWidget);
}
```
2. 第二步执行rebuild后，最终执行到Element树下子Element的performRebuild方法里完成rebuild。
	* 对于StatefulElement类型，重写了performRebuild，当前面_didChangeDependencies改为true时，就会调用State的didChangeDependencies方法，把widget的变化通知到state中去。
```dart
//StatefulElement类型的performRebuild
@override
void performRebuild() {
  if (_didChangeDependencies) {
    _state.didChangeDependencies();
    _didChangeDependencies = false;
  }
  super.performRebuild();
}
```

## InheritedWidget总结
### 整体理解
1. build向下构建Widget和Element节点树时，每个Element节点会维护一张InheritedElement表（key是InheritedWidget类型，value是InheritedElement），记录树上的父InheritedElement。
2. 展示性Widget build展示数据时通过Context（也就是当前Element）实现的dependOnInheritedWidgetOfExactType方法从InheritedElement表中取出上层父InheritedElement并做它和父InheritedElement之间的相互依赖关联，最后返回InheritedElement.widget（InheritedWidget）对象取出数据做展示。
3. InheritedElement在rebuild时，会调用update方法，内部会拿新widget的updateShouldNotify方法传入旧widget检查是否需要通知widget依赖变化的diff逻辑，需要的话就调用didChangeDependencies方法把依赖变化通知出去。
4. 对于StatefulElement类型，它重写了didChangeDependencies方法，并把内部的_didChangeDependencies先改为true表示依赖已更新，再在performRebuild时根据是否已更新（_didChangeDependencies为true）时调用state的didChangeDependencies方法（注意个Element的同名方法取分开）把通知通知到了state中去。

### 注意点
* setState所在的State的Widget必须是ShareDataWidget或者它的父级，才会引发ShareDataWidget及子级的刷新。
* build的情况，不一定会调用State的didChangeDependencies（一定是widget发生了变化，内部先被标记了_didChangeDependencies为true），但是调用didChangeDependencies，肯定伴随着build。
* StatefulWidget配合dependOnInheritedWidgetOfExactType才能同时有获取数据和数据改变后通知回调的效果，StatelessWidget，也能刷新界面，但不能也没有didChangeDependencies回调。
* getElementForInheritedWidgetOfExactType、findAncestorWidgetOfExactType等方法也能获取数据，但是不能建立依赖关系，即使是StatefulWidget也不能收到didChangeDependencies回调。
* 一般不需要重写didChangeDependencies，除非需要在依赖更新时做一些操作。

### 生命周期测试
* 首次build
	* flutter: _HYHomePageState build
	* flutter: HYCounterWidget init
	* flutter: _HYShowData01State didChangeDependencies
	* flutter: _HYShowData01State build
	* flutter: context.dependOnInheritedWidgetOfExactType()
* 点击按钮后
	* flutter: _HYHomePageState build
	* flutter: HYCounterWidget init
	* flutter: updateShouldNotify(true)
	* flutter: _HYShowData01State didChangeDependencies
	* flutter: _HYShowData01State build
	* flutter: context.dependOnInheritedWidgetOfExactType()
* 修改updateShouldNotify为false时：
	* flutter: _HYHomePageState build
	* flutter: HYCounterWidget init
	* flutter: updateShouldNotify(false)
	* flutter: _HYShowData01State build
	* flutter: context.dependOnInheritedWidgetOfExactType()

# Provider
基于InheritedWidget，更reactive
1. ChangeNotifier：用于向监听器发送通知，是flutter:foundation的一部分，你可以订阅它的状态变化，相当于Rx中的Observable可观察对象。（比如你写一个Model继承自ChangeNotifier，在model数据源增删变化后，调用ChangeNotifier 提供的notifyListeners()，就可以把数据变化通知出去）
```dart
class CartModel extends ChangeNotifier {
	final List<Item> _items = [];
	oid add(Item item) {
    _items.add(item);
    notifyListeners();
  }
  void removeAll() {
    _items.clear();
    notifyListeners();
  }
}
//测试ChangeNotifier的变化响应
test('adding item increases total cost', () {
  final cart = CartModel();
  final startingPrice = cart.totalPrice;
  cart.addListener(() { //响应回调
    expect(cart.totalPrice, greaterThan(startingPrice));
  });
  cart.add(Item('Dash')); //内部会调用notifyListeners()
});
```

2. ChangeNotifierProvider：使用这个widget 向其子孙节点暴露一个
ChangeNotifier，它属于provider package，它不会重复实例化
暴露的这个ChangeNotifier，并且如果该实例已经不会再被调用，
也会自动调用该ChangeNotifier实例的dispose()方法。MultiProvider可以用来暴露多个不同ChangeNotifier实例。
```dart
void main() {
  runApp(
    ChangeNotifierProvider(
      create: (context) => CartModel(),
      child: MyApp(),
    ),
  );
}
runApp(MultiProvider(
  providers: [
    ChangeNotifierProvider(create: (ctx) => CounterProvider()),
    ChangeNotifierProvider(create: (ctx) => UserProvider()),
  ],
  child: MyApp(),
));
```

3. Consumer：构造时传入 builder，当 ChangeNotifier 发生变化的时候会调用 builder 这个builder函数，相当于在构造Rx中的Observer订阅回调，也就是说当你在模型中调用 notifyListeners() 时，所有和 Consumer 相关的 builder 方法都会被调用。
	* 第一个参数是每个build方法都会有上下文，目的是知道当前树的位置。
	* 第二个参数是 ChangeNotifier 的实例，它是我们最开始Provider时创建的实例。
	* child是用于优化目的，如果 Consumer 下面有一个庞大的子树，当模型发生改变的时候，该子树并不会改变，那么你就可以仅仅创建它一次，然后通过 builder 获得该实例。
```dart
return Consumer<CartModel>(
  builder: (context, cart, child) => Stack(
        children: [
          // Use SomeExpensiveWidget here, without rebuilding every time.
          child,
          Text("Total price: ${cart.totalPrice}"),
        ],
      ),
  // Build the expensive widget here.
  child: SomeExpensiveWidget(),
);
```

**注意点：**
* 最好把 Consumer 放在 widget 树尽量低的位置上，避免UI 上任何一点小变化就全盘重新构建 widget 。
* 有的时候不需要模型中的数据来改变 UI，只是单纯需要访问该数据时，可以使用`Provider.of<CartModel>(context, listen: false).removeAll();`并且将listen设置为false。虽然Consumer也可以做到，但是重构了一个无需重构的 widget。
* 一般场景下使用Consumer比Provider.of更高效，因为Provider.of基于InheritedWidget，在一个setState数据改变后会从对应widget开始rebuild子树，这样也会重构很多无需变更的widget，而使用Consumer当数据变化时，只会进行Consumer的builder响应，并且可以使用child优化不需要改变的子widget树。
* 除了Consumer 还有一种Selector，是另外一种状态变更的响应形式。

Provider的更多用法参考[Flutter | 状态管理指南篇——Provider](https://juejin.cn/post/6844903864852807694#heading-27)

# 其他状态管理方式
[状态 (State) 管理参考  - Flutter 中文文档 - Flutter 中文资源](https://flutter.cn/docs/development/data-and-backend/state-mgmt/options)
[Flutter | 状态管理指南篇——Provider](https://juejin.cn/post/6844903864852807694#heading-27)
[Flutter状态管理 - 初探与总结](https://juejin.cn/post/6844903842992095240#heading-1)

