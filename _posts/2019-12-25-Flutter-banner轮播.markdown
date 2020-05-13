---
layout: post
title:  "Flutter-banner轮播"
date:   2019-12-25 11:09:22 +0800
categories: flutter
---

> 文章将同步更新到微信公众号：Android部落格

## 问题背景

因为最近做商城App，需要用到轮播，发现flutter的控件库里面没有这个控件（当然了，可能是我自己没有找到），于是就决定自己动手做一个banner轮播图片了。

## 框架
- 整体框架就是一个PageView，Indicator指示器，一个定时器。

- PageView用来展示需要播放的Widget，此处不一定必须限定死要展示Image.

- Indicator作为当前图片的指示器，需要给出当前获取焦点Page的指示。而且Indicator要可以设置半径，选中颜色，未选中的颜色。最后还要可以设置指示器的位置，左上右下。

- 定时器，当设置需要定时播放的时候，就按照设置的间隔时间播放对应的Widget。这里需要考虑的就是当手动点击的时候，需要取消定时循环，点击放开的时候，重新定时。当界面销毁的时候，销毁定时器。

- Page点击和被展示回调。只需要在Page被点击和被播放展示的时候回调到对应的方法就行了。

## 实现

#### 1、实现PageView
上代码：

```dart
_controller = PageController(
  initialPage: widget.initPage,
);
Widget pageView = GestureDetector(
  onHorizontalDragDown: _onTaped,
  onTap: _onPageClicked,
  child: PageView(
    children: widget.childWidget,
    scrollDirection: widget.scrollDirection,
    onPageChanged: onPageChanged,
    controller: _controller,
  ),
);
```

PageController是Page控制器，可以设置初始显示的Page，也可以在多个图片轮播的时候，是否缓存页面，也可以设置在滚动方向上，在视图显示区域的比例。initialPage可以给外部调用，默认设置为0。

#### 2、实现Indicator
上代码：

```dart
Widget indicatorWidget = Row(
  mainAxisAlignment: MainAxisAlignment.center,
  mainAxisSize: MainAxisSize.max,
  crossAxisAlignment: CrossAxisAlignment.center,
  children: widget.childWidget.map(
    (f) {
      int index = widget.childWidget.indexOf(f);
      Widget indicatorWidget = Container(
        width: Size.fromRadius(widget.indicatorRadius).width,
        height: Size.fromRadius(widget.indicatorRadius).height,
        margin: EdgeInsets.only(right: widget.indicatorSpaceBetween),
        decoration: new BoxDecoration(
          shape: BoxShape.circle,
          color: index == selectedPage
              ? widget.indicatorSelectedColor
              : widget.indicatorColor,
        ),
      );
      return indicatorWidget;
    },
  ).toList(),
);
```

当需要轮播多个Widget的时候，默认指示器按行排列，一字排开居中显示。单个子Indicator是个圆圈，颜色，半径可以外部设置。并且选中的Page的指示器要是选中的颜色，其他的为未选中颜色。

#### 3、合并Indicator和PageView
将这两个Widget合并起来，一般来说他们是上下级关系，Indicator在PageView之上：

```dart
Widget stackWidget = widget.enableIndicator
    ? Stack(
        alignment: AlignmentDirectional.bottomCenter,
        children: <Widget>[
          pageView,
          Positioned(
            bottom: 6,
            child: Align(
              alignment: widget.indicatorAlign,
              child: indicatorWidget,
            ),
          ),
        ],
      )
    : pageView;
```

当外部设置展示Indicator的时候，直接就展示PageView，否则的话，需要用Stack将这两个Widget堆积起来。PageView在下，Indicator在上，此时它在什么位置，也是可以外部设置，通过Align可以指定指示器的位置，一般是在底部偏上一点，所以这里将Positioned的bottom参数设置了一个值。当然了，我们也可以将这个位置和位置的偏移距离对外提供设置的接口。

