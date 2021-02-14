# iOS绘制相关
#iOS知识点/UI相关

# View绘制流程
![](iOS%E7%BB%98%E5%88%B6%E7%9B%B8%E5%85%B3/2EC6F3F6-08EB-4BE8-B619-222A32275837.png)
1. view.setNeedDisplay：内部会调用Layer.setNeedDisplay
2. layer.setNeedDisplay：将当前图层标记为需要重新绘制
3. layer.display：下一次loop循环到来时系统调用，内部会首先进行是否响应代理方法displayLayer的判断，如果代理View实现了这个方法，就可以拿到CALayer直接设置content寄宿图完成异步绘制，如果没有实现，就进入系统绘制的流程。

## 系统绘制阶段
![](iOS%E7%BB%98%E5%88%B6%E7%9B%B8%E5%85%B3/D27A8453-CF1C-4CF8-8010-594CB3B53CDD.png)
4. 首先Layer会创建一个合适尺寸的空寄宿图（尺寸由bounds和contentsScale决定，这也是平常大量重写View的drawRect方法并设置背景颜色布满画板时，内存会增大的原因（待考证））和一个Core Graphics的绘制上下文环境，为绘制寄宿图做准备。
5. 然后Layer会判断是否有代理，如果没有就会调用自身的drawInContext方法进行寄宿图绘制得到CGImage对象，设置Content后交给GPU渲染最后显示。
6. 如果Layer有代理存在，会传入绘制的上下文并调用代理实现的drawLayer：inContext方法来进行绘制，最后同样得到CGImage对象后设置Content交给GPU渲染最后显示。
7. UIView作为layer代理实现的drawLayer：inContext方法内部先把获取到的绘制上下push到系统上下文栈顶（这也是drawRect重绘View时通过UIGraphicsGetCurrentContext能拿到当前上下文的原因），并调用自身drawRect方法来完成绘制任务，结束时会进行上下文的pop操作。

## 异步绘制
一般就是把需要绘制的图形，提前在子线程处理好。将处理好的图像数据直接返给主线程使用，这样可以降低主线程的压力。
layer中有一个属性contents，绘制完成从画布中导出图片（CGImage），再把图片赋值给layer.contents就完成了显示。

### 异步绘制的时机
主要通过实现 layer代理的 displayLayer这个方法实现

# 视图绘制优化
1. 最好的方式是实现displayLayer代理方法，走异步的绘制流程而不是系统绘制流程，在子线程绘制好图形后，返回主线程设置layer的content属性。
2. 复杂的图形绘制，考虑CAShapeLayer来完成而不是layer，它是矢量图形而不是bitmap，也不会创建空寄宿图，并且使用的是GPU来进行绘制加速，而不是CPU。

# UIView自定义绘制注意点
![](iOS%E7%BB%98%E5%88%B6%E7%9B%B8%E5%85%B3/096A9BE5-17F1-4816-8DC4-AED25169A369.png)
1. drawRect系统调用是在控制器ViewWillAppear方法后。
2. drawRect主要作用是允许我们在系统绘制流程之上进行额外的自定义绘制。（可以看到我们是在初始化时发生重绘前设置的backgroundColor，重绘时没有对画布填充任何背景色，重绘结束后依然改变了背景颜色）

![](iOS%E7%BB%98%E5%88%B6%E7%9B%B8%E5%85%B3/1454F0A5-ABD7-46FB-B884-E694D870F5AB.png)
3. setNeedDisplay的本质是异步为当前视图打上需绘制标记，下一次loop到来时会触发drawRect进行绘制，初始化时会调用2次。
4. 在drawRect中单独设置UIView显示相关backgroundColor是无效的。原因在于你设置的是下一次loop循环所做的事情而不是本次loop绘制任务。

> 虽然会为background属性赋值，并且重写了drawRect后系统会调用setNeedDisplay方法打上需绘制的标记，但不会再次触发drawRect，即使使用layoutIfNeed手动立即刷新也不会有任何效果。  
>   
> 正确的设置应该是在init的时候或者外部其他地方设置backgroundColor。  

5. layoutIfNeed的本质是立即调用drawRect处理绘制任务或是进行视图布局，如果已经在绘制过程中了，该方法会失效，因为没有needdisplay的视图了。

