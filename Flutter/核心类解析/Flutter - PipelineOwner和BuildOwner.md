# Flutter - PipelineOwner和BuildOwner
#flutter/核心类分析

# BuildOwner
BuildOwer在 Element 状态管理上起到重要作用：
	* 在 UI 更新过程中跟踪、管理需要 rebuild 的 Element (dirty elements)。
	* 在有dirty elements时，及时通知引擎，以便在下一帧安排上对「dirty elements」执行 rebuild，从而去刷新 UI。
	* 管理处于 inactive状态的 Element。

BuildOwer在启动流程中WidgetsBinding做初始化准备时创建。
整棵Element Tree共享同一个BuildOwer实例，从根Element开始，在做挂载mount操作时由 parent 节点传递给 child element节点。

一般情况下并不需要我们手动实例化BuildOwer，除非需要离屏沉浸 (此时需要构建 off-screen element tree)。

BuildOwer两个关键成员变量，分别用于存储收集到的「Inactive Elements」、「Dirty Elements」。
```dart
final _InactiveElements _inactiveElements = _InactiveElements();
final List<Element> _dirtyElements = <Element>[];
```

## 如何管理Dirty Elements
### 通过scheduleBuildFor()收集
对于需要更新的 element，应该调用Element.markNeedsBuild方法，该方法会最终调用BuildOwer.scheduleBuildFor()方法，主要做了2件事情：
* 调用onBuildScheduled，该方法(其实是个callback)会通知 Engine 在下一帧需要做更新操作。
* 将Dirty Elements加入到_dirtyElements中。
```dart
void scheduleBuildFor(Element element) {
  assert(element.owner == this);
  onBuildScheduled();
  _dirtyElements.add(element);
  element._inDirtyList = true;
}
```

### 通过buildScope()更新
有2个比较重要的时机会调用BuildOwer.buildScope：
1. 在渲染循环里，新一帧绘制到来时，WidgetsBinding.drawFrame会在绘制前调用BuildOwer.buildScope(根Element)来更新。
2. 启动流程里有分析，首次根Element把根widget和renderObject关联起来时也会调用，并且在传入的回调中进行了根element的mount操作，创建了整颗element树。

BuildOwer.buildScope的实现逻辑主要如下：
* 如有回调，首先执行回调。
* 对dirty elements按在「Element Tree」上的深度排序 (即 parent 排在 child 前面) ，然后依次调用Element.rebuild()。
* 清理_dirtyElements。
```dart
void buildScope(Element context, [ VoidCallback callback ]) {
  if (callback == null && _dirtyElements.isEmpty)
    return;

  try {
    if (callback != null) {
      callback();
    }

    _dirtyElements.sort(Element._sort);
    int dirtyCount = _dirtyElements.length;
    int index = 0;
    while (index < dirtyCount) {
      _dirtyElements[index].rebuild();
      index += 1;
    }
  } finally {
    for (Element element in _dirtyElements) {
      element._inDirtyList = false;
    }
    _dirtyElements.clear();
  }
}
```

### 管理Inactive Elements
Inactive Element，是指 element 从「Element Tree」上被移除到 dispose 或被重新插入「Element Tree」间的一个中间状态。 

**设计 inactive 状态的主要目的是实现『带有「global key」的 element』可以带着『状态』在树上任意移动。**

BuildOwer 负责对「Inactive Element」进行管理，包括添加、删除以及对过期的「Inactive Element」执行 unmount 操作。


# PipelineOwner
![](Flutter%20-%20PipelineOwner%E5%92%8CBuildOwner/7B3A5080-0B7C-41F3-916B-ADC5F3CACAD8.png)
PipelineOwner在 Rendering Pipeline 中起到重要作用：
	* 随着 UI 的变化而不断收集 Dirty Render Objects
	* 随之驱动 Rendering Pipeline 刷新 UI

PipelineOwner也是RenderObject Tree与RendererBinding间的桥梁，在两者间起到沟通协调的作用。

启动流程中RendererBinding创建并持有PipelineOwner实例，同时还会创建RenderObject Tree的根节点（即RenderView）并将其赋值给PipelineOwner.rootNode。