这里做最后一个工作，给这个stackWidget设置一个文字方向，因为有些国家的文字阅读方向是从右往左，此时如果不设置的话，会报错，我这里写死了，从左往右：

```dart
Widget parent = Directionality(
    textDirection: TextDirection.ltr,
    child: Container(
        width: widget.width, height: widget.height, child: stackWidget));
```
当然Android是提供了判断方法的：
```kotlin
Configuration config = getResources().getConfiguration();
if(config.getLayoutDirection() == View.LAYOUT_DIRECTION_RTL) {
    //in Right To Left layout
}
```

至此，Widget层面算是写完了。

#### 4、定时器
定时触发，这在flutter里面是提供了方法的：

```dart
void _startTimer() {
    if (widget.autoDisplayInterval <= 0) {
      return;
    }
    var oneSec = Duration(seconds: widget.autoDisplayInterval);
    _timer = new Timer.periodic(oneSec, (Timer timer) {
      ++selectedPage;
      selectedPage = selectedPage % widget.childWidget.length;
      _controller?.jumpToPage(selectedPage);
      onPageChanged(selectedPage);
    });
}
```

autoDisplayInterval时间间隔由外部设置，不设置的话，默认不自动播放，如果设置大于0的数字就是自动播放了。时间间隔是秒。另外每次触发的时候，将选中播放的页面自加1，然后跟总的页面数取余，就是当前应该播放的页面，同时需要手动设置跳转到指定的页面，同时将播放页的序数回调给调用者。

在这里，当我们滑动页面的时候，如果不处理，就会出现滑动混乱的情况。这里我们需要做出处理。记得在PageView定义的时候，最外层包裹了一个GestureDetector，我们需要处理Page被点击和被拖住水平滑动的场景，于是就有了  处理onHorizontalDragDown和onTap这两个回调函数的场景了。

onTap用于处理点击，直接携带序号参数回调给调用者。

onHorizontalDragDown用于处理水平滑动按下，具体的处理是：

```dart
void _onHorizontalDragDown(DragDownDetails details) {
    if (_timer == null) {
      return;
    }
    _releaseTimer();
    Future.delayed(Duration(seconds: 2));
    _startTimer();
}
```

每次按下点击的时候，停止计时，将计时器释放掉，同时延时2秒，如果在这期间有新的拖动事件进来，再次取消。

如果延时2秒，没有其他处理，继续定时播放处理。

看看定时销毁的方法：

```dart
void _releaseTimer() {
    if (_timer != null && !_timer.isActive) {
      return;
    }
    _timer?.cancel();
    _timer = null;
}
```

#### 对外提供
下一步就是将定义的过程中需要调用者传递的参数对外暴露，通过构造的时候传递进来：

```dart
BannerWidget(
  {Key key,
  @required this.childWidget,//子视图列表
  this.enableIndicator = true,//是否允许显示指示器
  this.initPage = 0,//初始页面序号
  this.height = 36,//高度
  this.width = 340,//宽度
  this.indicatorRadius = 4,//指示器半径
  this.indicatorColor = Colors.white,//指示器默认颜色
  this.indicatorSelectedColor = Colors.red,//页面被选中指示器颜色
  this.scrollDirection = Axis.horizontal,//滚动方向（默认水平）
  this.indicatorSpaceBetween = 6,//指示器之间的间距
  this.autoDisplayInterval = 0,//自动播放时间间隔（不设置或设置为0，则不自动播放）
  this.onPageSelected,//页面被展示回调
  this.indicatorAlign = Alignment.bottomCenter,//指示器位置
  this.onPageClicked})//页面被点击回调
  : super(key: key);
```
被播放的子视图列表必须提供。从上到下依次是：是否允许显示指示器，初始页面序号，高度，宽度，指示器半径，指示器默认颜色，页面被选中指示器颜色，滚动方向（默认水平），指示器之间的间距，自动播放时间间隔（不设置或设置为0，则不自动播放），页面被展示回调，指示器位置，页面被点击回调。

