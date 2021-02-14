# Flutter - Element
#flutter/核心类分析

[深入浅出 Flutter Framework 之 Element](https://juejin.cn/post/6844904161427849223#heading-5)
[从源码看flutter（二）：Element篇](https://juejin.cn/post/6844904130142535694)


UI 的层级结构在 Element 间形成一棵真实存在的树「Element Tree」，Element 有 2 个主要职责：
* 根据 UI信息 (「Widget Tree」) 的变化来维护「Element Tree」，包括：节点的插入、更新、删除、移动等。
* 实现了BuildContext接口作为Widget 与 RenderObject 间通信的桥梁。

## Element分类
![](Flutter%20-%20Element/4D6C0FE6-B1F1-4FB7-A480-28174193CB57.png)
Element 根据特点可以分为 2 类：
1. **Component Element** —— 组合型 Element，「Component Widget」、「Proxy Widget」对应的 Element 都属于这一类型，其特点是子节点对应的 Widget 需要通过build方法去创建。同时，该类型 Element 都只有一个子节点 (single child)。
2. **Renderer Element** —— 渲染型 Element，对应「Renderer Widget」，其不同的子类型包含的子节点个数也不一样：
	* LeafRenderObjectElement 没有子节点。
	* RootRenderObjectElement、SingleChildRenderObjectElement 有一个子节点。
	* MultiChildRenderObjectElement 有多个子节点。

## 和其他对象的关系
![](Flutter%20-%20Element/47611C8B-A59E-4A95-866F-92FB251E0846.png)
Element和其他各种对象的关系如下：
	* Element 通过 parent、child 指针形成「Element Tree」。
	* Element 持有 Widget、Render Object。
	* State 是绑定在 Element 上的，而不是绑在Stateful Widget上(这点很重要)。
当然也不是所有类型的 Element 都有下面的关系，比如：
	* RenderObject只有RenderObject Element才有。
	* State 只有Stateful Element才有。 

## 生命周期理解
![](Flutter%20-%20Element/E8CCBD55-661A-4C28-B097-6FCC8D6DBA7F.png)
1. mount方法负责所有组合型节点的 build (这是一个递归的过程)，对于渲染型，mount方法需要负责将renderobject添加到render tree上。
2. element是复用的，并且在移除后当前动画桢结束之前并不会被立即销毁，还可以重新mount到element树上。

### 创建形成树
![](Flutter%20-%20Element/5CF28477-DF92-46BE-8B78-F994B80DD5D9.png)
Element创建流程的入口时机（`createElement()` 被调用）有两个地方：
	* 其一是作为根节点 Element 的 RenderObjectToWidgetElement 在 RenderObjectToWidgetAdapter 的 attachToRenderTree(…) 中被创建。
	* 另一个是其他所有 Element 在 inflateWidget(…) 方法中被创建（也可以说是在上层updateChild处理子节点时传入的child节点为空，也会调用inflateWidget创建子element节点）。

整体的创建过程可以理解就是不断根据widget创建element再mount到element树，再创建renderObject并挂载到render树上。具体讲就是从根element出发，然后根据左边的widget描述树的widget节点创建子element并mount挂载到element树上，子element的mount挂的操作根据类型不同（渲染型、组合型等）决定接下来行为：
	1. 组合型子element：向左接着通过调用performRebuild触发build操作（有状态的element通过state间接build）来build它组合的所有子widget描述树的新的叶子节点，有了叶子widget节点后再通过`updateChild`创建并mount挂载widget树新叶子结点对应的element节点到element树上。
	2. 渲染型子element：向右通过调用`widget.createRenderObject(this)`创建renderObject再插入到render树上。

另外创建后mount时，关于slot的理解如下：
	* `dynamic Element.slot`父节点用于确定其下子节点的排列顺序 (兄弟节点间的排序)，因此，对于单子节点的节点 (single child)，child.slot 通常为 null。
	* slot 的类型是动态的，不同类型的 Element 可能会使用不同类型的 slot，如：Sliver 系列使用的是 int 型的 index，MultiChildRenderObjectElement 用兄弟节点作为后一个节点的 slot。

被mount后，此时(child) element 处于 **active** 状态，其内容随时可能显示在屏幕上。

### 状态更新：父节点递归传递下来的更新操作updateChild
![](Flutter%20-%20Element/6981125B-434D-4711-9B18-1A0D8C6E9D35.png)
![](Flutter%20-%20Element/A638E736-0E16-4C34-A832-E26E65679108.png)
Element状态更新的场景是一个递归向下的更新循环，里面涉及到更新element节点的位置更新、节点信息更新、删除节点等不同情况。

整体理解就是从父节点调用updateChild处理当前需要被处理的子element开始，此时child也就是这个被处理的element肯定不为null，所以根据不同情况在updateChild方法内产生了如下不同行为：
	1. 更新element节点的位置：若 child.widget == newWidget，说明 child.widget 前后没有变化，若 child.slot != newSlot 表明子节点在兄弟结点间移动了位置，通过`updateSlotForChild`修改 child.slot 。
	2. 更新element节点信息：如果通过`Widget.canUpdate`判断是可以用 newWidget 修改 child element，若可以（新老 Widget.[key || runtimeType] 相等）则调用`update`方法首先更改这个子element节点对应的widget为新widget，然后根据当前已经被更新的element的类型不同子类`update`内又有如下不同行为：
		1. 组合型、代理型element向左最终都会先标记当前已经被更新的element为dirty，再调用rebuild流程向下（这里开始进行下层节点的更新处理）在performRebuild内重新build当前已经被更新element节点对应widget树节点下面的所有叶子节点形成新的子widget描述树，然后再拿新build出来的widget树的根节点继续作为参考树updateChild处理当前已被更新完毕的element节点的child节点的状态更新（进入递归的状态更新循环，从新build出来的widget参考树的根节点一直到叶子结点，以此来更新对应的element树节点的状态）。
		2. 渲染型element会向右通过`widget.updateRenderObject(this, renderObject)`来把需要更新对应renderObject节点的消息告诉给当前已经被更新element节点对应的新widget节点，从而更新render节点上的信息（自定义renderObjectWidget时就是在这里通过widget更新renderObject上的一些信息（重要点）），然后不会标记为dirty，更具体的子类`update`内还会有如下不同行为：
			1. SingleChildRenderObjectElement：继续updateChild处理child节点的状态更新。
			2. MultiChildRenderObjectElement：最终还是会继续updateChild处理所有child节点的状态更新。
	3. 移除element重新创建新节点：否则表示不可更新，会做如下处理：
		1. 父节点先将这个被处理的child element 进行deactivateChild，主要做了以下3 件事，之后该element 就处于 “inactive” 状态，并从屏幕上消失，该状态一直持续到当前帧动画结束，这3件事如下：
			* 从Element Tree中移除该 element 节点(将 parent 置为 null)。
			* 将相应的render节点从render tree上移除。
			* 将移除的element 节点添加到owner._inactiveElements中，在添加过程中会对『以该 element 为根节点的子树上所有节点』调用deactivate方法 (移除的是整棵子树)。
		2. 再通过`inflateWidget(newWidget, newSlot)`重新根据新widget节点配置子element树节点和render树节点，从上面element 进入 “inactive” 状态到当前帧动画结束期间，其还有被『抢救』的机会，根据『带有「global key」&& 被重新插入树中』有如下不同行为：
			1. 如果可以抢救：重新中_inactiveElements取出element
				* 该 element 将会从owner._inactiveElements中移除。
				* 对该 element subtree 上所有节点调用activate方法 (它们又复活了！)。
				* 将相应的「render object」重新插入「render tree」中。
				* 该 element subtree 又进入 “active” 状态，并将再次出现在屏幕上。
			2. 抢救不回来的话：`inflateWidget(newWidget, newSlot)`内重新`newWidget.createElement()`创建新的子element节点并`mount`到element树上（这时候相当于重新进入一开始的创建流程，递归创建下面的element、render子树）。
### 被销毁：Element 节点所在的子树随着 UI 的变化被移除时。
![](Flutter%20-%20Element/EE8B6E91-ED13-48DB-BF7E-4C10600A920F.png)
对于所有在当前帧动画结束时未能成功『抢救』回来的「Inactive Elements」，在下一桢绘制时，`WidgetsBinding.drawFrame`触发`BuildOwner.finalizeTree`，最终调用unmount，至此，element 生命周期圆满结束。

为了避免在一次动画执行过程中反复创建、移除某个特定element，“inactive”态的element在当前动画最后一帧结束前都会保留，如果在动画执行结束后它还未能重新变成“active”状态，Framework就才会调用其unmount方法将其彻底移除，这时element的状态为defunct，它将永远不会再被插入到树中。

## 理解Element的依赖 (Dependencies)
```dart
//Element基类
Map<Type, InheritedElement> _inheritedWidgets;//字典
Set<InheritedElement> _dependencies;//集合

void _updateInheritance() {
  _inheritedWidgets = _parent?._inheritedWidgets;
}

//InheritedElement
final Map<Element, Object> _dependents //字典：
```
###  _inheritedWidgets字典
用于收集从「Element Tree」的根节点到当前节点路径上所有的「Inherited Elements」，在mount方法结束的最后会调用_updateInheritance()方法更新。
### _updateInheritance()方法
用于更新_inheritedWidgets，在不同子类上的具体差别为：
	* Element的基类的实现是直接获得父节点的_inheritedWidgets。
	* InheritedElement类的实现是在父节点的基础上将自己加入到_inheritedWidgets中，以便其子孙节点的_inheritedWidgets包含它
### _dependencies集合
用于记录当前节点依赖了哪些「Inherited Elements」，通常我们调用`context.dependOnInheritedWidgetOfExactType<T>`时就会在当前节点与目标 Inherited 节点间形成依赖关系。
### _dependents字典
在InheritedElement中用于记录所有依赖于它的节点，在「Inherited Element」发生变化，需要通知依赖者时。
```dart
void notifyClients(InheritedWidget oldWidget) {
  for (Element dependent in _dependents.keys) {
    // check that it really depends on us
    assert(dependent._dependencies.contains(this));
    notifyDependent(oldWidget, dependent);
  }
}
```

## 重要方法分析
### updateChild：父节点通过该方法来更新子节点的状态
```dart
Element updateChild(Element child, Widget newWidget, dynamic newSlot) {
  if (newWidget == null) {
    if (child != null)
      deactivateChild(child);
    return null;
  }

  if (child != null) {
    if (child.widget == newWidget) {
      if (child.slot != newSlot)
        updateSlotForChild(child, newSlot);
      return child;
    }

    if (Widget.canUpdate(child.widget, newWidget)) {
      if (child.slot != newSlot)
        updateSlotForChild(child, newSlot);
      child.update(newWidget);
      assert(child.widget == newWidget);

      return child;
    }

    deactivateChild(child);
    assert(child._parent == null);
  }

  return inflateWidget(newWidget, newSlot);
}
```
![](Flutter%20-%20Element/4E4B2F7E-F5D9-4ECC-8B5D-1138404B0333.png)
`updateChild(Element child, Widget newWidget, dynamic newSlot)`
在「Element Tree」上，**父节点通过该方法来修改子节点对应的 Widget**，子类一般不需要重写该方法，由基类Element实现统一行为的模板方法，根据传入参数的不同，该方法有以下几种不同的行为：
	1. newWidget == null —— 说明子节点对应的 Widget 已被移除，直接 remove child element (如果有)。
	2. child == null —— 说明 newWidget 是新插入的，通过`inflateWidget`传入这个newWidget创建子element节点 。
	3. child != null —— 此时一般是更新场景，分为 3 种情况：
		1. 若 child.widget == newWidget，说明 child.widget 前后没有变化，若 child.slot != newSlot 表明子节点在兄弟结点间移动了位置，通过`updateSlotForChild`修改 child.slot 。
		2. 通过`Widget.canUpdate`判断是否可以用 newWidget 修改 child element，若可以，则调用`update`方法；
		3. 否则先将 child element 移除，并通过 newWidget 创建新的 element 子节点。
### inflateWidget：把传入的newWidget对应的element作为child挂载到当前element节点下并返回这个对应的element
```dart
Element inflateWidget(Widget newWidget, dynamic newSlot) {
  final Key key = newWidget.key;
  if (key is GlobalKey) {
    final Element newChild = _retakeInactiveElement(key, newWidget);
    if (newChild != null) {
      newChild._activateWithParent(this, newSlot);
      final Element updatedChild = updateChild(newChild, newWidget, newSlot);
      return updatedChild;
    }
  }

  final Element newChild = newWidget.createElement();
  newChild.mount(this, newSlot);
  return newChild;
}
```
属于模板方法，故一般情况下子类不用重写，主要调用路径来自上面介绍的updateChild方法，有如下2种行为：
	1. 如果传入的widget 带有 GlobalKey，首先在 Inactive Elements 列表中查找是否有处于 inactive 状态的节点 (即刚从树上移除)，如找到就直接复活该element节点，之后传入复活的element节点调用updateChild更新这个复活的element和newWidget的映射关系。
	2. 如果传入的widget 没有 GlobalKey或者有但是没有在 Inactive Elements 列表中找到，则会通过传入的newWidget 调它的`newWidget.createElement()`创建它对应的element，之后用新生成的element.mount方法把它挂到当前element节点下。

### update：子类重写该方法以处理具体的Widget更新逻辑
在更新流程中，若新老 Widget.[runtimeType && key] 相等，则会走到该方法，子类需要重写该方法以处理具体的Widget更新逻辑，子类重写该方法时必须调用 super。

**Element 基类**
```dart
@mustCallSuper
void update(covariant Widget newWidget) {
  _widget = newWidget;
}
```

**StatelessElement**
```dart
void update(StatelessWidget newWidget) {
  super.update(newWidget);
  _dirty = true;
  rebuild();
}
```
通过rebuild方法触发重建 child widget (第 4 行)，并以此来 update child element，期间会调用到StatelessWidget.build方法 (也就是我们写的 Flutter 代码)。

**StatefulElement**
```dart
void update(StatefulWidget newWidget) {
  super.update(newWidget);
  final StatefulWidget oldWidget = _state._widget;
  _dirty = true;
  _state._widget = widget;
  try {
    _state.didUpdateWidget(oldWidget) as dynamic;
  }
  finally {
  }
  rebuild();
}
```
相比StatelessElement，StatefulElement.update稍微复杂一些，需要处理State的更新，逻辑如下：
* 修改 State 的 _widget属性；
* 调用State.didUpdateWidget (熟悉么)。
最后，同样会触发rebuild操作，期间会调用到State.build方法。

**ProxyElement**
```dart
void update(ProxyWidget newWidget) {
  final ProxyWidget oldWidget = widget;
  super.update(newWidget);
  updated(oldWidget);
  _dirty = true;
  rebuild();
}
void updated(covariant ProxyWidget oldWidget) {
  notifyClients(oldWidget);
}
Widget build() => widget.child;
```
ProxyElement.update方法需要关注的是对updated的调用，其主要用于通知关联对象 Widget 有更新，具体通知逻辑在子类中处理，如：
	* InheritedElement会触发所有依赖者 rebuild (对于 StatefulElement 类型的依赖者，会调用State.didChangeDependencies)。
ProxyElement 的build操作很简单：直接返回widget.child。

**RenderObjectElement**
```dart
void update(covariant RenderObjectWidget newWidget) {
  super.update(newWidget);
  widget.updateRenderObject(this, renderObject);
  _dirty = false;
}
```
主要是调用了widget.updateRenderObject来更新「Render Object」

**SingleChildRenderObjectElement**
```dart
void update(SingleChildRenderObjectWidget newWidget) {
  super.update(newWidget);
  _child = updateChild(_child, widget.child, null);
}
```
通过newWidget.child调用updateChild方法递归修改子节点。

**MultiChildRenderObjectElement**
```dart
void update(MultiChildRenderObjectWidget newWidget) {
  super.update(newWidget);
  _children = updateChildren(_children, widget.children, forgottenChildren: _forgottenChildren);
}
```
在updateChildren方法中处理了子节点的插入、移动、更新、删除等所有情况。

### mount：把自己挂载到传入的Element节点下
当 Element 第一次被插入「Element Tree」上时，调用该方法。由于此时 parent 已确定，故在该方法中可以做依赖 parent 的初始化操作。经过该方法后，element 的状态从 “initial” 转到了 “active”，子类重写该方法时，必须调用 super。在基类中Element实现里会把父节点的 owner 传给了子节点。 如果对应的 Widget 带有 GlobalKey，进行相关的注册。 最后继承来自父节点的「Inherited Widgets」。

**Element**
```dart
@mustCallSuper
void mount(Element parent, dynamic newSlot) {
  _parent = parent;
  _slot = newSlot;
  _depth = _parent != null ? _parent.depth + 1 : 1;
  _active = true;
  if (parent != null) // Only assign ownership if the parent is non-null
    _owner = parent.owner;

  if (widget.key is GlobalKey) {
    final GlobalKey key = widget.key;
    key._register(this);
  }

  _updateInheritance();
}
```

然后子类不同有不同的如下行为：
**ComponentElement**
```dart
void mount(Element parent, dynamic newSlot) {
  super.mount(parent, newSlot);
  _firstBuild();
}
void _firstBuild() {
  rebuild();
}
```
组合型 Element 在第一次挂到element树上时会执行_firstBuild再进入rebuild流程，最终调用performRebuild。

**RenderObjectElement**
```dart
void mount(Element parent, dynamic newSlot) {
  super.mount(parent, newSlot);
  _renderObject = widget.createRenderObject(this);
  attachRenderObject(newSlot);
  _dirty = false;
}
```
在RenderObjectElement.mount中做的最重要的事就是拿当前element 对应的widget调用`widget.createRenderObject(this) `创建了对应的Render Object，再在attach内通过`insertRenderObjectChild`把创建的renderObject到插到render树上。

**SingleChildRenderObjectElement**
```dart
@override
void mount(Element parent, dynamic newSlot) {
  super.mount(parent, newSlot);
  _child = updateChild(_child, widget.child, null);
}
```
SingleChildRenderObjectElement在 super (RenderObjectElement) 的基础上，调用`updateChild`方法处理子节点。

**MultiChildRenderObjectElement**
```dart
void mount(Element parent, dynamic newSlot) {
  super.mount(parent, newSlot);
  _children = List<Element>(widget.children.length);
  Element previousChild;
  for (int i = 0; i < _children.length; i += 1) {
    final Element newChild = inflateWidget(widget.children[i], previousChild);
    _children[i] = newChild;
    previousChild = newChild;
  }
}
```
MultiChildRenderObjectElement在 super (RenderObjectElement) 的基础上，对每个子节点直接调用inflateWidget方法。

### markNeedsBuild：当前 Element 加入_dirtyElements中，以便在下一帧可以rebuild。
```dart
void markNeedsBuild() {
  if (!_active)
    return;
  if (dirty)
    return;
  _dirty = true;
  owner.scheduleBuildFor(this);
}
```
子类一般不必重写该方法，它的主要调用场景：
	* State.setState —— 状态变更时需要刷新然后rebuild。
	* Element.reassemble —— debug hot reload。
	* Element.didChangeDependencies —— 当这个Element依赖的「Inherited Widget」有变化时会导致依赖者 rebuild，就是从这里触发的。
	* StatefulElement.activate —— 当 Element 从 “inactive” 到 “active” 时，会调用该方法，因为StatefulElement有附带的 State，需要给它一个activate的机会。

### rebuild：对活跃的、脏节点调用performRebuild
```dart
void rebuild() {
  if (!_active || !_dirty)
    return;
  performRebuild();
}
```
在下面几种场景下被调用：
	* 对于被标记为 dirty 的element，在新一帧绘制过程中由BuildOwner.buildScope触发。
	* Component Element在 element 挂载时，在Element.mount内触发。
	* Component Element在update方法内被调用。

### performRebuild：需要由子类重写实际的rebuild操作
**ComponentElement**
```dart
void performRebuild() {
  Widget built;
  built = build();
  _child = updateChild(_child, built, slot);
}
```
对于ComponentElement，rebuild 过程其实就是先调用`Widget build()`方法生成子widget（平常外部代码里写的生成widget的方法），再把生成的这个子widget作为newWidget传入updateChild来更新child element。

有状态和无状态的widget对应的element在build子widget树时，差别就在无状态的element是直接调用widget.build，有状态的通过state间接build。

**RenderObjectElement**
```dart
void performRebuild() {
  widget.updateRenderObject(this, renderObject);
  _dirty = false;
}
```
在渲染型 Element 基类中只是拿当前element对应的widget的`updateRenderObject`方法更新了对应的Render Object，更具体的子类还进行了更具体的不同处理。




