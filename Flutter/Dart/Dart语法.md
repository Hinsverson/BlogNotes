# Dart语法
#flutter

[Flutter(三)之搞定Dart（一）](https://juejin.cn/post/6844903938697773063)
[Dart | 什么是Mixin - 简书](https://www.jianshu.com/p/a578bd2c42aa)
[Dart vs Swift](https://juejin.cn/post/6844903773039656968)
[一文了解Dart语法](https://juejin.cn/post/6844903773094019086#heading-26)
[Language tour | Dart](https://dart.cn/guides/language/language-tour)官方语法

在 Dart 中，未初始化的变量拥有一个默认的初始化值：null。即便数字也是如此，因为在 Dart 中一切皆为对象，数字也不例外。

Dart 支持两种 Number 类型，int 和 double 都是  [num](https://api.dart.dev/stable/dart-core/num-class.html)  的子类。

字符串和数字之间转换的方式：
```dart
var one = int.parse(‘1’);
String oneAsString = 1.toString();
```


可以在字符串中以 ${表达式} 的形式使用表达式，如果表达式是一个标识符，可以省略掉 {}。如果表达式的结果为一个对象，则 Dart 会调用该对象的 toString 方法来获取一个字符串。


因为所有类都是 Object 的子类。

??为dart中的null判断符号
```dart
// 当且仅当 b 为 null 时才赋值
b ??= value;
```


级联运算符（..）可以让你在同一个对象上连续调用多个对象的变量或方法，本质是一个dart的语法糖。
```dart
querySelector('#confirm') // 获取对象 (Get an object).
  ..text = 'Confirm' // 使用对象的成员 (Use its members).
  ..classes.add('important')
  ..onClick.listen((e) => window.alert('Confirmed!'));
```


Dart 提供了  [Exception](https://api.dart.dev/stable/dart-core/Exception-class.html)  和  [Error](https://api.dart.dev/stable/dart-core/Error-class.html)  两种类型的异常以及它们一系列的子类，你也可以定义自己的异常类型。但是在 Dart 中可以将任何非 null 对象作为异常抛出而不局限于 Exception 或 Error 类型。


两个使用相同构造函数相同参数值构造的编译时常量是同一个对象：
```dart
var a = const ImmutablePoint(1, 1);
var b = const ImmutablePoint(1, 1);

assert(identical(a, b)); // 它们是同一个实例 (They are the same instance!)
```

构造的思想同Swift和C++混合，先自己再父类。

如果你没有声明构造函数，那么 Dart 会自动生成一个无参数的构造函数并且该构造函数会调用其父类的无参数构造方法。（同C++类似）。

子类不会继承父类的构造函数，如果子类没有声明构造函数，那么只会有一个默认无参数的构造函数。

默认情况下，子类的构造函数会调用父类的匿名无参数构造方法，并且该调用会在子类构造函数的函数体代码执行前，如果子类构造函数还有一个 初始化列表，那么该初始化列表会在调用父类的该构造函数之前被执行，总的来说，这三者的调用顺序如下：
1. 初始化列表
2. 父类的无参数构造函数
3. 当前类的构造函数
4. 如果父类没有匿名无参数构造函数，那么子类必须调用父类的其中一个构造函数，为子类的构造函数指定一个父类的构造函数只需在构造函数体前使用（:）指定。

传递给父类构造函数的参数不能使用 this 关键字，因为在参数传递的这一步骤，子类构造函数尚未执行，子类的实例对象也就还未初始化，因此所有的实例成员都不能被访问，但是类成员可以。（同C++类似）