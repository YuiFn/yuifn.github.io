---
title: "Different Profiles in Flutter apps"
date: 2018-12-03T12:23:15+08:00
draft: false
tags: [ "Flutter", "Dart"]
categories: [ "Flutter", "Dart"]
keywords: ["Flutter build environments ", "Flutter按照profile加载配置"]
---

在日常的开发中需要根据不同的Profile/env来加载对应的配置信息，比如dev，qa， prod等。Flutter怎么实现这个功能呢？
浏览下文档无果，最后在其examples/catalog的例子中找到了一个可行的方法
```shell
flutter run lib/animated_list.dart
flutter run lib/app_bar_bottom.dart
flutter run lib/basic_app_bar.dart
...
```
Flutter生成的**main.dart**是默认的APP入口文件但可以通过命令修改为其他文件如上，所以只要按照不同的Profile创建不同的main的文件即可，如
main_dev.dart对应着的开发环境。用一个例子说明下

#### 01 - 生成一个默认的项目
```shell
flutter create profiles

# 在lib中的结构如下:
lib
└── main.dart
```

#### 02 - 分离main.dart中的类为独立的文件
```shell
├── main.dart
├── app.dart
└── my_home_page.dart
```
#### 03 - 添加配置信息
添加一个新类用于存储需要的配置信息
```dart
/// config/app_config.dart
import 'package:meta/meta.dart';

class AppConfig {
  final String appName;
  final String apiBaseUrl;

  AppConfig({@required this.appName, @required this.apiBaseUrl}):
      assert(appName!=null),
      assert(apiBaseUrl!=null);
}
```
那怎么在不同的Widget之间传递配置信息呢？一个直观的做法就是直接创建在不同的环境中创建一个全局的对象，然后直接访问比如
```dart
/// main_dev.dart
main(){
    final dev = AppConfig(
        appName: "app - dev",
        apiBaseUrl: "https://dev.app",
    );
    AppConfig.profile = dev;
}
```
这种方式当然是可行而且简单明了，但在Flutter中倡导一切都是Widget，可不可以通过Widget的方式去传递呢。答案是可以的，Flutter提供了名为**InheritedWidget**的类,它提供了在Widget Tree中获取最近实例的能力。在一些常见的设计模式中比如BLOC模式中也正常使用。

用**InheritedWidget**对app_config.dart进行改造，并提供一个of的静态方法来获取当前的配置
```dart
/// config/app_config.dart
import 'package:flutter/material.dart';
import 'package:meta/meta.dart';

class AppConfig extends InheritedWidget {
  final String appName;
  final String apiBaseUrl;
  final Widget child;

  AppConfig({
    @required this.appName,
    @required this.apiBaseUrl,
    @required this.child,
  })  : assert(appName != null),
        assert(apiBaseUrl != null),
        assert(child != null),
        super(child: child);

  @override
  bool updateShouldNotify(InheritedWidget oldWidget) => false; // 这里不需要更新

  static AppConfig of(BuildContext context) {
    return context.inheritFromWidgetOfExactType(AppConfig);
  }
}
```
然后就可以在app.dart中使用了
```dart
/// main_dev.dart
import 'package:flutter/material.dart';
import 'package:profiles/app.dart';
import 'package:profiles/config/app_config.dart';

main() {
  final devApp = AppConfig(
    appName: 'app - dev',
    apiBaseUrl: "https://dev.app",
    child: MyApp(),
  );
  runApp(devApp);
}
/// main_prod.dart
import 'package:flutter/material.dart';
import 'package:profiles/app.dart';
import 'package:profiles/config/app_config.dart';

main() {
  final devApp = AppConfig(
    appName: 'app - prod',
    apiBaseUrl: "https://prod.app",
    child: MyApp(),
  );
  runApp(devApp);
}

/// app.dart
import 'package:flutter/material.dart';
import 'package:profiles/config/app_config.dart';
import 'package:profiles/my_home_page.dart';

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // 获取对应的配置信息
    final config = AppConfig.of(context);
    return MaterialApp(
      title: config.appName,
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: MyHomePage(title: config.apiBaseUrl),
    );
  }
}
```
lib的目录结构:
```shell
.
├── app.dart
├── config
│   └── app_config.dart
├── main.dart
├── main_dev.dart
├── main_pord.dart
└── my_home_page.dart
```



#### 04 - 运行和打包
可以通过flutter的命令来确定当前运行的是哪个Profile
```shell
# 开发环境
flutter run -t lib/main_dev.dart
# 正是环境
flutter run -t lib/main_prod.dart

# android 打包
flutter build apk -t lib/main_dev.dart
# ios 打包
flutter build ios -t lib/main_dev.dart

```
#### 05 - IDE集成
这里保留了原来的main.dart作为默认的口(当不指定具体的一个Profile时),
```dart
import 'main_dev.dart' as dev;

void main() => dev.main();
```
当然也可以通过IDE相关的配置，来使用具体的入口文件，比如在IDEA中可以通过配置对应的config,重新指定一个文件
![map](/imgs/flutter-config.png)
