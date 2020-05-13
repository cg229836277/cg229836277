---
layout: post
title:  "Flutter-A glimpse of flutter"
date:   2019-3-27 11:09:22 +0800
categories: flutter
---

# 1、Official website
We can get numerous knowledge about flutter at [official website](https://flutter.dev/docs/resources/technical-overview).
The basic skeleton of flutter is below:
![](https://i.loli.net/2019/03/26/5c999af0bae79.png)
We can find out that it can be apply to Android and iOS platform,and package Android material design for Android developers.
What we care about is how we can develop application ignore platform difference,fortunately,google has taken this into account.

# 2、Platform invoke
Channel deliver a way to communicate with platform plugins using asynchronous method calls.
Including MethodChannel,EventChannel,BasicMessageChannel.
## (1)MethodChannel
[MethodChannel](https://docs.flutter.io/flutter/services/MethodChannel-class.html) provide a method for flutter to invoke Android method,Android kotlin codes:
```kotlin
MethodChannel(flutterView, CHANNEL).setMethodCallHandler { p0, p1 ->
    val method = p0!!.method
    MyLog.d(TAG, method)
    if (method == "showToast") {
        if (p0!!.hasArgument("msg") && !TextUtils.isEmpty(p0!!.argument<String>("msg") as String)) {
            context.toast(p0!!.argument<String>("msg") as String)
        } else {
            context.toast("toast text must not null")
        }
    } else if (method == "getData") {
        if(Looper.getMainLooper() == Looper.myLooper()){
            MyLog.d(TAG, "in main thread")
        }
        val type = p0!!.argument<Int>("type") as Int
        MyLog.d(TAG, "getData type:$type")
        if (type == GetDataManager.TYPE_YEAR) {
            if (!yearlyDataList.isEmpty()) {
                yearlyDataList.clear()
            }
        }
        getDataManager.getData(curTime, type)
    } else if (method == "getAppIcon") {
        val pkgName = p0!!.argument<String>("pkgName") as String
        val pkgIcon = SystemDataUtil.getAppIcon(this, pkgName)
        p1.success(pkgIcon)
    } else if (method == "getAppIconList") {
        val pkgNameList = p0!!.argument<List<String>>("pkgNameList") as List<String>
        if (pkgNameList != null && pkgNameList.isNotEmpty()) {
            val pkgIconList = mutableListOf<String>()
            pkgNameList.forEach {
                val pkgIcon = SystemDataUtil.getAppIcon(this, it)
                pkgIconList.add(pkgIcon!!)
            }
            p1.success(pkgIconList)
        }
    }
}
```
The codes define in onCreate method,and can not locate somewhere else,because methodchannel is unsafe thread.In addition,if we wanna return data from native to flutter,invoke:
```dart
p1.success(data)
```
And flutter invoke above codes with a name:
```dart
static const platform =
  const MethodChannel("com.example.test/mainActivity");
```
Initializing a methodchannel object first:
```dart
Future<void> showToast() async {
	try {
	  await platform.invokeMethod("showToast", {"msg": "Toast"});
	} on PlatformException catch (e) {
	  print(e.toString());
	}
}

Future<void> getData(int type) async {
	try {
	  await platform.invokeMethod("getData", {"type": type});
	} on PlatformException catch (e) {
	  print(e.toString());
	}
}

Future<String> getAppIcon(String pkgName) async {
	Future<String> result;
	try {
	  print("getAppIcon $pkgName");
	  result = await platform.invokeMethod("getAppIcon", {"pkgName": pkgName});
	} on PlatformException catch (e) {
	  print(e.toString());
	}
	return result;
}

Future<List<String>> getAppIconList(List<AppUsagePercent> tempAppUsagePercentList) async {
	if(tempAppUsagePercentList == null){
	  return null;
	}
	Future<List<String>> result;
	List<String> pkgNames = [];
	for (AppUsagePercent appUsagePercent in tempAppUsagePercentList) {
	  pkgNames.add(appUsagePercent.pkgName);
	}
	try {
	  var data = await platform.invokeMethod("getAppIconList", {"pkgNameList": pkgNames});
	  print("get app icon data finished");
	} on Exception catch (e) {
	  print(e.toString());
	}
	return result;
}
```
## (2)EventChannel
[EventChannel](https://docs.flutter.io/flutter/services/EventChannel-class.html) can deliver native message to flutter,such as broadcast or something else.
Defining the listen method in flutter first:
```dart
static const EventChannel eventChannel =
  const EventChannel("com.example.test/typeData");
 ```
 And then:
 ```dart
eventChannel.receiveBroadcastStream().listen(_onEvent, onError: _onError);
void addListener(InteractListener listener) {
    if (listener == null) {
      return;
    }
    listenerList.add(listener);
}

void _onEvent(Object event) {
    print("_onEvent is invoke$event");
    for (InteractListener listener in listenerList) {
      listener.onEvent(event);
    }
}

void _onError(Object error) {
    for (InteractListener listener in listenerList) {
      listener.onError(error);
    }
}
```
What's happened in native platform:
```kotlin
lateinit var listenEvents: EventChannel.EventSink
```
init a `EventChannel.EventSink` object to operate data.
Then register the event stream handler:
```kotlin
EventChannel(flutterView, DATA_RESULT_CHANNEL).setStreamHandler(object : EventChannel.StreamHandler {
    override fun onListen(arguments: Any?, events: EventChannel.EventSink?) {
        listenEvents = events!!
    }

    override fun onCancel(arguments: Any?) {
    }
})
```
At last,we can throw data from native to flutter:
```kotlin
@Subscribe(threadMode = ThreadMode.BACKGROUND)
fun receiveYearlyData(usageDetailForEventBusBean: UsageDetailForEventBusBean) {
    synchronized(GetDataManager::class.java) {
        MyLog.d(TAG, "receiveYearlyData:${usageDetailForEventBusBean.usageDetailBean}")
        if (usageDetailForEventBusBean.isFinish) {
            listenEvents!!.success(JsonUtil.toJson(yearlyDataList))
            return
        }
        yearlyDataList.add(usageDetailForEventBusBean.usageDetailBean!!)
    }
}
```
## (3)BasicMessageChannel
[BasicMessageChannel](https://api.flutter.dev//flutter/services/BasicMessageChannel-class.html) is similar to Method channel,The Dart type of messages sent and received is T, but only the values supported by the specified `MessageCodec` can be used,the difference is T data type,which supports basic, asynchronous message passing using a custom message codec. Further, you can use the specialized `BinaryCodec`, `StringCodec`, and `JSONMessageCodec` classes, or create your own codec.
The following table shows how Dart values are received on the platform side and vice versa:
|Dart	|Android | iOS |
| ----- | :--------:  | :---------: | :------: |
|null	|null	|nil (NSNull when nested)
|bool	|java.lang.Boolean	|NSNumber numberWithBool:|
|int	|java.lang.Integer	|NSNumber numberWithInt:|
|int, if 32 bits not enough	|java.lang.Long|	NSNumber numberWithLong:|
|double	|java.lang.Double	|NSNumber numberWithDouble:|
|String	|java.lang.String	|NSString|
|Uint8List	|byte[]	|FlutterStandardTypedData typedDataWithBytes:|
|Int32List|	int[]|	FlutterStandardTypedData typedDataWithInt32:|
|Int64List	|long[]|	FlutterStandardTypedData typedDataWithInt64:|
|Float64List	|double[]|	FlutterStandardTypedData typedDataWithFloat64:|
|List	|java.util.ArrayList|	NSArray|
|Map	|java.util.HashMap|	NSDictionary|
This [website](https://flutter.dev/docs/development/platform-integration/platform-channels#codec) can get some knowledge about platform channel.
And alibaba's xianyu tech team published some articles in yuque,the url is <https://www.yuque.com/xytech/flutter/>.

# 3、Widget
Widget includes `StatefulWidget` and `StatelessWidget` class.The lifecycle below:
- createState()
- mounted == true
- initState()
- didChangeDependencies()
- build()
- didUpdateWidget()
- setState()
- deactivate()
- dispose()
- mounted == false
## (1)[StatefulWidget](https://docs.flutter.io/flutter/widgets/StatefulWidget-class.html)
`StatefulWidget` includes `createState` method which init `State` class,`State` class can update widgets by `setState` method,and it contains `initState` and `build` method.
```dart
class TimeView extends StatefulWidget {
  @override
  _TimeViewState createState() => _TimeViewState();
}

class _TimeViewState extends State<TimeView> implements InteractListener {
  @override
  void initState() {
    super.initState();
  }

  @override
  Widget build(BuildContext context) {
    return Container(child:Text(""));
  }
}
 ```
## (2)[StatelessWidget](https://docs.flutter.io/flutter/widgets/StatelessWidget-class.html)
The build method of a stateless widget is typically only called in three situations: the first time the widget is inserted in the tree, when the widget's parent changes its configuration, and when an `InheritedWidget` it depends on changes.
```dart
class MyApp extends StatelessWidget {

  @override
  Widget build(BuildContext context) {
    return Container(child:Text(""));
  }
}
```
## (3)Push or Pop
We can find some useful information here:<https://flutter.io/docs/cookbook/navigation/navigation-basics>.`Navigator` is the bridge between two widgets while navigating.

- 🐫Common mode
```dart
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
- 🐫Named for Route
```dart
MaterialApp(
  // Start the app with the "/" named route. In our case, the app will start
  // on the FirstScreen Widget
  initialRoute: '/',
  routes: {
    // When we navigate to the "/" route, build the FirstScreen Widget
    '/': (context) => FirstScreen(),
    // When we navigate to the "/second" route, build the SecondScreen Widget
    '/second': (context) => SecondScreen(),
  },
);
```
When widget give a name to each routes,we must make sure that it does not contain home argument.And On pressing it will guide to second widget:
```dart
// Within the `FirstScreen` Widget
onPressed: () {
  // Navigate to the second screen using a named route
  Navigator.pushNamed(context, '/second');
}
```
Returning from second widget:
```dart
// Within the SecondScreen Widget
onPressed: () {
  // Navigate back to the first screen by popping the current route
  // off the stack
  Navigator.pop(context);
}
```
- 🐫 Returned with data

Directly with codes:
```dart
class SelectionButton extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return RaisedButton(
      onPressed: () {
        _navigateAndDisplaySelection(context);
      },
      child: Text('Pick an option, any option!'),
    );
  }

  // A method that launches the SelectionScreen and awaits the result from
  // Navigator.pop
  _navigateAndDisplaySelection(BuildContext context) async {
    // Navigator.push returns a Future that will complete after we call
    // Navigator.pop on the Selection Screen!
    final result = await Navigator.push(
      context,
      // We'll create the SelectionScreen in the next step!
      MaterialPageRoute(builder: (context) => SelectionScreen()),
    );
  }
}
```
In second widget:
```dart
class SelectionScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Pick an option'),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            Padding(
              padding: const EdgeInsets.all(8.0),
              child: RaisedButton(
                onPressed: () {
                  // Pop here with "Yep"...
                  Navigator.pop(context, 'Yep!');
                },
                child: Text('Yep!'),
              ),
            ),
            Padding(
              padding: const EdgeInsets.all(8.0),
              child: RaisedButton(
                onPressed: () {
                  // Pop here with "Nope"
                  Navigator.pop(context, 'Nope!');
                },
                child: Text('Nope.'),
              ),
            )
          ],
        ),
      ),
    );
  }
}
```
- 🐫 Pushed with data

The pushed-widget should contain constructor with arguments which can be pushed,for instance:
```dart
Navigator.push(
    context,
    MaterialPageRoute(
      builder: (context) => DetailScreen(todo: todos[index]),
    ),
);
```
The second widget:
```dart
class DetailScreen extends StatelessWidget {
  // Declare a field that holds the Todo
  final Todo todo;

  // In the constructor, require a Todo
  DetailScreen({Key key, @required this.todo}) : super(key: key);
}
```
**Article will be synced to wechat blog:Android部落格**