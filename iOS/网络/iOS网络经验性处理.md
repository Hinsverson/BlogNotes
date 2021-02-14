# iOS网络经验性处理
[iOS网络深度优化总结](https://juejin.im/post/6844903829209612296)

#计算机网络/ios相关

# 实现文件的断点下载
**基本原理：**
在request头部写入请求数据范围，每次请求时请求对应范围的数据即可。

只要设置HTTP请求头的Range属性, 就可以实现从指定位置开始下载
表示头100个字节：Range: bytes=0-99
表示第二个100字节：Range: bytes=100-199
表示最后100个字节：Range: bytes=-100
表示100字节以后的范围：Range: bytes=100-   
``` swift
//表示本次请求100-字节后的数据
request.setValue(“bytes=100-“, forHTTPHeaderField: “Range”)
```

**系统实现**
URLSessionDownloadTask有个cancel方法，取消下载时会有个Data类型的参数resumeData是用于记录下载的URL地址和已下载的总共的字节数两部分,而不是直接存储的已下载的数据.把resumeData保存下来后,使用URLSession自带的downloadTaskWithResumeData方法创建下一个下载即可实现断点续传.
``` swift
open func cancel(byProducingResumeData completionHandler: @escaping (Data?) -> Void)
```

# 大文件的上传下载
[NSFileHandle完成分段读写数据 - 简书](https://www.jianshu.com/p/7a3cf862b63d)
角度边下边写，分段读取上传

**边下边写基本原理：**
创建写文件的句柄，在每次HTTP请求头部设置Rang，请求完成后写入并更新写入位置，直到所有数据请求并写入。
         
**分段读取基本原理：**
创建读文件的句柄，每次读一段数据后后更新所读的位置后继续读取数据，直到结束。
``` swift
self.readHandle = [NSFileHandle fileHandleForReadingAtPath:path];//读到内存
self.offset = 0;
//将句柄移动到已读取内容的最后
[self.readHandle seekToFileOffset:self.offset];
//读取指定大小的内容（PackgeSize）
data = [self.readHandle readDataOfLength:PackgeSize];
//偏移量累加
self.offset += PackgeSize;
//最后结束时一定要关闭句柄
[self.readHandle closeFile];
```

# 请求的取消处理
1. 请求未发出，cancel后则不发送请求
2. 请求已发出，但还没有完成，cancel会立即回调failure
3. 请求已完成，此时cancel没有任何效果

### 取消指定网络请求后如何判断取消的处理？
[iOS取消网络请求的正确姿势 - 简书](https://www.jianshu.com/p/96272c18150e)
拿到SessionManager匹配URL，然后调用task的cancel。
请求cancel后会进入failure回调方法，而像网络不通、服务器宕机无法连接等错误也会进入failure方法，那么如何区分是否是因为cancel进入的？可以通过判断task的错误码来进行区分。

### 页面返回的时候，将网络请求取消
将请求URLSessionTask记录并存下来，根据state维护一个当前正在请求并还未完成的表，在控制器销毁时全部cancel掉。

### 同一个请求多次请求时，短时间忽略相同的请求
UI上的交互增加请求的阀值时间，或者KVO请求的状态，如果没有完成则不发起下一次请求。

### 同一个请求多次请求时，取消之前发出的请求（搜索场景）
还是把请求存在一张表里，可以对URL和参数进行哈希运算得到一个key，value为请求的时间+请求的缓存结果，请求时如果时间间隔过短则使用缓存结果。

### 发送的请求，多次尝试并确保成功。
主要是请求的生命周期问题要独立，考虑使用单例管理。

# Https适配和自签名证书使用
.p12文件一个复合文件，其中包装了私钥与证书信息，使用OpenSSL工具可以将其中的信息进行提取。
```
//CD到.p12文件所在的目录下，终端执行
//分离私钥
openssl pkcs12 -in huishao.p12 -nocerts -out privateKey.pem -nodes
//分离证书（包含公钥）
openssl pkcs12 -in huishao.p12 -nokeys -out cert.pem -nodes
//将.pem转为der或者cer格式
//penssl x509 -inform PEM -in cert.pem -outform DER -out cert.der
```

###  ATS策略配置
在iOS9之后，开发者可以在Info.plist文件中添加如下键：NSAppTransportSecurity（App Transport Security Settings）。这个键用来配置APP传输安全的相关策略，是字典类型，其中可以设置的键有五个：
	1. NSAllowsArbitraryLoads：布尔值，默认为NO，设置为YES则代表除了NSExceptionDomains中设置的域名外，其他所有请求的协议类型都不受限制，也就是说可以支持HTTP类型的请求，这个键的作用域是全局的，App内所有的请求都受影响，但是如果开发者设置为了YES，在提交审核时需要说明原因。
	2. NSAllowsArbitraryLoadsForMedia：布尔值，默认为NO，设置为YES的话，则应用程序内所有的媒体数据的加载将不受协议类型的限制，同样如果开发者设置为了YES，则在提交审核时需要说明原因。
	3. NSAllowsArbitraryLoadsInWebContent：布尔值，默认为NO。如果设置为YES，则应用程序内所有WebView的请求加载不受协议类型的限制，开发者设置为了YES，则在提交审核时需要说明原因。
	4. NSAllowsLocalNetworking：布尔值，默认为NO，如果设置为YES，则在加载本地资源时不受安全传输协议的限制。
	5. NSExceptionDomains：字典，其主要对某些特殊域名做限制。其中结构可以表示如下：
		1. NSIncludesSubdomains：布尔值，这个键的作用是设置此域名下的所有子域名是否采用和父域名相同的配置。
		2. NSExceptionAllowsInsecureHTTPLoads：布尔值，设置是否允许此域名使用自签名的证书进行请求，默认为NO，如果设置为YES，则在提交时需要说明原因。
		3. NSExceptionMinimumTLSVersion：设置所使用的TLS版本。
		4. NSExceptionRequiresForwardSecret：设置为NO，则不允许向前加密方式。
		5. NSRequiresCertificateTransparency：如果设置为YES，则服务端的证书要有有效的透明时间戳。
```
NSAppTransportSecurity : Dictionary {
    NSAllowsArbitraryLoads : Boolean
    NSAllowsArbitraryLoadsForMedia : Boolean
    NSAllowsArbitraryLoadsInWebContent : Boolean
    NSAllowsLocalNetworking : Boolean
    //对某些域名做特殊限制
    NSExceptionDomains : Dictionary {
        <domain-name-string> : Dictionary {
            NSIncludesSubdomains : Boolean
            NSExceptionAllowsInsecureHTTPLoads : Boolean
            NSExceptionMinimumTLSVersion : String
            NSExceptionRequiresForwardSecrecy : Boolean   // Default value is YES
            NSRequiresCertificateTransparency : Boolean
        }
    }
}
```

### 自签名证书处理
[OS安全系列之二：HTTPS进阶](https://www.kancloud.cn/digest/ios-security/67013#21__147)
1. 发起请求，需要证书校验时会调用URLSession代理的didReceiveChallenge方法，这里是处理证书校验的入口，校验的信息会通过URLAuthenticationChallenge对象给出来。
2. 通过`challenge.protectionSpace.authenticationMethod`取得保护空间要求我们认证的方式（server、client的校验还是其他）。
3. 系统默认的处理是根据发起请求时server返回的证书，去系统内置的证书链上做校验，通过后则建立连接，不通过则报错。
4. 使用自签名证书做校验时无非就是把本地的证书load加载到证书校验链上再做校验，具体处理如下[详细参考这里](https://www.jianshu.com/p/31bcddf44b8d)：
	1. 首先自定义的验证过程，需要先通过`challenge.protectionSpace.serverTrust`拿出一个 SecTrustRef 对象，它是一种执行信任链验证的抽象实体，包含着验证策略SecPolicyRef以及一系列受信任的锚点证书，而我们能做的也是修改这两样东西而已。
	2. 再加载本地bundle内的证书，使用`SecTrustSetAnchorCertificates(trust, (__bridge CFArrayRef)certificates)`将它设置为锚点证书进行证书链的验证。
	3. 但这样系统本身默认信任的 CA 证书就不能作为锚点验证证书链，要想恢复系统中 CA 证书作为锚点的功能还需要调用`SecTrustSetAnchorCertificatesOnly(trust, false)`，true 代表仅被传入的证书作为锚点，false 允许系统 CA 证书也作为锚点。
	4. 最后调用`SecTrustEvaluate(trust, &trustResult)`，系统会递归地从叶节点证书到根证书进行校验。

具体的自签证书使用实现参考：
[Swift - 使用URLSession通过HTTPS进行网络请求，及证书的使用](https://www.hangge.com/blog/cache/detail_991.html)
[深入理解HTTPS及在iOS系统中适配HTTPS类型网络请求(上) - 珲少的个人空间 - OSCHINA - 中文开源技术交流社区](https://my.oschina.net/u/2340880/blog/807358)
[深入理解HTTPS及在iOS系统中适配HTTPS类型网络请求(下) - 珲少的个人空间 - OSCHINA - 中文开源技术交流社区](https://my.oschina.net/u/2340880/blog/807863)
[Alamofire  https自签名证书验证 - 简书](https://www.jianshu.com/p/cbad827eeed7?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)
[IOS 使用自签名证书开发HTTPS文件传输_kyl282889543的博客-CSDN博客](https://blog.csdn.net/kyl282889543/article/details/102877244)