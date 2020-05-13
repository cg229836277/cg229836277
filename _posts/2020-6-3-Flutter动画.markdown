---
layout: post
title:  "Flutter动画"
date:   2020-6-3 11:09:22 +0800
categories: flutter
---

> 参考：https://flutter.io/docs/development/ui/animations

## 一、动画类型
动画分为两类：基于tween或基于物理的。
- 1）补间(Tween)动画

“介于两者之间”的简称。在补间动画中，定义了开始点和结束点、时间线以及定义转换时间和速度的曲线。然后由框架计算如何从开始点过渡到结束点。
- 2）基于物理的动画

在基于物理的动画中，运动被模拟为与真实世界的行为相似。例如，当你掷球时，它在何处落地，取决于抛球速度有多快、球有多重、距离地面有多远。

类似地，将连接在弹簧上的球落下（并弹起）与连接到绳子上的球放下的方式也是不同。

## 二、动画详解
Flutter中的动画系统基于Animation对象的，widget可以在build函数中读取Animation对象的当前值， 并且可以监听动画的状态改变。
### 1、基本概念
Animation对象是Flutter动画库中的一个核心类，它生成指导动画的值。

Animation对象知道动画的当前状态（例如，它是开始、停止还是向前或向后移动），但它不知道屏幕上显示的内容。
AnimationController管理Animation。

CurvedAnimation 将过程抽象为一个非线性曲线.
Tween在正在执行动画的对象所使用的数据范围之间生成值。例如，Tween可能会生成从红到蓝之间的色值，或者从0到255。
使用Listeners和StatusListeners监听动画状态改变。
### 2、Animation<double>
在Flutter中，Animation对象本身和UI渲染没有任何关系。Animation是一个抽象类，它拥有其当前值和状态（完成或停止）。其中一个比较常用的Animation类是Animation<double>。

Flutter中的Animation对象是一个在一段时间内依次生成一个区间之间值的类。Animation对象的输出可以是线性的、曲线的、一个步进函数或者任何其他可以设计的映射。

根据Animation对象的控制方式，动画可以反向运行，甚至可以在中间切换方向。

Animation还可以生成除double之外的其他类型值，如：Animation<Color> 或 Animation<Size>。

Animation对象有状态。可以通过访问其value属性获取动画的当前值。

Animation对象本身和UI渲染没有任何关系。
### 3、CurvedAnimation
CurvedAnimation 将动画过程定义为一个非线性曲线.
```java
final CurvedAnimation curve =
    new CurvedAnimation(parent: controller, curve: Curves.easeIn);
```
Curves 类类定义了许多常用的曲线，也可以创建自己的，例如：
```java
class ShakeCurve extends Curve {
  @override
  double transform(double t) {
    return math.sin(t * math.PI * 2);
  }
}
```
### 4、AnimationController
AnimationController是一个特殊的Animation对象，在屏幕刷新的每一帧，就会生成一个新的值。默认情况下，AnimationController在给定的时间段内会线性的生成从0.0到1.0的数字。 例如，下面代码创建一个Animation对象，但不会启动它运行：
```java
final AnimationController controller = new AnimationController(
    duration: const Duration(milliseconds: 2000), vsync: this);
```
AnimationController派生自Animation<double>，因此可以在需要Animation对象的任何地方使用。

但是，AnimationController具有控制动画的其他方法。例如，.forward()方法可以启动动画。数字的产生与屏幕刷新有关，因此每秒钟通常会产生60个数字，在生成每个数字后，每个Animation对象调用添加的Listener对象。

当创建一个AnimationController时，需要传递一个vsync参数，存在vsync时会防止屏幕外动画（译者语：动画的UI不在当前屏幕时）消耗不必要的资源。

通过将SingleTickerProviderStateMixin添加到类定义中，可以将stateful对象作为vsync的值。

vsync对象会绑定动画的定时器到一个可视的widget，所以当widget不显示时，动画定时器将会暂停，当widget再次显示时，动画定时器重新恢复执行，这样就可以避免动画相关UI不在当前屏幕时消耗资源。 如果要使用自定义的State对象作为vsync时，请包含TickerProviderStateMixin。

### 5、Tween，补间动画
默认情况下，AnimationController对象的范围从0.0到1.0。如果您需要不同的范围或不同的数据类型，则可以使用Tween来配置动画以生成不同的范围或数据类型的值。例如，以下示例，Tween生成从-200.0到0.0的值：
`final Tween doubleTween = new Tween<double>(begin: -200.0, end: 0.0);`