![](iOS%E7%BB%98%E5%88%B6%E7%9B%B8%E5%85%B3/2612B9A8-A755-4371-8AD0-53FF0BE79921.png)
6. drawRect重绘时既没有为画布填充背景颜色，也没有在任何其他地方设置View的backgroundColor，背景颜色会自动变黑。（因为CG画图画布的默认颜色就是黑色，同时可以看出在drawRect中设置backgroundColor依然没有显示）

![](iOS%E7%BB%98%E5%88%B6%E7%9B%B8%E5%85%B3/0A0DCD9A-549D-4F05-9689-CE2188332940.png)
7. drawRect重绘时为画布填充了背景颜色后，显示时会覆盖掉预先在其他地方设置的背景颜色。

![](iOS%E7%BB%98%E5%88%B6%E7%9B%B8%E5%85%B3/266304D7-3BEB-4CC3-B3AA-47F5543FF390.png)
![](iOS%E7%BB%98%E5%88%B6%E7%9B%B8%E5%85%B3/D615DF9D-B0D2-4DC0-8947-3CA5ADD2D308.png)
8. 这里在View显示2秒后手动调用setNeedDisplay，会在下一次loop到来时触发drawRect，由于先前初始化重绘中已经设置过背景为蓝色，所以本次loop任务会进行背景色的绘制（整个过程是刚显示时为黄色，2秒后为变为蓝色）


# CGContextSaveGState和UIGraphicsPushContext的区别
[CoreGraphics之CGContextSaveGState与UIGraphicsPush… - 简书](https://www.jianshu.com/p/be38212c0f79)

CGContextSaveGState：是压栈上下文的绘制状态，保存一些设置，便于之后恢复（CGContextRestoreGState）。
UIGraphicsPushContext：压栈当前的绘制对象context，使它成为当前上下文。
![](iOS%E7%BB%98%E5%88%B6%E7%9B%B8%E5%85%B3/971628C8-425A-4B7E-8860-D502F7149C9D.png)
![](iOS%E7%BB%98%E5%88%B6%E7%9B%B8%E5%85%B3/091E374D-0819-4DA4-A7B1-C22AFB68B165.png)
可以看到，**CGContextSaveGState**存储下来了当前红色和默认的线条状态，然后切换颜色到黄色和10粗度的线条画圈，然后在**CGContextRestoreGState**恢复到了红色和默认的线条状态进行画圈，这个就是存储当前绘制状态的意思*

![](iOS%E7%BB%98%E5%88%B6%E7%9B%B8%E5%85%B3/675AE283-F769-4AA3-8154-62A19BAE6771.png)
![](iOS%E7%BB%98%E5%88%B6%E7%9B%B8%E5%85%B3/6204DA29-37E2-426A-9896-6646E884C7E7.png)
如果你将UIGraphicsPushContext(ctx)，与UIGraphicsPopContext()，删去的话，是无法进行绘制的。
原因是，UIKit的绘制必须在当前的上下文中绘制，而UIGraphicsPushContext可以将当前的参数context转化为可以UIKit绘制的上下文，进行绘制图片。


# CoreAnimation
view作为layer的delegate，layer通过询问view的actionForLayer:forKey:方法来获得自己应该执行的CAAction。

UIView动画其实就是对Core Animation的一种封装，内部会转化为Core Animation交给layer。

# CAShapeLayer优点
[放肆的使用UIBezierPath和CAShapeLayer画各种图形 - 简书](https://www.jianshu.com/p/c5cbb5e05075)

* 渲染快速。CAShapeLayer使用了硬件加速（GPU），绘制同一图形会比用Core Graphics（CPU）快很多。
* 高效使用内存。一个CAShapeLayer不需要像普通CALayer一样创建一个寄宿图形，所以无论有多大，都不会占用太多的内存。
* 不会被图层边界剪裁掉。一个CAShapeLayer可以在边界之外绘制。你的图层路径不会像在使用Core Graphics的普通CALayer一样被剪裁掉（如我们在第二章所见）。
* 不会出现像素化。当你给CAShapeLayer做3D变换时，它不像一个有寄宿图的普通图层一样变得像素化。
