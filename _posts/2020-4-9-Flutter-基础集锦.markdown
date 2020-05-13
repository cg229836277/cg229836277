---
layout: post
title:  "Flutter-基础集锦"
date:   2020-4-9 11:09:22 +0800
categories: flutter
---

# View篇

## 有几种视图框架

总体来说有两种，Column和Row，前者表示竖直方向，后者表示水平方向。

## 怎么实现类似wrap_content和match_parent的效果

```dart
Widget parent = Container(
  width: 360,
  height: 360,
  color: Colors.lightGreen,
  child: Column(
    mainAxisSize: MainAxisSize.max,
    children: <Widget>[
      Container(
        width: 120,
        height: 86,
        color: Colors.brown,
      ),
      Container(
        width: 120,
        height: 86,
        color: Colors.red,
      ),
      Container(
        width: 120,
        height: 86,
        color: Colors.cyan,
      ),
    ],
  ),
);
```

可以看到Container里面包含的Column面积设置的多大就展示多大的区域，主要起作用的是mainAxisSize和mainAxisAlignment参数。

![](https://ftp.bmp.ovh/imgs/2020/01/9b586101e6b2f117.png)

#### mainAxisSize 

表示剩余空间的占用情况，是最大限度(max)还是最小限度(min)的占用剩余空间

#### mainAxisAlignment

表示Column或Row里面子布局相互之间的空间应该怎么分布。
- MainAxisAlignment.start 表示所有子控件往首部的区域集中，剩余空间集中在尾部
- MainAxisAlignment.spaceBetween 表示所有子控件之间平分剩余的空间，不包含首尾
- MainAxisAlignment.center 表示剩余的空间平均分配到子控件的首胃，而子控件之间没有空间
- MainAxisAlignment.spaceEvenly 表示子控件之间，包含首尾都要均分剩余空间
- MainAxisAlignment.end 表示所有子控件往尾部的区域集中，剩余空间集中在首部
- MainAxisAlignment.spaceAround 剩余空间除以子控件数量 + 1，子控件之间的间距是等分的，多出来的空间除以2，分布于子控件的首尾。比如剩余空间是42，有2个子控件，控件之间的间隔是42 / (2 + 1) = 14,控件之间的间距是14，而两个控件前后的间隔是14 / 2 = 7.

#### crossAxisAlignment

这个参数决定了与排布方向垂直方向子控件的位置。与mainAxisAlignment是相对的。

![](https://ftp.bmp.ovh/imgs/2020/01/3819f9f1125f5816.png)

还是利用上面的代码做例子说明：

```dart
Widget parent = Container(
  width: 360,
  height: 360,
  color: Colors.lightGreen,
  child: Column(
    mainAxisSize: MainAxisSize.max,
    mainAxisAlignment: MainAxisAlignment.spaceAround,
    crossAxisAlignment: CrossAxisAlignment.end,
    children: <Widget>[
      Container(
        width: 120,
        height: 86,
        color: Colors.brown,
      ),
      Container(
        width: 120,
        height: 86,
        color: Colors.red,
      ),
      Container(
        width: 120,
        height: 86,
        color: Colors.cyan,
      ),
    ],
  ),
);
```
子控件之间和首尾都有了空间，另外子控件对齐最右边。如下：
![](https://ftp.bmp.ovh/imgs/2020/01/04878d2ef53cf0e0.png)

到这里，我们就知道，Column的Main方向是竖直的，Cross方向是水平的，而Row正好相反，Main方向是水平的，Cross方向是竖直的。

综述一下，要实现wrap_content效果就设置mainAxisSize为min，并且mainAxisAlignment设置为start;要实现match_parent就设置mainAxisSize为max，并且mainAxisAlignment不要设置为start或end或center。

## 怎么设置视图的宽高

一般情况下使用Container里面的width和height设置宽高。

当然了，如果不想设置具体的宽高，只想设置一个大体的约束，可以设置Container里面的constraints属性，这个属性对应的类是BoxConstraints类，这个类有四个参数可以设置，如下：

```dart
BoxConstraints(
    minHeight: 360, 
    minWidth: 360, 
    maxWidth: 360, 
    maxHeight: 360);
```

## 怎么设置控件的背景颜色

Container里面有color参数可以设置背景颜色，前提是Container没有显式的设置decoration属性。

decoration属性用于为Container设置装饰，这个装饰属性对应的是BoxDecoration类，如果设置了这个属性就不能在Container中设置color的值了，只能通过decoration设置。

## 怎么设置点击之后的波纹效果

InkWell可以满足需求，父Widget用InkWell，然后child参数设置为要点击的子Widget。

## 怎么设置字体以及富文本

- 普通文本

```dart
Text(
    "flutter教程",
    textAlign: TextAlign.center,
    style: TextStyle(
      fontSize: 12,
      color: Colors.lightGreenAccent,
      fontWeight: FontWeight.bold,
      decoration: TextDecoration.lineThrough,
    ),
)
```

- 富文本

```dart
RichText(
    text: TextSpan(
        text: "first text",
        style: TextStyle(color: Colors.white, fontSize: 18),
        children: <TextSpan>[
          TextSpan(
            text: "second text",
            style: TextStyle(color: Colors.red, fontSize: 12),
          ),
        ],
    ),
);
```

## 怎么设置控件圆角

```dart
Container(
    decoration: BoxDecoration(
        borderRadius: BorderRadius.all(Radius.circular(12)))
);
```

## 怎么将控件背景设置为我想要的icon

```dart
Container(
  decoration: BoxDecoration(
    image: DecorationImage(
      image: AssetImage("images/your image source.png"),
      fit: BoxFit.cover,
    ),
  ),
);
```

## 怎么设置控件背景渐变

```dart
Container(
  decoration: BoxDecoration(
    gradient: LinearGradient(
        begin: Alignment.centerLeft,//起始方向
        end: Alignment.centerRight,//终止方向
        colors: [
          ColorUtil.hexToColor("#FF8602"),
          ColorUtil.hexToColor("#FE3F01")
    ]),
  ),
);
```

## 怎么为控件设置点击事件

```dart
Widget backWidget = GestureDetector(
  child: Image.asset(
    "images/your image source",
    width: 24,
    height: 24,
  ),
  onTap: () => Navigator.of(context).pop(),//your own action
);
```
## 控件怎么居中

- 第一种方法

```dart
Align(
    alignment: Alignment.center,
    child: Text(
      "flutter教程",
      textAlign: TextAlign.center,
      style: TextStyle(
        fontSize: 12,
        color: Colors.lightGreenAccent,
        fontWeight: FontWeight.bold,
        decoration: TextDecoration.lineThrough,
      ),
    ),
);
```

- 第二种方法

```dart
Center(
    child: Text(
      "flutter教程",
      textAlign: TextAlign.center,
      style: TextStyle(
        fontSize: 12,
        color: Colors.lightGreenAccent,
        fontWeight: FontWeight.bold,
        decoration: TextDecoration.lineThrough,
      ),
    ),
)
```

## 控件怎么设置上下左右的margin

```dart
Widget parent = Container(
  margin: EdgeInsets.only(top: 40, bottom:40, left:40, right:40),
  child: childWidget,
);
```

## 控件怎么设置padding

```dart
Padding(
    child: Text("your own widget")
    padding: EdgeInsets.only(top: 40, bottom:40, left:40, right:40),
),
```

## 图片怎么设置圆角

```dart
ClipRRect(
  borderRadius: new BorderRadius.circular(8.0),
  child: Image.network(
    goodsImage,
    width: 80,
    height: 80,
   ),
),
```

## 列表怎么用

- ListView.builder

```dart
ListView.builder(
    itemBuilder: (context, index) {
      return Text("your child list item widget");
    },
    itemCount: dataSet.length,
),
```

- ListView.separated可以设置子item之间的分割，separatorBuilder属性返回间隔widget

```dart
Widget parent = ListView.separated(
    itemBuilder: (BuildContext context, int index) {
      return Text("this is index $index");
    },
    separatorBuilder: (BuildContext context, int index) {
      return Container(
        width: 360,
        height: 8,
        color: Colors.green,
      );
    },
    itemCount: 20);
```

- ListView.custom可以自定义子item的展现模式

```dart
Widget parent = ListView.custom(
  childrenDelegate: SliverChildBuilderDelegate(
    (BuildContext context, int index) {
      return Text("this is index $index");
    },
    childCount: 20,
  ),
);
```

## 滚动视图怎么用

- SingleChildScrollView 只能包含有一个child

```dart
Widget parent = SingleChildScrollView(
  child: Column(
    children: <Widget>[
      Container(
        color: Colors.blue,
        child: Text("this is 1"),
        width: 360,
        height: 240,
      ),
      Container(
        color: Colors.red,
        child: Text("this is 2"),
        width: 360,
        height: 240,
      ),
      Container(
        color: Colors.green,
        child: Text("this is 3"),
        width: 360,
        height: 240,
      ),
      Container(
        color: Colors.cyan,
        child: Text("this is 4"),
        width: 360,
        height: 240,
      ),
      Container(
        color: Colors.yellow,
        child: Text("this is 5"),
        width: 360,
        height: 240,
      ),
    ],
  ),
);
```

- CustomScrollView 这个控件比较强大，里面可以有多个child widget,并且必须是sliver属性的控件。

> 不包含appBar

```dart
CustomScrollView(
    controller: _scrollController,
    slivers: <Widget>[
      SliverList(
          delegate: SliverChildBuilderDelegate(
        (context, index) {
          return Text("child item");
        },
        childCount: dataSet.length,
      )),
      SliverToBoxAdapter(child: footerView),
    ],
);
```

其实有sliver属性的控件还有SliverGrid，SliverAppBar等。

- NestedScrollView 有收缩伸展appBar能力的滚动控件

```dart
Widget childContent = NestedScrollView(
    headerSliverBuilder: (context, innerBoxIsScrolled) => [
          SliverOverlapAbsorber(
            child: SliverSafeArea(
              top: false,
              sliver: SliverAppBar(
                backgroundColor: Colors.white,
                expandedHeight: expandedHeight,
                flexibleSpace: featureParentWidget,
                pinned: true,
                floating: true,
                elevation: 179,
                bottom: PreferredSize(
                  child: sortItemWidget,
                  preferredSize: Size(double.infinity, 42),
                ),
              ),
            ),
            handle:
              NestedScrollView.sliverOverlapAbsorberHandleFor(context),
          ),
        ],
    body: body);
```
**expandedHeight** 指可以扩展的高度

**flexibleSpace** 在appbar里面还可以设置弹性空间的widget，比如希望appbar伸展之后的背景是一张图片，就可以在这里设置

**pinned**
appbar是否应该保留在滚动视图顶部

**floating**
用户向appbar滚动时，是否应立即显示appbar。

其他参数参考：
> https://api.flutter.dev/flutter/material/SliverAppBar-class.html

各个参数的示例图片如下：
![](https://flutter.github.io/assets-for-api-docs/assets/material/app_bar.png)

## TabLayout + ViewPager怎么实现
使用TabBar和TabBarView实现，具体实现方式如下：

- TabBar实现title

```dart
Widget titleBar = Container(
  height: 48,
  color: Colors.green,
  margin: EdgeInsets.only(top: 24),
  child: TabBar(
    tabs: titleArray.map((f) {
      return Text(
        f,
        style: TextStyle(color: Colors.red, fontSize: 14),
      );
    }).toList(),
    onTap: _tabClicked,
    controller: _controller,
    indicatorSize: TabBarIndicatorSize.label,
    isScrollable: false,
    indicatorColor: Colors.pink,
  ),
);
```

- TabBarView实现

```dart
Widget contentView = Container(
    color: Colors.amber,
    child: TabBarView(
      controller: _controller,
      children: titleArray.map((f) {
        return Center(
          child: Text(
            "this is $f",
            style: TextStyle(color: Colors.red, fontSize: 24),
          ),
        );
      }).toList(),
    ));
```
一个实现了Tab标题，一个实现了Tab下的具体页面内容，两者产生关系的纽带是_controller变量，这个变量是TabController类型。

全部实现代码如下：

```dart
class TabViewDemoWidget extends StatelessWidget {
  TabController _controller;
  var initSelectedIndex = 0;

  final titleArray = ["A", "B", "C", "D", "E"];

  _tabScrollListener() {
    if (_controller.index != initSelectedIndex) {
      //todo tab changed
    }
  }

  @override
  Widget build(BuildContext context) {
    _controller = TabController(
      initialIndex: initSelectedIndex, //初始选中页
      length: 5, //总页数
      vsync: ScrollableState(),
    );

    _controller.addListener(_tabScrollListener);

    Widget titleBar = Container(
      height: 48,
      color: Colors.green,
      margin: EdgeInsets.only(top: 24),
      child: TabBar(
        tabs: titleArray.map((f) {
          return Text(
            f,
            style: TextStyle(color: Colors.red, fontSize: 14),
          );
        }).toList(),
        onTap: _tabClicked,
        controller: _controller,
        indicatorSize: TabBarIndicatorSize.label,
        isScrollable: false,
        indicatorColor: Colors.pink,
      ),
    );

    Widget contentView = Container(
        color: Colors.amber,
        child: TabBarView(
          controller: _controller,
          children: titleArray.map((f) {
            return Center(
              child: Text(
                "this is $f",
                style: TextStyle(color: Colors.red, fontSize: 24),
              ),
            );
          }).toList(),
        ));
    return Scaffold(
      appBar: PreferredSize(
        child: titleBar,
        preferredSize: Size(double.infinity, 48),
      ),
      body: contentView,
    );
  }

  _tabClicked(int index) {}
}
```

## 底部导航栏怎么实现
Scaffold有一个bottomNavigationBar属性，设置这个属性就可以实现，具体实现如下：

```dart
Scaffold(
    body: Container(),//define your own body content
    bottomNavigationBar: BottomNavigationBar(
        currentIndex: _selectedIndex,
        type: BottomNavigationBarType.fixed,
        selectedFontSize: 12,
        unselectedFontSize: 12,
        unselectedItemColor: Color.fromARGB(0XFF, 0X40, 0X40, 0X40),
        selectedItemColor: Color.fromARGB(0XFF, 0XFE, 0X3F, 0X01),
        items: [
          BottomNavigationBarItem(
              activeIcon: Container(
                width: 24,
                height: 24,
                child: Image.asset("images/main_activity_main_pressed.png"),
              ),
              icon: Container(
                width: 24,
                height: 24,
                child: Image.asset("images/main_activity_main_normal.png"),
              ),
              title: Text(
                "主页",
              )),
          BottomNavigationBarItem(
              activeIcon: Container(
                width: 24,
                height: 24,
                child: Image.asset(
                    "images/main_activity_classify_pressed.png"),
              ),
              icon: Container(
                width: 24,
                height: 24,
                child:
                    Image.asset("images/main_activity_classify_normal.png"),
              ),
              title: Text(
                "分类",
              )),
        ],
        onTap: onItemTaped),
  );
```
onTap 属性要给一个item点击的响应事件，携带index参数。当这个方法触发之后可以做状态改变操作。

## AlertDialog怎么实现
> flutter自身有AlertDialog Widget，但是可能不满足我们的需求，可以自定义

Dialog Widget里面有一个child属性，可以设置自己想要的对话框形式，通过showDialog方法将对话框显示出来。

```dart
class DialogDemoWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return WillPopScope(
        onWillPop: () => _willPopScope(context),
        child: Scaffold(
          body: Center(
              child: Container(
            child: Text("show dialog demo"),
          )),
        ));
  }

  Future<bool> _willPopScope(BuildContext context) async {
    return showDialog(
            context: context,
            builder: (context) {
              //design your own dialog content
              Widget dialogChild = Container(
                width: 302,
                height: 242,
                child: Center(
                  child: Text("this is dialog"),
                ),
              );
              return Dialog(
                backgroundColor: null,
                elevation: 24,
                shape: RoundedRectangleBorder(
                    borderRadius: BorderRadius.all(Radius.circular(24))),
                child: dialogChild,
              );
            }) ??
        false;
  }
}
```

## ProgressDialog怎么实现

ProgressDialog可以通过PopupRoute实现，他的原理是在已有的路由上面再绘制一层。

> 借鉴别人的实现，加自己的改造

```dart
///加载弹框
class ProgressDialog {
  static bool _isShowing = false;

  ///展示
  static void showProgress(BuildContext context, {Widget child}) {
    if (child == null) {//define your own child
      child = Theme(
          data: Theme.of(context)
              .copyWith(accentColor: ColorUtil.hexToColor("#FFFE7501")),
          child: Stack(
            children: <Widget>[
              Center(
                child: Container(
                  width: 46,
                  height: 46,
                  child: new CircularProgressIndicator(
                    strokeWidth: 3,
                  ),
                ),
              ),
              Center(
                  child: Text(
                    "加载中",
                    style: TextStyle(
                        fontSize: 12, color: ColorUtil.hexToColor("#FF202020")),
                  )),
            ],
          ));
    }

    if (!_isShowing) {
      _isShowing = true;
      Navigator.push(
        context,
        _PopRoute(
          child: _Progress(
            child: child,
          ),
        ),
      );
    }
  }

  ///隐藏
  static void dismiss(BuildContext context) {
    if (context == null) {
      return;
    }
    if (_isShowing) {
      Navigator.of(context).pop();
      _isShowing = false;
    }
  }
}

///Widget
class _Progress extends StatelessWidget {
  final Widget child;

  _Progress({
    Key key,
    @required this.child,
  })  : assert(child != null),
        super(key: key);

  @override
  Widget build(BuildContext context) {
    return Material(
        color: Colors.transparent,
        child: Center(
          child: child,
        ));
  }
}

///Route
class _PopRoute extends PopupRoute {
  final Duration _duration = Duration(milliseconds: 300);
  Widget child;

  _PopRoute({@required this.child});

  @override
  Color get barrierColor => null;

  @override
  bool get barrierDismissible => true;

  @override
  String get barrierLabel => null;

  @override
  Widget buildPage(BuildContext context, Animation<double> animation,
      Animation<double> secondaryAnimation) {
    return child;
  }

  @override
  Duration get transitionDuration => _duration;
}
```

## 点击控件之后怎么响应事件更新视图
Widget从大体上可以分为StatelessWidget和StatefullWidget,其中StatefullWidget可以根据setState方法触发对整个视图的刷新。

粗略地，StatefullWidget的生命周期可以分为initState,build,dispose。

在调用了setState方法之后，会触发build方法使子widget重新构建。

方法如下：

```dart
class SetStateDemoWidget extends StatefulWidget {
  @override
  State<StatefulWidget> createState() {
    return SetStateDemoWidgetState();
  }
}

class SetStateDemoWidgetState extends State<SetStateDemoWidget> {
  var index = 0;

  @override
  void initState() {
    super.initState();
  }

  @override
  Widget build(BuildContext context) {
    Widget numberText = Text(
      "$index",
      style: TextStyle(fontSize: 18, color: Colors.cyan),
    );
    Widget addButton = Container(
      margin: EdgeInsets.only(top: 16),
      child: FlatButton(
        onPressed: _buttonClicked,
        child: Icon(
          Icons.add,
          size: 36,
        ),
      ),
    );

    return Scaffold(
      body: Container(
        width: double.infinity,
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          mainAxisSize: MainAxisSize.max,
          crossAxisAlignment: CrossAxisAlignment.center,
          children: <Widget>[
            numberText,
            addButton,
          ],
        ),
      ),
    );
  }

  _buttonClicked() {
    setState(() {
      ++index;
    });
  }
}
```

在_buttonClicked方法响应了点击事件之后，再调用setState方法将index自增，此时会触发build重新渲染view tree.

## InheritedWidget怎么用
InheritedWidget用来解决父Widget的数据变化时通知子Widget。

具体用法是，父Widget继承自InheritedWidget，并定义一个static方法用来对外提供对象实例，另外有一个child属性用于设置子Widget，还要设置数据，用于给这个子widget引用。同时覆写是否刷新数据的方法。当数据变化时，子widget就收到通知了。使用方法如下：

```dart
///先继承InheritedWidget
class InheritedData extends InheritedWidget {
  final String data;//用于给子Widget使用

  InheritedData({
    this.data,
    Widget child,//子Widget
  }) : super(child: child);
  //提供static方法给外部使用
  static InheritedData of(BuildContext context) {
    return context.dependOnInheritedWidgetOfExactType<InheritedData>();
  }
  //判断是否通知子Widget
  @override
  bool updateShouldNotify(InheritedData old) => true;
}

///子Widget定义
class TestInheritedDataWidget extends StatefulWidget {
  @override
  _TestInheritedDataWidgetState createState() =>
      _TestInheritedDataWidgetState();
}

class _TestInheritedDataWidgetState extends State<TestInheritedDataWidget> {
  @override
  Widget build(BuildContext context) {
    return Center(
      child: Text(
      //引用父Widget的数据
        "${InheritedData.of(context).data}",
        style: TextStyle(fontSize: 18, color: Colors.pink),
      ),
    );
  }

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    //收到通知
    print("didChangeDependencies invoked");
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
    //点击按钮之后数据发生变化
      data = "in white";
    });
  }
}
```

## LinearLayout的sumWeight以及单个child的weight怎么实现

用Expanded Widget可以实现这种效果。源码里面针对Expanded的注释是它作为Row，Column，Flex的子Widget，里面可以含有多个子Widget，并且每个子widget通过flex来分配剩余的空间。

用下面这个例子说明：

```dart
class FlexWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Container(
        width: double.infinity,
        child: Row(
          mainAxisAlignment: MainAxisAlignment.center,
          mainAxisSize: MainAxisSize.max,
          crossAxisAlignment: CrossAxisAlignment.center,
          children: <Widget>[
            Expanded(
              child: Container(
                color: Colors.pink,
              ),
              flex: 1,
            ),
            Expanded(
              child: Container(
                color: Colors.lightGreenAccent,
              ),
              flex: 2,
            ),
          ],
        ),
      ),
    );
  }
}
```
运行下看效果可以知道，粉色占据了三分之一，浅绿色占据了三分之二的空间。

## 视图比较复杂，写的代码比较长，导致代码结尾有海量的反括号，怎么优化

还是以上面实现weight效果的代码做示例，可以看到代码后边有9个反括号，看起来比较累，容易造成心理压力。

```dart
class FlexWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    var child1 = Expanded(
      child: Container(
        color: Colors.pink,
      ),
      flex: 1,
    );

    var child2 = Expanded(
      child: Container(
        color: Colors.lightGreenAccent,
      ),
      flex: 2,
    );

    var child3 = Row(
      mainAxisAlignment: MainAxisAlignment.center,
      mainAxisSize: MainAxisSize.max,
      crossAxisAlignment: CrossAxisAlignment.center,
      children: <Widget>[
        child1,
        child2,
      ],
    );

    var child4 = Container(
      width: double.infinity,
      child: child3,
    );

    return Scaffold(
      body: child4,
    );
  }
}
```
把每个child拆出来作为一个widget赋值给一个变量，这样就可以将一些小的逻辑分出来，避免每个子widget的业务逻辑堆积在一起，使得逻辑看起来比较清晰。

## FrameLayout怎么实现
Stack配合Positioned使用，可以实现FrameLayout的效果。

```dart
class StackDemoWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    var child = Stack(
      children: <Widget>[
        Container(
          width: 480,
          height: 480,
          color: Colors.pink,
        ),
        Positioned(
          child: Container(
            width: 360,
            height: 360,
            color: Colors.amber,
          ),
          top: 84,
          bottom: 84,
        ),
        Positioned(
          child: Container(
            width: 240,
            height: 240,
            color: Colors.deepOrange,
          ),
          right: 12,
          left: 12,
        )
      ],
    );

    var child1 = Center(
      child: Container(
        child: child,
        width: 480,
        height: 480,
      ),
    );
    return Scaffold(
      body: child1,
    );
  }
}
```
Positioned的左右上下属性是相对于Stack边缘的偏移，如果设置左右或上下会影响到Positioned的宽高。

# 数据处理篇
## Android的SharePreference在这里怎么用

在工程下的pubspec.yaml文件中添加引用：

```dart
dependencies:
  shared_preferences: ^0.5.6
```

然后在命令行执行：
```dart
flutter pub get
```

然后在代码中就可以使用SharePreference的功能了。flutter中的SharePreference跟Android里面的一样，可能也会阻塞UI线程，需要使用async修饰相关方法。使用举例：

```dart
import 'package:shared_preferences/shared_preferences.dart';

class SharedPreferenceUtil {
  static SharedPreferences _prefs;

  SharedPreferenceUtil._privateConstructor();

  static final SharedPreferenceUtil instance =
      SharedPreferenceUtil._privateConstructor();

  Future<SharedPreferences> get prefs async {
    if (_prefs != null) return _prefs;
    _prefs = await SharedPreferences.getInstance();
    return prefs;
  }

  static const String KEY_SEARCH_HISTORY = "search_history";
  static const String KEY_USER_TOKEN = "token";

  void setValue(String key, String value) async {
    SharedPreferences sharedPreferences = await instance.prefs;
    sharedPreferences.setString(key, value);
  }

  Future<String> getValue(String key) async {
    SharedPreferences sharedPreferences = await instance.prefs;
    return sharedPreferences.getString(key);
  }
}

```
在这里我们把SharedPreferences抽象出来，做一个单例对外提供。所有的set和get必须用async修饰，因为他是阻塞式的。调用的地方举例：

```dart
SharedPreferenceUtil.instance
    .getValue(SharedPreferenceUtil.KEY_USER_TOKEN)
    .then((value) {
  //do something
});
```
这里getValue之后，然后调用then方法，意思是执行完了，在then方法里面获取到返回的值，可以做接下来的操作了。

## Android的数据库怎么用
在工程下的pubspec.yaml文件中添加引用：

```dart
dependencies:
  sqflite: ^1.1.7+3
```

然后在命令行执行：
```dart
flutter pub get
```
然后再代码中就可以使用数据库的相关功能了。

使用示例：

```dart
import 'package:flutter_hello/database/user_info_impl.dart';
import 'package:sqflite/sqflite.dart';
import 'package:path/path.dart';

class DataBaseHelper {
  static Database _database;

  static final _databaseName = "coupon.db";
  static final _databaseVersion = 1;

  static const String _CREATE_TABLE = "CREATE TABLE IF NOT EXISTS ";

  DataBaseHelper._privateConstructor();

  static final DataBaseHelper instance = DataBaseHelper._privateConstructor();

  Future<Database> get database async {
    if (_database != null) return _database;
    _database = await _initDatabase();
    return _database;
  }

  _initDatabase() async {
    String path = join(await getDatabasesPath(), _databaseName);
    return await openDatabase(path,
        version: _databaseVersion, onCreate: _onCreate);
  }

  Future _onCreate(Database db, int version) async {
    await db.execute(_CREATE_TABLE +
        UserInfoImpl.TABLE_USER_INFO +
        UserInfoImpl.generateCreateSql());
  }

  Future<int> insert(String dbName, Map<String, dynamic> data) async {
    Database db = await instance.database;
    return db.insert(
      dbName,
      data,
    );
  }

  Future<List<Map<String, dynamic>>> query(String dbName) async {
    Database db = await instance.database;
    final List<Map<String, dynamic>> maps = await db.query(dbName);
    return maps;
  }

  Future<void> update(String tableName, Map<String, dynamic> data, String keyId,
      String equalValue) async {
    Database db = await instance.database;
    await db.update(
      tableName,
      data,
      where: "$keyId = ?",
      whereArgs: [equalValue],
    );
  }

  Future<void> delete(String tableName, String keyId, String equalValue) async {
    Database db = await instance.database;
    await db.delete(
      tableName,
      where: "$keyId = ?",
      whereArgs: [equalValue],
    );
  }
}
```
这是一个数据库操作的类，将增删改查抽象出来，做具体的操作。在做操作之前获取数据库操作句柄database，其实他也是一个单例，使用了单例模式。

具体操作类实现之后，如果我们要操作具体的某个表就比较方便了。比如针对user_info这个表的操作类UserInfoImpl，我们可以这样操作：

```dart
insertUserInfo(UserInfoBean userInfoBean) {
    DataBaseHelper.instance.insert(TABLE_USER_INFO, userInfoBean.toJson());
}

Future<UserInfoBean> getUserInfo() async {
    List<Map<String, dynamic>> lists =
        await DataBaseHelper.instance.query(TABLE_USER_INFO);
    if (lists != null && lists.length > 0) {
      return UserInfoBean.fromJson(lists[0]);
    }
    return null;
}

deleteUserInfo(String userId) {
    DataBaseHelper.instance.delete(TABLE_USER_INFO, "id", userId);
}

updateUserInfo(UserInfoBean bean) {
    DataBaseHelper.instance
        .update(TABLE_USER_INFO, bean.toJson(), "id", bean.id.toString());
}
```

## Json数据怎么处理

在工程下的pubspec.yaml文件中添加引用：

```dart
//库版本可能已经更新，可以在https://pub.dev/packages/网站搜索最新的版本
dev_dependencies:
  build_runner: ^1.0.0
  json_serializable: ^2.0.0
```

然后在命令行执行：
```dart
flutter pub get
```

接下来就可以处理json数据了。处理json数据主要分三个步骤：
- 根据网络请求的字段，定义对应的类及属性字段
- 在类中添加序列化和反序列化接口
- 通过flutter指令生成对应的赋值处理

用以下json数据举例说明：

```json
{
    "total_count":12,
    "total_page":1,
    "page":1,
    "data":[]
}
```
首先定义类及属性字段，文件名是withdraw_detail_bean.dart

```dart
import 'package:json_annotation/json_annotation.dart';

part 'withdraw_detail_bean.g.dart';

@JsonSerializable()
class WithdrawDetailBean {
  WithdrawDetailBean(this.data, this.totalPage, this.totalCount, this.page);

  @JsonKey(name: "total_count")
  int totalCount;
  @JsonKey(name: "total_page")
  int totalPage;
  int page;
  List<WithDrawItemBean> data;

  factory WithdrawDetailBean.fromJson(Map<String, dynamic> json) =>
      _$WithdrawDetailBeanFromJson(json);

  Map<String, dynamic> toJson() => _$WithdrawDetailBeanToJson(this);
}
```

> 首先要添加part
>
> 'withdraw_detail_bean.g.dart'，因为part关键字可以帮助创建一个模块化的，可共享的代码库。后面执行指令的时候可以生成一个withdraw_detail_bean.g.dart文件，里面包含了当前数据序列化相关的处理代码。
>
> 其次在定义的类前面添加JsonSerializable注解。
>
> 定义构造函数，定义与返回数据对应的key，通过JsonKey注解，可以简化key对应的变量命名。
>
> 定义序列化和反序列化的接口，用于外部调用

最后一步就是执行生成处理json的代码，用指令实现，分两种。

一种是一次生成，后面再有修改的话，再执行，再生成：

*flutter packages pub run build_runner build*

另外一种是持续生成，一旦修改了json定义类，就会马上自动生成对应的处理类：

*flutter packages pub run build_runner watch*

最后可以看到data数组里面定义了一个WithDrawItemBean类，同样的WithDrawItemBean类，需要再像上面这样定义一遍就可以了。

# 网络请求
网络请求一般借助现有的库实现，http库对于各种方式的网络请求支持比较好，网址是：https://pub.dev/packages/http。

## get请求
这里以最复杂的情况做例子，query + header:

```dart
Future<NetResponseBean> _getResponse(
      String baseUrl, String api, Map<String, String> queryMap) async {
    var _client = http.Client();
    if (!api.contains("?")) {
      api = api + "?";
    }
    try {
      var url = baseUrl + api + _getQuery(queryMap);
    
      Map<String, String> headers = Map();
      _getCommonHeaders(headers);
      http.Response response = await _client.get(url, headers: headers);
      if (response.statusCode == HttpStatus.ok) {
        var body = response.body;
        Map<String, dynamic> data = jsonDecode(body);
        NetResponseBean responseData = NetResponseBean.fromJson(data);
        if (responseData != null) {
          return responseData;
        }
      }
    } catch (exception, stackTrace) {
      print("_getResponse stackTrace:$stackTrace");
    } finally {
      if (_client != null) {
        _client.close();
      }
    }
    return null;
}
```

第一步，获取client对象，这个对象在finally里面必须关掉，跟Android的操作是一样的。

第二步，拼接query到url中。

第三步，填充header到一个Map集合中。

这几步走完了调用对应的接口，填充对应的参数就可以了。

检查返回码，ok的话，获取body数据，转换成json格式，然后用对应json类序列化就可以使用返回的数据了。

## post请求
这里以最复杂的情况举例，header + request body

```dart
Future<NetResponseBean> _getPostResponse(
      String api, Map<String, String> parameters) async {
    var _client = http.Client();
    var url = _BAE_URL + api;
    try {
      //设置header
      Map<String, String> headersMap = new Map();
      _getCommonHeaders(headersMap);
      http.Response response = await _client.post(url,
          headers: headersMap, body: parameters, encoding: Utf8Codec());
      if (response.statusCode == 200) {
        String bodyData = response.body;
        if (bodyData != null) {
          Map<String, dynamic> data = jsonDecode(bodyData);
          NetResponseBean response = NetResponseBean.fromJson(data);
          return response;
        }
      } else {
        print('_getPostResponse error');
      }
    } catch (exception) {
      print("_getPostResponse fail:" + exception.toString());
    } finally {
      if (_client != null) {
        _client.close();
      }
    }
    return null;
}
```
跟上面get请求的流程差别不大，主要是接口调用不同，另外需要注意的是这里body是dynamic类型，传的是Map集合。当然也可以传json字符串，一个常见的场景是bean类转化为json，然后传递，对应的处理是：

```dart
var body = json.encode(jsonBean);
```

## 上传

```dart
Future requestUploadFile(File file, Function callBack) async {
    var url = _URL + "your sub url";
    var client = http.MultipartRequest("post", Uri.parse(url));
    http.MultipartFile.fromPath(
      'file',
      file.path,
    ).then((http.MultipartFile file) {
      _getOSCommonHeaders(client.headers);
      client.files.add(file);
      client.fields["description"] = "descriptiondescription";
      client.send().then((http.StreamedResponse response) {
        if (response.statusCode == 200) {
          response.stream.transform(utf8.decoder).join().then((String string) {
            print("requestUploadFile:$string");
            Map<String, dynamic> data = jsonDecode(string);
            if (data != null && data.length > 0) {
              NetResponseBean uploadResponse = NetResponseBean.fromJson(data);
              if (uploadResponse != null) {
                if (uploadResponse.code == 0) {
                  UploadImageResultBean uploadImageResultBean =
                      UploadImageResultBean.fromJson(uploadResponse.data);
                  callBack(uploadImageResultBean.fileUrl, 0);
                } else {
                  callBack("", uploadResponse.code);
                }
              }
            }
          });
        }
      }).catchError((error) {
        print("requestUploadFile error:$error");
      });
    });
  }
```
代码如上，一般性的上传操作。

## 下载

```dart
Future<File> _downloadFile(String url, String filename) async {
    http.Client _client = new http.Client();
    try {
      var req = await _client.get(Uri.parse(url));
      var bytes = req.bodyBytes;
      String dir = (await getApplicationDocumentsDirectory()).path;
      File file = new File('$dir/$filename');
      await file.writeAsBytes(bytes);
      return file;
    } finally {
      _client.close();
    }
}
```
上面是比较简单的下载，下面从别处抄一个带进度的：
> https://gist.github.com/ajmaln/c591cfb71d66bb6e688fe7027cbbe606

```dart
import 'dart:typed_data';
import 'dart:io';

import 'package:http/http.dart';
import 'package:path_provider/path_provider.dart';


downloadFile(String url, {String filename}) async {
  var httpClient = http.Client();
  var request = new http.Request('GET', Uri.parse(url));
  var response = httpClient.send(request);
  String dir = (await getApplicationDocumentsDirectory()).path;
  
  List<List<int>> chunks = new List();
  int downloaded = 0;
  
  response.asStream().listen((http.StreamedResponse r) {
    
    r.stream.listen((List<int> chunk) {
      // Display percentage of completion
      debugPrint('downloadPercentage: ${downloaded / r.contentLength * 100}');
      
      chunks.add(chunk);
      downloaded += chunk.length;
    }, onDone: () async {
      // Display percentage of completion
      debugPrint('downloadPercentage: ${downloaded / r.contentLength * 100}');
      
      // Save the file
      File file = new File('$dir/$filename');
      final Uint8List bytes = Uint8List(r.ntentLength);
      int offset = 0;
      for (List<int> chunk in chunks) {
        bytes.setRange(offset, offset + chunk.length, chunk);
        offset += chunk.length;
      }
      await file.writeAsBytes(bytes);   
      return;       
   });
}
```

**未完待续**




