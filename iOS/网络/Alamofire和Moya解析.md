# Alamofire和Moya解析 
#iOS知识点/第三方库  #计算机网络/ios相关

 [Alamofire（1）— URLSession必备技能](https://juejin.im/post/6844903913401892878) 
 [Alamofire（2）— 后台下载](https://juejin.im/post/6844903918145634311) 
 [Alamofire（3）— Request](https://juejin.im/post/6844903920091807752) 
 [Alamofire（4）— 你需要知道的细节](https://juejin.im/post/6844903921379442695) 
 [Alamofire（5）— Response](https://juejin.im/post/6844903923224936455) 
 [Alamofire（6）— 多表单上传](https://juejin.im/post/6844903933178019847) 
 [Alamofire（7）— 安全认证](https://juejin.im/post/6844903933698113543) 
 [Alamofire（8）— 终章（网络监控&通知&下载器封装）](https://juejin.im/post/6844903934155309069) 

[Alamofire源码解读系列(十二)之请求(Request) - 马在路上 - 博客园](https://www.cnblogs.com/machao/p/6856603.html)
[Alamofire官方文档的翻译](https://github.com/Lebron1992/learning-notes/blob/master/docs/alamofire5/02%20-%20Alamofire%205%20的使用%20-%20基本用法.md)

# Alamofire
## 整体理解
核心作用就是帮我们便利的去进行网络请求调度，提供了很多和网络通信相关的辅助功能，便于我们更方便的进行网络请求和结果处理，比如：
::1. 对请求的参数编码、::
::2. 响应的验证、::
::3. 序列化、::
::4. 请求到响应中的错误处理、::
::5. 上传下载进度的监听、::
::6. 网络状态管理和安全策略管理。::

::不用再去自己创建并持有session及实现其代理，只需要传入请求的URL及响应的回调，就可以完成一次请求。::

![](Alamofire%E5%92%8CMoya%E8%A7%A3%E6%9E%90/B6E3FCA3-C95E-4661-A231-F7CAFEFF29B0.png)

## 构成模块
1. API接口部分：整个框架的表现层，提供请求调用的API声明
2. 请求模块：构建请求相关类、参数编码、自定义表单、服务器验证等
3. 响应模块：构建响应相关类、响应序列化、验证、结果表示和错误处理等
4. 底层Session会话模块：NSURLSession的管理和交互相关处理
5. 其他部分：网络状态监听、消息通知、时间描述相关框架辅助

### 接口部分
Alamofire.swift
整个框架的表现层，提供请求调用的API声明，主要包括：
1. request：简单URL请求和允许自定义配置方法参数的2种请求形式。
2. downoad：断点下载、简单URL下载、和允许自定义配置方法参数信息这3种下载方式。
3. upload：以文件、data、stream、多种格式的数据类型这4种上传形式。

### 请求模块
Request.swift // 请求类，用于构建请求
ParameterEncoding.swift // 参数编码
MultipartFormData.swift // 自定义表单类
ServerTrustPolicy.swift // 服务器验证

### 响应模块
Response.swift // 相应类，用于构建响应
ResponseSerialization.swift // 响应数据序列化
Validation.swift // 响应数据验证
Result.swift // 请求结果表示
AFError.swift // 错误类型

### 底层Session会话模块
SessionManager.swift // 请求session的管理类，底层使用NSURLSession实现
SessionDelegate.swift // 请求Session的代理对象，主要实现NSURLSession的代理方法以及回调闭包
TaskDelegate.swift // 请求Task任务的代理对象，主要实现NSURLDataTask的代理方法
DispatchQueue+Alamofire.swift //GCD扩展，定义多个不同功能的队列

### 网络状态监听、消息通知
NetworkReachabilityManager.swift // 网络状态监听类
Notifications.swift // 定义通知
Timeline.swift //描述请求有关的时间 结构体

## 请求过程的理解
![](Alamofire%E5%92%8CMoya%E8%A7%A3%E6%9E%90/A3036AEA-50E3-4660-A5ED-0FDFE044A926.png)

# Moya
[深入理解Moya设计 - 个人文章 - SegmentFault 思否](https://segmentfault.com/a/1190000012997081)
[Swift-Moya 源码解析](https://juejin.cn/post/6899362830114357256)

## 理解：管理网络请求的抽象层
POP网络：面向协议编程的网络能够大大降低耦合度！网络层下沉，业务层上浮。中间利用 POP网络的Moya 隔开。如果你的项目是 RxSwift 函数响应式的也没有关系！因为有 RxMoya

是一个网络抽象层，它在底层将Alamofire进行封装，对外提供更简洁的接口供开发者调用。Moya可以说是非常Swift式的一个框架，最大的优点是使用面向协议的思想，让使用者能以搭积木的方式配置自己的网络抽象层。提供了插件机制，在保持主干网络请求逻辑的前提下，让开发者根据自身业务需求，定制自己的插件，在合适的位置加入到网络请求的过程中。

## 用法：
[Swift - 网络抽象层库Moya的使用详解1（安装配置、基本用法）](https://www.hangge.com/blog/cache/detail_1797.html)

1. 创建枚举，遵守TargetType协议，实现规定的属性。
2. 初始化provider = MoyaProvider<Myservice>()
3. 调用provider.request，在闭包里处理请求结果。
![](Alamofire%E5%92%8CMoya%E8%A7%A3%E6%9E%90/F877AA9C-C111-4323-94EE-9A9C7F3C9381.png)
![](Alamofire%E5%92%8CMoya%E8%A7%A3%E6%9E%90/74ABD718-7946-4C6C-998F-100EF24F9CEE.png)
![](Alamofire%E5%92%8CMoya%E8%A7%A3%E6%9E%90/9446D627-D8C3-49DD-8CEA-AC8F58ED2CDD.png)