Tween是一个无状态(stateless)对象，需要begin和end值。Tween的唯一职责就是定义从输入范围到输出范围的映射。输入范围通常为0.0到1.0，但这不是必须的。

Tween继承自Animatable<T>，而不是继承自Animation<T>。Animatable与Animation相似，不是必须输出double值。例如，ColorTween指定两种颜色之间的过渡。

`final Tween colorTween =
    new ColorTween(begin: Colors.transparent, end: Colors.black54);`

- 1）Tween对象不存储任何状态。相反，它提供了evaluate(Animation<double> animation)方法将映射函数应用于动画当前值。 Animation对象的当前值可以通过value()方法取到。evaluate函数还执行一些其它处理，例如分别确保在动画值为0.0和1.0时返回开始和结束状态。
- 2）Tween.animate
要使用Tween对象，请调用其animate()方法，传入一个控制器对象。例如，以下代码在500毫秒内生成从0到255的整数值。
```java
final AnimationController controller = new AnimationController(
    duration: const Duration(milliseconds: 500), vsync: this);
Animation<int> alpha = new IntTween(begin: 0, end: 255).animate(controller);
```
注意animate()返回的是一个Animation，而不是一个Animatable。
以下示例构建了一个控制器、一条曲线和一个Tween：
```java
final AnimationController controller = new AnimationController(
    duration: const Duration(milliseconds: 500), vsync: this);
final Animation curve =
    new CurvedAnimation(parent: controller, curve: Curves.easeOut);
Animation<int> alpha = new IntTween(begin: 0, end: 255).animate(curve);
```

### 6、动画通知
一个Animation对象可以拥有Listeners和StatusListeners监听器，可以用addListener()和addStatusListener()来添加。 只要动画的值发生变化，就会调用监听器。一个Listener最常见的行为是调用setState()来触发UI重建。动画开始、结束、向前移动或向后移动（如AnimationStatus所定义）时会调用StatusListener。

## 三、动画示例
概述：
如何通过addListener()和setState()给widget添加基础的动画。
每次动画生成一个新数字时，监听函数都会调用setState()。
如何使用必需的vsync参数定义AnimatedController
### 1、渲染动画
没有添加动画的代码如下：
```java
import 'package:flutter/material.dart';

class LogoApp extends StatefulWidget {
  _LogoAppState createState() => new _LogoAppState();
}

class _LogoAppState extends State<LogoApp> {
  Widget build(BuildContext context) {
    return new Center(
      child: new Container(
        margin: new EdgeInsets.symmetric(vertical: 10.0),
        height: 300.0,
        width: 300.0,
        child: new FlutterLogo(),
      ),
    );
  }
}

void main() {
  runApp(new LogoApp());
}
```
添加AnimationController，传入一个vsync对象，实现一个逐渐放大的动画显示logo，高亮部分为修改的代码：
```java
import 'package:flutter/animation.dart';
import 'package:flutter/material.dart';

class LogoApp extends StatefulWidget {
  _LogoAppState createState() => new _LogoAppState();
}

class _LogoAppState extends State<LogoApp> with SingleTickerProviderStateMixin {
  Animation<double> animation;
  AnimationController controller;

  initState() {
    super.initState();
    controller = new AnimationController(
        duration: const Duration(milliseconds: 2000), vsync: this);
    animation = new Tween(begin: 0.0, end: 300.0).animate(controller)
      ..addListener(() {//依赖animate返回的值调用addListener方法
        setState(() {
          // the state that has changed here is the animation object’s value
        });
      });
    controller.forward();
  }

  Widget build(BuildContext context) {
    return new Center(
      child: new Container(
        margin: new EdgeInsets.symmetric(vertical: 10.0),
        height: animation.value,
        width: animation.value,
        child: new FlutterLogo(),
      ),
    );
  }

  dispose() {
    controller.dispose();
    super.dispose();
  }
}

void main() {
  runApp(new LogoApp());
}
```
### 2、用AnimatedWidget简化
概述：
使用AnimatedWidget创建一个可重用动画的widget。要从widget中分离出动画过渡，请使用AnimatedBuilder。

Flutter API提供的关于AnimatedWidget的示例包括：AnimatedBuilder、AnimatedModalBarrier、DecoratedBoxTransition、FadeTransition、PositionedTransition、RelativePositionedTransition、RotationTransition、ScaleTransition、SizeTransition、SlideTransition。

