---
layout: post
title:  "Flutter-HelloWorld"
date:   2019-5-16 11:09:22 +0800
categories: flutter
---

## 一、Flutter初识
Flutter有什么优势？它可以帮助你：
#### 提高开发效率
- 同一份代码开发iOS和Android
- 用更少的代码做更多的事情
- 轻松迭代
- 在应用程序运行时更改代码并重新加载（通过热重载）（IDE部署的时候，并不是在移动设备上运行的时候，与插件化的热更新不是一个概念）
- 修复崩溃并继续从应用程序停止的地方进行调试
#### 创建美观，高度定制的用户体验
- 受益于使用Flutter框架提供的丰富的Material Design和Cupertino（iOS风格）的widget
- 实现定制、美观、品牌驱动的设计，而不受原生控件的限制
### 1）核心原则
Flutter包括一个现代的响应式框架、一个2D渲染引擎、现成的widget和开发工具。这些组件可以帮助您快速地设计、构建、测试和调试应用程序。
#### 一切皆为widget
Widget是Flutter应用程序用户界面的基本构建块。每个Widget都是用户界面一部分的不可变声明。

与其他将视图、控制器、布局和其他属性分离的框架不同，Flutter具有一致的统一对象模型：widget。

Widget可以被定义为:
- 一个结构元素（如按钮或菜单）
- 一个文本样式元素（如字体或颜色方案）
- 布局的一个方面（如填充）
- 等等…
### 2）分层的框架
Flutter框架是一个分层的结构，每个层都建立在前一层之上。
![](http://i68.tinypic.com/1gpqfp.png)
## 二、Flutter源码解析
参考：https://yq.aliyun.com/articles/672478
### 1、Application
在AndroidManifest.xml文件里面定义的Application路径是：io.flutter.app.FlutterApplication，发现这个Application定义在flutter.jar包里面，点进去看看实现的主要方法在onCreate中调用：
```dart
FlutterMain.startInitialization(this);
具体做了这几件事：
initConfig(applicationContext);//1
initAot(applicationContext);//2
initResources(applicationContext);//3
System.loadLibrary("flutter");//4
```
#### 1）初始化配置
#### 2）初始化AOT(ahead of time) 

使用 AOT 编译后的应用，不再包含任何 HTML 片段，取而代之的是编译生成的 TypeScript 代码，这样的话 TypeScript 编译器就能提前发现错误。总而言之，采用 AOT 编译模式，我们的模板是类型安全的。适用于部署发布。
#### 3）初始化资源

主要实现一个AsyncTask，先在线程池中删除data目录下的文件，再将assets目录下的文件复制释放到data目录下
#### 4）jni的常规操作，加载so库
### 2、MainActivity

常规的入口函数是MainActivity，继承于FlutterActivity，在这个函数里面主要初始化了这几个函数：
#### 1）FlutterActivityDelegate，FlutterActivity的代理

其实就是代理接管了Activity的生命周期还有一些重写的方法。
初始化核心代码是:
```dart
String[] args = getArgsFromIntent(this.activity.getIntent());
FlutterMain.ensureInitializationComplete(this.activity.getApplicationContext(), args);
this.flutterView = this.viewFactory.createFlutterView(this.activity);
if (this.flutterView == null) {
    FlutterNativeView nativeView = this.viewFactory.createFlutterNativeView();
    this.flutterView = new FlutterView(this.activity, (AttributeSet)null, nativeView);
    this.flutterView.setLayoutParams(matchParent);
    this.activity.setContentView(this.flutterView);
    this.launchView = this.createLaunchView();
    if (this.launchView != null) {
        this.addLaunchView();
    }
}

if (!this.loadIntent(this.activity.getIntent())) {
    if (!this.flutterView.getFlutterNativeView().isApplicationRunning()) {
        String appBundlePath = FlutterMain.findAppBundlePath(this.activity.getApplicationContext());
        if (appBundlePath != null) {
            FlutterRunArguments arguments = new FlutterRunArguments();
            arguments.bundlePath = appBundlePath;
            arguments.entrypoint = "main";
            this.flutterView.runFromBundle(arguments);
        }
    }
}
```
#### 2）FlutterView
继承自SurfaceView，第一次初始化的时候为空，在这个View里面初始化了FlutterNativeView，整个FlutterView的初始化在确保FlutterNativeView的创建成功和一些必要的view设置之外，主要做了两件事： 
- 1、注册SurfaceHolder监听，其中surfaceCreated回调会作为Flutter的第一帧回调使用。 
- 2、初始化了Flutter系统需要用到的一系列桥接方法

例如：localization、navigation、keyevent、system、settings、platform、textinput。 

另外FlutterActivityDelegate代理的onStart和onPause方法会通知给FlutterView，FlutterView自己不处理，交给FlutterNativeView通知给native部分，应该是引擎做绘制相关操作。
- 3、FlutterNativeView

实现BinaryMessenger了接口，可见这个view是个信使，专门传递消息给底层引擎。

另外这个类里面初始化了FlutterPluginRegistry类，这个类有两个核心的方法：attach，detach。

在FlutterView构造函数初始化的时候会调用mNativeView.attachViewAndActivity，这里就调用了FlutterPluginRegistry.attach，这个方法具体作用是：
通过MethodChannel调用flutter/platform_views方法，添加视图；

FlutterView里面也调用了detach方法，也是通过MethodChannel向抵用flutter/platform_views方法，刷新视图状态，detach视图。
- 4、实现了加载launchView的能力

如果需要自定义launchView需要在AndroidManifest.xml中入口Activity处设置开关metaData:io.flutter.app.android.SplashScreenUntilFirstFrame,Activity定义的Theme就是splash screen的视图。

源码如下：
```java
//判断是否设置
private Boolean showSplashScreenUntilFirstFrame() {
        try {
            ActivityInfo activityInfo = this.activity.getPackageManager().getActivityInfo(this.activity.getComponentName(), 129);
            Bundle metadata = activityInfo.metaData;
            return metadata != null && metadata.getBoolean("io.flutter.app.android.SplashScreenUntilFirstFrame");
        } catch (NameNotFoundException var3) {
            return false;
        }
    }
//获取launchView
private Drawable getLaunchScreenDrawableFromActivityTheme() {
        TypedValue typedValue = new TypedValue();
        if (!this.activity.getTheme().resolveAttribute(16842836, typedValue, true)) {
            return null;
        } else if (typedValue.resourceId == 0) {
            return null;
        } else {
            try {
                return this.activity.getResources().getDrawable(typedValue.resourceId);
            } catch (NotFoundException var3) {
                Log.e("FlutterActivityDelegate", "Referenced launch screen windowBackground resource does not exist");
                return null;
            }
        }
    }
```
- 5、loadIntent

源码方法如下：
```java
if (!this.loadIntent(this.activity.getIntent())) {
    if (!this.flutterView.getFlutterNativeView().isApplicationRunning()) {
        String appBundlePath = FlutterMain.findAppBundlePath(this.activity.getApplicationContext());
        if (appBundlePath != null) {
            FlutterRunArguments arguments = new FlutterRunArguments();
            arguments.bundlePath = appBundlePath;
            arguments.entrypoint = "main";
            this.flutterView.runFromBundle(arguments);
        }
    }
}
```
核心是：`this.flutterView.runFromBundle(arguments);`

这一句最终调用到：`FlutterNativeView.runFromBundle(FlutterRunArguments args)`，在底层发现调用的是：
```java
bool DartController::SendStartMessage(Dart_Handle root_library,
                                      const std::string& entrypoint) {
  // other codes ...
  // Get the closure of main().
  Dart_Handle main_closure = Dart_GetClosure(
  root_library, Dart_NewStringFromCString(entrypoint.c_str()));
  // other codes ...
  // Grab the 'dart:isolate' library.
  Dart_Handle isolate_lib = Dart_LookupLibrary(ToDart("dart:isolate"));
  DART_CHECK_VALID(isolate_lib);
  // Send the start message containing the entry point by calling
  // _startMainIsolate in dart:isolate.
  const intptr_t kNumIsolateArgs = 2;
  Dart_Handle isolate_args[kNumIsolateArgs];
  isolate_args[0] = main_closure;
  isolate_args[1] = Dart_Null();
  Dart_Handle result = Dart_Invoke(isolate_lib, ToDart("_startMainIsolate"),
                                   kNumIsolateArgs, isolate_args);
  return LogIfError(result);
}
```
主要做了三件事： 
1. 获取Flutter入口方法（例如main方法）的closure。 
2. 获取FlutterLibrary。 
3. 发送消息来调用Flutter的入口方法。
由此处终于知道Android是如何启动flutter相关的代码了。
## 三、Flutter页面跳转
FlutterView中定义了两个重要的方法：
```java
public void pushRoute(String route) {//添加路由
        this.mFlutterNavigationChannel.invokeMethod("pushRoute", route);
    }
public void popRoute() {//删除路由
        this.mFlutterNavigationChannel.invokeMethod("popRoute", (Object)null);
    }
```
每一个页面都可以抽象成Widget，每一个Widget跳转的识别符号是路由。

### 从页面A跳转到页面B：
#### 1、不定义route名称
```java
import 'package:flutter/material.dart';

void main() {
  runApp(MaterialApp(
    title: 'Navigation Basics',
    home: FirstScreen(),
  ));
}

class FirstScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('First Screen'),
      ),
      body: Center(
        child: RaisedButton(
          child: Text('Launch screen'),
          onPressed: () {
            Navigator.push(
              context,
              MaterialPageRoute(builder: (context) => SecondScreen()),
            );
          },
        ),
      ),
    );
  }
}

class SecondScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("Second Screen"),
      ),
      body: Center(
        child: RaisedButton(
          onPressed: () {
            Navigator.pop(context);
          },
          child: Text('Go back!'),
        ),
      ),
    );
  }
}
```
#### 2、定义route名称跳转
```java
import 'package:flutter/material.dart';

void main() {
  runApp(MaterialApp(
    title: 'Named Routes Demo',
    // Start the app with the "/" named route. In our case, the app will start
    // on the FirstScreen Widget
    initialRoute: '/',
    routes: {
      // When we navigate to the "/" route, build the FirstScreen Widget
      '/': (context) => FirstScreen(),
      // When we navigate to the "/second" route, build the SecondScreen Widget
      '/second': (context) => SecondScreen(),
    },
  ));
}

class FirstScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('First Screen'),
      ),
      body: Center(
        child: RaisedButton(
          child: Text('Launch screen'),
          onPressed: () {
            // Navigate to the second screen using a named route
            Navigator.pushNamed(context, '/second');
          },
        ),
      ),
    );
  }
}

class SecondScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("Second Screen"),
      ),
      body: Center(
        child: RaisedButton(
          onPressed: () {
            // Navigate back to the first screen by popping the current route
            // off the stack
            Navigator.pop(context);
          },
          child: Text('Go back!'),
        ),
      ),
    );
  }
}
```
> 详情参考：https://flutter.io/docs/development/ui/navigation