在『RenderObject Tree』构建过程中，每插入一个新节点，就会将PipelineOwner实例 attach 到该节点上，即『RenderObject Tree』上所有结点共享同一个PipelineOwner实例。

正常情况下在 Flutter 运行过程中只有一个PipelineOwner实例，并由RendererBinding持有，用于管理所有『 on-screen RenderObjects 』。 然而，如果有『 off-screen RenderObjects 』，则可以创建新的PipelineOwner实例来管理它们。 『on-screen PipelineOwner』与 『 off-screen PipelineOwner 』完全独立，后者需要创建者自己维护、驱动。

![](Flutter%20-%20PipelineOwner%E5%92%8CBuildOwner/E235A191-8D50-4DF9-B2C6-7C60F528F4A9.png)
![](Flutter%20-%20PipelineOwner%E5%92%8CBuildOwner/4C81FAEA-347C-49DA-8A51-17D0385582CA.png)
### 如何管理并收集Dirty RenderObjects
首先Render Object 有4种『 Dirty State』 需要 PipelineOwner 去维护、收集起来，具体如下：
	* Needing Layout：Render Obejct 需要重新 layout，调用`markNeedsLayout`方法，该方法会将当前 RenderObject 加入 PipelineOwner的_nodesNeedingLayout或传给父节点去处理。
	* Needing Compositing Bits Update：Render Obejct 合成标志位(Compositing)有变化，调用`markNeedsCompositingBitsUpdate`方法，该方法会将当前RenderObject 加入 PipelineOwner的_nodesNeedingCompositingBitsUpdate或传给父节点去处理。
	* Needing Paint：Render Obejct 需要重新绘制，调用`markNeedsPaint`方法，该方法会将当前 RenderObject 加入PipelineOwner._nodesNeedingPaint或传给父节点处理。
	* Needing Semantics：Render Object 辅助信息有变化，调用`markNeedsSemanticsUpdate`方法，该方法会将当前 RenderObject 加入 PipelineOwner._nodesNeedingSemantics或传给父节点去处理。
### 如何驱动 Rendering Pipeline 刷新 UI
* 上面的4个markNeeds方法除了`markNeedsCompositingBitsUpdate`之外最后都会调用`PipelineOwner.requestVisualUpdate`方法（markNeedsCompositingBitsUpdate不会调用是因为它不会单独出现，一定是伴随其他3个一起的），随着 `PipelineOwner.requestVisualUpdate -> RendererBinding.scheduleFrame -> Window.scheduleFrame`调用链，UI 需要刷新的信息最终传递到了 Engine 层。
* 之后Window.scheduleFrame向 Engine 请求下一帧需要刷新。
* Engine 在接收到 UI 需要更新后，在下一帧刷新时会调用Window.onBeginFrame和Window.onDrawFrame ，然后会回到RendererBinding向widow注册的回调_handleBeginFrame和_handleDrawFrame中，通过提前注册好的PersistentFrameCallback，最终调用到RendererBinding.drawFrame 方法触发UI刷新。
```dart
//RendererBinding
void drawFrame() {
  pipelineOwner.flushLayout();
  pipelineOwner.flushCompositingBits();
  pipelineOwner.flushPaint();
  renderView.compositeFrame(); 
  pipelineOwner.flushSemantics(); 
}
```

**Flush Layout**
首先，PipelineOwner对于收集到的『 Needing Layout RenderObjects 』按其在『 RenderObject Tree 』上的深度升序排序，主要是为了避免子节点重复 Layout (因为父节点 layout 时，也会递归地对子树进行 layout)。
其次，对排好序的且满足条件的 RenderObjects 依次调用_layoutWithoutResize来执行 layout 操作。

**Flush Compositing Bits**
同理，先对『 Needing Compositing Bits RenderObjects 』排序，再调用RenderObjects#_updateCompositingBits

**Flush Paint**
对于 Paint 操作来说，父节点需要用到子节点绘制的结果，故子节点需要先于父节点被绘制。 因此，不同于前两个 flush 操作，此时需要对『 Needing Paint RenderObjects 』按深度降序排序。

之后进入PipelineOwner.flushPaint到PaintingContext内部操作的过程。

**Flush Semantics**
Flush Semantics 所做操作与 Flush Layout 完全相似。