AnimatedWidget类允许您从setState()调用中的动画代码中分离出widget代码。AnimatedWidget不需要维护一个State对象来保存动画。

在下面的重构示例中，LogoApp现在继承自AnimatedWidget而不是StatefulWidget。AnimatedWidget在绘制时使用动画的当前值。LogoApp仍然管理着AnimationController和Tween。
```java
// Demonstrate a simple animation with AnimatedWidget

import 'package:flutter/animation.dart';
import 'package:flutter/material.dart';

class AnimatedLogo extends AnimatedWidget {
  AnimatedLogo({Key key, Animation<double> animation})
      : super(key: key, listenable: animation);

  Widget build(BuildContext context) {
    final Animation<double> animation = listenable;
    return new Center(
      child: new Container(
        margin: new EdgeInsets.symmetric(vertical: 10.0),
        height: animation.value,
        width: animation.value,
        child: new FlutterLogo(),
      ),
    );
  }
}

class LogoApp extends StatefulWidget {
  _LogoAppState createState() => new _LogoAppState();
}

class _LogoAppState extends State<LogoApp> with SingleTickerProviderStateMixin {
  AnimationController controller;
  Animation<double> animation;

  initState() {
    super.initState();
    controller = new AnimationController(
        duration: const Duration(milliseconds: 2000), vsync: this);
    animation = new Tween(begin: 0.0, end: 300.0).animate(controller);
    controller.forward();
  }

  Widget build(BuildContext context) {
    return new AnimatedLogo(animation: animation);
  }

  dispose() {
    controller.dispose();
    super.dispose();
  }
}

void main() {
  runApp(new LogoApp());
}
```
LogoApp将Animation对象传递给基类并用animation.value设置容器的高度和宽度，AnimatedWidget(基类)中会自动调用addListener()和setState()。
### 3、监视动画的过程
概述：
使用addStatusListener来处理动画状态更改的通知，例如启动、停止或反转方向。
当动画完成或返回其开始状态时，通过反转方向来无限循环运行动画.

知道动画何时改变状态通常很有用的，如完成、前进或倒退。你可以通过addStatusListener()来得到这个通知。通过修改上述示例的代码实现：
```java
// Demonstrate a simple animation with AnimatedWidget

import 'package:flutter/animation.dart';
import 'package:flutter/material.dart';

class AnimatedLogo extends AnimatedWidget {
  AnimatedLogo({Key key, Animation<double> animation})
      : super(key: key, listenable: animation);

  Widget build(BuildContext context) {
    final Animation<double> animation = listenable;
    return new Center(
      child: new Container(
        margin: new EdgeInsets.symmetric(vertical: 10.0),
        height: animation.value,
        width: animation.value,
        child: new FlutterLogo(),
      ),
    );
  }
}

class LogoApp extends StatefulWidget {
  _LogoAppState createState() => new _LogoAppState();
}

class _LogoAppState extends State<LogoApp> with SingleTickerProviderStateMixin {
  AnimationController controller;
  Animation<double> animation;

  initState() {
    super.initState();
    controller = new AnimationController(
        duration: const Duration(milliseconds: 2000), vsync: this);
    animation = new Tween(begin: 0.0, end: 300.0).animate(controller);
    animation.addStatusListener((status) {
      if (status == AnimationStatus.completed) {
        controller.reverse();
      } else if (status == AnimationStatus.dismissed) {
        controller.forward();
      }
    });
    controller.forward();
  }

  Widget build(BuildContext context) {
    return new AnimatedLogo(animation: animation);
  }

  dispose() {
    controller.dispose();
    super.dispose();
  }
}

void main() {
  runApp(new LogoApp());
}
```

### 4、用AnimatedBuilder重构
概述：

1. AnimatedBuilder

不知道如何渲染widget，也不知道如何管理Animation对象。
使用AnimatedBuilder将动画描述为另一个widget的build方法的一部分。如果你只是想用可复用的动画定义一个widget，请使用AnimatedWidget。

Flutter API中AnimatedBuilder的示例包括: `BottomSheet`、`ExpansionTile`、 `PopupMenu`、`ProgressIndicator`、`RefreshIndicator`、`Scaffold`、`SnackBar`、`TabBar`、`TextField`。

