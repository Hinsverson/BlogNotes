# Flutter启动到显示分析
#flutter

https://github.com/Messiahfy/Notes/blob/master/Flutter/Flutter应用启动流程.md#scheduleattachrootwidget
http://gityuan.com/2019/06/29/flutter_run_app/
[14.4：Flutter从启动到显示 · 《Flutter实战》](https://book.flutterchina.club/chapter14/flutter_app_startup.html)

# 启动流程
[Flutter Internals 时序图](https://juejin.cn/post/6898228981372616717) - 流程图很丰富

![](Flutter%E5%90%AF%E5%8A%A8%E5%88%B0%E6%98%BE%E7%A4%BA%E5%88%86%E6%9E%90/E33B52CC-44A4-47EC-8E01-2E13D4BEDDC4.png)
```dart

void runApp(Widget app) {
  WidgetsFlutterBinding.ensureInitialized()
    ..attachRootWidget(app)
    ..scheduleWarmUpFrame();
}
```

Flutter应用层的启动入口从main开始`void main() => runApp(MyApp());`，之后分为了三步走，依次是：
1. 初始化WidgetsFlutterBinding胶水类，同时通过mixin形式把各个子模块对系统时机的绑定混入到这个胶水类中（Flutter engine层向ui层提供的系统时机主要暴露在windows上）各个子模块initInstances准备时主要有以下几个关键点：
	1. GestureBinding：设置了对window.onPointerDataPacket的回调，是Framework事件模型与底层事件的绑定入口。
	2. SchedulerBinding：设置了对window.onBeginFrame和window.onDrawFrame的回调从而监听刷新事件，充当绘制相关操作的调度者角色，其中有transientCallbacks、persistentCallbacks、postFrameCallbacks三个回调队列， 在一次刷新过程中，这三个回调队列内的回调会放在不同的渲染时机下被执行。
	3. RendererBinding：设置了window.onMetricsChanged、window.onTextScaleFactorChanged屏幕尺寸、密度等的变化回调，同时作为渲染树与Flutter engine的桥梁，会创建renderView向SchedulerBinding的persistentCallbacks队列中添加处理布局与绘制相关的持久回调，用以完成渲染流程（非常重要）
	4. WidgetsBinding：设置了window.onLocaleChanged、onBuildScheduled等变化回调，同时作为Flutter widget层与engine的桥梁，创建BuildOwner设置BuildOwner.onBuildScheduled的回调
2. 把根Widget(通过RenderObjectToWidgetAdapter桥接类)和根RenderObject通过内部创建（首次创建，过后复用）的根RenderObjectToWidgetElement关联起来，并且设置了在BuildOwner.buildScope时机触发根element的mount挂载操作，意味着每次buildScope（非首次渲染会调用）时机会触发从根到子Widget的rebuild、Element、RenderObject这三颗树的构造映射。
3. 调度执行各个渲染队列中加入的渲染操作回调：
	1. 先handleBeginFrame执行transientCallbacks队列中的临时回调（做动画相关的事）。
	2. 再handleDrawFrame执行persistentCallbacks持续性队列（做布局渲染相关的事，在renderBinding准备时已经加入）和postFrameCallbacks队列中的回调。在非首次渲染时还会先调用BuildOwner.buildScope先引发关根element进行mount挂载操作更新pipelineOwner的_nodesNeedingLayout列表。
	3. 布局渲染回调内先pipelineOwner.flushLayout()正向遍历需要layout的renderObject节点并做performLayout引发从根到子的RenderObject确定位置和尺寸，并把需要绘制的节点更新到pipelineOwner的_nodesNeedingPaint中。
	4. pipelineOwner.flushPaint()反向遍历对需要Paint的renderObject节点做paint引发绘制（类比iOS的drawRect时机），绘制命令将存储在layer中，后面发给底层处理。

	* 创建BuildOwner，管理Widget，包括Element，它跟踪哪些widget需要重新构建。
	* 设置BuildOwner.onBuildScheduled的回调，也就是Element标记为dirty时，每次构建过程的回调，也就是渲染流程。
	* 设置了window.onLocaleChanged、onBuildScheduled等回调。
	* 设置处理路由相关函数。

### 重要时机总结
* Animate: 遍历执行_transientCallbacks中的回调函数，动画相关的Ticker会往这里面添加回调函数。
* Build: buildScope函数引发Widget、Element、RenderObject三棵树的构造。
* Layout: flushLayout引发RenderObject确定位置和尺寸。
* Paint: flushPaint引发绘制流程，绘制命令将存储在layer中，后面会发给底层处理。

# 初始化WidgetsFlutterBinding胶水类
![](Flutter%E5%90%AF%E5%8A%A8%E5%88%B0%E6%98%BE%E7%A4%BA%E5%88%86%E6%9E%90/C5736737-66D7-4AA6-A896-916A70CE5DB3.png)
```dart
//WidgetsFlutterBinding
class WidgetsFlutterBinding extends BindingBase with GestureBinding, ServicesBinding, SchedulerBinding, PaintingBinding, SemanticsBinding, RendererBinding, WidgetsBinding {
  static WidgetsBinding ensureInitialized() {
    if (WidgetsBinding.instance == null)
      WidgetsFlutterBinding(); //初始化方法
    return WidgetsBinding.instance; //单利
  }
}
//BindingBase
BindingBase() {
  //...
  initInstances(); //何种binding的初始化准备
  // 注册各种扩展用于调试，不具体分析
  initServiceExtensions();
  //...
}
```
首先ensureInitialized()初始化WidgetsFlutterBinding这个绑定widget 框架和Flutter engine的胶水类，执行父类BindingBase 的构造方法时进行各种mixin进来的binding类的initInstances和initServiceExtensions操作，最终返回创建的WidgetsBinding单例并返回。

根据mixin的多态表现形式，后面的会覆盖前面的同名方法，所以先执行WidgetsBinding中的initInstances方法，但因为每个Binding类的initInstances方法中又会先执行super. initInstances，所以最终的执行顺序又变成从头（GestureBinding）开始执行。

Window是Flutter Framework连接宿主操作系统的接口，内部包含了当前设备和系统的一些信息以及Flutter Engine给出来的一些接口回调，WidgetsFlutterBinding混入的各种Binding基本都是在监听并处理Window对象的一些事件，然后将这些事件按照Framework的模型包装、抽象然后分发，各个Binding的initInstances内具体做了以下事情：
* GestureBinding：设置了window.onPointerDataPacket回调，是Framework事件模型与底层事件的绑定入口。
* SchedulerBinding：设置了window.onBeginFrame和window.onDrawFrame回调从而监听刷新事件，充当绘制相关的调度者角色，在SchedulerBinding类中有三个FrameCallback回调队列， 在一次刷新过程中，这三个回调队列会放在不同时机被执行：
	* transientCallbacks：用于存放一些临时回调，一般存放动画回调，可以通过SchedulerBinding.instance.scheduleFrameCallback添加回调。
	* persistentCallbacks：用于存放一些持久的回调，持久回调一经注册则不能移除，一般是布局和渲染回调，可通过SchedulerBinding.instance.addPersitentFrameCallback添加回调，注意不能在此类回调中再请求新的绘制帧。
	* postFrameCallbacks：在一个Frame结束时只会被调用一次，调用后会被系统移除，可通过SchedulerBinding.instance.addPostFrameCallback添加回调，注意不要在此类回调中再触发新的Frame，这可以会导致循环刷新。
* ServicesBinding ：设置了window.onPlatformMessage 回调， 用于绑定平台消息通道（message channel），主要处理原生和Flutter通信。
* PaintingBinding：绑定绘制库，主要用于处理图片缓存。
* SemanticsBinding：语义化层与Flutter engine的桥梁，主要是辅助功能的底层支持。
* RendererBinding: 它是渲染树与Flutter engine的桥梁，详细如下：
	* 创建PipelineOwner，管理渲染流水线的整个过程。
	* 设置了window.onMetricsChanged、window.onTextScaleFactorChanged屏幕尺寸、密度等的变化回调。
	* 内部initRenderView方法创建并初始化renderView，而renderView重写了setter，会把renderView设置为_pipelineOwner的rootNode，且renderView.attach指定owner为_pipelineOwner完成根渲染节点和_pipelineOwner的关联。
	* 	向SchedulerBinding的persistentCallbacks队列中添加处理布局与绘制相关的持久回调，用以完成渲染流程（非常重要）。
* WidgetsBinding：它是Flutter widget层与engine的桥梁，详细如下：
	* 创建BuildOwner，管理Widget，包括Element，它跟踪哪些widget需要重新构建。
	* 设置BuildOwner.onBuildScheduled的回调，也就是Element标记为dirty时，每次构建过程的回调，也就是渲染流程。
	* 设置了window.onLocaleChanged、onBuildScheduled等回调。
	* 设置处理路由相关函数。

# 关联根Widget和RenderView并构造3颗树
![](Flutter%E5%90%AF%E5%8A%A8%E5%88%B0%E6%98%BE%E7%A4%BA%E5%88%86%E6%9E%90/BE35514D-256F-49D5-8418-FCAC34593A95.png)
```dart
//WidgetsBinding
void scheduleAttachRootWidget(Widget rootWidget) {
  Timer.run(() {
    attachRootWidget(rootWidget);
  });
}
void attachRootWidget(Widget rootWidget) {
  _readyToProduceFrames = true;
  _renderViewElement = RenderObjectToWidgetAdapter<RenderBox>(
    container: renderView,
    debugShortDescription: '[root]',
    child: rootWidget,
  ).attachToRenderTree(buildOwner, renderViewElement as RenderObjectToWidgetElement<RenderBox>);
}

//RenderObjectToWidgetAdapter
RenderObjectToWidgetElement<T> createElement() => RenderObjectToWidgetElement<T>(this);

@override
RenderObjectWithChildMixin<T> createRenderObject(BuildContext context) => container;

RenderObjectToWidgetElement<T> attachToRenderTree(BuildOwner owner, [RenderObjectToWidgetElement<T> element]) {
  if (element == null) {
    owner.lockState(() {
      element = createElement(); // 创建Element
      assert(element != null);
      element.assignOwner(owner); // 指定BuildOwner
    });
    owner.buildScope(element, () {
      element.mount(null, null); //设置回调，执行Element的挂载操作并rebuild所有标记为dirty的element
    });
  } else {
    element._newWidget = this;
    element.markNeedsBuild();
  }
  return element;
}
```

RenderObjectToWidgetAdapter桥接对象的child才是外部runApp里传入的widget，这个桥接widget的：
	* createElement方法创建RenderObjectToWidgetElement对象。
	* createRenderObject方法直接返回的是创建时传入的RendererBinding里创建的根renderView。

scheduleAttachRootWidget在下一个消息循环中去执行
attachRootWidget添加rootWidget的操作，该操作主要是通过桥接类RenderObjectToWidgetAdapter的attachToRenderTree方法把外部的根Widget和RendererBinding中创建的继承自RenderObject的绘制树的根RenderView通过RenderObjectToWidgetElement关联起来，并返回，重点注意下面几个细节：
	* 传入的BuildOwner就是WidgetsBinding在initInstances时创建的Widget管理者，可以追拿到buildScope时机（后面非首次渲染会触发）。
	* 这个element 只会创建一次，后面会进行复用，创建Element时会将RenderObjectToWidgetAdapter保存在父类Element的widget成员变量中，如果element 已经创建过了，attachToRenderTree方法则将根element 中关联的widget 设为新的。
	* 设置了在BuildOwner.buildScope时机下进行element.mount(null, null)挂载操作，意味着会引发rootWidget的widget树递归构造Element树和RenderObject树。第一次调用应该还没有标记为dirty的element。

scheduleAttachRootWidget流程的关键，就是把根Widget(通过RenderObjectToWidgetAdapter)和RenderView以及内部创建的Element一一关联起来，并且把我们的rootWidget加入到了根Widget树中，一起构造了Widget、Element、RenderObject三颗树。

# scheduleWarmUpFrame调度执行渲染操作
```dart
//在SchedulerBinding中有实现
void scheduleWarmUpFrame() {
  ...
  Timer.run(() {
    handleBeginFrame(null); 
  });
  Timer.run(() {
    handleDrawFrame();  
    resetEpoch();
  });
  // 锁定事件
  lockEvents(() async {
    await endOfFrame;
    Timeline.finishSync();
  });
 ...
}
```
把根Widget(通过RenderObjectToWidgetAdapter)和RenderView关联起来构建完3颗树后，开始渲染操作，在此次绘制结束前，该方法会锁定事件分发，也就是说在本次绘制结束完成之前Flutter将不会响应各种事件，这可以保证在绘制过程中不会再触发新的重绘。

从这个方法注释可知，调用这个方法会主动构建视图数据。这样做的好处是因为 Flutter 依赖 Dart 的 MicroTask 来进行帧数据构建任务的 schedule，这里通过主动调用进行整个周期的 “热身”，这样最近的下次 VSync 信号同步时就有视图数据可提供，而不用等到 MicroTask 的 next Tick。也就是说scheduleWarmUpFrame被调用后会立即进行一次绘制准备数据，而不是等待”vsync” 信号。

先handleBeginFrame处理transientCallbacks队列中的回调（比如动画）
再handleDrawFrame处理persistentCallbacks和postFrameCallbacks队列中的回调，前面RendererBinding在initInstances时已经向persistentCallbacks队列内添加了绘制回调，这里就会执行，回调内主要是调用drawFrame。
```dart
//WidgetsBinding
@override
void drawFrame() {
 ...//省略无关代码
  try {
    if (renderViewElement != null)
      buildOwner.buildScope(renderViewElement); 
    super.drawFrame(); //调用RendererBinding的drawFrame()方法
    buildOwner.finalizeTree();
  } 
}
//RendererBinding
void drawFrame() {
  assert(renderView != null);
  pipelineOwner.flushLayout(); //布局
  pipelineOwner.flushCompositingBits(); //重绘之前的预处理操作，检查RenderObject是否需要重绘
  pipelineOwner.flushPaint(); // 重绘
  renderView.compositeFrame(); // 将需要绘制的比特数据发给GPU
  pipelineOwner.flushSemantics(); // this also sends the semantics to the OS.
}
//pipelineOwner.flushLayout()
void flushLayout() {
   ...
    while (_nodesNeedingLayout.isNotEmpty) {
      final List<RenderObject> dirtyNodes = _nodesNeedingLayout;
      _nodesNeedingLayout = <RenderObject>[];
      for (RenderObject node in 
           dirtyNodes..sort((RenderObject a, RenderObject b) => a.depth - b.depth)) {
        if (node._needsLayout && node.owner == this)
          node._layoutWithoutResize();
      }
    }
  } 
}
//RenderObject
void _layoutWithoutResize() {
  //...
  try {
    performLayout();
    markNeedsSemanticsUpdate();
  } catch (e, stack) {
    //...
  }
  //...
  _needsLayout = false;
  markNeedsPaint();
}
```
drawFrame主要做了以下几个事情，注意由于RendererBinding只是一个mixin，在调用RendererBinding.drawFrame()方法前会先调用WidgetsBinding中的实现，所以会调用buildOwner.buildScope() （非首次绘制时）被标记为“dirty” 的 element 进行rebuild()（前面讲到过）：
1. pipelineOwner.flushLayout()：正向遍历（根到子）_nodesNeedingLayout列表里需要布局的RenderObject，`node._needsLayout && node.owner == this`通过的话就调用node._layoutWithoutResize()执行布局流程，在RendererBinding的initInstances方法中，会调用initRenderView，其中会把RenderView添加到_nodesNeedingLayout列表中，所以初始应该只有
RenderView在列表中，_layoutWithoutResize主要做了：
	* performLayout操作进行重新布局，performLayout()由RenderObject自行重写实现，其中会设置自己的大小，如果有子节点，那么还需要调用子节点的layout方法，引发整个树的布局。
	* markNeedsPaint()将当前RenderObject添加到PipelineOwner的_nodesNeedingPaint列表中，待绘制时使用。
2. pipelineOwner.flushCompositingBits()：检查RenderObject是否需要重绘，然后更新RenderObject.needsCompositing属性，如果该属性值被标记为
True则需要重绘。
3. pipelineOwner.flushPaint()：反向遍历（子到根）_nodesNeedingPaint列表里需要重绘的RenderObject，`node._needsPaint && node.owner == this`通过的话就调用PaintingContext.repaintCompositedChild(node)，该方法最终会调用到RenderObject的_paintWithContext函数，该函数核心就是调用了paint方法，RenderObject子类将重写paint方法，在其中完成实际绘制工作（PaintingContext中可以获得canvas，可以类比UIView的drawRect）
4. renderView.compositeFrame()：这个方法中有一个Scene对象，Scene对象是一个数据结构，保存最终渲染后的像素信息。这个方法将Canvas画好的Scene传给window.render()方法，该方法会直接将scene信息发送给Flutter engine，最终由engine将图像画在设备屏幕上。
5. pipelineOwner.flushSemantics()：语义化处理（不具体分析）。