#### 如何使用

```dart
var images = {
  "https://ws1.sinaimg.cn/large/610dc034ly1fitcjyruajj20u011h412.jpg",
  "https://ws1.sinaimg.cn/large/610dc034ly1fjfae1hjslj20u00tyq4x.jpg",
  "http://ww1.sinaimg.cn/large/610dc034ly1fjaxhky81vj20u00u0ta1.jpg",
  "https://ws1.sinaimg.cn/large/610dc034ly1fivohbbwlqj20u011idmx.jpg",
  "https://ws1.sinaimg.cn/large/610dc034ly1fj78mpyvubj20u011idjg.jpg",
  "https://ws1.sinaimg.cn/large/610dc034ly1fj3w0emfcbj20u011iabm.jpg",
  "https://ws1.sinaimg.cn/large/610dc034ly1fiz4ar9pq8j20u010xtbk.jpg",
  "https://ws1.sinaimg.cn/large/610dc034ly1fis7dvesn6j20u00u0jt4.jpg",
  "https://ws1.sinaimg.cn/large/610dc034ly1fir1jbpod5j20ip0newh3.jpg",
  "https://ws1.sinaimg.cn/large/610dc034ly1fik2q1k3noj20u00u07wh.jpg",
  "https://ws1.sinaimg.cn/large/610dc034ly1fiednrydq8j20u011itfz.jpg",
  "http://ww1.sinaimg.cn/large/610dc034ly1ffmwnrkv1hj20ku0q1wfu.jpg",
};

void main() => runApp(BannerWidget(
      width: 340,
      height: 56,
      autoDisplayInterval: 2,
      childWidget: images.map((f) {
        return Image.network(
          f,
          width: 340,
          height: 48,
        );
      }).toList(),
    ));
```

#### 最后一公里

