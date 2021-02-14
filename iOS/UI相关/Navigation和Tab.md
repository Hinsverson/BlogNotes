# Navigation和Tab
#iOS知识点/UI相关

[iOS系统中导航栏的转场解决方案与最佳实践 - 美团技术团队](https://tech.meituan.com/2018/10/25/navigation-transition-solution-and-best-practice-in-meituan.html)
[iOS导航栏之UINavigationController,UINavigationbar和UINavigationItem的关系 - 简书](https://www.jianshu.com/p/6803ead6df48)
[iOS-UINavigationController官方文档分析大总结 | Lianpeng’s Sky](https://xulianpeng.github.io/2016/07/28/iOS-UINavigationController%E5%AE%98%E6%96%B9%E6%96%87%E6%A1%A3%E5%88%86%E6%9E%90%E5%A4%A7%E6%80%BB%E7%BB%93/)
[非常详细的 navigationController 的使用_FFCoding的博客-CSDN博客](https://blog.csdn.net/FFCoding/article/details/73610696)
[UINavigationController与UINavigationBar详解 - 简书](https://www.jianshu.com/p/b2ae4d211499)
[NavigationController 已经洗干净了, 就等你来 - iOS](https://juejin.im/entry/6844903471028633607)

# 系统UINavigationController的理解
![](Navigation%E5%92%8CTab/CD480A5F-5FF4-4AA0-95FE-3A50A649D02C.png)
![](Navigation%E5%92%8CTab/152A1DB7-2BFB-4229-AA6A-637FFA23B6A2.png)

## UINavigationController
作为UINavigationBar和UIToolBar的代理，负责创建、配置及管理Bar对象，并分别通过ViewController的UINavigationItem和UIToolbarItems作为模型数据源在导航过程中渲染并展示内容到UINavigationbar和UIToolBar上。其中Toolbar默认隐藏。

## UINavigationBar
是一个在层级结构中起导航作用的视觉控件，结构如下，虽然继承自UIView，但是其并非通过addSubview方法来添加子视图,，而是通过其管理的UINavigationItem堆栈来决定展示在UINavigationBar中的内容。
![](Navigation%E5%92%8CTab/B5848F3C-18E9-4B7A-BDFD-6BCDA42D9258.png)
![](Navigation%E5%92%8CTab/274D93D9-55ED-4B69-8654-20CD40591F14.png)
1. 可以利用系统提供的方法向UINavigationBar的UINavigationItem堆栈中Push、Pop一个UINavigationItem，也可以直接设置UINavigationItem堆栈中的全部UINavigationItem。
2. UINavigationController就是通过实现UINavigationBar的代理方法和Push、Pop操作来管理视图的展示。
3. UINavigationBar类对象的appearance可以对所有UINavigationBar进行统一风格设置。

* UITabBar 上展示的是: UITabBarItem( 其类继承关系为–> UIBarItem –> NSObject)
* UIToolBar 上展示的是: UIToolBarItem (其类继承关系为–>UIBarButtonItem –> UIBarItem–> NSObject)
* UINavigationBar 上展示的是: UINavigationItem(包括:titleView leftBarButtonItem leftBarButtonItems rightBarButtonItem rightBarButtonItems) (其类继承关系为–> UIBarButtonItem –> UIBarItem–> NSObject)

设置UInavigationController堆栈中的viewControllers时，当animated参数为YES时，系统会根据如下三种情况选择采用何种动画形式：
	* 当数组中的最后一个元素在当前堆栈中, 且是位于栈顶, 则采用无动画形式
	* 当数组中的最后一个元素在当前堆栈中, 且不是位于栈顶, 则采用Pop动画形式
	* 当数组中的最后一个元素不在当前堆栈中, 则采用Push动画形式

# 技巧
[UINavigationBar 使用总结 - 简书](https://www.jianshu.com/p/f0d3df54baa6)

## 实现全屏幕的侧滑返回手势
新加入一个手势，拿到系统侧滑返回手势所在的View，action、target对象进行替换。
``` objc
@interface LPNavigationController : UINavigationController

@property(nullable, nonatomic, readonly) UIGestureRecognizer *fullInteractivePopGestureRecognizer;

@end

@interface LPNavigationController () <UIGestureRecognizerDelegate>

@property(nullable, nonatomic, readwrite) UIGestureRecognizer *fullInteractivePopGestureRecognizer;

@end

@implementation LPNavigationController

- (void)viewDidLoad
{
    [super viewDidLoad];
    // Do any additional setup after loading the view.
    
    self.interactivePopGestureRecognizer.enabled = NO;
    
    UIView *view = self.interactivePopGestureRecognizer.view;
    id target = self.interactivePopGestureRecognizer.delegate;
    SEL action = NSSelectorFromString(@"handleNavigationTransition:");
    
    self.fullInteractivePopGestureRecognizer = [[UIPanGestureRecognizer alloc] initWithTarget:target action:action];
    self.fullInteractivePopGestureRecognizer.delaysTouchesBegan = YES;
    self.fullInteractivePopGestureRecognizer.delegate = self;
    [view addGestureRecognizer:self.fullInteractivePopGestureRecognizer];
}

- (BOOL)gestureRecognizerShouldBegin:(UIGestureRecognizer *)gestureRecognizer
{
    // 左滑时可能与UITableView左滑删除手势产生冲突
    CGPoint translation = [(UIPanGestureRecognizer *)gestureRecognizer translationInView:gestureRecognizer.view];
    if (translation.x <= 0)
    {
        return NO;
    }
    
    // 跟视图控制器不响应手势
    return ([self.viewControllers count] == 1) ? NO : YES;
}

@end
```


# UITabBarController
[你真的了解UITabBarController吗？ - 踏浪帅 - 博客园](https://www.cnblogs.com/wujy/p/5832714.html)

* UITabBarController和UINavigationController一样是用来管理试图控制器的，与导航控制器不同，tabBarController控制器使用数组而不是栈来管理子控制器的，并且子试图之间是平等关系。
* UITabBarController作为UITabBar的代理，负责创建、配置及管理Bar对象，并分别通过ViewController的UITabBarItem作为模型数据源在导航过程中渲染并展示内容到UITabBar上。
* UITabBar通过数组的形式管理UITabBarItem。


# 生命周期
![](Navigation%E5%92%8CTab/7A74339C-799C-4BAD-AB80-6B9CC89E862B.png)

## NavigationController
从A Push B：
```
A: viewWillDisappear
B: viewWillAppear
A: viewDidDisappear
B: viewDidAppear
```
从A Pop B：
```
B: viewWillDisappear
A: viewWillAppear
B: viewDidDisappear
A: viewDidAppear
```

## UITabBarController
从A切换到B：
```
B: viewWillAppear
A: viewWillDisappear
A: viewDidDisappear
B: viewDidAppear
```
从B切换到A：
```
A: viewWillAppear
B: viewWillDisappear
B: viewDidDisappear
A: viewDidAppear
```

