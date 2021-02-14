# Flutter网络处理
#flutter

[Flutter网络请求](https://mp.weixin.qq.com/s?__biz=Mzg5MDAzNzkwNA==&mid=2247483765&idx=1&sn=1c44e47844e920ae6169111a74086720&chksm=cfe3f28af8947b9c4428968c514cbf87940f21c7b5fe0b18105651c7a919bc811ee381f95d94&scene=178&cur_album_id=1566028536430247937#rd)
[JSON 和序列化数据  - Flutter 中文文档 - Flutter 中文资源](https://flutter.cn/docs/development/data-and-backend/json#code-generation)
[Flutter如何JSON转Model](https://mp.weixin.qq.com/s?__biz=Mzg5MDAzNzkwNA==&mid=2247483770&idx=1&sn=5f4498179fcadf90a10e930ba9a17c1e&chksm=cfe3f285f8947b938cf4e8860f95ebd7b8599b44f871ce219aef0710e9e1ad3113c3f007747a&scene=178&cur_album_id=1566028536430247937#rd)

# 基本网络请求
1. 使用dart的io包中自带的HttpClient，封装了网络相关的基础处理。
2. Dart 官方提供的另一个网络请求类http，相比于 HttpClient，易用性提升了不少，但没有默认集成到Dart的SDK中，使用时需要先在pubspec中添加依赖。
3. 使用第三方库dio，快速方便的实现了比如拦截器、取消请求、文件上传/下载、超时设置等等常用开发需求。
```dart
  // 1.创建Dio请求对象
  final dio = Dio();
  // 2.发送网络请求
  final response = await dio.get("请求url");
  // 3.打印请求结果
  if (response.statusCode == HttpStatus.ok) {
    print(response.data);
  } else {
    print("请求失败：${response.statusCode}");
  }
```

# 模型转换
1. 拿到JSON字符串decode成字典后，手动映射字段和值。
2. json_serializable：采用关键字+注解的方式标记模型类，然后运行命令生成Model类和JSON的自动转换代码。