AnimatedBuilder是渲染树中的一个独立的类。 与AnimatedWidget类似，AnimatedBuilder自动监听来自Animation对象的通知，并根据需要将该控件树标记为脏(dirty)，因此不需要手动调用addListener()。

2. AnimatedBuilder可以将职责分离：

- 显示logo
- 定义Animation对象
- 渲染过渡效果

渲染流程图如下：
> 图片来自其他网站，忘记出处了，后续找到了加上去

![](https://i.loli.net/2019/06/03/5cf4db0e3537379684.png)

图中的中间三个块都在GrowTransition的build()方法中创建。GrowTransition本身是无状态的，并拥有定义过渡动画所需的最终变量集合。 

- 1、build()函数创建并返回AnimatedBuilder
- 2、将（匿名构建器）方法和LogoWidget对象作为参数
- 3、渲染转换的工作实际上发生在（匿名构建器）方法中， 该方法创建一个适当大小的Container来强制缩放LogoWidget。
```java
class GrowTransition extends StatelessWidget {
  GrowTransition({this.child, this.animation});

  final Widget child;
  final Animation<double> animation;

  Widget build(BuildContext context) {
    return new Center(
      child: new AnimatedBuilder(
          animation: animation,
          builder: (BuildContext context, Widget child) {
            return new Container(
                //child:要播放动画的widget对象
                height: animation.value, width: animation.value, child: child);
          },
          //child:要插入的widget对象
          child: child),
    );
  }
}
```
child看起来像被指定了两次。但实际发生的事情是，将外部引用child传递给AnimatedBuilder，AnimatedBuilder将其传递给匿名构造器， 然后将该对象用作其子对象。最终的结果是AnimatedBuilder插入到渲染树中的两个widget之间。
```java
class LogoApp extends StatefulWidget {
  _LogoAppState createState() => new _LogoAppState();
}

class _LogoAppState extends State<LogoApp> with TickerProviderStateMixin {
  Animation animation;
  AnimationController controller;

  initState() {
    super.initState();
    controller = new AnimationController(
        duration: const Duration(milliseconds: 2000), vsync: this);
    final CurvedAnimation curve =
        new CurvedAnimation(parent: controller, curve: Curves.easeIn);
    animation = new Tween(begin: 0.0, end: 300.0).animate(curve);
    controller.forward();
  }

  Widget build(BuildContext context) {
    return new GrowTransition(child: new LogoWidget(), animation: animation);
  }

  dispose() {
    controller.dispose();
    super.dispose();
  }
}

void main() {
  runApp(new LogoApp());
}
```
initState()方法创建一个AnimationController和一个Tween，然后通过animate()绑定它们。在build()方法中，返回一个带有LogoWidget作为子对象的GrowTransition对象，以及一个用于驱动过渡的动画对象。

完整的代码如下：
```java
import 'package:flutter/animation.dart';
import 'package:flutter/material.dart';

class LogoWidget extends StatelessWidget {
  // Leave out the height and width so it fills the animating parent
  build(BuildContext context) {
    return new Container(
      margin: new EdgeInsets.symmetric(vertical: 10.0),
      child: new FlutterLogo(),
    );
  }
}

class GrowTransition extends StatelessWidget {
  GrowTransition({this.child, this.animation});

  final Widget child;
  final Animation<double> animation;

  Widget build(BuildContext context) {
    return new Center(
      child: new AnimatedBuilder(
          animation: animation,
          builder: (BuildContext context, Widget child) {
            return new Container(
                height: animation.value, width: animation.value, child: child);
          },
          child: child),
    );
  }
}

class LogoApp extends StatefulWidget {
  _LogoAppState createState() => new _LogoAppState();
}

class _LogoAppState extends State<LogoApp> with TickerProviderStateMixin {
  Animation animation;
  AnimationController controller;

  initState() {
    super.initState();
    controller = new AnimationController(
        duration: const Duration(milliseconds: 2000), vsync: this);
    final CurvedAnimation curve =
        new CurvedAnimation(parent: controller, curve: Curves.easeIn);
    animation = new Tween(begin: 0.0, end: 300.0).animate(curve);
    controller.forward();
  }

  Widget build(BuildContext context) {
    return new GrowTransition(child: new LogoWidget(), animation: animation);
  }

  dispose() {
    controller.dispose();
    super.dispose();
  }
}

void main() {
  runApp(new LogoApp());
}
```

### 5、并行动画
在同一个动画控制器上使用多个Tween，其中每个Tween管理动画中的不同效果。如果在生产代码中需要使用不透明度和大小变化的Tween，则可能会使用FadeTransition和SizeTransition。

每一个Tween管理动画的一种效果。例如：
```java
final AnimationController controller =
    new AnimationController(duration: const Duration(milliseconds: 2000), vsync: this);
final Animation<double> sizeAnimation =
    new Tween(begin: 0.0, end: 300.0).animate(controller);
final Animation<double> opacityAnimation =
    new Tween(begin: 0.1, end: 1.0).animate(controller);
```
可以通过sizeAnimation.value来获取大小，通过opacityAnimation.value来获取不透明度，但AnimatedWidget的构造函数只接受一个动画对象。 为了解决这个问题，该示例创建了自己的Tween对象并显式计算了这些值。

其build方法.evaluate()在父级的动画对象上调用Tween函数以计算所需的size和opacity值。

完整代码如下:
```java
import 'package:flutter/animation.dart';
import 'package:flutter/material.dart';

class AnimatedLogo extends AnimatedWidget {
  // The Tweens are static because they don't change.
  static final _opacityTween = new Tween<double>(begin: 0.1, end: 1.0);
  static final _sizeTween = new Tween<double>(begin: 0.0, end: 300.0);

  AnimatedLogo({Key key, Animation<double> animation})
      : super(key: key, listenable: animation);

  Widget build(BuildContext context) {
    final Animation<double> animation = listenable;
    return new Center(
      child: new Opacity(
        opacity: _opacityTween.evaluate(animation),
        child: new Container(
          margin: new EdgeInsets.symmetric(vertical: 10.0),
          height: _sizeTween.evaluate(animation),
          width: _sizeTween.evaluate(animation),
          child: new FlutterLogo(),
        ),
      ),
    );
  }
}

class LogoApp extends StatefulWidget {
  _LogoAppState createState() => new _LogoAppState();
}

class _LogoAppState extends State<LogoApp> with TickerProviderStateMixin {
  AnimationController controller;
  Animation<double> animation;

  initState() {
    super.initState();
    controller = new AnimationController(
        duration: const Duration(milliseconds: 2000), vsync: this);
    animation = new CurvedAnimation(parent: controller, curve: Curves.easeIn);

    animation.addStatusListener((status) {
      if (status == AnimationStatus.completed) {
        controller.reverse();
      } else if (status == AnimationStatus.dismissed) {
        controller.forward();
      }
    });

    controller.forward();
  }

  Widget build(BuildContext context) {
    return new AnimatedLogo(animation: animation);
  }

  dispose() {
    controller.dispose();
    super.dispose();
  }
}

void main() {
  runApp(new LogoApp());
}
```

### 6、AnimatedOpacity
这是一个widget，有几个子属性需要定义：

- opacity: 0.0 (不可见) 到 1.0 (可见).
- duration: 动画多长时间执行完
- child: 执行动画的widget

完整示例：
```java
import 'package:flutter/material.dart';

void main() => runApp(MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final appTitle = 'Opacity Demo';
    return MaterialApp(
      title: appTitle,
      home: MyHomePage(title: appTitle),
    );
  }
}

// The StatefulWidget's job is to take in some data and create a State class.
// In this case, our Widget takes in a title, and creates a _MyHomePageState.
class MyHomePage extends StatefulWidget {
  final String title;

  MyHomePage({Key key, this.title}) : super(key: key);

  @override
  _MyHomePageState createState() => _MyHomePageState();
}

// The State class is responsible for two things: holding some data we can
// update and building the UI using that data.
class _MyHomePageState extends State<MyHomePage> {
  // Whether the green box should be visible or invisible
  bool _visible = true;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.title),
      ),
      body: Center(
        child: AnimatedOpacity(
          // If the Widget should be visible, animate to 1.0 (fully visible). If
          // the Widget should be hidden, animate to 0.0 (invisible).
          opacity: _visible ? 1.0 : 0.0,
          duration: Duration(milliseconds: 500),
          // The green box needs to be the child of the AnimatedOpacity
          child: Container(
            width: 200.0,
            height: 200.0,
            color: Colors.green,
          ),
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          // Make sure we call setState! This will tell Flutter to rebuild the
          // UI with our changes!
          setState(() {
            _visible = !_visible;
          });
        },
        tooltip: 'Toggle Opacity',
        child: Icon(Icons.flip),
      ), // This trailing comma makes auto-formatting nicer for build methods.
    );
  }
}
```
