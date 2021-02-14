# Flutter - PaintingContext和Layer
#flutter/核心类分析

[深入浅出 Flutter Framework 之 PaintingContext](https://juejin.cn/post/6844904175344549901)

![](Flutter%20-%20PaintingContext%E5%92%8CLayer/A2E82249-CB3B-4F89-B83C-AD748795EDB2.png)
* PaintingContext：绘制上下文，简单的理解就是为绘制操作 (Paint) 提供了场所或者说环境 (上下文)，其主要职责包括：
	* 在绘制流程中按需引入新的 Layer(主要依据 Repaint Boundary、need compositing)。
	* 管理Layer Tree，每个 PaintingContext 实例都会生成一棵 Layer Sub Tree。
	* 管理 Canvas，对底层细节进行抽象、封装。
* PaintingContext继承自ClipContext，ClipContext是抽象类，主要提供了几个与裁剪 (Clip) 有关的辅助方法。
* PictureLayer _currentLayer、ui.PictureRecorder _recorder以及Canvas _canvas用于具体的绘制操作。
* `ContainerLayer _containerLayer`，Layer Subtree的根节点，由PaintingContext构造函数传入，一般传入的是RenderObject._layer。

### Canvas原生基本绘制操作
通过直接操作 Canvas，可以在屏幕上画了一个⭕️。
```dart
void main() {
  PictureRecorder recorder = PictureRecorder();
  // 初始化 Canvas 时，传入 PictureRecorder 实例
  // 用于记录发生在该 canvas 上的所有操作
  Canvas canvas = Canvas(recorder);

  Paint circlePaint= Paint();
  circlePaint.color = Colors.blueAccent;

  // 调用 Canvas 的绘制接口，画一个圆形
  canvas.drawCircle(Offset(400, 400), 300, circlePaint);

  // 绘制结束，生成Picture
  Picture picture = recorder.endRecording();

  SceneBuilder sceneBuilder = SceneBuilder();
  sceneBuilder.pushOffset(0, 0);
  // 将 picture 送入 SceneBuilder
  sceneBuilder.addPicture(Offset(0, 0), picture);
  sceneBuilder.pop();

  // 生成 Scene
  Scene scene = sceneBuilder.build();

  // 将 scene 送入 Engine 层进行渲染显示
  window.onDrawFrame = () {
    window.render(scene);
  };
  window.scheduleFrame();
}
```

### 绘制流程
![](Flutter%20-%20PaintingContext%E5%92%8CLayer/B42426AC-635F-44BF-B1CD-F60220CDA199.png)
* 在 UI Frame 刷新时，通过RendererBinding.drawFrame->PipelineOwner.flushPaint触发RenderObject.paint。
* RenderObject.paint调用PaintingContext.canvas提供的图形操作接口(draw、clip、transform等)完成绘制任务。上述绘制操作被 PictureRecorder 记录下来，在绘制结束时生成 picture，并被添加到 PictureLayer (_currentLayer)上。
* 随后如果有child renderobject的话，RenderObject 通过PaintingContext.paintChild递归地绘制子节点， 在绘制子节点时，根据子节点是否是「Repaint Boundary」而采用不同的策略：
	* 是「Repaint Boundary」— 为子节点生成新的 PaintingContext，从而子节点可以独立进行绘制，绘制结果就是一颗「Layer subTree」，最后将该子树 append 接到父节点生成的「Layer Tree」上。
	* 不是「Repaint Boundary」— 子节点直接绘制在当前PaintingContext.canvas上，即 RenderObject 与 Layer 是多对一的关系。
* 之后整个绘制流程结束时就得到了一棵「Layer Tree」，其后通过 SceneBuilder 生成 Scene，再经window.render送入 Engine 层，最终 GPU 对其进行光栅化处理，显示在屏幕上。

# Layer
[深入浅出 Flutter Framework 之 Layer](https://juejin.cn/post/6897973450485268493)
[从源码看flutter（四）：Layer篇](https://juejin.cn/post/6844904133728665614)

![](Flutter%20-%20PaintingContext%E5%92%8CLayer/CA909E8F-7760-42B8-9DE1-627AC462CE97.png)
Layer Tree 是 Flutter Framework 最终的输出产物，之后的流程就进入到 Flutter Engine 了。

在build过程中，由 Element Tree 生成 RenderObject Tree，RenderObjectElement 才会有对应的 RenderObject。

在paint阶段，由 RenderObject Tree 生成 Layer Tree ，只有当RenderObject的isRepaintBoundary为true时才会生成独立的 Layer 节点。

Layer通过`rootLayer.addToScene(sceneBuilder)`方法加入到

### Layer的分类
![](Flutter%20-%20PaintingContext%E5%92%8CLayer/6A884E88-9E97-419E-9447-2DF1234ABAD8.png)
Flutter中 Layer是抽象基类，其内部实现了基本的 Layer Tree 的管理逻辑以及对渲染结果复用的控制逻辑。 具体的 Layer 大致可以分为2类：
	* Container Layer：正如其名，作为 Layer 容器，用于管理一组 Layers，是唯一可以拥有 child layer 的 Layer；
	* 非 Container Layer：真正用于承载渲染结果的 layer，在 Layer Tree 中属于叶结点，如：PictureLayer承载的是图片的渲染结果，TextureLayer承载的是纹理的渲染结果。

### Layer的状态管理理解
抽象基类Layer一个非常重要的职责就是管理 Layer Tree 的状态，什么情况下可以复用 engine 在前一帧渲染的结果，什么情况下需要刷新，即需要 engine 重新渲染。

每个 Layer 实例都有一个与之对应的EngineLayer实例，其属于 engine 层范畴，对 framework 来说是个黑盒，可以简单理解 EngineLayer 为 engine 渲染的结果。

Layer上的内容需要借助SceneBuilder生成Scene，之后才能被渲染在屏幕上。 这一过程被称之为addToScene。

Layer内有一个非常重要的变量：_needsAddToScene，用于记录该 Layer 自上次渲染后(addToScene)是否发生了变化。 即，该 layer 背后的 EngineLayer 是否可以复用。在Layer刚初始化时，_needsAddToScene为true，在第一次调用addToScene后置为false，之后有几种情况可能会被再次置为true，表明该 layer需要 engine 重新渲染，如下图所示：
	* 自身发生了变化，如PictureLayer.picture、TransformLayer.transform被重新赋值。
	* 子节点有增删。
	* 子节点的needsAddToScene变为true。
![](Flutter%20-%20PaintingContext%E5%92%8CLayer/4C57F00F-FEB4-409F-B1FF-8FEEE18BD909.png)

Layer.alwaysNeedsAddToScene为true时，表示该 layer 在每帧刷新时都需要重新渲染。

addToScene是Layer中最重要的方法之一，用于将 layer 送入 engine 进行渲染， 由具体的子类去实现该方法，Container 类型的 Layer 与非 Container 类型的 Layer 在实现上还是有较大区别。

RenderView 是 RenderObject Tree 的根节点，在每帧刷新时都会调用其compositeFrame方法去合成新的帧，RenderView 对应的 Layer 是ContainerLayer。注意：
	* ContainerLayer.buildScene()方法首先去更新needsAddToScene标志位 (对 Layer Tree 进行深度遍历)，子节点的值会影响父节点 (子节点有更新时，父节点肯定也要刷新)；
	* 之后，调用ContainerLayer.addToScene()方法，该方法会对子节点进行递归操作；
	* 注意，在ContainerLayer._addToSceneWithRetainedRendering()方法中，当_ needsAddToScene为false且_engineLayer!=nil时直接复用上次的渲染结果。

### Layer的渲染流程
![](Flutter%20-%20PaintingContext%E5%92%8CLayer/4DD10F74-83FB-4DDC-BB59-C3C495308D4F.png)

Layer最终会通过sceneBuilder生成screen，然后调用window的render方法进行合成渲染。


```dart
//ContainerLayer
  ui.Scene buildScene(ui.SceneBuilder builder) {
    ...
    updateSubtreeNeedsAddToScene();
    addToScene(builder);
    ...
    _needsAddToScene = false;
    ...
    return scene;
  }
```

```dart
//TransformLayer
  @override
  void addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    ...
    engineLayer = builder.pushTransform(
      _lastEffectiveTransform.storage,
      oldLayer: _engineLayer as ui.TransformEngineLayer,
    );
    addChildrenToScene(builder);
    builder.pop();
  }
```