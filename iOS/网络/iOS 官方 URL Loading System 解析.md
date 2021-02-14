# iOS 官方 URL Loading System 解析
#计算机网络/ios相关

官方文档：[URL Loading System](https://developer.apple.com/documentation/foundation/url_loading_system)
对整个iOS网络层有一个总体的认知参考：[iOS 网络：URL Loading System官方文档详细的翻译](https://zhuanlan.zhihu.com/p/345302514)

[URL Loading System 概览](https://juejin.cn/post/6844903555858448391)
 [iOS 网络：URL Loading System（译）](https://zhuanlan.zhihu.com/p/345302514) 
 [iOS 网络：网络库概览](https://zhuanlan.zhihu.com/p/341686279) 
 [iOS 网络：Handling an Authentication Challenge（译）](https://zhuanlan.zhihu.com/p/343870368) 
 [iOS 网络：Performing Manual Server Trust Authentication（译）](https://zhuanlan.zhihu.com/p/343916763) 

![](iOS%20%E5%AE%98%E6%96%B9%20URL%20Loading%20System%20%E8%A7%A3%E6%9E%90/CB3E7376-C03F-4700-984B-56320CC29985.png)

URL Loading System 核心在于基于通过 URL 这个类来访问网络资源，所支持的协议有：【ftp://】、【http://】、【https://】、【file://】、【data://】，另外它还支持代理服务和网关处理。除了加载 URL 的核心类NSURLSession 之外，其他相关辅助API类分为 5 类（如图所示）：
	* 协议支持（protocol support）
	* 认证和证书（authentication and credentials）
	* cookie 存储（cookie storage）
	* 请求配置（configuration management）
	* 缓存管理（cache management）

# 网络活动过程
使用URLSession来进行HTTP/HTTPS请求的时候，实际过程理解如下：
![](iOS%20%E5%AE%98%E6%96%B9%20URL%20Loading%20System%20%E8%A7%A3%E6%9E%90/621D73FE-1D93-4310-A4C1-47A7DC18251E.png)

# 基本使用理解
### 通过URLRequest封装请求
URLRequest：这个类用于对请求的封装，封装了加载URL请求的两个基本数据元素：
	* 用于数据请求的地址URL。
	* 用于请求过程中的配置信息（请求报文的字段构造、本次请求的缓存策略、请求控制与服务相关设置，具体细节参考[URLRequest详解](https://blog.csdn.net/die_word/article/details/82701545)）。

### 配置URLSession并发起请求
1. 整体API的设计，首先使用URLSessionConfiguration提供的三种类方法创建3种不同的SessionConfiguration实例对象，通过它来配置并创建不同的Session对象，不同的Session对象具备不同的网络活动行为，重要区别如下：
	* Default sessions：支持磁盘缓存，并且会把凭证（credentials）保存到 keychain 中。
	* Ephemeral sessions：不会存储任何数据到磁盘上，所有的缓存、凭证都只会随着 session 保存在内存中，而且一旦这个 session 被清除了，这些缓存同时也会被清除。
	* Background sessions：跟default sessions的行为基本类似，但是还支持后台驻留进程的数据传输，也就是说，当 app 被挂起时，还可以在后台继续传输数据。

URLSessionConfiguration还可以进行更多的详细配置：[详细参考](https://juejin.cn/post/6844903913401892878)
	* 设置Cookie政策
	* 安全策略
	* 设置缓存策略
	* 支持后台转移
	* 支持自定义协议
	* 支持多路径TCP
	* HTTPShouldUsePipelining（开启管线化，默认为NO）

2. 然后通过URLSession作为工厂（可以复用）创建一个或多个URLSessionTask，进行实际的数据传输任务，Session会保持对Task的一个强引用，直到Task完成或者出错才会释放，所以一般不需要自己持有和维护，具体的子Task类和特点有： 
	* URLSessionDataTask：主要用来发送短暂的、有交互性的请求，下载数据到内存里，数据的格式是NSData。Data tasks 可以分次返回数据，也可以一次性返回所有的数据。
	* URLSessionDownloadTask：获取文件形式的数据，把文件直接download到本地磁盘，支持后台下载，支持断点续传（下载到一半的时候暂停，重启后继续下载，前提下载的服务器支持断点续传）。
	* URLSessionUploadTask：以文件的形式发送数据给服务器，也支持后台上传。
	* URLSessionStreamTask：建立一个端到端的TCP流进行传输。
	* URLSessionWebSocketTask：iOS13新增，处理WebSocket。

URLSessionTask在使用上有2种形式，只关注结果的Block回调和能拿到所有细节过程的Delegate通知（通过设置Session的Delegate）

URLSessionTask本身代表一次网络活动，有具体的State状态（创建后默认挂起），通过启动、取消、挂起等动作来维护，其他属性方法参考[URLSessionTask使用详解](https://leomobiledeveloper.blog.csdn.net/article/details/44565115)，Task的所有属性都支持KVO。

3. 最后通过SessionDelegate把整个网络活动的细节行为暴露出来，提供具体的业务控制处理，使用时系统内部根据不同类型的Task任务来触发不同的SessionDelegate子类回调，按照协议的继承关系主要是如下层面：[详细的各种SessionDelegate事件参考查阅这里）](https://www.jianshu.com/p/4e69bc24795d)
	* URLSessionDelegate：顶层抽象协议类，处理Session层共性事件。
		* URLSessionTaskDelegate：处理所有Task层次的共性事件 。
			* URLSessionDataDelegate：处理 data task、upload task 任务级事件方法。
			* 	URLSessionDownloadDelegate：处理 download task 任务级事件方法。
			* URLSessionStreamDelegate：对应流task。
			* URLSessionWebSocketDelegate：对WebSocket。

### 通过URLResponse处理响应
URLResponse：对于请求到的**描述内容数据的元数据**和**内容数据本身**的封装：
	* 描述内容数据的元数据：描述内容数据的元数据往往是请求协议定义的，大部分协议的元数据包括 MIME type、 expected content length、text encoding (where applicable)以及这个响应对应的 URL。
	* 内容数据本身：也就是要请求的数据。

HTTPURLResponse：URLResponse的子类，代表对于HTTP请求的报文响应封装，主要多了statusCode和allHeaderFields2个部分。

### 一些注意点
* 在 iOS 中，为 background session 创建 upload task 时，系统会将文件复制到临时目录，然后从临时目录上传。
* 某些情况下，configuration 指定的策略会被任务的URLRequest对象重写。默认采用 request 指定的策略，除非 session 的策略更为严格。例如，sesseion configuration 指定禁止使用蜂窝网络，则URLRequest对象不能使用蜂窝网络进行请求。

# 认证和授权处理
官方demo：[iOS 网络：Handling an Authentication Challenge（译）](https://zhuanlan.zhihu.com/p/343870368) 

有些服务器会对某些特定的内容限制访问权限，只对提供了信任证书通过认证的用户提供访问资格。对于 web 服务器来说，受保护的内容被聚集到一个需要凭证才能访问的区域。在客户端上，有时也需要根据凭证来确定是否信任要访问的服务器。URL Loading System 提供了封装凭证（credentials）、封装保护区域（protected areas）和保存安全凭证（secure credential）的类：
	* URLCredential：封装一个含有认证信息（比如用户名和密码等）和持久化存储行为的凭证（credential）。
	* URLCredentialStorage：管理凭证的存储以及 NSURLCredential 和相应的 NSURLProtectionSpace 之间的映射关系。
	* URLProtectionSpace：server上需要认证的区域，定义了认证质询的一系列信息，这些信息确定了应开发者应该如何响应质询，提供怎样的 URLCredential 。
	* URLAuthenticationChallenge：在客户端向有限制访问权限的服务器发起请求时，服务器会询问凭证信息，包括凭证、保护空间、认证错误信息、认证响应等，这个类会将这些信息封装起来。URLAuthenticationChallenge 实例通常被 NSURLProtocol 子类用来通知 URL Loading System 需要认证，以及在 NSURLSession 的代理方法中用来处理认证。

C-S间认证方法和授权的一些基本理解：[参考这里](https://leomobiledeveloper.blog.csdn.net/article/details/45286951)
iOS认证和授权API使用和相关概念，以及alamofire中的认证处理部分分析：[参考这里](https://juejin.cn/post/6844904056767381518#heading-11)

# 缓存管理
官方demo：[iOS 网络：Accessing Cached Data（译）](https://zhuanlan.zhihu.com/p/345858093) 
其他参考：
[iOS NSCache & NSURLCache 机制原理探究 (一)](https://juejin.cn/post/6844903938584477704#heading-10)
[iOS NSCache & NSURLCache 机制原理探究 (二)](https://juejin.cn/post/6844903944918040584)
[iOS网络缓存扫盲篇 - 使用两行代码就能完成80%的缓存需求 - SegmentFault 思否](https://segmentfault.com/a/1190000004356632)

URL Loading System 提供了 app 级别的 HTTP 响应缓存，在使用 URLSession 发起请求时，我们可以通过设置 URLRequest 和 URLSessionConfiguration 的缓存策略（cache policy）来决定是否缓存以及如何处理缓存本次请求到的数据，之后系统会自动处理缓存无需其他额外的设置，当然还可以通过实现 `URLSession:dataTask:willCacheResponse:completionHandler: `方法来针对特定的 URL 设置缓存策略。

如果URLRequest配置使用了缓存，系统默认是把请求到的数据包成CachedURLResponse对象并建立和URLRequest的哈希映射关系，通过URLCache单例对象来管理，而把实际的缓存数据都保存在本地 [数据库](http://lib.csdn.net/base/14) 中：
	* URLCache：通过这个类可以设置缓存大小和位置，以及读取和存储各个请求的 CachedURLResponse。[NSURLCache详细使用参考](https://www.cnblogs.com/feng9exe/p/7203354.html)
	* CachedURLResponse：封装了请求元数据（一个  URLResponse 对象）和实际响应内容（一个 Data 对象）。

不是所有请求的响应都能被缓存起来，URL Loading System 目前只支持对 http 和 https 请求的响应进行缓存（get请求）。

URL Loading System 默认的缓存策略是根据标准http协议规定，服务器通过Cache-Control: max-age字段来告诉URLCache是否需要缓存数据已经怎么缓存，Cache-Control头具有如下选项：
```
Cache-Control头具有如下选项:
	•	public: 指示可被任何区缓存
	•	private
	•	no-cache: 指定该响应消息不能被缓存
	•	no-store: 指定不应该缓存
	•	max-age: 指定过期时间
	•	min-fresh:
	•	max-stable:
```
![](iOS%20%E5%AE%98%E6%96%B9%20URL%20Loading%20System%20%E8%A7%A3%E6%9E%90/56F472D3-6690-4E3F-8916-728C52E5EB41.png)

### 网络请求常见的缓存处理原理
[彻底弄懂HTTP缓存机制及原理](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/chenqf/p/6386163.html) 
1. 文件的缓存控制，一般按照Http协议标准做法有2种方式，iOS里会默认按照http标准进行自动处理（当然前提是server支持）：
	1. Last-Modified/If-Modified-Since：基于时间
		1. Last-Modified 是由服务器返回响应头，标识资源的最后修改时间.
		2. If-Modified-Since 则由客户端发送，标识客户端所记录的，资源的最后修改时间。服务器接收到带有该请求头的请求时，会使用该时间与资源的最后修改时间进行对比，如果发现资源未被修改过，则直接返回HTTP 304而不返回包体，告诉客户端直接使用本地的缓存。否则响应完整的消息内容。
	2. If-None-Match/Etag：基于哈希（推荐server用这种）
		1. Etag 是一个hash值，由服务器发送，告之当资源在服务器上的一个唯一标识符，每次资源变更后会更新。
		2. 客户端请求时，如果发现资源过期(使用Cache-Control的max-age)，发现资源具有Etag声明，这时请求服务器时则带上If-None-Match头，服务器收到后则与资源的标识进行对比，决定返回200还是304。
		3. 因为修改资源文件后该值会立即变更。这也决定了 ETag 在断点下载时非常有用。
2. json通信的业务数据缓存，一般是结合业务场景，通常添加一个接口，客户端将本地缓存的最后一条数据的的时间戳或 id 报给服务端，然后服务端做diff，没有新增则返回 nil 或 304。

# cookie 存储
URL Loading System 提供了 app 级别的 cookie 存储机制。URL Loading System 中涉及到 cookie 操作的两个类分别是：
	* HTTPCookieStorage：这个类提供了管理 cookie 存储的功能。
	* HTTPCookie：用来封装 cookie 数据和属性的类。
URLRequest 提供了 HTTPShouldHandleCookies 属性来设置请求发起时，是否需要 cookie manager 自动处理 cookie。在 UIWebView 中，系统会通过 cookie manager 自动将 cookie 缓存起来。

一些注意点：
1. URLRequest在请求发送的时候,，会自动带上已经存储在Cookie Storage中的cookies，除非我们设置了NSURLRequest的HTTPShouldHandleCookies属性为不处理Cookie。
2. URLResponse会根据当前APP中的HTTPCookieStorage的cookieAcceptPolicy，选择是否将响应头中的Cookie存储在Cookie Storage中。
3. 每个WKWebView的实例对象都有一个与它自己关联的 cookie storage!!!, 我们可以参考WKHTTPCookieStore.(与APP 的Shared Cookie Storage互相独立!!!!)

[认识HTTP----Cookie和Session篇](https://zhuanlan.zhihu.com/p/27669892) 
[iOS Cookie 的一些注意点参考](https://www.jianshu.com/p/eae8638ecff0)

# 协议支持
URL Loading System 本身只支持 http、https、file、ftp 和 data 协议。NSURLProtocol 是一个抽象类，提供了URL Loading System 处理 URL 加载的基础设施。通过实现自定义的 NSURLProtocol 子类，可以让我们的 app 支持自定义的数据传输协议。

另外，对于 NSURLProtocol 核心功能，官方文档中并没有着重提到，但是却是最重要的一点：**借助它，你不必改动应用在网络调用上的其他部分，就可以改变 URL 加载行为的全部细节**。运用这一点，我们可以自由发挥，做很多想做的事情，比如：
	*  [拦截图片加载请求，转为从本地文件加载](http://stackoverflow.com/questions/5572258/ios-webview-remote-html-with-local-image-files) 
	*  [在 UIWebView 中加载 webp 图片](https://github.com/cysp/STWebPDecoder) 
	*  [通过缓存静态资源实现 UIWebView 的预加载优化](https://github.com/ShannonChenCHN/iOSLevelingUp/issues/55#issuecomment-300365305) 
	*  [UIWebView 离线缓存](https://github.com/rnapier/RNCachingURLProtocol) 
	*  [为了测试对HTTP返回内容进行mock和stub](https://draveness.me/%5Bhttps://github.com/AliSoftware/OHHTTPStubs%5D) 
	*  [实现一个 In-App 网络抓包工具](https://github.com/Flipboard/FLEX/tree/master/Classes/Network) 


# 统计分析
对于网络检查分析提供了URLSessionTaskMetrics类，URLSessionTaskMetrics是对应URLSessionTaskDelegate的，每个task结束时都会回调下面的方法，并且可以获得一个metrics对象，这个对象记录了请求发起到结束时的一些信息。通过URLSessionTaskMetrics可以帮助我们分析网络请求的过程（比如找耗时原因等），URLSessionTaskMetrics能获取到的详细参考信息[参考这里](https://juejin.cn/post/6847009772034588680#heading-15)。
```objc
- (void)URLSession:(NSURLSession *)session 
              task:(NSURLSessionTask *)task 
didFinishCollectingMetrics:(NSURLSessionTaskMetrics *)metrics;
```

# 错误处理
 [URLError](https://developer.apple.com/documentation/foundation/urlerror) ：URL加载API返回的错误代码。
 [URL Loading System Error Info Keys](https://developer.apple.com/documentation/foundation/url_loading_system/url_loading_system_error_info_keys) ：从URL加载API生成的错误对象的用户信息字典中识别这些键。

# 官方的指南参考
### 基本使用
[iOS 网络：Fetching Website Data into Memory（译）](https://zhuanlan.zhihu.com/p/345303067) ：通过从URL会话创建数据任务，将数据直接接收到内存中。

### 上传
 [iOS 网络：Uploading Data to a Website（译）](https://zhuanlan.zhihu.com/p/345585333) ：将数据从你的应用发送到服务器。
 [iOS 网络：Uploading Streams of Data（译）](https://zhuanlan.zhihu.com/p/345586144) ：将数据流发送到服务器。

### 下载
 [iOS 网络：Downloading Files from Websites（译）](https://zhuanlan.zhihu.com/p/345593349) ：直接下载文件到文件系统。
 [iOS 网络：Pausing and Resuming Downloads（译）](https://zhuanlan.zhihu.com/p/345852507) ：允许用户继续下载而无需重新开始。
 [iOS 网络：Downloading Files in the Background（译）](https://zhuanlan.zhihu.com/p/345853487) ：创建在应用不活动时下载文件的任务。


[NSURLSession 上传下载实战](https://juejin.cn/post/6847009772034588680#heading-18)

[深度理解 NSURLProtocol - FiTeen’s Blog](https://blog.fiteen.top/2020/hijacking-webview-request-with-nsprotocol)
[NSURLProtocal使用](https://www.cnblogs.com/junhuawang/p/8465982.html)
[iOS - NSURLProtocol详解和应用 - 俊华的博客 - 博客园](https://www.cnblogs.com/junhuawang/p/8465982.html)

[Swift的网络监控方案](https://github.com/pmusolino/Wormholy)
[即壳iOS中的网络调试](https://juejin.cn/post/6844904185268273159)（对理解NSURLProtocal很有帮助，同时也介绍了网络监控方案的实现思路）

[iOS app秒开H5优化探索](https://juejin.cn/post/6844903809521549320#heading-8)
[iOS 流量监控分析](https://juejin.cn/post/6844903617623752711)

[iOS 设置代理（Proxy）方案总结](https://juejin.cn/post/6844903774440407047#heading-6)