```dart
import 'dart:async';

import 'package:flutter/material.dart';

class BannerWidget extends StatefulWidget {
  BannerWidget(
      {Key key,
      @required this.childWidget,
      this.enableIndicator = true,
      this.initPage = 0,
      this.height = 36,
      this.width = 340,
      this.indicatorRadius = 4,
      this.indicatorColor = Colors.white,
      this.indicatorSelectedColor = Colors.red,
      this.scrollDirection = Axis.horizontal,
      this.indicatorSpaceBetween = 6,
      this.autoDisplayInterval = 0,
      this.onPageSelected,
      this.indicatorAlign = Alignment.bottomCenter,
      this.onPageClicked})
      : super(key: key);

  final double width;
  final double height;

  final List<Widget> childWidget;
  final Axis scrollDirection;

  final ValueChanged<int> onPageSelected;
  final ValueChanged<int> onPageClicked;

  final int initPage;

  final bool enableIndicator;

  final Color indicatorSelectedColor;
  final Color indicatorColor;
  final double indicatorRadius;

  final double indicatorSpaceBetween;

  final int autoDisplayInterval;

  final Alignment indicatorAlign;

  @override
  __BannerWidgetState createState() => __BannerWidgetState();
}

class __BannerWidgetState extends State<BannerWidget> {
  int selectedPage = 0;
  PageController _controller;
  Timer _timer;
  int lastTapDownTime = 0;

  void onPageChanged(int index) {
    setState(() {
      selectedPage = index;
    });
    widget?.onPageSelected(index);
  }

  void _startTimer() {
    if (widget.autoDisplayInterval <= 0) {
      return;
    }
    var oneSec = Duration(seconds: widget.autoDisplayInterval);
    _timer = new Timer.periodic(oneSec, (Timer timer) {
      ++selectedPage;
      selectedPage = selectedPage % widget.childWidget.length;
      _controller?.jumpToPage(selectedPage);
      onPageChanged(selectedPage);
    });
  }

  void _releaseTimer() {
    if (_timer != null && !_timer.isActive) {
      return;
    }
    _timer?.cancel();
    _timer = null;
  }

  void _onHorizontalDragDown(DragDownDetails details) {
    if (_timer == null) {
      return;
    }
    _releaseTimer();
    Future.delayed(Duration(seconds: 2));
    _startTimer();
  }

  void _onPageClicked() {
    widget?.onPageClicked(selectedPage);
  }

  @override
  void initState() {
    super.initState();
    _startTimer();
  }

  @override
  Widget build(BuildContext context) {
    _controller = PageController(
      initialPage: widget.initPage,
    );
    Widget pageView = GestureDetector(
      onHorizontalDragDown: _onHorizontalDragDown,
      onTap: _onPageClicked,
      child: PageView(
        children: widget.childWidget,
        scrollDirection: widget.scrollDirection,
        onPageChanged: onPageChanged,
        controller: _controller,
      ),
    );

    Widget indicatorWidget = Row(
      mainAxisAlignment: MainAxisAlignment.center,
      mainAxisSize: MainAxisSize.max,
      crossAxisAlignment: CrossAxisAlignment.center,
      children: widget.childWidget.map(
        (f) {
          int index = widget.childWidget.indexOf(f);
          Widget indicatorWidget = Container(
            width: Size.fromRadius(widget.indicatorRadius).width,
            height: Size.fromRadius(widget.indicatorRadius).height,
            margin: EdgeInsets.only(right: widget.indicatorSpaceBetween),
            decoration: new BoxDecoration(
              shape: BoxShape.circle,
              color: index == selectedPage
                  ? widget.indicatorSelectedColor
                  : widget.indicatorColor,
            ),
          );
          return indicatorWidget;
        },
      ).toList(),
    );

    Widget stackWidget = widget.enableIndicator
        ? Stack(
            alignment: AlignmentDirectional.bottomCenter,
            children: <Widget>[
              pageView,
              Positioned(
                bottom: 6,
                child: Align(
                  alignment: widget.indicatorAlign,
                  child: indicatorWidget,
                ),
              ),
            ],
          )
        : pageView;

    Widget parent = Directionality(
        textDirection: TextDirection.ltr,
        child: Container(
            width: widget.width, height: widget.height, child: stackWidget));
    return parent;
  }

  @override
  void dispose() {
    super.dispose();
    _releaseTimer();
  }
}

var images = {
  "https://ws1.sinaimg.cn/large/610dc034ly1fitcjyruajj20u011h412.jpg",
  "https://ws1.sinaimg.cn/large/610dc034ly1fjfae1hjslj20u00tyq4x.jpg",
  "http://ww1.sinaimg.cn/large/610dc034ly1fjaxhky81vj20u00u0ta1.jpg",
  "https://ws1.sinaimg.cn/large/610dc034ly1fivohbbwlqj20u011idmx.jpg",
  "https://ws1.sinaimg.cn/large/610dc034ly1fj78mpyvubj20u011idjg.jpg",
  "https://ws1.sinaimg.cn/large/610dc034ly1fj3w0emfcbj20u011iabm.jpg",
  "https://ws1.sinaimg.cn/large/610dc034ly1fiz4ar9pq8j20u010xtbk.jpg",
  "https://ws1.sinaimg.cn/large/610dc034ly1fis7dvesn6j20u00u0jt4.jpg",
  "https://ws1.sinaimg.cn/large/610dc034ly1fir1jbpod5j20ip0newh3.jpg",
  "https://ws1.sinaimg.cn/large/610dc034ly1fik2q1k3noj20u00u07wh.jpg",
  "https://ws1.sinaimg.cn/large/610dc034ly1fiednrydq8j20u011itfz.jpg",
  "http://ww1.sinaimg.cn/large/610dc034ly1ffmwnrkv1hj20ku0q1wfu.jpg",
};

void main() => runApp(BannerWidget(
      width: 340,
      height: 56,
      autoDisplayInterval: 2,
      childWidget: images.map((f) {
        return Image.network(
          f,
          width: 340,
          height: 48,
        );
      }).toList(),
    ));

```