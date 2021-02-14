# UI常见问题记录
#iOS知识点/UI相关

# UIView和CALayer的理解？
UIView：负责用户的交互事件
CALayer：负责图像和动画的渲染

UIView相当于CALayer的进一步封装，每个UIView都有一个图层layer，View大部分显示相关的属性都是对应于layer。
UIVIew并作为CALayer的代理，负责创建并且管理这个图层。

为什么要区分：单一职责的系统设计原则。

# UIViewController中UIView的生命周期？
**加载阶段：**
1. 在loadView阶段（内存加载阶段），先是把自己本身都加到superView上面，再去寻找自己的subView并依次添加。到此为止，只和addSubview操作有关，和是否显示无关。
2. 等到所有的subView都在内存层面加载完成了，会调用一次viewWillAppear，然后会把加载好的一层层view，从父到子layoutSubview，再分别从父到子绘制到window上面，加载即完成。

**viewDidLoad时调用addSubView初始化视图层级 -> viewWillAppear后开始显示（首先layoutSubview确认布局再DrawRect绘制）**

**移除阶段：**
1. 会先调用moveToWindow移除本view的，然后依次调用moveToWindow移除所有子视图，view就在window上消失不见了。
2. 然后在removeFromSuperView，然后dealloc。dealloc之后再removeSubview。（但不理解为什么dealloc之后再removeSubview）

# UIView的几个方法的理解？
## 视图布局方法
### layoutSubviews方法
这个方法默认没有做任何事情，需要子类进行重写，一般是在内部对需要对子视图进行精确的位置布局时重写，如果要在外部通过约束去布局则不应该重写这个方法。

**触发方式**
1. 调用self.setNeedsLayout的时候。
2. addSubview的时候。
3. view的size发生改变的时候，前提是frame的值设置前后发生了变化。
4. 滑动UIScrollView的时候。

### setNeedsLayout方法：
标记为需要重新布局，不会立即刷新，异步等待下次loop循环到来时调用layoutSubviews方法。layoutIfNeeded可以马上刷新布局。

### layoutIfNeeded方法：
如果有需要刷新的标记，立即调用layoutSubviews进行布局（如果没有标记，不会调用layoutSubviews）

> layoutIfNeeded遍历的不是superview链，是subviews链  

## 视图绘制方法
### drawRect:(CGRect)rect方法：
获取图形上下文，执行自定义的重绘任务。重写这个方法后会生成一张寄宿图，在方法内利用Core Graphics完成绘制后，内容就会缓存起来，等待下次调用setNeedsDisplay时再进行更新。

**系统调用时机**
drawRect调用是在Controller->loadView,Controller->viewWillAppear->viewDidLayoutSubviews 方法之后调用的。

