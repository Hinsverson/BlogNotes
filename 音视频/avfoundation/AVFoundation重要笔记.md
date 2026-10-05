# AVFoundation重要笔记


# AVFuodation常用数据结构理解
CMSampleBuffer：存放编解码前后的视频图像的容器数据结构
![](AVFoundation%E9%87%8D%E8%A6%81%E7%AC%94%E8%AE%B0/6989C9A8-C85F-4DD3-9C69-426802E6647A.png)
调用 AVCaptureSession 拍摄输出的每一帧图像都会被包装成 CMSampleBuffer 对象，通过这个 CMSampleBuffer 对象你就可以获取到未压缩的 CVPixelBuffer 对象；如果读取 H.264 文件你也可以获取数据生成压缩的 CMBlockBuffer 对象并创建一个 CMSampleBuffer 对象给 VideoToolbox 来解码。也就是说，CMSampleBuffer 既可以作为 CVPixelBuffer 对象的容器，也可以作为 CMBlockBuffer 对象的容器，CVPixelBuffer 可以说是未压缩的图像数据容器，而 CMBlockBuffer 则是压缩图像数据容器。VideoToolBox的编解码核心过程就是上面这张图里2个sampleBuffer的拼装。
CVPixelBuffer：编码前和解码后的图像数据结构
CMBlockBuffer：编码后图像的数据结构
CMVideoFormatDescription：图像存储方式，编解码器等格式描述
pixelBufferAttributes：可能包含了视频的宽高，像素格式类型（32RGBA, YCbCr420），是否可以用于OpenGL ES等相关信息
CMTime：以帧的概念表示时间的数据结构
	1. 分母：表述精度，比如600就是1秒600帧（一般都用600，因为它是常见视频帧率的最大公倍数）。
	2. 分子：表述第几个帧，比如1。
 AVAsset：指代一个或者多个音视频媒体资源数据的集合。

