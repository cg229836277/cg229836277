---
layout: post
title:  "Flutter-CustomBarChartWidget"
date:   2019-3-25 11:09:22 +0800
categories: flutter
---

# Flutter-CustomBarChartWidget
## 1、Scene
We hold our mobile phone everywhere and almost everytime,naturally a application which can show our usage detail is born.Like this:
- Daily
![daily](https://i.loli.net/2019/03/22/5c94c95a8f09d.png)
- Weekly
![daily](https://i.loli.net/2019/03/22/5c94c95a90d39.png)
- Monthly
![daily](https://i.loli.net/2019/03/22/5c94c95a8e186.png)
-Yearly
![daily](https://i.loli.net/2019/03/22/5c94c95a834d0.png)
One of the most vital work is to accomplish the bar chart,and the charts_flutter library is taken into my consideration firstly,and this [link](https://google.github.io/charts/flutter/example/time_series_charts/with_bar_renderer) seems meet my requirement,but the x axis's value was limited to date,you can not self-design the x-axis value and y-axis value,so we need to deliver requirement in detail.

## 2、Requirement
The custom bar chart should be highly self-design,for instance,the x-axis value and y-axis value,the y-axis value's position,which can be placed at left or right in bar chart,what's more, the bar's color and style,how many lines,lines's color,each lines gap and so on ,which can be set by other widget.

## 3、Skeleton
Brief skeleton below:
![](https://i.loli.net/2019/03/22/5c94d0e47ca0f.png)
Parent view is a stack,its children can be stacked from bottom to top,this children are bars,lines,y-units,x-units.
Each children can be placed by point exactly,and factors which related to points can be decided by other widgets,including gap between each lines,the x-units number,y-units number which must equal to lines's.
With the help of CustomPaint class,we can accomplish this canvas by paint method.
## 4、CustomPainter
[CustomPainter](https://docs.flutter.io/flutter/rendering/CustomPainter-class.html),the interface provides a paint method,that we can self-design what we want to draw.
Then we define a BarChartPaint extends CustomPainter class,and make repaint return true,let canvas redraw while new is coming.
## 5、Coding
Coding is working than words.
### (1)Define the critical parameters
- Draw type
```dart
enum DRAW_TYPE { TYPE_LINE, TYPE_X, TYPE_Y, TYPE_BAR }
```
Four items decide whole chart appearance.
### (2) Label
Including x label and y label
```dart
class Label {
  String label;
  int color;

  Label(this.label, this.color);
}
```
It decide label's color and value.
### (3) LinePosition
```dart
class LinePosition {
  double x1;
  double y1;

  double x2;
  double y2;

  LinePosition(this.x1, this.y1, this.x2, this.y2);
}
```
From start to end.
### (4) LabelPointPosition
```dart
class LabelPointPosition {
  final double x;
  final double y;
  final double value;

  LabelPointPosition(this.x, this.y, this.value);
}
```
Including label position and value.
### (5) LabelPointPosition
```dart
enum LabelPosition { POSITION_LEFT, POSITION_RIGHT }
```
Y-label can be placed at left or right.
### (6) BarStyle
```dart
class BarStyle {
  final double radius;
  final int color;
  final double barWidth;

  // ignore: invalid_required_param
  const BarStyle(
      @required this.radius, @required this.color, @required this.barWidth);
}
```
Bar contains radius for four angle and its width and color can be set.
### (7)ChartData
```dart
class ChartData {
  double yData;
  double xData;

  ChartData(this.xData, this.yData);
}
```
chart data decides bar's height and y label value.
### (8)ChartData
```dart
class Margin {
  final double left;
  final double right;
  final double top;
  final double bottom;

  const Margin(this.left, this.right, this.top, this.bottom);
}
```
Charts's margin from all sides is determined.
### (9)Start Drawing
The parent widget is a stack,and each children is drew with given parameters.
```dart
return GestureDetector(
  child: Container(
    height: widget.lineMargin.top * (widget.lineNum - 1) +
        widget.margin.top +
        widget.margin.bottom +
        widget.heightOffset,
    width: actualWidth,
    color: Colors.white,
    margin: EdgeInsets.only(
        left: widget.margin.left,
        top: widget.margin.top,
        right: widget.margin.right,
        bottom: widget.margin.bottom),
    child: Stack(
      children: drawSkeleton(),
    ),
  ),
  onTap: () {
    widget.onTap("clicked");
  },
);
```
Container here do a convenience to margin,background,size.And GestureDetector wrap them to respond to tap.
- drawLine
line position parameters init below:
```dart 
_linePosition.clear();
Margin lineMargin = widget.lineMargin;
for (int index = 0; index < widget.lineNum; index++) {
  LinePosition tempPosition = new LinePosition(
      lineMargin.left,
      lineMargin.top * index,
      width - lineMargin.right,
      lineMargin.bottom * index);
  _linePosition.add(tempPosition);
}

if (_linePosition == null ||
    _linePosition.length == 0 ||
    _linePosition.length != widget.yLabels.length) {
  return Text(
    "data init fail",
    style: TextStyle(color: Colors.red),
  );
}
```
From init method,we can find that line positions are determined by margin,left and right margin control gap from edge of screen,top and bottom margin control distance with each line from top to bottom.

then we can draw line:
```dart
for (LinePosition linePosition in _linePosition) {
  if (widget.yLabelPosition == LabelPosition.POSITION_LEFT) {
    linePosition.x1 = linePosition.x1 + widget.margin.left;
    linePosition.x2 = linePosition.x2 - widget.margin.right;
  } else {
    linePosition.x1 = linePosition.x1 + widget.margin.left;
    linePosition.x2 = linePosition.x2 - widget.margin.right;
  }
  LinePosition curLinePosition = new LinePosition(
      linePosition.x1, linePosition.y1, linePosition.x2, linePosition.y2);
  barChartPaint = new BarChartPaint(
      widget, linePaint, curLinePosition, DRAW_TYPE.TYPE_LINE, null, 0);
  xWidgets.add(CustomPaint(
    painter: barChartPaint,
    willChange: true,
  ));
}
```
Because BarChartPaint extends CustomPainter,and when it commit a new object of BarChartPaint with painter parameter,it will start draw line from start point to end.

Draw method is easy:
```dart
if (DRAW_TYPE.TYPE_LINE == _drawType) {
//      print("x1 = ${_linePosition.x1}" + ",y1 = ${_linePosition.y1}");
  if (widget.yLabelPosition == LabelPosition.POSITION_LEFT) {
    canvas.drawLine(Offset(_linePosition.x1, _linePosition.y1),
        Offset(_linePosition.x2, _linePosition.y2), _paint);
  } else {
    canvas.drawLine(Offset(_linePosition.x1, _linePosition.y1),
        Offset(_linePosition.x2, _linePosition.y2), _paint);
  }
}
```
- draw y label
```dart
int positionSize = _linePosition.length;
int yLabelSize = widget.yLabels.length;
if (positionSize == yLabelSize) {
  for (int index = yLabelSize - 1; index >= 0; index--) {
    LinePosition curLinePosition = _linePosition[positionSize - index - 1];
    LabelPointPosition labelPointPosition;
    Label yLabel = widget.yLabels[index];
    if (widget.yLabelPosition == LabelPosition.POSITION_LEFT) {
      curLinePosition.x1 = curLinePosition.x1 + widget.margin.left;
      labelPointPosition = new LabelPointPosition(curLinePosition.x1,
          curLinePosition.y1, double.parse(yLabel.label));
      curLinePosition.y1 = curLinePosition.y1 - 7.0;
    } else {
      curLinePosition.x2 = curLinePosition.x2 - widget.margin.right;
      labelPointPosition = new LabelPointPosition(curLinePosition.x2,
          curLinePosition.y2, double.parse(yLabel.label));
      curLinePosition.y2 = curLinePosition.y2 - 7.0;
    }
    widget.yLabelPositions.add(labelPointPosition);
    LinePosition tempLinePosition = new LinePosition(curLinePosition.x1,
        curLinePosition.y1, curLinePosition.x2, curLinePosition.y2);
    barChartPaint = new BarChartPaint(widget, linePaint, tempLinePosition,
        DRAW_TYPE.TYPE_Y, yLabel, yLabelSize);
    xWidgets.add(CustomPaint(
      painter: barChartPaint,
      willChange: true,
    ));
  }
}
```
Y label accompanies line,it locates left or right of line,and can keep a little distance from start or end of line.
Starting draw y label:
```dart
if (DRAW_TYPE.TYPE_Y == _drawType) {
  TextSpan span = new TextSpan(
      style: new TextStyle(color: Color(label.color)),
      text: label.label + widget.yLabelUnit);
  TextPainter tp = new TextPainter(
      text: span,
      textAlign: TextAlign.left,
      textDirection: TextDirection.ltr);
  tp.layout();

  if (widget.yLabelPosition == LabelPosition.POSITION_LEFT) {
    tp.paint(canvas, new Offset(_linePosition.x1, _linePosition.y1));
  } else {
    tp.paint(canvas, new Offset(_linePosition.x2, _linePosition.y2));
  }
}
```
- draw x label
```dart
int xLabelSize = widget.xLabels.length;
LinePosition lastLinePosition = _linePosition[positionSize - 1];
double xLabelGap = (actualWidth - 18) / xLabelSize;
double initPosition = widget.margin.left;
double yPosition = lastLinePosition.y1;
double xLabelValue = 0;
for (int index = 0; index < xLabelSize; index++) {
  Label xLabel = widget.xLabels[index];
  double xPosition =
      lastLinePosition.x1 + initPosition + index * (xLabelGap);
  LinePosition labelPosition = new LinePosition(xPosition, yPosition, 0, 0);
  barChartPaint = new BarChartPaint(widget, linePaint, labelPosition,
      DRAW_TYPE.TYPE_X, xLabel, xLabelSize);
  if (widget.isMultiData) {
    xLabelValue = double.parse(xLabel.label);
  } else {
    xLabelValue = (index + 1).toDouble();
  }
  LabelPointPosition labelPointPosition =
      new LabelPointPosition(xPosition, yPosition, xLabelValue);
  widget.xLabelPositions.add(labelPointPosition);
  var paint = CustomPaint(
    painter: barChartPaint,
    willChange: true,
  );
  xWidgets.add(paint);
}
```
X label's x-axis start position is line's,and y-axis position is the last y-axis position of line's.The distance between each x-label is based on the screen width decrease margin and then divide x-label size.The critical parameter is isMultiData,which means whether the x-label is coordinated with value,if true,the bar can be draw between two x-labels,or one x-label by one bar.
Draw x label:
```dart
if (DRAW_TYPE.TYPE_X == _drawType) {
  TextSpan span = new TextSpan(
      style: new TextStyle(color: Color(label.color)),
      text: label.label + widget.xLabelUnit);
  TextPainter tp = new TextPainter(
      text: span,
      textAlign: TextAlign.left,
      textDirection: TextDirection.ltr);
  tp.layout();
  tp.paint(canvas, new Offset(_linePosition.x1, _linePosition.y1));
}
```
- draw data
```dart
if (widget.data != null) {
  barChartPaint = new BarChartPaint(
      widget, barPaint, null, DRAW_TYPE.TYPE_BAR, null, null);
  barChartPaint.barData = widget.data;
  barChartPaint.initDrawData();
  xWidgets.add(CustomPaint(
    painter: barChartPaint,
    willChange: true,
  ));
}
```
Before drawing bar,we need to invoke initDrawData first,because of the bar position is uncertain.
```dart
void initDrawData() {
    xLabelPositions = widget.xLabelPositions;
    yLabelPositions = widget.yLabelPositions;
    if (xLabelPositions != null && xLabelPositions.length > 0) {
      xLabelSize = xLabelPositions.length;
      if (widget.isMultiData) {
        xLabelMaxValue = xLabelPositions[xLabelSize - 1].value;
        xAverageDistance =
            (xLabelPositions[xLabelSize - 1].x - xLabelPositions[0].x) /
                xLabelMaxValue;
        firstXLabelX = xLabelPositions[0].x;
      }
    }
    if (yLabelPositions != null &&
        yLabelPositions.length > 0 &&
        _curData != null &&
        _curData.length > 0) {
      dataSize = _curData.length;
      yLabelSize = yLabelPositions.length;
      yLabelMaxValue = yLabelPositions[0].value;
      yAverageDistance =
          (yLabelPositions[yLabelSize - 1].y - yLabelPositions[0].y) /
              yLabelMaxValue;
      lastYLabelY = yLabelPositions[yLabelSize - 1].y;
    }
}
```
Pixel Distance coordinate with actual data value is vital,which decide where the bar is. xAverageDistance and yAverageDistance are born before bars to be drew.

**xAverageDistance = (x end position - x start position) / x label with max value**

**yAverageDistance = (y start position - y end position) / y label with max value**

Beginning draw data:
```dart
if (DRAW_TYPE.TYPE_BAR == _drawType && _curData != null) {
  bool barStyleValid =
      widget.barStyle != null && widget.barStyle.length == dataSize;
  for (int index = 0; index < dataSize; index++) {
    ChartData chartData = _curData[index];
    double yData = chartData.yData;
    if (yData > yLabelMaxValue) {
      break;
    }
    double dataY = 0;
    for (int yIndex = 0; yIndex < yLabelSize; yIndex++) {
      double value = yLabelPositions[yLabelSize - yIndex - 1].value;
      double yLabelY = yLabelPositions[yLabelSize - yIndex - 1].y;
//            print("value = $value" + ",yData = $yData ,yLabelY = $yLabelY");
      if (yData == 0 && value == 0) {
        dataY = lastYLabelY;
        break;
      }
      if (yData <= value) {
        dataY = lastYLabelY - yData * yAverageDistance;
//            print("value = $value" + ",yData = $yData ,yLabelY = $yLabelY");
        break;
      }
    }
    double dataX = 0;
    if (widget.isMultiData) {
      dataX = getXPosition(chartData);
    } else {
      if (index >= xLabelSize) {
        break;
      }
      double xValue = 0;
      for (LabelPointPosition tempXLabelPosition in xLabelPositions) {
        if (tempXLabelPosition.value == chartData.xData) {
          dataX = tempXLabelPosition.x;
          xValue = tempXLabelPosition.value;
          break;
        }
      }
      String valueStr = xValue.toInt().toString();
      int valueLength = valueStr.length;
      double offset = valueLength % 2 == 0
          ? 2 * valueLength.toDouble()
          : valueLength * 3 / 2.0;
      dataX += offset;
    }
    if (dataX < 0) {
      break;
    }
    print("dataX = $dataX" +
        ",dataY = $dataY" +
        ",yData = $yData ,lastYLabelY = $lastYLabelY ,yAverageDistance = $yAverageDistance");
    if (barStyleValid) {
      RRect rRect = new RRect.fromLTRBR(
          dataX,
          dataY,
          dataX + widget.barStyle[index].barWidth,
          lastYLabelY,
          Radius.circular(widget.barStyle[index].radius));
      _paint.color = Color(widget.barStyle[index].color);
      canvas.drawRRect(rRect, _paint);
    } else {
      RRect rRect = new RRect.fromLTRBR(
          dataX,
          dataY,
          dataX + widget.barStyle[0].barWidth,
          lastYLabelY,
          Radius.circular(widget.barStyle[0].radius));
      _paint.color = Color(widget.barStyle[0].color);
      canvas.drawRRect(rRect, _paint);
    }
  }
}
```
Multi data mode seems complex,
```dart
double getXPosition(ChartData chartData) {
    double xData = chartData.xData;
    double dataX = -1;
    for (int xIndex = 0; xIndex < xLabelSize; xIndex++) {
      double value = xLabelPositions[xIndex].value;
      double xLabelX = xLabelPositions[xIndex].x;
    //            print("value = $value" + ",yData = $yData ,yLabelY = $yLabelY");
      if (xData <= value) {
        String valueStr = value.toInt().toString();
        int valueLength = valueStr.length;
        double offset = valueLength % 2 == 0
            ? 2 * valueLength.toDouble()
            : valueLength * 3 / 2.0;
        dataX = firstXLabelX + xData * xAverageDistance + offset;
    //        print("value = $value" + "yLabelY = $xLabelX");
        break;
      }
    }
    return dataX;
}
```
the codes are a little bit long,the main idea is to clarify how to get bar's x and y position.If it is not multi mode,the bar's x-point equal to x-label's x point the y-point equal to y-average-distance-per-value multi value.And getXPosition method apply to multi mode.

## 6、Packge Interface
```dart
const CustomBarChart({
    Key key,
    this.lineColor = 0x7FA1A1A1,
    this.lineWidth = 1,
    this.lineNum = 3,
    @required this.xLabels,
    @required this.yLabels,
    @required this.data,
    this.onTap,
    this.xLabelUnit = "",
    this.yLabelUnit = "",
    this.yLabelPosition = LabelPosition.POSITION_RIGHT,
    this.barStyle = const [BarStyle(0, 0xFF35CABE, 6)],
    this.xLabelColor = 0xFFBCBCBC,
    this.yLabelColor = 0xFFBCBCBC,
    this.margin = const Margin(0, 0, 0, 0),
    this.isMultiData = false,
    this.lineMargin = const Margin(16, 32, 50, 50),
});
```
All parameters in this constructor can be customized,and other requirements such as chart data should be above bar,each bar can be touched with different color and so on.
## 7、How to use
```dart
Widget _getSportsDayBarChartView(
      List<SportsTypeDetail> sportsTypeDetailList) {
    var barStyles = [BarStyle(2, 0xFFFF7404, 4)];
    var chartList = _getChartData(sportsTypeDetailList);
    return CustomBarChart(
      xLabels: BarChartCommon.getXLabels(0),
      yLabels: BarChartCommon.getYLabel(chartList, ChartYLabelType.TYPE_STEP),
      data: chartList,
      lineNum: 3,
      yLabelUnit: "",
      isMultiData: true,
      yLabelPosition: LabelPosition.POSITION_RIGHT,
      barStyle: barStyles,
      lineMargin: Margin(16, 32, 64, 64),
      onTap: (value) {
        Navigator.push(
          context,
          MaterialPageRoute(
            builder: (context) => SingleDaySportsListView(
                sportsTypeDetailList: sportsTypeDetailList),
          ),
        );
      },
    );
  }
```
## 8、flutter codes
```dart
import 'package:flutter/material.dart';
import 'dart:ui';

///CustomBarChart
class CustomBarChart extends StatefulWidget {
  final int lineColor;
  final double lineWidth;
  final int lineNum;

  final List<Label> xLabels;

  ///y label number should equal to lineNum
  final List<Label> yLabels;
  final List<ChartData> data;
  final String xLabelUnit;
  final String yLabelUnit;
  final LabelPosition yLabelPosition;
  final List<BarStyle> barStyle;

  final int xLabelColor;
  final int yLabelColor;

  final Margin margin;

  ///line left and right margin should consider y unit position
  ///and line top and bottom margin related to gap between each line
  final Margin lineMargin;

  ///true: labels are not coordinate with data,bar can be draw between labels
  ///false:labels are related with data from one by one
  final bool isMultiData;

  static List<LabelPointPosition> _xLabelPositions = [];
  static List<LabelPointPosition> _yLabelPositions = [];

  get xLabelPositions => _xLabelPositions;

  get yLabelPositions => _yLabelPositions;

  final ValueChanged<String> onTap;
  final int heightOffset = 18;

  const CustomBarChart({
    Key key,
    this.lineColor = 0x7FA1A1A1,
    this.lineWidth = 1,
    this.lineNum = 3,
    @required this.xLabels,
    @required this.yLabels,
    @required this.data,
    this.onTap,
    this.xLabelUnit = "",
    this.yLabelUnit = "",
    this.yLabelPosition = LabelPosition.POSITION_RIGHT,
    this.barStyle = const [BarStyle(0, 0xFF35CABE, 6)],
    this.xLabelColor = 0xFFBCBCBC,
    this.yLabelColor = 0xFFBCBCBC,
    this.margin = const Margin(0, 0, 0, 0),
    this.isMultiData = false,
    this.lineMargin = const Margin(16, 32, 50, 50),
  });

  @override
  _CustomBarChartState createState() => _CustomBarChartState();

  void clearLabelPointPosition() {
    _xLabelPositions.clear();
    _yLabelPositions.clear();
  }
}

class _CustomBarChartState extends State<CustomBarChart> {
  Paint linePaint;
  Paint barPaint;
  double width;
  double actualWidth = 0;
  List<LinePosition> _linePosition = [];

  @override
  void initState() {
    super.initState();
    linePaint = new Paint();
    linePaint.color = Color(widget.lineColor);
    linePaint.strokeWidth = widget.lineWidth;

    barPaint = new Paint();
    if (widget.barStyle.length == 1) {
      barPaint.color = Color(widget.barStyle[0].color);
      barPaint.strokeWidth = widget.barStyle[0].barWidth;
    }
  }

  @override
  Widget build(BuildContext context) {
    width = MediaQuery.of(context).size.width;
    if (widget.lineNum == 0 ||
        widget.xLabels == null ||
        widget.xLabels.length == 0 ||
        widget.yLabels == null ||
        widget.yLabels.length == 0 ||
        widget.yLabels.length != widget.lineNum) {
      return Text(
        "data init fail",
        style: TextStyle(color: Colors.red),
      );
    }
    _linePosition.clear();
    Margin lineMargin = widget.lineMargin;
    for (int index = 0; index < widget.lineNum; index++) {
      LinePosition tempPosition = new LinePosition(
          lineMargin.left,
          lineMargin.top * index,
          width - lineMargin.right,
          lineMargin.bottom * index);
      _linePosition.add(tempPosition);
    }

    if (_linePosition == null ||
        _linePosition.length == 0 ||
        _linePosition.length != widget.yLabels.length) {
      return Text(
        "data init fail",
        style: TextStyle(color: Colors.red),
      );
    }
    actualWidth = width - widget.margin.left - widget.margin.right;
    return GestureDetector(
      child: Container(
        height: widget.lineMargin.top * (widget.lineNum - 1) +
            widget.margin.top +
            widget.margin.bottom +
            widget.heightOffset,
        width: actualWidth,
        color: Colors.white,
        margin: EdgeInsets.only(
            left: widget.margin.left,
            top: widget.margin.top,
            right: widget.margin.right,
            bottom: widget.margin.bottom),
        child: Stack(
          children: drawSkeleton(),
        ),
      ),
      onTap: () {
        widget.onTap("clicked");
      },
    );
  }

  List<Widget> drawSkeleton() {
    List<Widget> xWidgets = [];
    BarChartPaint barChartPaint;
    for (LinePosition linePosition in _linePosition) {
      if (widget.yLabelPosition == LabelPosition.POSITION_LEFT) {
        linePosition.x1 = linePosition.x1 + widget.margin.left;
        linePosition.x2 = linePosition.x2 - widget.margin.right;
      } else {
        linePosition.x1 = linePosition.x1 + widget.margin.left;
        linePosition.x2 = linePosition.x2 - widget.margin.right;
      }
      LinePosition curLinePosition = new LinePosition(
          linePosition.x1, linePosition.y1, linePosition.x2, linePosition.y2);
      barChartPaint = new BarChartPaint(
          widget, linePaint, curLinePosition, DRAW_TYPE.TYPE_LINE, null, 0);
      xWidgets.add(CustomPaint(
        painter: barChartPaint,
        willChange: true,
      ));
    }
    widget.clearLabelPointPosition();
    int positionSize = _linePosition.length;
    int yLabelSize = widget.yLabels.length;
    if (positionSize == yLabelSize) {
      for (int index = yLabelSize - 1; index >= 0; index--) {
        LinePosition curLinePosition = _linePosition[positionSize - index - 1];
        LabelPointPosition labelPointPosition;
        Label yLabel = widget.yLabels[index];
        if (widget.yLabelPosition == LabelPosition.POSITION_LEFT) {
          curLinePosition.x1 = curLinePosition.x1 + widget.margin.left;
          labelPointPosition = new LabelPointPosition(curLinePosition.x1,
              curLinePosition.y1, double.parse(yLabel.label));
          curLinePosition.y1 = curLinePosition.y1 - 7.0;
        } else {
          curLinePosition.x2 = curLinePosition.x2 - widget.margin.right;
          labelPointPosition = new LabelPointPosition(curLinePosition.x2,
              curLinePosition.y2, double.parse(yLabel.label));
          curLinePosition.y2 = curLinePosition.y2 - 7.0;
        }
        widget.yLabelPositions.add(labelPointPosition);
        LinePosition tempLinePosition = new LinePosition(curLinePosition.x1,
            curLinePosition.y1, curLinePosition.x2, curLinePosition.y2);
        barChartPaint = new BarChartPaint(widget, linePaint, tempLinePosition,
            DRAW_TYPE.TYPE_Y, yLabel, yLabelSize);
        xWidgets.add(CustomPaint(
          painter: barChartPaint,
          willChange: true,
        ));
      }
    }
    int xLabelSize = widget.xLabels.length;
    LinePosition lastLinePosition = _linePosition[positionSize - 1];
    double xLabelGap = (actualWidth - 18) / xLabelSize;
    double initPosition = widget.margin.left;
    double yPosition = lastLinePosition.y1;
    double xLabelValue = 0;
    for (int index = 0; index < xLabelSize; index++) {
      Label xLabel = widget.xLabels[index];
      double xPosition =
          lastLinePosition.x1 + initPosition + index * (xLabelGap);
      LinePosition labelPosition = new LinePosition(xPosition, yPosition, 0, 0);
      barChartPaint = new BarChartPaint(widget, linePaint, labelPosition,
          DRAW_TYPE.TYPE_X, xLabel, xLabelSize);
      if (widget.isMultiData) {
        xLabelValue = double.parse(xLabel.label);
      } else {
        xLabelValue = (index + 1).toDouble();
      }
      LabelPointPosition labelPointPosition =
          new LabelPointPosition(xPosition, yPosition, xLabelValue);
      widget.xLabelPositions.add(labelPointPosition);
      var paint = CustomPaint(
        painter: barChartPaint,
        willChange: true,
      );
      xWidgets.add(paint);
    }

    if (widget.data != null) {
      barChartPaint = new BarChartPaint(
          widget, barPaint, null, DRAW_TYPE.TYPE_BAR, null, null);
      barChartPaint.barData = widget.data;
      barChartPaint.initDrawData();
      xWidgets.add(CustomPaint(
        painter: barChartPaint,
        willChange: true,
      ));
    }

    return xWidgets;
  }
}

class BarChartPaint extends CustomPainter {
  final Paint _paint;
  final LinePosition _linePosition;
  final DRAW_TYPE _drawType;
  final Label label;
  final int labelSize;
  final CustomBarChart widget;

  BarChartPaint(this.widget, this._paint, this._linePosition, this._drawType,
      this.label, this.labelSize);

  List<ChartData> _curData;
  int xLabelSize = 0;
  double xLabelMaxValue = 0;
  double xAverageDistance = 0;
  double firstXLabelX = 0;
  List<LabelPointPosition> xLabelPositions;
  List<LabelPointPosition> yLabelPositions;

  int dataSize = 0;
  int yLabelSize = 0;
  double yLabelMaxValue = 0;
  double yAverageDistance = 0;
  double lastYLabelY = 0;

  set barData(List<ChartData> data) {
    _curData = data;
  }

  void initDrawData() {
    xLabelPositions = widget.xLabelPositions;
    yLabelPositions = widget.yLabelPositions;
    if (xLabelPositions != null && xLabelPositions.length > 0) {
      xLabelSize = xLabelPositions.length;
      if (widget.isMultiData) {
        xLabelMaxValue = xLabelPositions[xLabelSize - 1].value;
        xAverageDistance =
            (xLabelPositions[xLabelSize - 1].x - xLabelPositions[0].x) /
                xLabelMaxValue;
        firstXLabelX = xLabelPositions[0].x;
      }
    }
    if (yLabelPositions != null &&
        yLabelPositions.length > 0 &&
        _curData != null &&
        _curData.length > 0) {
      dataSize = _curData.length;
      yLabelSize = yLabelPositions.length;
      yLabelMaxValue = yLabelPositions[0].value;
      yAverageDistance =
          (yLabelPositions[yLabelSize - 1].y - yLabelPositions[0].y) /
              yLabelMaxValue;
      lastYLabelY = yLabelPositions[yLabelSize - 1].y;
    }
  }

  double getXPosition(ChartData chartData) {
    double xData = chartData.xData;
    double dataX = -1;
    for (int xIndex = 0; xIndex < xLabelSize; xIndex++) {
      double value = xLabelPositions[xIndex].value;
      double xLabelX = xLabelPositions[xIndex].x;
//            print("value = $value" + ",yData = $yData ,yLabelY = $yLabelY");
      if (xData <= value) {
        String valueStr = value.toInt().toString();
        int valueLength = valueStr.length;
        double offset = valueLength % 2 == 0
            ? 2 * valueLength.toDouble()
            : valueLength * 3 / 2.0;
        dataX = firstXLabelX + xData * xAverageDistance + offset;
//        print("value = $value" + "yLabelY = $xLabelX");
        break;
      }
    }
    return dataX;
  }

  @override
  void paint(Canvas canvas, Size size) {
    if (DRAW_TYPE.TYPE_LINE == _drawType) {
//      print("x1 = ${_linePosition.x1}" + ",y1 = ${_linePosition.y1}");
      if (widget.yLabelPosition == LabelPosition.POSITION_LEFT) {
        canvas.drawLine(Offset(_linePosition.x1, _linePosition.y1),
            Offset(_linePosition.x2, _linePosition.y2), _paint);
      } else {
        canvas.drawLine(Offset(_linePosition.x1, _linePosition.y1),
            Offset(_linePosition.x2, _linePosition.y2), _paint);
      }
    }
    if (DRAW_TYPE.TYPE_Y == _drawType) {
      TextSpan span = new TextSpan(
          style: new TextStyle(color: Color(label.color)),
          text: label.label + widget.yLabelUnit);
      TextPainter tp = new TextPainter(
          text: span,
          textAlign: TextAlign.left,
          textDirection: TextDirection.ltr);
      tp.layout();

      if (widget.yLabelPosition == LabelPosition.POSITION_LEFT) {
        tp.paint(canvas, new Offset(_linePosition.x1, _linePosition.y1));
      } else {
        tp.paint(canvas, new Offset(_linePosition.x2, _linePosition.y2));
      }
    }
    if (DRAW_TYPE.TYPE_X == _drawType) {
      TextSpan span = new TextSpan(
          style: new TextStyle(color: Color(label.color)),
          text: label.label + widget.xLabelUnit);
      TextPainter tp = new TextPainter(
          text: span,
          textAlign: TextAlign.left,
          textDirection: TextDirection.ltr);
      tp.layout();
      tp.paint(canvas, new Offset(_linePosition.x1, _linePosition.y1));
    }
    if (DRAW_TYPE.TYPE_BAR == _drawType && _curData != null) {
      bool barStyleValid =
          widget.barStyle != null && widget.barStyle.length == dataSize;
      for (int index = 0; index < dataSize; index++) {
        ChartData chartData = _curData[index];
        double yData = chartData.yData;
        if (yData > yLabelMaxValue) {
          break;
        }
        double dataY = 0;
        for (int yIndex = 0; yIndex < yLabelSize; yIndex++) {
          double value = yLabelPositions[yLabelSize - yIndex - 1].value;
          double yLabelY = yLabelPositions[yLabelSize - yIndex - 1].y;
//            print("value = $value" + ",yData = $yData ,yLabelY = $yLabelY");
          if (yData == 0 && value == 0) {
            dataY = lastYLabelY;
            break;
          }
          if (yData <= value) {
            dataY = lastYLabelY - yData * yAverageDistance;
//            print("value = $value" + ",yData = $yData ,yLabelY = $yLabelY");
            break;
          }
        }
        double dataX = 0;
        if (widget.isMultiData) {
          dataX = getXPosition(chartData);
        } else {
          if (index >= xLabelSize) {
            break;
          }
          double xValue = 0;
          for (LabelPointPosition tempXLabelPosition in xLabelPositions) {
            if (tempXLabelPosition.value == chartData.xData) {
              dataX = tempXLabelPosition.x;
              xValue = tempXLabelPosition.value;
              break;
            }
          }
          String valueStr = xValue.toInt().toString();
          int valueLength = valueStr.length;
          double offset = valueLength % 2 == 0
              ? 2 * valueLength.toDouble()
              : valueLength * 3 / 2.0;
          dataX += offset;
        }
        if (dataX < 0) {
          break;
        }
        print("dataX = $dataX" +
            ",dataY = $dataY" +
            ",yData = $yData ,lastYLabelY = $lastYLabelY ,yAverageDistance = $yAverageDistance");
        if (barStyleValid) {
          RRect rRect = new RRect.fromLTRBR(
              dataX,
              dataY,
              dataX + widget.barStyle[index].barWidth,
              lastYLabelY,
              Radius.circular(widget.barStyle[index].radius));
          _paint.color = Color(widget.barStyle[index].color);
          canvas.drawRRect(rRect, _paint);
        } else {
          RRect rRect = new RRect.fromLTRBR(
              dataX,
              dataY,
              dataX + widget.barStyle[0].barWidth,
              lastYLabelY,
              Radius.circular(widget.barStyle[0].radius));
          _paint.color = Color(widget.barStyle[0].color);
          canvas.drawRRect(rRect, _paint);
        }
      }
    }
  }

  @override
  bool shouldRepaint(CustomPainter oldDelegate) {
    return true;
  }
}

enum DRAW_TYPE { TYPE_LINE, TYPE_X, TYPE_Y, TYPE_BAR }

class Label {
  String label;
  int color;

  Label(this.label, this.color);
}

class LinePosition {
  double x1;
  double y1;

  double x2;
  double y2;

  LinePosition(this.x1, this.y1, this.x2, this.y2);
}

class LabelPointPosition {
  final double x;
  final double y;
  final double value;

  LabelPointPosition(this.x, this.y, this.value);
}

enum LabelPosition { POSITION_LEFT, POSITION_RIGHT }

class BarStyle {
  final double radius;
  final int color;
  final double barWidth;

  // ignore: invalid_required_param
  const BarStyle(
      @required this.radius, @required this.color, @required this.barWidth);
}

class ChartData {
  double yData;
  double xData;

  ChartData(this.xData, this.yData);
}

class Margin {
  final double left;
  final double right;
  final double top;
  final double bottom;

  const Margin(this.left, this.right, this.top, this.bottom);
}
```

## 9、java codes
```dart
package com.qiku.healthguard.view;

import android.content.Context;
import android.content.res.Resources;
import android.graphics.Bitmap;
import android.graphics.Canvas;
import android.graphics.Paint;
import android.graphics.Point;
import android.graphics.Rect;
import android.graphics.RectF;
import android.util.AttributeSet;
import android.util.DisplayMetrics;
import android.util.TypedValue;
import android.view.View;

import com.qiku.healthguard.R;
import com.qiku.healthguard.bean.ChartDataBean;
import com.qiku.healthguard.sport.common.util.SDKUtils;
import com.qiku.healthguard.util.SystemDataUtil;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * Created by chuck chan-iri on 2017/5/19.
 */

public class CustomBarChartView extends View {
    public final static int TYPE_TIME = 0;
    public final static int TYPE_WEEK = 1;
    public final static int TYPE_MONTH = 2;
    public final static int TYPE_YEAR = 3;

    private static DisplayMetrics mMetrics;
    float yMaxData, yDataForAveragePosition;
    private String TAG = "HG-CustomBarChartView";
    private float offsetLeft = 18;
    private float offsetRight = 18;
    private float offsetBottom = 20;
    private float leftOffsetInPixel = 0;
    private float bottomOffsetInPixel = 0;
    private int width = 0;
    private int height = 0;
    private int positionLeft = 0;
    private int positionRight = 0;
    private int positionTop = 0;
    private int positionBottom = 0;
    private float xGap, yGap, xStartPosition, xEndPosition, yStartPosition,
            yEndPosition, yBaseStartPosition, yBaseEndPosition, rectGap;
    private Paint xPaint = null;
    private Paint yPaint = null;
    private Paint linePaint = null;
    private Paint rectPaint = null;
    private List<String> xItem;
    private int xType = -1;
    private List<String> yItem;
    private int lineColor = R.color.color_A1;
    private int labelColor = R.color.color_BC;
    private List<ChartDataBean> dataList;
    private RectF drawRectF;
    private float rectWidth = 4f;
    private int rectColor = R.color.color_35;
    private Map<String, Point> pointMap = new HashMap<>();
    private boolean isMultiData = false;
    private String[] WEEK_X_ARRAY = null;
    private String yLabelUnit = "h";
    //draw x label only for empty data set
    private boolean isDrawXLabelOnly = false;

    public CustomBarChartView(Context context) {
        this(context, null);
    }

    public CustomBarChartView(Context context, AttributeSet attrs) {
        super(context, attrs);
        Resources res = context.getResources();
        mMetrics = res.getDisplayMetrics();
        xPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        float xLabelSize = TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_SP,
                10, mMetrics);
        xPaint.setTextSize(xLabelSize);
        xPaint.setColor(getResources().getColor(labelColor));
        yPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        float yLabelSize = TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_SP,
                10, mMetrics);
        yPaint.setTextSize(yLabelSize);
        yPaint.setColor(getResources().getColor(labelColor));
        linePaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        linePaint.setColor(getResources().getColor(lineColor));
        rectPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        rectPaint.setColor(getResources().getColor(rectColor));
        leftOffsetInPixel = dp2Pixel(offsetLeft);
        bottomOffsetInPixel = dp2Pixel(offsetBottom);
        rectWidth = dp2Pixel(rectWidth);

        drawRectF = new RectF();
        setWeekArray(res);
    }

    private void setWeekArray(Resources resource) {
        WEEK_X_ARRAY = new String[]{
                resource.getString(R.string.week_1), resource.getString(R.string.week_2),
                resource.getString(R.string.week_3), resource.getString(R.string.week_4),
                resource.getString(R.string.week_5), resource.getString(R.string.week_6),
                resource.getString(R.string.week_7)};
    }

    public List<ChartDataBean> getData() {
        return dataList;
    }

    public void setData(List<ChartDataBean> list) {
        dataList = list;
    }

    private float dp2Pixel(float dp) {
        return dp * mMetrics.density;
    }

    public void setXLabelData(List<String> xItem) {
        this.xItem = xItem;
    }

    public void setXLabelDataType(int type) {
        xType = type;
    }

    public void setYLabelData(List<String> yItem) {
        this.yItem = yItem;
    }

    public void setRectWidth(int rectWidth) {
        this.rectWidth = dp2Pixel(rectWidth);
    }

    public void setRectColor(int rectColor) {
        this.rectColor = rectColor;
        rectPaint.setColor(rectColor);
    }

    public void setIsMultiData(boolean isMultiData) {
        this.isMultiData = isMultiData;
    }

    public void setyLabelUnit(String yLabelUnit) {
        this.yLabelUnit = yLabelUnit;
    }

    private String getXLabel(String label, int index) {
        if (index > xItem.size()) {
            return "";
        }
        if (xType == TYPE_TIME) {
            if (label.equals("24")) {
                label = "0";
            }
            return label;
        } else if (xType == TYPE_WEEK) {
            return WEEK_X_ARRAY[index];
        } else {
            return xItem.get(index);
        }
    }

    public void setDrawXLabelOnly(boolean drawXLabelOnly) {
        isDrawXLabelOnly = drawXLabelOnly;
    }

    @Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        super.onSizeChanged(w, h, oldw, oldh);
        width = w;
        height = h;
        positionLeft = 0;
        positionTop = 0;
        positionRight = positionLeft + width;
        positionBottom = positionTop + height;
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        if (xItem == null || yItem == null) {
            return;
        }
        pointMap.clear();
        xStartPosition = positionLeft + leftOffsetInPixel;
        xEndPosition = positionRight - leftOffsetInPixel;
        xGap = (xEndPosition - xStartPosition) / (xItem.size());

        yStartPosition = positionBottom - positionTop;
        yEndPosition = 0;
        yBaseStartPosition = yStartPosition - getResources().getDimensionPixelOffset(R.dimen.view_margin_40dp);
        yBaseEndPosition = yEndPosition;

        if (isDrawXLabelOnly) {
            drawX(canvas);
            return;
        }
        if (dataList == null || dataList.size() == 0) {
            return;
        }

        yMaxData = Float.parseFloat(yItem.get(yItem.size() - 1));
        yDataForAveragePosition = (yBaseStartPosition - yBaseEndPosition) / yMaxData;
        yGap = (yBaseStartPosition - yBaseEndPosition) / (yItem.size() - 1);

        float yBaseStartPositionTmp = yBaseStartPosition;
        try {
            for (ChartDataBean mChartDataBean : dataList) {
                if (mChartDataBean.getDataY() <= 0 || mChartDataBean.getResIds() == null || mChartDataBean.getResIds().size() <= 0)
                    continue;
                if (mChartDataBean.getResIds() != null && mChartDataBean.getResIds().size() >= 1) {
                    for (Integer resId : mChartDataBean.getResIds()) {
                        Bitmap bitmap = SystemDataUtil.getBitmap(mContext, resId);
                        if (bitmap == null) {
                            continue;
                        }
                        yBaseStartPositionTmp -= bitmap.getHeight() + SDKUtils.dip2px(mContext, 4);
                    }
                }
                float tmp = (yBaseStartPositionTmp - yBaseEndPosition) / mChartDataBean.getDataY();
                if (yDataForAveragePosition > tmp) yDataForAveragePosition = tmp;
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

        drawX(canvas);
        drawY(canvas);

        drawData(canvas);
    }

    private void drawX(Canvas canvas) {
        float xBasePosition = 0f;
        int xItemSize = xItem.size();
        if (xType == TYPE_YEAR) {
            xBasePosition = xStartPosition + getResources().getDimensionPixelOffset(R.dimen.view_margin_4dp);
        } else if (xType == TYPE_MONTH || xType == TYPE_WEEK) {
            xBasePosition = xStartPosition + getResources().getDimensionPixelOffset(R.dimen.view_margin_12dp);
        } else {
            xBasePosition = xStartPosition + getResources().getDimensionPixelOffset(R.dimen.view_margin_24dp);
        }
        float yBasePosition = yStartPosition - bottomOffsetInPixel;
        for (int i = 0; i < xItemSize; ++i) {
            String tempXItem = xItem.get(i);
            float x = xBasePosition + xGap * i;
            float y = yBasePosition;
//            Log.d(TAG, "x = " + x + ",y = " + y);
            pointMap.put(tempXItem, new Point((int) x, (int) y));
            tempXItem = getXLabel(tempXItem, i);
            canvas.drawText(tempXItem + "", x, y, xPaint);
        }
    }

    private void drawData(Canvas canvas) {
        int nextKeyValue = -1;
        int previousValue = -1;
        for (int j = 0; j < dataList.size(); ++j) {
            ChartDataBean data = dataList.get(j);
            int dataOne = data.getDataX();
            float dataTwo = data.getDataY();
//            MyLog.d(TAG, "drawData dataOne = " + dataOne + ",dataTwo = " + dataTwo);

            float left = 0f;
            float top = 0f;
            float right = 0f;
            float bottom = 0f;
            boolean canDraw = false;
            int[] dataGap = getProperValueGap(dataOne);
            previousValue = dataGap[0];
            nextKeyValue = dataGap[1];

            Point tempPoint = pointMap.get("" + previousValue);
            if (tempPoint == null) {
                continue;
            }
            int x = tempPoint.x;
            int y = tempPoint.y;
            if (isMultiData) {
                if (dataOne >= previousValue && dataOne <= nextKeyValue) {
                    canDraw = true;
                }
                rectGap = (xGap - (rectWidth * (nextKeyValue - previousValue))) / (nextKeyValue - previousValue);
                left = x + (dataOne - previousValue) * (rectWidth + rectGap);
                top = yBaseStartPosition - yDataForAveragePosition * dataTwo;
                right = left + rectWidth;
                bottom = yBaseStartPosition;
            } else {
                canDraw = true;
                left = x;
                top = yBaseStartPosition - yDataForAveragePosition * dataTwo;
                right = left + rectWidth;
                bottom = yBaseStartPosition;
            }
            if (canDraw && dataTwo >= 0f) {
                drawRectF.set(left, top, right, bottom);
                canvas.drawRoundRect(drawRectF, 12f, 12f, rectPaint);
            }
            try {
                if (data.getResIds() != null && data.getResIds().size() >= 1) {
                    for (Integer resId : data.getResIds()) {
                        Bitmap bitmap = SystemDataUtil.getBitmap(mContext, resId);
                        if (bitmap == null) {
                            continue;
                        }
                        float mBpWidth = bitmap.getWidth() / 2f;
                        top = top - bitmap.getHeight() - SDKUtils.dip2px(mContext, 4);
                        canvas.drawBitmap(bitmap, left + (rectWidth / 2) - mBpWidth, top, null);
                    }
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    private int[] getProperValueGap(int currentData) {
        int nextKeyValue = -1;
        int previousValue = -1;
        int size = xItem.size();
        if (!isMultiData) {
            return new int[]{currentData, currentData};
        }
        for (int i = 0; i < xItem.size(); ++i) {
            String tempLabel = xItem.get(i);
            int nextKey = i + 1;
            int curHour = Integer.parseInt(tempLabel);
            if (nextKey < size) {
                previousValue = curHour;
                nextKeyValue = Integer.parseInt(xItem.get(nextKey));
            } else {
                nextKeyValue = curHour;
                if (currentData == curHour && i == size - 1) {
                    previousValue = Integer.parseInt(xItem.get(i - 1));
                } else {
                    previousValue = Integer.parseInt(xItem.get(nextKey - 1));
                }
            }
            if (currentData >= previousValue && currentData < nextKeyValue) {
                break;
            }
        }
//        MyLog.d(TAG, "getProperValueGap currentData = " + currentData + ",previousValue = " + previousValue + ",nextKeyValue = " + nextKeyValue);
        return new int[]{previousValue, nextKeyValue};
    }

    private void drawY(Canvas canvas) {
        int offset = SDKUtils.dip2px(getContext(), 2);
        for (int i = 0; i < yItem.size(); ++i) {
            String tempYItem = yItem.get(i);
            float yLinePosition = yBaseStartPosition - yGap * i;

            Rect rect = new Rect();
            yPaint.getTextBounds((tempYItem + yLabelUnit), 0, (tempYItem + yLabelUnit).length(), rect);
            int w = rect.width();
            int h = rect.height();

            canvas.drawText(tempYItem + yLabelUnit, xEndPosition - w + offset, yLinePosition + offset + h, yPaint);
//            Log.d(TAG, "drawY x = " + xEndPosition + ",yLinePosition = " + yLinePosition);
            canvas.drawLine(xStartPosition, yLinePosition, xEndPosition, yLinePosition, linePaint);
        }
    }
}
```