**drawRect注意点**
1. 默认不做任何事情，直接创建UIView的子类可以不用调用super，如果创建的是不同的view类，你需要在实现代码的某个时候调用super。
2. 若使用UIView绘图，只能在drawRect：方法中获取相应的contextRef并绘图。在其他方法中将获取到一个invalidate的ref并且不能用于画图。（因为实际View的drawRect是在其drawLayer:inContext代理方法里调用的，而drawLayer:inContext方法内部会把系统维护的当前图层的上下文栈给push到栈顶，在drawLayer:inContext完成后会进行pop，drawRect的执行刚好被包裹在其中）
3. 方法不能手动显示调用，必须通过调用setNeedsDisplay或者setNeedsDisplayInRect，由系统自动调该方法。
4. drawRect重绘时在方法内部设置一些图层显示相关的属性比如backgroundColor、cornerRadius（即使是不含子视图情况下）是无效的。
5. 若使用Calayer绘图，只能在drawInContext绘制，或者在delegate中的相应方法绘制，同样也是调用setNeedDisplay等间接调用以上方法。
6. 若要实时画图，不能使用gestureRecognizer，只能使用touchbegan等方法来调用setNeedsDisplay实时刷新屏幕。
7. 随意使用drawRect可能引入内存暴增的问题（待定），参考 [内存恶鬼drawRect](http://bihongbo.com/2016/01/03/memoryGhostdrawRect/) 

**官方翻译：**
该方法的默认实现并不会做任何事情。子类使用诸如Core Graphics和UIKit技术绘制其控件的内容应该重写该方法，并且把实现的代码写在该方法中。如果你控件的内容是用其他方式设置的，那么你就不需要重写该方法。例如，如果你的控件仅仅只是展示背景颜色，则不需要重写该方法或者你的控件内容是直接使用layer对象设置的，也不需要调用该方法。
当调用该方法时，UIKit框架已经为你的控件配置好合适的绘制环境，你可以轻松的调用任何绘制方法和函数来渲染你的控件内容。特别地，UIKit会创建并配置一个图形上下文，然后调整该上下文的形变，以使该图形上下文的原点与你控件的bounds的原点相匹配。你可以通过调用UIGraphicsGetCurrentContext函数获得图形上下文的引用，但是不要对该图形上下文建立强引用,因为多次调用drawRect:方法期间，图形上下文会改变。
如果你直接创建UIView的子类，该方法的实现不需要调用super。但是，如果你创建的是不同的view类，你需要在实现代码的某个时候调用super。
当一个view第一次显示或当一个事件发生，该事件导致view的可视部分无效，永远不要手动调用该方法。为了使控件的某个部分失效，并且因此导致某个部分重绘，应该调用setNeedsDisplay或者setNeedsDisplayInRect:而不是drawRect方法。

### setNeedsDisplay方法：
标记为需要重绘，不会立即重绘，等待下次loop循环到来时调用drawRect。

## 计算大小方法
### sizeThatFits: 
会计算出最合适的 size 但是不会改变自己的 size。
### sizeToFit: 
会计算出最合适的 size 而且会改变自己的 size。
> sizeToFit相当于调用sizeThatFits方法返回size后设置了frame  

# cornerRadius和maskToBounds？
cornerRadius：官方解释只是对view的背景颜色和边框做圆角，对于一些像ImageView以及Label等含有内部子视图的就不起作用了。

**无效原因：**
对于没有子图层的layer，直接设置主图层的cornerRadius是能有圆角效果的，而且不会触发离屏渲染。但是如果有子图层的存在，比如UIImageView这种，设置的图片其实就是设置在子图层的content，所以无效。

maskToBounds：裁剪超出主图层范围内的内容

# Bounds和Frame的区别？
Bounds针对的是自身坐标系，影响子视图位置
Frame针对的是父视图坐标系，影响自身位置
anchroPoint：默认（0.5，0.5）在layer的中间
position：锚点在supLayer坐标系中的位置

> anchorPoint和position的修改互不影响，但是会影响frame.origin，所以修改时会移动视图。  


# loadView方法的理解？
作用：负责创建UIViewController的view。
调用时机：每次访问UIViewController的view，当view为nil，就会调用loadView方法。

**注意点**
在重写loadView方法的时候，一般不调用父类的方法。（原因是没有意义）
返回View之前，不能使用self.view，否则会死循环（系统调用时机的问题）。

# initWithFrame、initWithCoder、awakeFromNib
initWithCoder、awakeFromNib是从xib加载的初始化方法。
initWithFrame是纯代码加载的初始化方法。

# ViewController生命周期？
![](UI%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98%E8%AE%B0%E5%BD%95/060D5B91-D735-49A2-A274-275F37CD0F60.png)
![](UI%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98%E8%AE%B0%E5%BD%95/943BA0AF-9CA0-4F54-AA28-F5182A86F1BF.png)
1. init 或 awakeFromNib：初始化创建。
2. loadView：创建或加载一个view并把赋值给UIViewController的view属性。
3. viewDidLoad：此时整个视图层次(view hierarchy)已经放到内存中，可以移除一些视图，修改约束，加载数据等。
4. viewWillAppear：视图加载完成，并即将显示在屏幕上viewWillLayoutSubviews：即将开始子视图位置布局
5. viewDidLayoutSubviews：用于通知视图的位置布局已经完成
6. viewDidAppear：视图已经展示在屏幕上，可以对视图做一些关于展示效果方面的修改。
7. viewWillDisappear：视图即将消失
8. viewDidDisappear：视图已经消失
9. deinit：视图销毁的时候调用

# Present和Dismiss
presentingViewController：上一个节点
presentedViewController：下一个节点

present只能用当前顶层，最上面节点vc来present。
dismiss的原理是通知父节点dismiss子节点，多个子节点都会被dismiss掉，比如A present B，B present C，B中使用dismiss，BC都会弹出。

# UIButton的父类是什么？UILabel的父类又是什么？
* UIButton -> UIControl -> UIView -> UIResponder
* UILabel -> UIView -> UIResponder

区别在于UIControlState状态、UIControlEvent的事件描述
UIControl可以用sendAction实现手动发出事件。

# UIWindow的理解？
1. UIWindow是一种特殊的UIView，通常在一个app中至少会有一个UIWindow，只有一个keywindow。
2. 启动完毕后，创建的第一个视图控件就是UIWindow，调用makeKeyAndVisible使它成为keyWindow，接着创建控制器的View，最后将控制器的View添加到UIWindow上，于是控制器的View就显示在屏幕上。

# keyWindow和自定义window的使用
[iOS开发·UIWindow与视图层级调整技巧（makeKeyWindow，resignKeyWindow，makeKeyAndVisible，keyWindow，windowLevel，UIWind - 云+社区 - 腾讯云](https://cloud.tencent.com/developer/article/1332201)

设置keyWindow与否并不影响视图层级显示，keyWindow仅来接收键盘及其它非触摸事件。

app的keyWindow与是否在最上层显示没有任何关系。比如，你如果想通过[[UIApplication sharedApplication] keyWindow]获取正在显示的UIWindow是不准确的

Windows的显示通过hidden属性控制，并且不需要用自动布局计算位置，系统会自动计算位置信息，并做横竖屏旋转处理。

windowLevel数值越大的显示在窗口栈的越上面，同一windowLevel上显示最后hidden=false的window。

# CALayer动画的暂停/恢复？
![](UI%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98%E8%AE%B0%E5%BD%95/538FEB74-C539-499D-B19C-38C36C75597E.png)

# Alpha、Opacity、opaque的区别
Opacity：layer的不透明度，1为不透明，会影响sublayer。

alpha：UIView的不透明通道，1为不透明，会影响subview。
opaque：UIView图层颜色混合优化开关，默认为true表示不透明，渲染时会进行优化（必须应该保证为true时alpha为1）。
hidden：隐藏视图。

# iOS动画、模型、呈现树
P就像是瞎子，M就像是瘸子，瞎子背着瘸子，瞎子每走一步（也就是每次屏幕刷新的时候）都要去问瘸子应该怎样走。
而当一个CAAnimation（以下称为A）加到了layer上面后，A就把M从P身上挤下去了。现在P背着的是A，P同样在每次屏幕刷新的时候去问他背着的那个家伙，A就指挥它从fromValue到toValue来改变值。
动画结束后，A会自动被移除，这时P没有了指挥，就只能大喊“M你在哪”，M说我还在原地没动呢，于是P就顺声回到M的位置了，这就是为什么动画结束后我们看到这个视图又回到了原来的位置。

解决办法是动画后设置M为动画后的状态。

# Xib原理和使用
[一天一点xib:10说说原理、优化方面的东西吧 - 简书](https://www.jianshu.com/p/2f9e71ef7f52)
[xib 原理、嵌套、可视化、继承 - 简书](https://www.jianshu.com/p/50ee2ce6d513)

initWithCoder：反序列化Nib文件时调用，
awakeFromNib：initWithCoder之后，SB或者XIB上的IBOutlet已经连接准备好时才调用，awakeFromNib后才能拿到XIB上的实例对象。

如果将顶级对象view的类名绑定为CustonView，那么在xib解档过程中，对顶级对象view解档后，返回的就是CustonView这个实例。

如果将File Owner的类名绑定为CustonView，那么CustonView类是CustonView.xib的管理者，CustonView.xib只负责给CustonView类提供打包好的view。也就是loadNibNamed: owner:self options:这个方法返回的数组中，你拿到的就是UIView的实例。

绑顶级对象view的类名：只能用于代码创建此类
绑File Owner的类名：代码创建此类、嵌套到其他xib中都可以（常用）