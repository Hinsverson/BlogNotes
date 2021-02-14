# iOS应用架构
#iOS知识点/设计模式和架构

本质只是数据来驱动 GUI 的开发模式

# MVC
![](iOS%E5%BA%94%E7%94%A8%E6%9E%B6%E6%9E%84/1D8BF83B-E654-4C33-8678-964AD1E23416.png)
TableView就是典型的MVC架构。

优点：View和model可以重复利用
缺点：controller会越来越臃肿

## MVC 改进
Controller只做以下事情：
* 在初始化时，构造相应的 View 和 Model。
* 监听 Model 层的事件，将 Model 层的数据传递到 View 层。
* 监听 View 层的事件，并且将 View 层的事件转发到 Model 层。

其他逻辑，拆到单独类中管理实现，比如：
1. 网络逻辑
2. 交互逻辑

# MVVM：
![](iOS%E5%BA%94%E7%94%A8%E6%9E%B6%E6%9E%84/6F44E58F-98B9-4027-A25D-4FEFFA528FC9.png)
![](iOS%E5%BA%94%E7%94%A8%E6%9E%B6%E6%9E%84/CD2D3296-5363-420E-9D32-0D94D1A1C37D.png)
View <-> C <-> ViewModel <-> Model
严格来说MVVM其实是MVCVM，Controller夹在View和ViewModel之间做的其中一个主要事情就是将View和ViewModel进行绑定。
在逻辑上，Controller知道应当展示哪个View，Controller也知道应当使用哪个ViewModel，然而View和ViewModel它们之间是互相不知道的。

## 绑定通信的方式：
Rxswift、KVO，Notification，block，delegate和target-action等

## 理解：只是写UI的一种规范形式
![](iOS%E5%BA%94%E7%94%A8%E6%9E%B6%E6%9E%84/34318048-4FCD-4047-9A9B-50948CFC876D.png)
![](iOS%E5%BA%94%E7%94%A8%E6%9E%B6%E6%9E%84/B5AD0CC8-FE30-4B6F-8DEB-F2E2BD7D3E61.png)
MVVM的核心思想是通过VM来处理UI状态并维护展示的model数据，通过Rxswift双向绑定实现mode和view的自动变化响应和更新。

1. Controller只负责view和VM的状态、数据绑定
2. VM只负责处input view的状态变化，并output mode数据给到view。
3. VM直接从service、DB、neteork层面获取mode。
4. service、DB、neteork层通过回调、delegate、通知和vm盲通信。


# 3层架构
![](iOS%E5%BA%94%E7%94%A8%E6%9E%B6%E6%9E%84/3544BB42-5EFC-471D-A27C-A06139EE116A.png)

# 4层架构
界面层、业务层、网络层、本地数据层
![](iOS%E5%BA%94%E7%94%A8%E6%9E%B6%E6%9E%84/D0B06BB1-BCC2-4C08-8185-FAD06A7B86C6.png)

