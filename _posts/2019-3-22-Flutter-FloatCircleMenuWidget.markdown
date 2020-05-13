---
layout: post
title:  "Flutter-FloatCircleMenuWidget"
date:   2019-3-22 11:09:22 +0800
categories: flutter
---

# Flutter-FloatCircleMenu
## 1、Requirement
To implements this view, in Android platform,we can self-design a custom circle menu by canvas,furthermore,the material design can do a favor.The view contains animations,including scale,rotate,translate. On center icon opening,it turn to close icon with scale animation,and child items spread out to pointed angle with rotation and translate animation and re-clicked center icon,the child animation are executed in reversed.

Finally,we need to package our widget,and expose partial interface to other widgets.
the final destination is taken on below:![image](https://i.loli.net/2019/03/21/5c937e9018f80.png)
Obviously,we need implements it in flutter.Fortunately,i search the similar library in github and other websites [and this guy's blog save my days](https://fireship.io/lessons/flutter-radial-menu-staggered-animations/).Whereas his codes can not apply directly,so we need to rebuild.
## 2、Coding
While reading the fuck source code,we will start coding which base on the origin code.

### (1)Animation
Basically,the fundation of this function is animation,we design the animation first.

- AnimationController

```java
controller = new AnimationController(
    duration: const Duration(milliseconds: 200), vsync: this);
```

- scale animation

```java
scale = Tween<double>(
  begin: 1,
  end: 0.0,
).animate(
  CurvedAnimation(parent: controller, curve: Curves.fastOutSlowIn),
)..addListener(() {
    setState(() {});
  });
```

-rotate animation

```java
rotation = Tween<double>(
  begin: 0.0,
  end: 360.0,
).animate(
  CurvedAnimation(
    parent: controller,
    curve: Interval(
      0.3,
      0.9,
      curve: Curves.decelerate,
    ),
  ),
)..addListener(() {
    setState(() {});
  });
```

- translate animation

```java
translation = Tween<double>(
  begin: 0.0,
  end: 100.0,
).animate(
  CurvedAnimation(parent: controller, curve: Curves.linear),
)..addListener(() {
    setState(() {});
  });
```

Now,we copy the origin codes,however we add listener to each animation,which contains setState method,it can trigger widget to update itself with variable values which produced by runnning-animation.

### (2)Child Item Position
The child item can be one or more,and it can be placed around center icon with 8 positions,including left,right,top,bottom,left-top,left-bottom,right-top,right-bottom.Each item must execute rotate and transform animation, additionally,it can be responded to clicked.
the codes below:

```java
var childWidget = Transform(
  transform: Matrix4.identity()
    ..translate(
        (translation.value) * cos(rad), (translation.value) * sin(rad)),
  child: Transform.rotate(
      // Add rotation
      angle: radians(rotation.value),
      child: FloatingActionButton(
          child: _itemDetail.childIcon,
          onPressed: () {
            widget.itemClicked(_itemDetail.itemOption);
          },
          elevation: 0)));
```

In above codes,the angle should be converted to radians,the principle is:

```math
Rad = PI / 180 * Degree
```

Factly,we take a few radians into consideration:x * PI/4,x are integreted and from zero to seven.
the child item translation point dx is cos,and dy is sin,and then Multiply by base value.The rotate animation was placed at child parameters for transform animation,then they can be execute simultaneously.

In order to clarify the child position,we create a enum:

```java
enum ItemOption {
  POSITION_LEFT,
  POSITION_RIGHT,
  POSITION_TOP,
  POSITION_BOTTOM,
  POSITION_LEFT_TOP,
  POSITION_RIGHT_TOP,
  POSITION_LEFT_BOTTOM,
  POSITION_RIGHT_BOTTOM,
}
```

each item relates a radians,below are relationships:

```java
double tempAngle = 0;
  switch (itemParameters.itemOption) {
    case ItemOption.POSITION_LEFT:
      tempAngle = 180;
      break;
    case ItemOption.POSITION_RIGHT:
      tempAngle = 0;
      break;
    case ItemOption.POSITION_TOP:
      tempAngle = 270;
      break;
    case ItemOption.POSITION_BOTTOM:
      tempAngle = 90;
      break;
    case ItemOption.POSITION_LEFT_TOP:
      tempAngle = 225;
      break;
    case ItemOption.POSITION_RIGHT_TOP:
      tempAngle = 315;
      break;
    case ItemOption.POSITION_LEFT_BOTTOM:
      tempAngle = 135;
      break;
    case ItemOption.POSITION_RIGHT_BOTTOM:
      tempAngle = 45;
      break;
  }
```

and center icon in the center of icon.

### (2)Center Icon
Center icon can be touched to execute opening and closing action,which corresponding to show and hide child items.
We decide to define two center icons in stack widget,and opening widget at top level,and closing widget placed below level,on open-widget clicking,it scale to zero with animation,and close-widget scale to one at the same way,within animation,the child items are spreading out and 
moving back.
we define center icon:

```java
var centerCloseWidget = Transform.scale(
      scale: scale.value - 1,
      // subtract the beginning value to run the opposite animation
      child: FloatingActionButton(
          child: centerCloseIcon,
          onPressed: close,
          backgroundColor: widget.centerIconBackground),
    );
    var centerOpenWidget = Transform.scale(
      scale: scale.value,
      child: FloatingActionButton(
          child: centerOpenIcon,
          backgroundColor: widget.centerIconBackground,
          onPressed: open),
    );
```

### (3)Parent Widget
Finishing the below steps,we should design the top widget which Stack widget meets our requirements,its child widget are placed in the same position from bottom to top,otherwise including positioned widget.

```java
return AnimatedBuilder(
    animation: controller,
    builder: (context, builder) {
      return Stack(
          alignment: widget.alignment,
          children: getChildItemWidgets(itemDetails));
    });
```

### (4)Trailing
Designing widget interface is vital,

```java
final ValueChanged<ItemOption> itemClicked;
final List<ItemParameters> itemParameterList;
final String centerIconOpenSourcePath;
final String centerIconCloseSourcePath;
final Color centerIconBackground;
final Alignment alignment;

const CircleMenuWidget({
    @required this.itemParameterList,
    this.itemClicked,
    this.centerIconOpenSourcePath,
    this.centerIconCloseSourcePath,
    this.alignment = Alignment.center,
    this.centerIconBackground = const Color(0xFFFF7404),
});
```

and ItemParameters define below:

```java
class ItemParameters {
  ItemOption itemOption;
  String sourcePath;
  double itemWidth;
  double itemHeight;

  ItemParameters(
      this.itemOption, this.sourcePath, this.itemWidth, this.itemHeight);
}
```

itemClicked is responded to item clicked,itemParameterList include child item's position,image source path,width and height,centerIconOpenSourcePath,centerIconBackground and centerIconCloseSourcePath relate to center icon,alignment decide center-icon's position in screen.

## 3、How to use

```java
List<String> path = [
  "resources/images/home_task_cycle_normal.png",
  "resources/images/home_task_run_normal.png",
  "resources/images/home_task_walk_normal.png"
];
List<ItemParameters> itemParas = [];
ItemParameters itemParameters0 =
    ItemParameters(ItemOption.POSITION_LEFT, path[0], 56, 56);
ItemParameters itemParameters1 =
    ItemParameters(ItemOption.POSITION_LEFT_TOP, path[1], 56, 56);
ItemParameters itemParameters2 =
    ItemParameters(ItemOption.POSITION_TOP, path[2], 56, 56);
itemParas.add(itemParameters0);
itemParas.add(itemParameters1);
itemParas.add(itemParameters2);

return Flexible(
    child: Scaffold(
  floatingActionButton: SizedBox.expand(
      child: CircleMenuWidget(
    itemParameterList: itemParas,
    itemClicked: itemClicked,
    alignment: Alignment.bottomRight,
  )),
  body: ListView(
    children: <Widget>[...])
));

void itemClicked(ItemOption option) {
    print("option desc ${option.index}");
}
```

## One more thing 

Simultaneously,i will paste the origin implements.

### My rebuild code

```java
import 'package:flutter/material.dart';
import 'dart:math';
import 'package:vector_math/vector_math.dart' show radians;

class CircleMenuWidget extends StatefulWidget {
  final ValueChanged<ItemOption> itemClicked;
  final List<ItemParameters> itemParameterList;
  final String centerIconOpenSourcePath;
  final String centerIconCloseSourcePath;
  final Color centerIconBackground;
  final Alignment alignment;

  const CircleMenuWidget({
    @required this.itemParameterList,
    this.itemClicked,
    this.centerIconOpenSourcePath,
    this.centerIconCloseSourcePath,
    this.alignment = Alignment.center,
    this.centerIconBackground = const Color(0xFFFF7404),
  });

  static _CircleMenuWidgetState curState;

  @override
  State<StatefulWidget> createState() {
    curState = new _CircleMenuWidgetState();
    return curState;
  }

  void open() {
    curState.open();
  }

  void close() {
    curState.close();
  }

  bool getStatus() {
    return curState.closed;
  }
}

class _CircleMenuWidgetState extends State<CircleMenuWidget>
    with SingleTickerProviderStateMixin {
  AnimationController controller;
  Animation<double> scale;
  Animation<double> translation;
  Animation<double> rotation;

  bool closed = true;

  @override
  void initState() {
    super.initState();
    controller = new AnimationController(
        duration: const Duration(milliseconds: 200), vsync: this);

    translation = Tween<double>(
      begin: 0.0,
      end: 100.0,
    ).animate(
      CurvedAnimation(parent: controller, curve: Curves.linear),
    )..addListener(() {
        setState(() {});
      });
    rotation = Tween<double>(
      begin: 0.0,
      end: 360.0,
    ).animate(
      CurvedAnimation(
        parent: controller,
        curve: Interval(
          0.3,
          0.9,
          curve: Curves.decelerate,
        ),
      ),
    )..addListener(() {
        setState(() {});
      });
    scale = Tween<double>(
      begin: 1,
      end: 0.0,
    ).animate(
      CurvedAnimation(parent: controller, curve: Curves.fastOutSlowIn),
    )..addListener(() {
        setState(() {});
      });
  }

  @override
  Widget build(context) {
    List<_ItemDetail> itemDetails = [];
    if (widget.itemParameterList == null ||
        widget.itemParameterList.length == 0) {
      return Text("itemParameterList fail");
    }
    for (ItemParameters itemParameters in widget.itemParameterList) {
      double width =
          itemParameters.itemWidth == 0 ? 56 : itemParameters.itemWidth;
      double height =
          itemParameters.itemHeight == 0 ? 56 : itemParameters.itemHeight;
      String sourcePath = itemParameters.sourcePath;
      double tempAngle = 0;
      switch (itemParameters.itemOption) {
        case ItemOption.POSITION_LEFT:
          tempAngle = 180;
          break;
        case ItemOption.POSITION_RIGHT:
          tempAngle = 0;
          break;
        case ItemOption.POSITION_TOP:
          tempAngle = 270;
          break;
        case ItemOption.POSITION_BOTTOM:
          tempAngle = 90;
          break;
        case ItemOption.POSITION_LEFT_TOP:
          tempAngle = 225;
          break;
        case ItemOption.POSITION_RIGHT_TOP:
          tempAngle = 315;
          break;
        case ItemOption.POSITION_LEFT_BOTTOM:
          tempAngle = 135;
          break;
        case ItemOption.POSITION_RIGHT_BOTTOM:
          tempAngle = 45;
          break;
      }
      _ItemDetail _itemDetail = new _ItemDetail(
          tempAngle,
          Image(
            image: AssetImage(sourcePath),
            width: width,
            height: height,
          ),
          itemParameters.itemOption);
      itemDetails.add(_itemDetail);
    }
    return AnimatedBuilder(
        animation: controller,
        builder: (context, builder) {
          return Stack(
              alignment: widget.alignment,
              children: getChildItemWidgets(itemDetails));
        });
  }

  List<Widget> getChildItemWidgets(List<_ItemDetail> itemDetails) {
    List<Widget> widgets = [];
    for (_ItemDetail _itemDetail in itemDetails) {
      final double rad = radians(_itemDetail.angle);
      var childWidget = Transform(
          transform: Matrix4.identity()
            ..translate(
                (translation.value) * cos(rad), (translation.value) * sin(rad)),
          child: Transform.rotate(
              // Add rotation
              angle: radians(rotation.value),
              child: FloatingActionButton(
                  child: _itemDetail.childIcon,
                  onPressed: () {
                    widget.itemClicked(_itemDetail.itemOption);
                  },
                  elevation: 0)));
      widgets.add(childWidget);
    }

    Widget centerOpenIcon;
    if (widget.centerIconOpenSourcePath == null) {
      centerOpenIcon = Icon(Icons.add);
    } else {
      centerOpenIcon =
          Image(image: AssetImage(widget.centerIconOpenSourcePath));
    }

    Widget centerCloseIcon;
    if (widget.centerIconCloseSourcePath == null) {
      centerCloseIcon = Icon(Icons.close);
    } else {
      centerCloseIcon =
          Image(image: AssetImage(widget.centerIconCloseSourcePath));
    }

    var centerCloseWidget = Transform.scale(
      scale: scale.value - 1,
      // subtract the beginning value to run the opposite animation
      child: FloatingActionButton(
          child: centerCloseIcon,
          onPressed: close,
          backgroundColor: widget.centerIconBackground),
    );
    var centerOpenWidget = Transform.scale(
      scale: scale.value,
      child: FloatingActionButton(
          child: centerOpenIcon,
          backgroundColor: widget.centerIconBackground,
          onPressed: open),
    );
    widgets.add(centerCloseWidget);
    widgets.add(centerOpenWidget);
    return widgets;
  }

  open() {
    closed = false;
    controller.forward();
  }

  close() {
    closed = true;
    controller.reverse();
  }
}

enum ItemOption {
  POSITION_LEFT,
  POSITION_RIGHT,
  POSITION_TOP,
  POSITION_BOTTOM,
  POSITION_LEFT_TOP,
  POSITION_RIGHT_TOP,
  POSITION_LEFT_BOTTOM,
  POSITION_RIGHT_BOTTOM,
}

class ItemParameters {
  ItemOption itemOption;
  String sourcePath;
  double itemWidth;
  double itemHeight;

  ItemParameters(
      this.itemOption, this.sourcePath, this.itemWidth, this.itemHeight);
}

class _ItemDetail {
  double angle;
  Widget childIcon;
  ItemOption itemOption;

  _ItemDetail(this.angle, this.childIcon, this.itemOption);
}
```

### Origin implements

```java
import 'package:flutter/material.dart';
import 'dart:math';
import 'package:vector_math/vector_math.dart' show radians;
import 'package:font_awesome_flutter/font_awesome_flutter.dart';

void main() => runApp(MyApp());

// The parent Material App
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
        home: Scaffold(body: SizedBox.expand(child: RadialAnimation())));
  }
}

class RadialAnimation extends StatefulWidget {
  @override
  State<StatefulWidget> createState() {
    return new _RadiaAnimationState();
  }
}

class _RadiaAnimationState extends State<RadialAnimation>
    with SingleTickerProviderStateMixin {
  AnimationController controller;
  Animation<double> scale;
  Animation<double> translation;
  Animation<double> rotation;

  @override
  void initState() {
    super.initState();
    controller = new AnimationController(
        duration: const Duration(milliseconds: 500), vsync: this);

    translation = Tween<double>(
      begin: 0.0,
      end: 100.0,
    ).animate(
      CurvedAnimation(parent: controller, curve: Curves.linear),
    )..addListener(() {
        setState(() {});
      });
    rotation = Tween<double>(
      begin: 0.0,
      end: 360.0,
    ).animate(
      CurvedAnimation(
        parent: controller,
        curve: Interval(
          0.3,
          0.9,
          curve: Curves.decelerate,
        ),
      ),
    )..addListener(() {
        setState(() {});
      });
    scale = Tween<double>(
      begin: 1,
      end: 0.0,
    ).animate(
      CurvedAnimation(parent: controller, curve: Curves.fastOutSlowIn),
    )..addListener(() {
        setState(() {});
      });
  }

  @override
  Widget build(context) {
    print(
        "FloatCircleMenuView RadialAnimation build rotation:${rotation.value} and scale:${scale.value} and translate:${translation.value}");
    return AnimatedBuilder(
        animation: controller,
        builder: (context, builder) {
          return Transform.rotate(
            // Add rotation
            angle: radians(rotation.value),
            child: Stack(alignment: Alignment.center, children: [
              _buildButton(0,
                  color: Colors.red, icon: FontAwesomeIcons.thumbtack),
              _buildButton(45,
                  color: Colors.green, icon: FontAwesomeIcons.sprayCan),
              _buildButton(90,
                  color: Colors.orange, icon: FontAwesomeIcons.fire),
              _buildButton(135,
                  color: Colors.blue, icon: FontAwesomeIcons.kiwiBird),
              _buildButton(180,
                  color: Colors.black, icon: FontAwesomeIcons.cat),
              _buildButton(225,
                  color: Colors.indigo, icon: FontAwesomeIcons.paw),
              _buildButton(270,
                  color: Colors.pink, icon: FontAwesomeIcons.bong),
              _buildButton(315,
                  color: Colors.yellow, icon: FontAwesomeIcons.bolt),
              Transform.scale(
                scale: scale.value - 1,
                // subtract the beginning value to run the opposite animation
                child: FloatingActionButton(
                    child: Icon(Icons.close),
                    onPressed: _close,
                    backgroundColor: Color(0xFFFF7404)),
              ),
              Transform.scale(
                scale: scale.value,
                child: FloatingActionButton(
                    child: Icon(Icons.add),
                    backgroundColor: Color(0xFFFF7404),
                    onPressed: _open),
              )
            ]),
          );
        });
  }

  _buildButton(double angle, {Color color, IconData icon}) {
    final double rad = radians(angle);
    return Transform(
        transform: Matrix4.identity()
          ..translate(
              (translation.value) * cos(rad), (translation.value) * sin(rad)),
        child: FloatingActionButton(
            child: Icon(icon),
            backgroundColor: color,
            onPressed: _openItemView(),
            elevation: 0));
  }

  _open() {
    controller.forward();
  }

  _close() {
    controller.reverse();
  }

  _openItemView() {}
}
```

add font_awesome_flutter: ^8.4.0 to pubspec.yaml below dependencies.

article will sync to wechat blog

微信公众号:Android部落格
