---
layout: post
title:  "Flutter InheritedWidget机制"
date:   2020-5-19 18:19:25 +0800
categories: flutter
---


> 微信公众号：Android部落格，文末有二维码


## 1、用法
用法示例：

```dart
class InheritedData extends InheritedWidget {
  final String data;

  InheritedData({
    this.data,
    Widget child,
  }) : super(child: child);

  static InheritedData of(BuildContext context) {
    return context.dependOnInheritedWidgetOfExactType<InheritedData>();
  }

  @override
  bool updateShouldNotify(InheritedData old) => true;
}

class TestInheritedDataWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Center(
      child: Text(
        "${InheritedData.of(context).data}",
        style: TextStyle(fontSize: 18, color: Colors.pink),
      ),
    );
  }
}

class ParentWidget extends StatefulWidget {
  @override
  _ParentWidgetState createState() => _ParentWidgetState();
}

class _ParentWidgetState extends State<ParentWidget> {
  var data = "you are beautiful";

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: GestureDetector(
        child: Container(
          color: Colors.cyan,
          width: double.infinity,
          child: InheritedData(
            data: data,
            child: TestInheritedDataWidget(),
          ),
        ),
        onTap: _buttonClicked,
      ),
    );
  }

  _buttonClicked() {
    setState(() {
      data = "in white";
    });
  }
}
```

## 2、ProxyWidget
InheritedWidget继承自ProxyWidget，在ProxyWidget中createElement调用的时候，创建了一个InheritedElement对象。
### 2.1 InheritedElement
InheritedElement继承自ProxyElement，ProxyElement继承自ComponentElement，ComponentElement继承自Element。

在InheritedElement中定义了一个全局Map对象：

```dart
final Map<Element, Object> _dependents = HashMap<Element, Object>();
```

同时还定义了一个_updateInheritance方法。在Element和
InheritedElement中都有定义：
> Element

```dart
void _updateInheritance() {
    _inheritedWidgets = _parent?._inheritedWidgets;
}
```
```Map<Type, InheritedElement> _inheritedWidgets```定义在Element中，是一个全局变量。

> InheritedElement

```dart
@override
void _updateInheritance() {
    final Map<Type, InheritedElement> incomingWidgets = _parent?._inheritedWidgets;
    if (incomingWidgets != null)
      _inheritedWidgets = HashMap<Type, InheritedElement>.from(incomingWidgets);
    else
      _inheritedWidgets = HashMap<Type, InheritedElement>();
    _inheritedWidgets[widget.runtimeType] = this;
}
```
InheritedElement中这个方法的主要逻辑是：先判断当前Element的_inheritedWidgets是否存在，不存在，新建一个，否则将已有的添加到_inheritedWidgets中，同时将当前Element添加到_inheritedWidgets中。

Element中的_updateInheritance方法被执行意味着，所有的Widget对应的Element都持有一个_inheritedWidgets，而InheritedWidget的child就可以通过Element的_updateInheritance方法直接持有InheritedElement。

### 2.2 保存InheritedElement节点
那么_updateInheritance是哪里调用的呢？

参考之前的文章《flutter 绘制过程 系列2-布局》，在Element类的mount方法中，会调用_updateInheritance方法，最终就调用到这里了。

## 3、ProxyElement中更新
当InheritedWidget中的data被setState更新之后，依赖data的Widget也会被更新。

### 3.1 ProxyElement update方法

```dart
@override
void update(ProxyWidget newWidget) {
    final ProxyWidget oldWidget = widget;
    super.update(newWidget);
    updated(oldWidget);
    _dirty = true;
    rebuild();
}
```
到updated方法，在InheritedElement方法中：
> InheritedElement

```dart
@override
void updated(InheritedWidget oldWidget) {
    if (widget.updateShouldNotify(oldWidget))
      super.updated(oldWidget);
}
```
如果我们InheritedWidget中的updateShouldNotify返回为true的话，会执行到InheritedElement的updated方法中：

```dart
@protected
void updated(covariant ProxyWidget oldWidget) {
    notifyClients(oldWidget);
}
```
notifyClients执行到了子类InheritedElement的notifyClients方法。

### 3.2 InheritedElement notifyClients方法

```dart
@override
void notifyClients(InheritedWidget oldWidget) {
    for (final Element dependent in _dependents.keys) {
      notifyDependent(oldWidget, dependent);
    }
}
```
这里的_dependents里面保存的是依赖InheritedWidget数据的节点。

### 3.3 _dependents保存依赖数据的Element

本章最开始的时候InheritedData提供了一个静态的of方法，依赖者(InheritedWidget的child Widget)调用dependOnInheritedWidgetOfExactType方法。在Element类中实现，BuildContext类中定义:

> **Element实现了BuildContext接口**

> Element

```dart
@override
T dependOnInheritedWidgetOfExactType<T extends InheritedWidget>({Object aspect}) {
    final InheritedElement ancestor = _inheritedWidgets == null ? null : _inheritedWidgets[T];
    if (ancestor != null) {
      return dependOnInheritedElement(ancestor, aspect: aspect) as T;
    }
    _hadUnsatisfiedDependencies = true;
    return null;
}
```
这里先找到InheritedElement，通过_inheritedWidgets的key（runtimeType）找到对应的value（InheritedElement）。接下来调用dependOnInheritedElement方法：

> Element

```dart
@override
InheritedWidget dependOnInheritedElement(InheritedElement ancestor, { Object aspect }) {
    _dependencies ??= HashSet<InheritedElement>();
    _dependencies.add(ancestor);
    ancestor.updateDependencies(this, aspect);
    return ancestor.widget;
}
```
在Element类中，先判断_dependencies是否为空，为空的话新建一个HashSet，把这个InheritedElement添加进去，然后执行updateDependencies方法。这个this指代的就是InheritedWidget的child Widget的Element。

这个updateDependencies定义在对应的InheritedElement类中：
> InheritedElement

```dart
@override
void updateDependencies(Element dependent, Object aspect) {
    final Set<T> dependencies = getDependencies(dependent) as Set<T>;
    if (dependencies != null && dependencies.isEmpty)
      return;
    
    if (aspect == null) {
      setDependencies(dependent, HashSet<T>());
    } else {
      setDependencies(dependent, (dependencies ?? HashSet<T>())..add(aspect as T));
    }
}

@protected
void setDependencies(Element dependent, Object value) {
    _dependents[dependent] = value;
}
```
到这里基本上清楚了基本的脉络，_dependents里面保存的就是是InheritedWidget的child Widget的Element。

### 3.4 提醒依赖Element更新
回到InheritedElement notifyClients方法，这里会遍历_dependents，然后执行notifyDependent方法：
> InheritedElement

```dart
@protected
void notifyDependent(covariant InheritedWidget oldWidget, Element dependent) {
    dependent.didChangeDependencies();
}

@mustCallSuper
void didChangeDependencies() {
    markNeedsBuild();
}
```

到这里重新绘制渲染依赖InheritedWidget数据的Widget。

![微信公众号](https://ftp.bmp.ovh/imgs/2020/04/f472fa7e9c62a8c7.jpg)


