# Flutter事件处理
#flutter

# 基本使用
[Flutter事件监听](https://mp.weixin.qq.com/s?__biz=Mzg5MDAzNzkwNA==&mid=2247483795&idx=1&sn=4ea10f4d6987a592b84885a271219849&chksm=cfe3f26cf8947b7a75e567280cd86270bb7f32abdfb3c596e325459ce8599884396328efcc6e&scene=178&cur_album_id=1566028536430247937#rd)

Flutter手势有2个层面，因为设计思想的缘故，区别于iOS创建对象并绑定target和设置响应事件方法这种用法，在使用时是通过使用特定的widget对象把事件的作用widget（类似iOS事件的target）当作child包起来，并把各种事件的响应回调传入进去（可选参数实现不同状态事件的回调传入），在合适的系统io时机从触发回调的响应，主要有2个层面：
1. 低级一点的Pointer Events，使用Listener Widget来监听
	* PointerDownEvent指针在特定位置与屏幕接触
	* PointerMoveEvent指针从屏幕的一个位置移动到另外一个位置
	* PointerUpEvent指针与屏幕停止接触
	* PointerCancelEven指针因为一些特殊情况被取消
2. 高级层面封装的Gesture Detector，使用GestureDetector Widget来监听
**点击**：
	* onTapDown：用户发生手指按下的操作
	* onTapUp：用户发生手指抬起的操作
	* onTap：用户点击事件完成
	* onTapCancel：事件按下过程中被取消
	**双击：**
	* onDoubleTap：快速点击了两次
	**长按：**
	* onLongPress：在屏幕上保持了一段时间
	**纵向拖拽：**
	* onVerticalDragStart：指针和屏幕产生接触并可能开始纵向移动；
	* onVerticalDragUpdate：指针和屏幕产生接触，在纵向上发生移动并保持移动；
	* onVerticalDragEnd：指针和屏幕产生接触结束；
	**横线拖拽：**
	* onHorizontalDragStart：指针和屏幕产生接触并可能开始横向移动；
	* onHorizontalDragUpdate：指针和屏幕产生接触，在横向上发生移动并保持移动；
	* onHorizontalDragEnd：指针和屏幕产生接触结束；
	**移动：**
	* onPanStart：指针和屏幕产生接触并可能开始横向移动或者纵向移动。如果设置了 onHorizontalDragStart 或者 onVerticalDragStart，该回调方法会引发崩溃；
	* onPanUpdate：指针和屏幕产生接触，在横向或者纵向上发生移动并保持移动。如果设置了 onHorizontalDragUpdate 或者 onVerticalDragUpdate，该回调方法会引发崩溃。
	* onPanEnd：指针先前和屏幕产生了接触，并且以特定速度移动，此后不再在屏幕接触上发生移动。如果设置了 onHorizontalDragEnd 或者 onVerticalDragEnd，该回调方法会引发崩溃。
```dart
class HomeContent extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Center(
      child: Listener( //基本事件监听器
        child: Container( //作用范围target
          width: 200,
          height: 200,
          color: Colors.red,
        ),
        //各种事件响应
        onPointerDown: (event) => print("手指按下:$event"),
        onPointerMove: (event) => print("手指移动:$event"),
        onPointerUp: (event) => print("手指抬起:$event"),
      ),
    );
  }
}
class HomeContent extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("手势测试"),
      ),
      body: GestureDetector( //事件监听器
        child: Container( //作用范围target
          width: 200,
          height: 200,
          color: Colors.red,
        ),
        //各种事件响应
        onTap: () {
        },
        onTapDown: (detail) {
          print(detail.globalPosition);
          print(detail.localPosition);
        },
        onTapUp: (detail) {
          print(detail.globalPosition);
          print(detail.localPosition);
        }
      ),
    );
  }
}
```

# 基本原理
[Flutter 事件处理源码剖析_血色@残阳的专栏-CSDN博客](https://blog.csdn.net/yingshukun/article/details/107814213)
[Flutter完整开发实战详解(十三、全面深入触摸和滑动原理)](https://juejin.cn/post/6844903841742192648#heading-7)
[flutter事件分发原理详解_xiatiandefeiyu的博客-CSDN博客_flutter 事件分发](https://blog.csdn.net/xiatiandefeiyu/article/details/105449484)

## 原生事件向Flutter的传递
1. Flutter框架层在WidgetsFlutterBinding.ensureInitialized里会初始化GestureBinding，GestureBinding胶水类的initInstances()函数中会给window设置触摸事件的回调_handlePointerDataPacket，这里完成了原生事件信息到Flutter事件处理层的传递。
2. GestureBinding的 _handlePointerDataPacket 方法内将原生传来的原始手势数据PointerData转化为Dart对应的PointerEvent对象并加入到一个queue中进行存储，然后会依次从queue中出队PointerEvent对象并在 _handlePointerEvent里进行PointerEvent的处理。
![](Flutter%E4%BA%8B%E4%BB%B6%E5%A4%84%E7%90%86/847E8852-B8BA-4D4F-8BE3-0FA3B085ED59.png)
3. _handlePointerEvent对PointerEvent的处理就是先hitTest，根据你按下（PointerDownEvent类型的event）的坐标位置找出view树中哪些控件处在你点击的范围内，可以认定它为候选处理事件的人，找候选人的事在手指按下的时候找，手指在移动和抬起的时候都复用当前点击找到的事件，区别在于不同的手指有不同的索引值（在原生的那一层被赋值），具体如下：
	1. 对于按下：hitTest遍历得到参与响应的hitTestResult。
	2. 对于move：不再需要像down一样执行命中测试，而是直接使用down事件对应的hitTestResult，然后和down事件一样，分发处理。
	3. 对于up/cancel：同样得到down对应的hitTestResult，分发处理，但为删掉该hitTestResult，因为事件流程已结束。
``` dart
// GestureBinding中
  void initInstances() {
    super.initInstances();
    _instance = this;
    window.onPointerDataPacket = _handlePointerDataPacket;
  }
  void _handlePointerDataPacket(ui.PointerDataPacket packet) {
    // 将指针数据转换为逻辑像素，这样就可以以独立于设备的方式定义触摸斜率
   _pendingPointerEvents.addAll(PointerEventConverter.expand(packet.data, window.devicePixelRatio));
    if (!locked)
      _flushPointerEventQueue();
  }
  void _flushPointerEventQueue() {
    assert(!locked);
    while (_pendingPointerEvents.isNotEmpty)
     _handlePointerEvent(_pendingPointerEvents.removeFirst());
  }

  void _handlePointerEvent(PointerEvent event) {
    assert(!locked);
    HitTestResult hitTestResult;
    if (event is PointerDownEvent || event is PointerSignalEvent) {
      hitTestResult = HitTestResult();
      ///开始碰撞测试了，会添加各个控件，得到一个需要处理的控件成员列表
      hitTest(hitTestResult, event.position); //MARK1
      if (event is PointerDownEvent) {
        _hitTests[event.pointer] = hitTestResult;
      }
    } else if (event is PointerUpEvent || event is PointerCancelEvent) {
      ///复用机制，抬起和取消，不用hitTest，移除
      hitTestResult = _hitTests.remove(event.pointer);
    } else if (event.down) {
      ///复用机制，手指处于滑动中，不用hitTest
      hitTestResult = _hitTests[event.pointer];
    }
    if (hitTestResult != null ||
        event is PointerHoverEvent ||
        event is PointerAddedEvent ||
        event is PointerRemovedEvent) {
      ///开始分发事件
      dispatchEvent(event, hitTestResult);
    }
  }
```

### 重要理解总结：

* Flutter对原生event信息的接收入口回调是在run后的GestureBinding初始化时设置的。
* 原生event在转化为dart 的PointerEvent时会进行独立于设备的触摸斜率处理保证设备无关性。
* 触发一次对原生event的回调后就会把PointerEvent加入GestureBinding中的queue中，然后就不断从queue里拿出来处理，直到为空。
* PointerEvent处理里总先是PointerDownEvent，也就是手指按下的事件是真正的在hitTest拿响应对象，up和cancel、以及滑动过程中都是对Down结果的复用（想象一下实际的触摸效果）。

## hitTest命中测试 
数据结构、类关系

* HitTestResult：存储HitTestEntry集合，用于事件分发
* HitTestEntry：通过HitTestTarget初始化并存储
* HitTestTarget：包含handleEvent函数的处理对象，GestureBinding和RenderObject都实现了HitTestTarget接口，所以大部分时候是RenderObject。

* HitTestable：抽象协议，提供hitTest(HitTestResult result, Offset position)方法，参与hitTest的命中测试。
renderView里是hitTest(result, position: position);


这个过程整体类似iOS里事件的hitTest，通过hitTest递归碰撞，最终得到一个HitTestResult对象，内部包含用于分发和竞争事件的List<HitTestEntry>列表，每个HitTestEntry都由HitTestTarget初始化并存储到内部target属性上，GestureBinding和RenderObject都实现了HitTestTarget接口，所以这里存储的target大部分时候是RenderObject。

先理解上面GestureBinding里MARK1标记的hitTest是HitTestable抽象类的方法，GestureBinding和RendererBinding，但是因为它们都是mixins在WidgetsFlutterBinding这个入口类上使用，mixin的顺序是RendererBinding在后面，所以虽然这里是在GestureBinding内调用hitTest，而其实执行是先调用RendererBinding的hitTest方法（理解Dart的mixin机制，同swift的默认extension差不多）。
```dart
// RendererBinding#hitTest
void hitTest(HitTestResult result, Offset position) {
  renderView.hitTest(result, position: position);
  super.hitTest(result, position);//调用下面super，也就是会在添加完命中的控件后，最后添加GestureBinding自身到HitTestResult中
}

// GestureBinding#hitTest
void hitTest(HitTestResult result, Offset position) {
  result.add(HitTestEntry(this));
}
```

### hitTest整个过程如下（以点击事件为例）：
* 首先手指按下时，马上开始处理PointerDownEvent，首先进入RendererBinding内调用重写的hitTest，RendererBinding的hitTest先调用renderView的hitTest，renderView的hitTest内又调用child.hitTest，再`result.add(HitTestEntry(this))`把renderView自己加到HitTestResult中，再回到RendererBinding中调用super.hitTest（按照mixin规则其实super就是GestureBinding）把GestureBinding加都HitTestResult中。
* 而child.hitTest（实际中renderView的child是RenderBox）里的实现同iOS hitTest思想差不多，开始递归收集参与hitTest的子节点，逻辑如下：
	* 先拿传进来的position判断自己是否属于响应区域，不是默认返回false，不参与命中测试。
	* 是的话继续先递归hitTestChildren方法，再调用hitTestSelf（默认返回false），如果2者有一个命中返回true就添加自己到HitTestResult中。
* 所以这里hitTest最后就自下而上的得到了一个HitTestResult的相应控件列表，最底下的参与hitTest的各种Child控件在最上面，然后是renderView、GestureBinding。
```dart
//renderView内
bool hitTest(HitTestResult result, { required Offset position }) {
  if (child != null)
    child!.hitTest(BoxHitTestResult.wrap(result), position: position);
  result.add(HitTestEntry(this));
  return true;
}

// RenderBox内
bool hitTest(BoxHitTestResult result, { required Offset position }) {
  if (_size!.contains(position)) {
    if (hitTestChildren(result, position: position) || hitTestSelf(position)) {
      result.add(BoxHitTestEntry(this, position));
      return true;
    }
  }
  return false; //RenderBox默认不参与
}
// RenderBox的子类实现，一般我们写的widget对应的RenderObject是RenderBox的子类，比如RenderShiftedBox。
bool hitTestChildren(BoxHitTestResult result, { required Offset position }) {
  if (child != null) {
    final BoxParentData childParentData = child!.parentData as BoxParentData; //这里先计算offset再进行hitTest
    return result.addWithPaintOffset(
      offset: childParentData.offset,
      position: position,
      hitTest: (BoxHitTestResult result, Offset? transformed) {
        return child!.hitTest(result, position: transformed!);
      },
    );
  }
  return false; //子控件也默认不参与
}
```

### PointerEvent的点击位置转换处理
由于renderView的child，也就是平常写的根Widget对应的RenderObject是强制铺满屏幕的，所以是hitTest时直接传触摸事件的position，不需要考虑偏移，但是向下child做hitTest的时候（RenderBox 内hitTestChildren），会先结合Parent计算偏移（类似iOS hitTest中的触摸点测试前先父子坐标系转换），再用_size!.contains(position)来判断触摸事件点击点是否在自己的范围内。

### 重要理解总结：

hitTest得到的target顺序：最底下的参与hitTest的各种Child控件（RenderObject）在最上面，然后是renderView、GestureBinding。

hitTest的递归逻辑：就是同iOS大致一致，自上而下递归调用，先转换坐标再判断是否在自身响应范围，接着透传到child里。

从renderView的child（RenderBox）开始 hitTestSelf，默认返回false，表示如果你要自定义控件的话，为了有手势事件你需要让hitTestSelf返回true，一般情况下为子部件添加点击事件，当然我们添加点击事件都是为嵌套一层GestureDetector和Listener。（还不是很确定这里说的什么意思）。

renderView和它的child（RenderBox）这2次hitTest无需坐标转换，再深层次的child在hitTest前需要坐标转换再判断范围。

## DispatchEvent事件分发

```dart
// GestureBinding中
void dispatchEvent(PointerEvent event, HitTestResult? hitTestResult) {
  assert(!locked);
  // No hit test information implies that this is a pointer hover or
  // add/remove event. These events are specially routed here; other events
  // will be routed through the `handleEvent` below.
  if (hitTestResult == null) {
    try {
      pointerRouter.route(event);
    } catch (exception, stack) {
      //...
    }
    return;
  }
  for (final HitTestEntry entry in hitTestResult.path) {
    try {
      entry.target.handleEvent(event.transformed(entry.transform), entry);
    } catch (exception, stack) {
      //...
    }
  }
}
```

收集到HitTestResult后，会进入GestureBinding的dispatchEvent方法内分发，然后循环调用HitTestTarget的handleEvent方法处理事件，在dispatchEvent时会先判断是否有HitTestResult结果，没有命中测试信息，则通过 pointerRouter.route 将事件分发到全局处理，一般情况下是存在的，所以直接执行循环entry.target.handleEvent进行事件处理。

hitTest过程里拿到的HitTestEntry.target主要是RenderObject，RenderObject及其子类RenderBox都是空实现handleEvent方法，其中主要处理事件的RenderObject子类是RenderPointerListener。所以不是所有widget都能消耗事件，而是只有几个特定（比如一般我们要处理事件，会使用Listener、GestureDetector（内部也会使用Listener ）等Widget），GestureDetector中设置的回调通过在GestureDetector、GestureRecognizer、Listener、_PointerListener、RenderPointerListener中的传递最终在RenderPointerListener的handleEvent中被处理。

### 以GestureDetector为例子具体流程分析如下：
* 首先GestureDetector在build时根据外部设置的回调会创建大量的不同GestureRecognizer对象（点击、双击等）并把不同的回调处理传到GestureRecognizer上，最后把这些gestures包成一个RawGestureDetector对象返回。
* RawGestureDetector继承自StatefulWidget，在它state的build内会创建Listener返回，并在Listener构造的onPointerDown回调上设置了_handlePointerDown这个内部处理函数（很重要，是后续把gestures添加到竞技场的入口）。
* Listener的build中又会创建_PointerListener返回，并把回调交给_PointerListener。
* widget控件_PointerListener在createRenderObject这个时机创建了RenderPointerListener对象，并把传递进来的（外部设置的事件回调）保存起来提供给RenderPointerListener的handleEvent方法调用响应事件，如上完成了GestureDetector widget设置的回调到RenderPointerListener.handleEvent事件处理的闭环。
```dart
//RenderPointerListener的handleEvent函数：
void handleEvent(PointerEvent event, HitTestEntry entry) {
  if (onPointerDown != null && event is PointerDownEvent)
    return onPointerDown!(event);
  if (onPointerMove != null && event is PointerMoveEvent)
    return onPointerMove!(event);
  if (onPointerUp != null && event is PointerUpEvent)
    return onPointerUp!(event);
  if (onPointerCancel != null && event is PointerCancelEvent)
    return onPointerCancel!(event);
  if (onPointerSignal != null && event is PointerSignalEvent)
    return onPointerSignal!(event);
}
```

### 重要理解总结：
* 事件的消耗对象：RenderObject对象的子类RenderPointerListener。所以不是所有的widget都具备处理的能力（也就是说大部分widget不处理）。
* RenderPointerListener的创建时机：在_PointerListener（Listener 对应的Element对象）的createRenderObject时被创建。
* 多个Gesture竞争的入口：RawGestureDetector state的build内会创建Listener，并在Listener构造的onPointerDown回调上设置了_handlePointerDown这个内部处理函数，也就是按下时会把gestures添加到竞技场。
* 平常使用的GestureDetector相当于一个容器widget，内部会根据外部设置的回调情况，通过工厂方法创建大量GestureRecognizer来分别处理一些响应行为。
* GestureDetecotor的最终实现也是Listener，由Listener驱动处理各种GestureDetecotor创建的Gesture具体行为对象。
* GestureDetector widget设置的回调到RenderPointerListener.handleEvent进行事件处理经过了如下中间类GestureRecognizer、Listener、_PointerListener的层层传递。
* 只要通过了hitTest，控件就能收到分发来的事件，而且就是直接遍历，并没有事件冲突、拦截之类的处理（比如一个parent内有一个child，这两个widget都是Listenr的子widget，parent的面积比child大；手指触摸child，parent和child对应的Listenr都会收到事件，这也证明了没有事件冲突处理的事实）。

## GestureDetecotor的事件冲突处理
### Flutter原生的事件拦截
前面分析的结论只要通过了hitTest，控件就能收到分发来的事件，如果在hitTest中返回false，当前控件就不会收到分发的事件，如下两个Widget就是通过重写控制其对应的RenderObject的hitTest函数来控制是否接收事件：
* IgnorePointer 忽略事件，包括它自己。
* AbsorbPointer 拦截事件，不会传递给子节点，但自己能收到。

以上的方式，只是直接的控制不接收事件，不适合动态的判断谁来接收事件。比如parent和child都能接收事件，但点击child的时候，只应该响应child的点击事件，parent同理；这种情况是不适合用IgnorePointer之类的方式来处理的。那么在Flutter如何像Android中那样，可以动态处理事件冲突呢？这就要使用到GestureDetecotor，GestureDetecotor可以识别多种手势，并且能处理事件冲突。下面我们来分析GestureDetecotor是如何做到可以处理事件冲突的。

### 重要概念和数据结构
Flutter 在设计事件竞争的时候，定义了一个很有趣的概念：**通过一个竞技场，各个控件参与竞争，直接胜利的或者活到最后的第一位，你就获胜得到了胜利。** 那么为了分析接下来的“战争”，我们需要先看几个概念：
* **GestureRecognizer** ：手势识别器基类，基本上 RenderPointerListener 中需要处理的手势事件，都会分发到它对应的 GestureRecognizer，并经过它处理和竞技后再分发出去，常见有 ：OneSequenceGestureRecognizer 、 MultiTapGestureRecognizer 、VerticalDragGestureRecognizer、TapGestureRecognizer 等等。
* **GestureArenaManagerr** ：手势竞技管理，它管理了整个“战争”的过程，原则上竞技胜出的条件是 ：**第一个竞技获胜的成员或最后一个不被拒绝的成员。**
* **GestureArenaEntry** ：提供手势事件竞技信息的实体，内封装参与事件竞技的成员。
* **GestureArenaMember**：参与竞技的成员抽象对象，内部有 acceptGesture 和 rejectGesture 方法，它代表手势竞技的成员，默认 GestureRecognizer 都实现了它，**所有竞技的成员可以理解为就是****GestureRecognizer****之间的竞争。**
* **_GestureArena**：GestureArenaManager 内的竞技场，内部持参与竞技的 members 列表，官方对这个竞技场的解释是： **如果一个手势试图在竞技场开放时(isOpen=true)获胜，它将成为一个带有“渴望获胜”的属性的对象。当竞技场关闭(isOpen=false)时，竞技场将寻找一个“渴望获胜”的对象成为新的参与者，如果这时候刚好只有一个，那这一个参与者将成为这次竞技场胜利的青睐存在。**

GestureArenaManager中存在Map<int, _GestureArena>类型的属性_arenas，表示每个手指对应一个手势竞技场_GestureArena，每个
_GestureArena内包含多个GestureRecognizer 。member参数就是当前示例的TapGestureRecognizer，这里就会把当前TapGestureRecognizer添加到该手指对应的_GestureArena中。

Gesture继承关系
![](Flutter%E4%BA%8B%E4%BB%B6%E5%A4%84%E7%90%86/F6F9DE9C-5485-4A21-9CEF-9831149A84C9.png)

### Down时机时注册竞争的Gesture并设置primaryPointer
首先回到上面事件分发时循环执行entry.target.handleEvent的过程中，会先触发最底层具备相应能力的控件对应RenderPointerListener对象的handleEvent函数（如果实际场景中只使用了一个GestureDetecotor对象， 那么HitTestResult过程内相当于只有一个GestureDetecotor作为child参与事件处理），而事件流中第一个事件一般都会是PointerDownEvent。PointerDownEvent的流程在事件竞争中相当关键，因为它会触发GestureRecognizer.addPointer。GestureRecognizer 只有通过 addPointer 方法将每个到来的PointerDownEvent事件和自己绑定，并添加到  GestureBinding的 PointerRouter 事件路由和 GestureArenaManager 事件竞技管理中心去，后续的事件这个 GestureRecognizer 才能响应和参与竞争，具体过程为：
* 取出一个entry.target执行handleEvent（GestureDetecotor）
* RenderPointerListener接受处理onPointerDown，最终会响应到RawGestureDetector的state中Listener的onPointerDown回调上设置的 _handlePointerDown方法内。
* _handlePointerDown内循环遍历包起来的gestures拿到准备参与竞技的每一个GestureRecognizer，调用recognizer.addPointer在触摸开始时把GestureRecognizer加入到GestureBinding持有的GestureArenaManager竞技场中参与竞争，recognizer.addPointer过程有几个重要点：
	* 初始状态也就是收到down的时候，初始化相关属性，并且修改gesture状态为possible，设置primaryPointer = event.pointer（重要）。
	* startTrackingPointer内注册路由，就是记录当前手指ID并把对应gesture的handleEvent函数，存储到GestureBinding的PointerRouter的_routeMap表中存起来，后续gesture就能收到回调响应。

Flutter把手势进行了抽象，集中在GestureDetecotor中并处了手势冲突，和iOS还是有区别的。

### 路由阶段的处理：
```dart
//GestureBinding
void handleEvent(PointerEvent event, HitTestEntry entry) {
  // 1路由阶段
  pointerRouter.route(event);
  // 2竞技场阶段
  if (event is PointerDownEvent) {
    gestureArena.close(event.pointer);
  } else if (event is PointerUpEvent) {
   gestureArena.sweep(event.pointer);
  } else if (event is PointerSignalEvent) {
    pointerSignalResolver.resolve(event);
  }
}
```
HitTestResult的list中最后一个HitTestTarget是GestureBinding自己，在前面把所有的参与竞技的GestureDetecotor中的GestureRecognizer加入竞技场后，会回到GestureBinding的handleEvent内处理，首先就是1路由阶段，pointerRouter.route(event)会执行前面在
startTrackingPointer中向GestureBinding的pointerRouter中注册的所有
GestureRecognizer中的handleEvent函数：
* 对于手指down时机，直接来到handlePrimaryPointer(event)，在BaseTapGestureRecognizer中不会做处理。
* 对于手指Up时机，直接来到handlePrimaryPointer(event)，到BaseTapGestureRecognizer层触发_checkUp，_checkUp内又会触发handleTapUp（准确说依赖acceptGesture后的_wonArenaForPrimaryPointer标记，后面会讲到），TapGestureRecognizer内handleTapUp触发onTap（类似iOS里的TouchUpInSide）完成点击响应（想象一下实际中就是点按后，需要最后松手才触发onTap）。
```dart
// PrimaryPointerGestureRecognizer
void handleEvent(PointerEvent event) {
  if (state == GestureRecognizerState.possible && event.pointer == primaryPointer) {
    //...
    if (event is PointerMoveEvent && (isPreAcceptSlopPastTolerance || isPostAcceptSlopPastTolerance)) {
      // 如果是滑动时机event，且滑动距离超过阈值，那么对于点击来说，视为取消，
      resolve(GestureDisposition.rejected);
      stopTrackingPointer(primaryPointer!); //MARK2
    } else {
      // 对于down、up、cancle时机event将执行这里
      handlePrimaryPointer(event);
    }
  }
  stopTrackingIfPointerNoLongerDown(event); //如果是up或者cancle，就会取消这根手指注册的handleEvent函数
}
// BaseTapGestureRecognizer
void handlePrimaryPointer(PointerEvent event) {
  if (event is PointerUpEvent) {
    _up = event;
    _checkUp();
  } else if (event is PointerCancelEvent) {
    resolve(GestureDisposition.rejected);
    if (_sentTapDown) {
      _checkCancel(event, '');
    }
    _reset();
  } else if (event.buttons != _down!.buttons) {
    resolve(GestureDisposition.rejected);
    stopTrackingPointer(primaryPointer!);
  }
}
@override
void acceptGesture(int pointer) {
  super.acceptGesture(pointer);
  if (pointer == primaryPointer) {
    _checkDown();
    _wonArenaForPrimaryPointer = true;
    _checkUp();
  }
}
@override
void rejectGesture(int pointer) {
  super.rejectGesture(pointer);
  if (pointer == primaryPointer) {
    // Another gesture won the arena.
    assert(state != GestureRecognizerState.possible);
    if (_sentTapDown)
      _checkCancel(null, 'forced');
    _reset();
  }
}
void _checkDown() {
  if (_sentTapDown) {
    return;
  }
  handleTapDown(down: _down!);
  _sentTapDown = true;
}
void _checkUp() {
  if (!_wonArenaForPrimaryPointer || _up == null) {
    return;
  }
  handleTapUp(down: _down!, up: _up!);
  _reset();
}

// TapGestureRecognizer
void handleTapUp({ required PointerDownEvent down, required PointerUpEvent up}) {
  //...
  switch (down.buttons) {
    case kPrimaryButton:
      //...
      if (onTap != null)
        invokeCallback<void>('onTap', onTap!);
      break;
    //...
    default:
  }
}
```

### 竞技场阶段的处理
然后执行2：根据PointerEvent手指触摸时机进行了不同的如下处理：
	* PointerDownEvent：gestureArena.close 关闭竞技场。
	* PointerUpEvent：gestureArena.sweep 强行得到结果。
	* PointerSignalEvent：pointerSignalResolver.resolve(event);
* 首先PointerDownEvent手指按下时，gestureArena.close 会关闭本次的竞技场，内部逻辑就是先`_GestureArena state = _arenas[pointer]`从_arenas表中拿到本次event的竞技场（包含addPointer时添加的所有参与竞技的成员封装信息），接着`state.isOpen = false`表示不再接收新的竞技成员，最后调用_tryToResolveArena开始直接尝试得到胜利者，逻辑如下：
	1. if `state.members.length == 1`说明只有一个成员不存在竞争，直接胜利_resolveByDefault响应本次触摸的所有后续所有事件（Down后的移动、抬起等事件）。
	2. else if `(state.members.isEmpty)`则_arenas.remove(pointer)跳过处理。
	3. else if `(state.eagerWinner != null)`如果竞技场内有eagerWinner对象，则_resolveInFavorOf(pointer, state, state.eagerWinner)让它胜出，其他成员全部拒绝，逻辑如下：
		1. eagerWinner进行acceptGesture。
		2. state.members的所有非eagerWinner进行rejectGesture。
	4. 如果前面判断都没过（多个手势且没有eagerWinner）则不处理，等待后续处理。

* PointerUpEvent手指抬起时，gestureArena.sweep函数内先移除本次的竞技场，之后选取state.members.first作为胜出者（也就是hitTest结果里控件树最里面的child），胜利的member 会通过members.first.acceptGesture(pointer) 最终回调到 TapGestureRecognizer.acceptGesture中处理。换句话说当手指抬起前Down时机如果还没能决策出到底谁该处理事件的分发的话（说明有多个Gesture），则调用sweep强行得到结果，将会默认采用最底层的叶子节点控件作为事件处理者，也就是说最内层的那个控件将消耗事件。也就是说你如果你用多个GestureRecognizer的话，最终事件会被最内层的GestureRecognizer消耗，这个和iOS单个控件消耗事件差不多了，因为Flutter自带滚动控件都是类似实现了GestureRecognizer，所以嵌套滚动总是先滚动内层，先被内层消耗。

### 竞技失败
在竞技场竞争失败的成员会被移出竞技场，移除后就没办法参加后面事件的竞技了 ，比如 TapGestureRecognizer 在接受到 PointerMoveEvent 事件时就会被直接 rejected , 并触发 rejectGesture ，之后定时器会被关闭，并且触发 onTapCancel ，然后重置标志位。
选出胜利者后，其他的都会竞技失败，触发 rejectGesture。不参与后面事件的竞技。

### TagGesture的_checkDown、_checkUp发出时机分析：

TapGesture中发出_checkUp并响应成功onTapUp，分别是在2个时机并依赖于2个东西（_up不为空和_wonArenaForPrimaryPointer为true）：
	1. acceptGesture时置_wonArenaForPrimaryPointer为true，尝试发出。
	2. Up时机的路由处理时（handlePrimaryPointer），把当前event保存在_up中，尝试发出。
TagGesture中发出_checkDown并响应onTapDown，主要是有2个时机：
	1. didExceedDeadline时机，前面 Down时机addPointer时TapGestureRecognizer会创建一个定时器，这个定时器的时间时 kPressTimeout = 100毫秒 ，如果我们点后长按住的话，就会等待到触发didExceedDeadline去自动执行_checkDown并响应onTapDown。
	2. acceptGesture调用时直接执行_checkDown并响应onTapDown。

**普通按下**
正常点击完后马上抬起手指（间隔小于100毫秒）
1. 单个TagGesture：
	1. Down时机的 route 过程内不会做处理，而后在竞技场处理 close 时因为只有一个Gesture则直接获胜，acceptGesture时执行_checkDown、_checkUp，但因为_up=nil，所以不会真正的响应onTapUp，只会响应onTapDown。
	2. 手指Up时机的路由处理时，发出 _checkUp真正响应onTapUp，而后stopTrackingIfPointerNoLongerDown(event)判断通过取消手指的注册。
2. 有多个 TapGestureRecognizer ：
	1. Down 时机的route 过程内每个Gesture都不会做处理，而后在竞技场 close 不会竞出胜利者。
	2. 手指Up时机的路由处理时，最外层最后的Gesture先处理发出 _checkUp但不会真正的响应onTapUp，而后stopTrackingIfPointerNoLongerDown(event)判断通过取消手指的注册（意味着外层后面的Gesture路由阶段不会再调用自己的handleEvent进行事件处理），之后经过竞技场处理 sweep 选取排在第一个位置的为胜利者（最外层最后的Gesture），调用 acceptGesture，只执行胜利者的 _checkDown 和 _checkUp，依次响应onTapDown和onTapUp。
	
**长按之后抬起：**
1. 单个TagGesture：和普通按下的区别在于，是在didExceedDeadline时马上发出_checkDown并响应onTapDown。
_checkDown外其他和普通按下基本没区别。
2. 多个 TapGestureRecognizer ：和普通按下的区别在于在didExceedDeadline发出的时机在Up事件到来前，所以所有的Gesture都会收到_checkDown响应onTapDown。

### 重要总结
GestureDetecotor冲突的解决原理：手指Up时机的路由处理时，最外层最后的Gesture先处理，而后stopTrackingIfPointerNoLongerDown(event)判断通过，取消手指的注册，意味着外层后面的Gesture路由阶段不会再调用自己的handleEvent进行事件处理，也就解决了冲突。

事实上Down事件在 Flutter 中一般都是用来做添加判断的，如果存在竞争时，大部分时候是不会直接出结果的，而 Move事件在不同 GestureRecognizer 中会表现不同，而 UP事件手指抬起之后，一般会强制得到一个结果。

事件处理的流程层次是先根据event时机为维度，先对gesture路由再竞技处理（过程里针对event会做不同处理）。

## 滑动事件DragGestureRecognizer执行分析
[十三、全面深入触摸和滑动原理 · GitBook](https://guoshuyu.cn/home/wx/Flutter-13.html)

滑动事件也同样是先在 Down 流程中addPointer。然后 Move 流程中，通过在PointerRouter.route之后执行DragGestureRecognizer.handleEvent进行处理。在DragGestureRecognizer.handleEvent里，会调用_hasSufficientPendingDragDeltaToAccept判断是否符合条件，即`pendingDragOffset.dy.abs() > kTouchSlop`，符合条件就直接执行
resolve(GestureDisposition.accepted)，将流程回到竞技场里。
然后执行acceptGesture，触发onStart和onUpdate。

所以如果是一个上下滑动的可点击列表，一开始没有产生 MOVE 事件，所以 DragGestureRecognizer 没有被接受，而Item 作为 Child 第一位，所以响应点击。如果有 MOVE 事件， DragGestureRecognizer 会被 acceptGesture，而点击 GestureRecognizer会被移除事件竞争，也就没有后续 UP 事件了。

### onUpdate如何滚动？
onUpdate最后会调用到Scrollable 的_handleDragUpdate，这时候会执行
Drag.update。

ListView 的Drag 实现是ScrollDragController，它在Scrollable中是和
ScrollPositionWithSingleContext关联在一起的。而继承关系`ScrollPositionWithSingleContext : ScrollPosition : ViewportOffset : ChangeNotifier`

执行 Drag.update后 ，最终会调用到 ScrollPositionWithSingleContext 的 applyUserOffset，导致内部确定位置的 pixels 发生改变，并执行父类 ChangeNotifier 的方法notifyListeners 通知更新。

而在 ListView 内部 RenderViewportBase 中，这个 ViewportOffset 是通过 _offset.addListener(markNeedsLayout); 绑定的，so ，触摸滑动导致Drag.update，最终会执行到RenderViewportBase中的markNeedsLayout触发页面更新。

# 如何解决冲突
[深入进阶-如何解决Flutter上的滑动冲突？](https://juejin.cn/post/6900751363173515278)

# 跨组件通信
1. EventBus：类似iOS里的通知，是一种订阅者模式的设计思想，通过一个全局的对象来管理
```dart
final eventBus = EventBus(); //全局创建
//发出
final info = UserInfo("why", 18);
eventBus.fire(info);
//响应
eventBus.on<UserInfo>().listen((data) {
  setState(() { //更新数据
    message = "${data.nickname}-${data.level}";
  });
});
```