# CVPixelBuffer
[深入理解 CVPixelBufferRef - 知乎](https://zhuanlan.zhihu.com/p/24762605)
[从RGB数组到CVPixelBuffer和UIImage再到CoreML - 知乎](https://zhuanlan.zhihu.com/p/125728613)
CVPixelBuffer是CoreVideo中用来描述一张图片原始像素数据的底层格式。包含很多图片相关属性，比较重要的有 width，height，PixelFormatType等。比如 kCVPixelFormatType_420YpCbCr8BiPlanarFullRange，这种YUV多平面的数据格式，这个类型里 BiPlanar 表示双平面，说明它是一个 NV12的YUV，包含一个Y平面和一个UV平面。
	* 通过CVPixelBufferGetBaseAddressOfPlane可以得到每个平面的数据指针提取数据进行处理（在操作前需要调用CVPixelBufferLockBaseAddress加锁）

显示一个CVPixelBuffer的思路是转化
	1. CoreImage把CVPixelBuffer转为CIImage、CGImage、UIImage
	2. 通过CoreVideo，转为OpenGLES的纹理对象，再通过OpenGLESTextCache对象进行纹理缓存管理，然后通过OpenGLES把这个纹理绘制出来（一个平面对应一个纹理）。
	3. 同样也可以转为Metal的纹理对象并绘制出来。
创建一个CVPixelBuffer的思路无非就是针对不同的存储格式，获取数据指针后按规则填入像素数据。


# AVFoudation理解
* 音频播放和记录—— [AVAudioPlayer](https://developer.apple.com/documentation/avfoundation/avaudioplayer)  和  [AVAudioRecorder](https://developer.apple.com/documentation/avfoundation/avaudiorecorder) 
* 媒体文件检查—— [AVMetadataItem](https://developer.apple.com/documentation/avfoundation/avmetadataitem) 
* 视频播放—— [AVPlayer](https://developer.apple.com/documentation/avfoundation/avplayer)  和  [AVPlayerItem](https://developer.apple.com/documentation/avfoundation/avplayeritem) 
* 媒体捕捉—— [AVCaptureSession](https://developer.apple.com/documentation/avfoundation/avcapturesession) 
* 媒体编辑：AVComposition编辑
* 媒体处理—— [AVAssetReader](https://developer.apple.com/documentation/avfoundation/avassetreader) 、 [AVAssetWriter](https://developer.apple.com/documentation/avfoundation/avassetwriter)

# Core Audio理解
[Core Audio音频基础概述 - 掘金](https://juejin.im/post/5cca9e99f265da03a54c2bc0)

1. magic cookie：被附加到压缩音频数据(文件或流)中的元数据，可以为解码器提供了正确解码文件或流所需要的详细信息，可以通过API进行复制，读取，使用元数据包含的信息。
2. audio converter：音频格式相互转换，本质时ABSD的转换。
3. Audio File Service：当需要对文件进行读取、写入处理时。
4. audio file stream：parse音频流数据。
5. Audio Queue Services：低级别、高性能的采集和播放方式（使用原理）
6. OpenAL：游戏场景使用，通过audio unit实现最低延迟的音频播放。
7. Audio Unit：最低阶别的音频处理API

# AVCaptureSession的理解
![](AVFoundation%E9%87%8D%E8%A6%81%E7%AC%94%E8%AE%B0/ADB50540-7CE5-4996-A008-10544FE01961.png)
[iOS视频流采集概述(AVCaptureSession) - 掘金](https://juejin.im/post/5cb1f987f265da039d3274c3)
device进行相机硬件参数设置（注意配置过程中有加锁和解锁操作）
session管理输入输出会话
Input对象表示具体的的输入端的硬件设备
Output设置代理方法产生视频帧
connect进行输入输出端的参数配置。

需要进行相机运行中突然出错的监听处理。

# Audio Session 理解
[iOS音频播放(一) - 云+社区 - 腾讯云](https://cloud.tencent.com/developer/article/1608475)
[Audio Session:系统与应用程序的中介 - 掘金](https://juejin.im/post/6844903834385383431)
[iOS - AVAudioSession详解 - 俊华的博客 - 博客园](https://www.cnblogs.com/junhuawang/p/7920989.html)

audio sessions充当着各个app与系统间访问处理音频硬件资源的中介。
1. 通过category、option、mode：向系统传达你将如何使用麦克风、播放器等音频资源以及和其他App竞争时的处理。
2. 添加监听并响应重要的audio session通知，音频中断与硬件线路改变。
3. 配置音频采样率，声道数、选择配置麦克风等附加信息，使系统可以根据设备特性进行优化处理。

**注意点：**
1. 好习惯是先检查aduio session是否被激活，同时修改或者配置完audio session后也应该重新激活。
2. 中断前应保存状态和相关上下文，中断后再恢复状态并需要手动重新激活audio sesseion。

> 高级别的AVfuodation录制与播放在中断发生并结束后系统会进行重新的激活，低级别的需要手动重新激活。（Siri引发的中断比较特殊，因为它可以是中断中要求停止我们App的session，所以要根据实际判断是否需要重新激活）  

## 理解category、option、mode
![](AVFoundation%E9%87%8D%E8%A6%81%E7%AC%94%E8%AE%B0/B17E5FEF-2643-46B5-B16C-2E8C76E23A0C.png)
![](AVFoundation%E9%87%8D%E8%A6%81%E7%AC%94%E8%AE%B0/41F5CD1A-409F-45B7-AACD-8201FC878A7F.png)
![](AVFoundation%E9%87%8D%E8%A6%81%E7%AC%94%E8%AE%B0/D96F94D3-BC21-462F-A495-0B79C4B7B5AD.png)
Ambient：默认，不独占，后台锁屏静音。
Playback：后台播放（还需info配置）
PlayAndRecord：录音和播放共存

MixWithOthers：和其他App混合
DuckOthers：压低其他声音

1. 系统默认是SoloAmbient，
2. setActive:withOptions 可以激活并通过Options配置当解除激活后是否恢复其他AppSession的激活状态

## 通知处理
1. 监听线路改变的通知AVAudioSessionRouteChangeNotification
![](AVFoundation%E9%87%8D%E8%A6%81%E7%AC%94%E8%AE%B0/E9DF40AE-D235-44F5-933F-831262458C99.png)
2. 监听Session被打断的的通知AVAudioSessionInterruptionNotification
	1. Begin：打断开始
	2. Ended：结束打断
3. 检查别的Audio是否正在播放使用AVAudioSessionSilenceSecondaryAudioHintNotification
	1. Begin：表示其他App开始占据Session
	2. End：表示其他App开始释放Session

**实际参考：**
1. 后台混音播放
```swift
AVAudioSession.sharedInstance().setCategory(AVAudioSessionCategoryPlayback, with: .mixWithOthers)
```

# AudioStreamBasicDescription（ASBD）
[iOS音频开发之AudioStreamBasicDescription](https://www.cnblogs.com/huahuahu/p/iOS-yin-pin-kai-fa-zhiAudioStreamBasicDescription.html) 
1. sample rate：PCM采样频率，1s采样多少次。
2. channel：一次采样可以得到若干采样数据，对应存储到多个channel上。
3. frame：一次采样点得到的所有channel上的数据组合起来叫做一个frame。
4. packet：多个frame组合起来的数据叫做一个packet，用来表示一段时间有意义的音频数据的单位。
5. mBitsPerChannel：一次采样的数据存储位数。
6. CBR（PCM）和VBR（MP3、ACC）的相同点在于，所有packet都具有相同的帧数，不同点在于VBR每一个frame的位数大小不固定。

**PCM数据常见ABSD设置参考**
``` objectivec
AudioStreamBasicDescription streamFormat;

streamFormat.mFormatID = kAudioFormatLinearPCM;
streamFormat.mFormatFlags = kAudioFormatFlagIsSignedInteger | kAudioFormatFlagsNativeEndian | kAudioFormatFlagIsPacked;

streamFormat.mSampleRate = sampleRate;
streamFormat.mBitsPerChannel = bitsPerChannel;
streamFormat.mChannelsPerFrame = channelsPerFrame;
streamFormat.mFramesPerPacket = 1;

int bytes = (bitsPerChannel / 8) * channelsPerFrame;
streamFormat.mBytesPerFrame = bytes;
streamFormat.mBytesPerPacket = bytes;
```
**streamFormat.mFormatFlags设置的一些理解**
Sign : 表示样本数据是否是有符号位。
Byte Ordering : 大小端存储字节序。
Integer Or Floating Point : 使用整形或浮点型进行存储，一般PCM样本数据使用整形表示，对精度要求高可使用浮点类型。

**注意点：**
1. 描述参数在特定场景下不唯一时需为0，比如压缩音频格式每个sample使用不同数量的bits，mBitsPerChannel成员的值需要为0。
2. 如果有些值是你不知道的，可以赋0，Core Audio将自动选择适当的值。

# AudioQueue
[Audio Queue录制 播放原理](https://juejin.cn/post/6844903834574159879)
[AudioQueue实现音频流实时播放实战](https://juejin.cn/post/6844903877091786759)
整个API使用层面遵循生产者和消费者的思想，通过AudioQueue管理的AudioBuffer进行音频播放和录制。
	1. 录制时不断把buffer Enqueue，录音器采集到的音频数据会不断填充到buffer上，并通过回调函数传出来，在回调里面对采集到的音频数据进行下一步处理并把buffer 再次 Enqueue，播放时反过来同理。
	2. 一般建议用3个buffer即可，正常情况下两个即可，使用第三个是当有延迟情况出现时作为补偿数据。

1. 录制、播放流程：
	1. 通过ABSD创建Input或者output queue，并设置数据回调。
	2. 创建buffer并enqueue
	3. 设置检查audiosession的状态后start queue
2. 音量处理：设置queue对应的参数就好
3. 编解码处理：通过设置音频数据格式，系统自动选择合适的编解码器处理。

# Audio Unit（了解即可）
[(强烈推荐)移动端音视频从零到上手 - 掘金](https://juejin.im/post/5d29d884f265da1b971aa220)
![](AVFoundation%E9%87%8D%E8%A6%81%E7%AC%94%E8%AE%B0/7B2DD1C9-B07F-4A1B-9E96-8B060122A1EE.png)
最底层的audio API，除非你需要实时播放同步的声音，低延迟的输入输出或是一些音频优化的其他特性，否则应该使用上层框架。

**使用场景**
* 以最低延迟的方式同步音频的输入输出,如VoIP.
* 手动同步音视频,如游戏,直播类软件
* 使用特定的audio unit:如回声消除,混音,音调均衡
* 一种处理链架构:将音频处理模块组装成灵活的网络。这是iOS中唯一提供此功能的音频API。

**生命周期**
* 运行时，获取对动态可链接库的引用，该库定义您要使用的audio unit
* 新建一个audio unit实例
* 根据需求配置audio unit
* 初始化audio unit以准备处理音频
* 开启audio unit
* 控制audio unit
* 用完后释放audio unit




