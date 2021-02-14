# Native交互
#flutter


# 平台通道
[撰写双端平台代码（插件编写实现）  - Flutter 中文文档 - Flutter 中文资源](https://flutter.cn/docs/development/platform-integration/platform-channels?tab=ios-channel-swift-tab)
dart内注册channel，invoke发送消息，原生内实现回调。

共享原生代码给Flutter使用：通过package实现，package内通过平台通道和原生平台交互，然后把这一部分功能继承在一个package内，再共享给flutter使用，有2种packages：
	* Dart packages ：用 Dart 编写的传统 package，比如  [path](https://pub.flutter-io.cn/packages/path) 。其中一些可能包含 Flutter 的特定功能，因此依赖于 Flutter 框架，其使用范围仅限于 Flutter。
	* Plugin packages：是 Dart package 的特别版本，按需使用 Java 或 Kotlin、ObjC 或 Swift 分别在 Android 和/或 iOS 平台实现的 package。
通过module在现有iOS中集成Flutter：创建flutter的module后，再通过pod集成就好，注意：
	* 当你在 my_flutter/pubspec.yaml 改变了 Flutter plugin 依赖，需要在 Flutter module 目录运行 flutter pub get，来更新会被podhelper.rb 脚本用到的 plugin 列表，然后再次在你的应用目录 some/path/MyApp 运行 pod install.
	* podhelper.rb 脚本会把你的 plugins， Flutter.framework，和 App.framework 集成到你的项目中。
	* 你应用的 Debug 和 Release 编译配置，将会集成相对应的 Debug 或 Release 的  [编译产物](https://flutter.cn/docs/testing/build-modes) ，可以增加一个 Profile 编译配置用于在 profile 模式下测试应用。
	* Flutter.framework 是 Flutter engine 的框架， App.framework 是你的 Dart 代码的编译产物。
```
# Uncomment the next line to define a global platform for your project
# platform :ios, '9.0'


# 添加模块所在路径
flutter_application_path = '../flutter_module'
load File.join(flutter_application_path, '.ios', 'Flutter', 'podhelper.rb')


target 'HPlayer' do
  # Comment the next line if you don't want to use dynamic frameworks
  use_frameworks!

  # Pods for HPlayer

  # 安装Flutter模块
  install_all_flutter_pods(flutter_application_path)

  target 'HPlayerTests' do
    inherit! :search_paths
    # Pods for testing
  end

end
```


![](Native%E4%BA%A4%E4%BA%92/92AB4BC3-E172-4494-BC95-5A749EF9799C.png)


新一代Flutter-Native混合解决方案。 FlutterBoost是一个Flutter插件，它可以轻松地为现有原生应用程序提供Flutter混合集成方案。FlutterBoost的理念是将Flutter像Webview那样来使用。在现有应用程序中同时管理Native页面和Flutter页面并非易事。 FlutterBoost帮你处理页面的映射和跳转，你只需关心页面的名字和参数即可（通常可以是URL）。

[flutter_boost/README_CN.md at master · alibaba/flutter_boost · GitHub](https://github.com/alibaba/flutter_boost/blob/master/README_CN.md)

[Flutter：基于video_player实现视频相关手势控制、全屏播放](https://juejin.cn/post/6844904039612678158#heading-18)