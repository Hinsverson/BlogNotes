# TableView相关
#iOS知识点/UI相关

# TableView的优化
1. 最基本的肯定Cell复用，没什么好说的。
2. 设置预估高度能避免cellForRow之前多次heightForRow的调用计算。
3.  复杂页面考虑进行cell的高度缓存，当平常使用UITableViewAutomaticDimension的方式撑开Cell时，可以在willDisplayCell时获取到已经被系统正确计算过的cell的size，把它存下来，一般就存在对应的model内，然后在heightForRow方法里根据indexpath取model，如果model内有值则直接使用，如果没有才返回UITableViewAutomaticDimension让系统计算。
4. 设置代理方法后，相同的属性设置会失效，固定情况尽量用属性设置（减少代理方法的调用次数）。
5. 大量图片的场景一定要做异步解码绘制，主线程刷新界面。
6. 刷新时进行diff局部刷新，而不是reloadAll。

[iOS-谈一谈自适应Cell的高度缓存 - 简书](https://www.jianshu.com/p/684d897be084)


# TabView重用机制的理解
滑动TableView时，并不会无限的创建每一个Cell。当某个cell移出窗口时，UITableView会将Cell放入一个重用池中等待重用。当需要出现新Cell时，从重用池中拿取并对它设置数据源进行显示。

## 实现TableView重用池的思路
1. 抽象一个重用池管理类，声明2个集合，用来描述一个待使用的View池和正在使用的View队列，并定义取View、添加View、重置重用池View（把所有使用中的View从正在使用加入到待使用的View池）的操作。
2. 定一个数据源协议，用来为TableView提供数据。
3. 子类一个UITableView类，声明弱引用的数据源delegate和一个强引用的重用池管理对象，并重写reloadData方法。
4. 在reloadData内部先用数据源方法获取数据，根据场景添加判断逻辑。再从重用池中去取View，如果没有取到就创建View并加入到重用池。

## 重用时数据错乱处理
重写Cell的prepareForReuse方法，在每次重用时清除原来的数据源。

## 重用Cell的获取方式和区别
1. 当使用register方法注册Cell后，重用Cell为空判断和创建系统会自动执行，需要使用Cell时直接dequeueReusableCellWithIdentifier拿到即可。

2. 没有注册Cell的话，需要判断在dequeueReusableCellWithIdentifier后判断取出的Cell是否为空，为空则手动创建新的Cell。

> dequeueReusableCellWithIdentifier：不需要注册  
> dequeueReusableCellWithIdentifier indexPath: 必须进行cell的注册  

# 多线程数据源同步问题
多线程共同访问数据源会涉及到数据源的同步问题，原则上保证在UI操作的更新后，下一次更新前刷新数据源即可，主要有2种解决思路：

## 并发访问，数据拷贝
![](TableView%E7%9B%B8%E5%85%B3/5BDE825C-27EC-43C0-BB39-F2A75E5ACC42.png)
利：由于并发操作，所以UI上无延迟。
弊：每次的操作做记录以保证同步，同时需要内存的开销来拷贝数据。

## 串行访问
![](TableView%E7%9B%B8%E5%85%B3/45676F38-D1ED-4305-A91F-5CF8B54AF3D8.png)
利：不需要内存的开销来拷贝数据。
弊：由于UI删除、刷新和数据删除在一个串行队列，可能存在一定延迟。

# 重点理解
reloadData：该方法刷新可见的所有Cell，是runloop异步的。
reloadDataIndexPath：该方法刷新指定的Cell，也是runloop异步的。
cellForRow在willdisplay之前调用，willdisplay时高度，内容都应该确定了。

## UITableViewAutomaticDimension原理
系统调用systemLayoutSizeFittingSize方法获取进行内容填充后的cell的size，但是计算结果并不回缓存，意味着每一次cell来回出现都需要重新计算。

# 属性、方法生命周期
初始化方法：
initWithFrame：—————设置表的大小和位置
initWithFrame：style---------设置表的大小，位置和样式（组，单一）
setEditing：—————表格进入编辑状态，无动画
setEditing： animated：————表格进入编辑状态，有动画
reloadData---------------刷新整个表视图
reloadSectionIndexTitles--------刷新索引栏
numberOfSections-----------获取当前所有的组
numberOfRowsInSection：————获取某个组有多少行
rectForSection：—————获取某个组的位置和大小
rectForHeaderInSection：————获取某个组的头标签的位置和大小
rectForFooterInSection：—————获取某个组的尾标签的位置和大小
rectForRowAtIndex：—————获取某一行的位置和大小
indexPathForRowAtPoint-------------点击某一个点，判断是在哪一行上的信息。
indexPathForCell：——————获取单元格的信息
indexPathsForRowsInRect：————在某个区域里会返回多个单元格信息
cellForRowAtIndexPath：——————通过单元格路径得到单元格
visibleCells-----------返回所有可见的单元格
indexPathsForVisibleRows--------返回所有可见行的路径
headerViewForSection：————设置头标签的视图
footerViewForSection；—————设置尾标签的视图
beginUpdates--------只添加或删除才会更新行数
endUpdates---------添加或删除后会调用添加或删除方法时才会更新
insertSections：withRowAnimation：—————插入一个或多个组，并使用动画
insertRowsIndexPaths：withRowAnimation：———插入一个或多个单元格，并使用动画
deleteSections：withRowAnimation：————删除一个或多个组，并使用动画
deleteRowIndexPaths：withRowAnimation：————删除一个或多个单元格，并使用动画
reloadSections：withRowAnimation：————更新一个或多个组，并使用动画
reloadRowIndexPaths：withRowAnimation：——————更新一个或多个单元格，并使用动画
moveSection：toSection：——————移动某个组到目标组位置
moveRowAtIndexPath：toIndexPath：—————移动个某个单元格到目标单元格位置
indexPathsForSelectedRow----------返回选择的一个单元格的路径
indexPathsForSelectedRows---------返回选择的所有的单元格的路径
selectRowAtIndexPath：animation：scrollPosition---------设置选中某个区域内的单元格
deselectRowAtIndexPath：animation：—————取消选中的单元格

重用机制
dequeueReusableCellWithIdentifier：————获取重用队列里的单元格

UITableViewDataSource代理方法：
方法：
numberOfSectionsInTableView：——————设置表格的组数
tableView：numberOfRowInSection：—————设置每个组有多少行
tableView：cellForRowAtIndexPath：————设置单元格显示的内容
tableView：titleForHeaderInSection：————设置组表的头标签视图
tableView：titleForFooterInSection：—————设置组表的尾标签视图
tableView：canEditRowAtIndexPath：————设置单元格是否可以编辑
tableView：canMoveRowAtIndexPath：————设置单元格是否可以移动
tableView：sectionIndexTitleForTableView：atIndex：———设置指定组的表的头标签文本
tableView：commitEditingStyle：forRowAtIndexPath：—————编辑单元格（添加，删除）
tableView：moveRowAtIndexPath：toIndexPath-------单元格移动

UITableViewDelegate代理方法：

tableView：willDisplayCell： forRowAtIndexPath：—————设置当前的单元格
tableView： heightForRowAtIndexPath：—————设置每行的高度
tableView：tableView heightForHeaderInSection：—————设置组表的头标签高度
tableView：tableView heightForFooterInSection：——————设置组表的尾标签高度
tableView： viewForHeaderInSection：—————自定义组表的头标签视图
tableView： viewForFooterInSection： —————自定义组表的尾标签视图
tableView： accessoryButtonTappedForRowWithIndexPath：—————设置某个单元格上的右指向按钮的响应方法
tableView： willSelectRowAtIndexPath：—————获取将要选择的单元格的路径
tableView： didSelectRowAtIndexPath：—————获取选中的单元格的响应事件
tableView： tableView willDeselectRowAtIndexPath：——————获取将要未选中的单元格的路径
tableView： didDeselectRowAtIndexPath：—————获取未选中的单元格响应事件

## 执行顺序
第一轮：
1、numberOfSectionsInTableView ：假如section=2，此函数只执行一次，假如section=0，下面函数不执行，默认为1
2、heightForHeaderInSection ，执行两次，此函数执行次数为section数目
3、heightForFooterInSection ，函数属性同上，执行两次
4、numberOfRowsInSection ，此方法执行一次
5、heightForHeaderInSection ，此方法执行了两次
6、heightForFooterInSection ，此方法执行两次
7、numberOfRowsInSection，执行一次
8、heightForRowAtIndexPath ，行高，先执行section=0，对应的row次数
第二轮：
1、numberOfSectionsInTableView ，一次
2、heightForHeaderInSection ，section次数
3、heightForFooterInSection ，section次数
4、numberOfRowsInSection ，一次
5、heightForHeaderInSection ，执行section次数
6、heightForFooterInSection，执行section次数
7、numberOfRowsInSection，执行一次
8、heightForRowAtIndexPath，行高，先执行一次
9、cellForRowAtIndexPath

10、willDisplayCell