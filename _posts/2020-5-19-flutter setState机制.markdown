---
layout: post
title:  "flutter setState机制"
date:   2020-5-19 18:19:25 +0800
categories: flutter
---


> 微信公众号：Android部落格，文末有二维码


## 1、定义

setState方法只能定义在State类中，执行这个方法之后，能够更新State限定的StatefulWidget及其子Widget树。

在StatefulWidget执行createElement方法创建StatefulElement对象的时候，会回调到StatefulWidget的createState方法，也就回到了我们自定义Widget继承StatefulWidget的createState方法，从而创建了一个State对象。

这个State持有一个_element和一个_widget对象。

## 2、State中执行
在State类中定义了setState方法：

```dart
@protected
void setState(VoidCallback fn) {
    final dynamic result = fn() as dynamic;
    _element.markNeedsBuild();
}
```

### 2.1 标记为dirty
先执行我们的闭包函数fn()，然后执行markNeedsBuild，定义在Element类中：

```dart
void markNeedsBuild() {
    if (!_active)
        return;
    if (dirty)
        return;
    _dirty = true;
    owner.scheduleBuildFor(this);
}
```

### 2.2 BuildOwner调度重建

当前Element节点被标记为dirty，同时调用owner的scheduleBuildFor方法，owner是BuildOwner类型，看看scheduleBuildFor方法：

```dart
void scheduleBuildFor(Element element) {
    if (element._inDirtyList) {
        _dirtyElementsNeedsResorting = true;
        return;
    }
    if (!_scheduledFlushDirtyElements && onBuildScheduled != null) {
        _scheduledFlushDirtyElements = true;
        onBuildScheduled();
    }
    _dirtyElements.add(element);
    element._inDirtyList = true;
}
```
BuildOwner用来管理那些需要更新的Widget。这个owner最开始被初始化的地方在WidgetsBinding的initInstances方法中，随后初始化了onBuildScheduled方法，对应执行的是_handleBuildScheduled，定义在WidgetsBinding类中，看看这个方法：

```dart
void _handleBuildScheduled() {
    ensureVisualUpdate();
}
```
ensureVisualUpdate方法定义在SchedulerBinding类中：

```dart
void ensureVisualUpdate() {
    switch (schedulerPhase) {
      case SchedulerPhase.idle:
      case SchedulerPhase.postFrameCallbacks:
        scheduleFrame();
        return;
      case SchedulerPhase.transientCallbacks:
      case SchedulerPhase.midFrameMicrotasks:
      case SchedulerPhase.persistentCallbacks:
        return;
    }
}
```

### 2.2 调度绘制
在提交下一帧绘制的时候会调用到scheduleFrame方法，提交给引擎绘制，看看scheduleFrame方法，也定义在SchedulerBinding类中：

```dart
void scheduleFrame() {
    if (_hasScheduledFrame || !framesEnabled)
      return;
    ensureFrameCallbacksRegistered();
    window.scheduleFrame();//native 'Window_scheduleFrame'
    _hasScheduledFrame = true;
}

@protected
void ensureFrameCallbacksRegistered() {
    window.onBeginFrame ??= _handleBeginFrame;
    window.onDrawFrame ??= _handleDrawFrame;
}
```
提交给引擎绘制之后，会收到onDrawFrame的回调，最终执行到_handleDrawFrame方法中，对应的是handleDrawFrame方法，定义在SchedulerBinding类中：

```dart
void handleDrawFrame() {
    Timeline.finishSync(); // end the "Animate" phase
    try {
      // PERSISTENT FRAME CALLBACKS
      _schedulerPhase = SchedulerPhase.persistentCallbacks;
      for (final FrameCallback callback in _persistentCallbacks)
        _invokeFrameCallback(callback, _currentFrameTimeStamp);
    
      // POST-FRAME CALLBACKS
      _schedulerPhase = SchedulerPhase.postFrameCallbacks;
      final List<FrameCallback> localPostFrameCallbacks =
          List<FrameCallback>.from(_postFrameCallbacks);
      _postFrameCallbacks.clear();
      for (final FrameCallback callback in localPostFrameCallbacks)
        _invokeFrameCallback(callback, _currentFrameTimeStamp);
    } finally {
      _schedulerPhase = SchedulerPhase.idle;
      Timeline.finishSync(); // end the Frame
      _currentFrameTimeStamp = null;
    }
}
```
这个_persistentCallbacks在之前的章节有讲到过，在RendererBinding的initInstances方法中添加了一个回调到这个List中，对应的是RendererBinding的drawFrame方法，对对应的节点进行绘制渲染操作。

## 3、绘制
### 3.1 开始绘制渲染
但是在WidgetsBinding类中，也覆写了drawFrame方法。在之前的绘制过程章节，可以看到WidgetsBinding相当于是继承了RendererBinding接口，所以先调用WidgetsBinding中的drawFrame方法：

```dart
@override
void drawFrame() {
    TimingsCallback firstFrameCallback;
    if (_needToReportFirstFrame) {
      firstFrameCallback = (List<FrameTiming> timings) {
        if (!kReleaseMode) {
          developer.Timeline.instantSync('Rasterized first useful frame');
          developer.postEvent('Flutter.FirstFrame', <String, dynamic>{});
        }
        SchedulerBinding.instance.removeTimingsCallback(firstFrameCallback);
        firstFrameCallback = null;
        _firstFrameCompleter.complete();
      };
      // Callback is only invoked when [Window.render] is called. When
      // [sendFramesToEngine] is set to false during the frame, it will not
      // be called and we need to remove the callback (see below).
      SchedulerBinding.instance.addTimingsCallback(firstFrameCallback);
    }
    
    try {
      if (renderViewElement != null)
        buildOwner.buildScope(renderViewElement);
      super.drawFrame();
      buildOwner.finalizeTree();
    } finally {
    }
    if (!kReleaseMode) {
      if (_needToReportFirstFrame && sendFramesToEngine) {
        developer.Timeline.instantSync('Widgets built first useful frame');
      }
    }
    _needToReportFirstFrame = false;
    if (firstFrameCallback != null && !sendFramesToEngine) {
      SchedulerBinding.instance.removeTimingsCallback(firstFrameCallback);
    }
}
```

看看这里的buildScope方法，定义在BuildOwner方法中：

```dart
void buildScope(Element context, [ VoidCallback callback ]) {
    if (callback == null && _dirtyElements.isEmpty)
        return;
    Timeline.startSync('Build', arguments: timelineWhitelistArguments);
    _dirtyElements.sort(Element._sort);
    _dirtyElementsNeedsResorting = false;
    int dirtyCount = _dirtyElements.length;
    int index = 0;
    while (index < dirtyCount) {
        try {
            _dirtyElements[index].rebuild();
        } catch (e, stack) {
        }finally {
            for (final Element element in _dirtyElements) {
                element._inDirtyList = false;
            }
            _dirtyElements.clear();
            _scheduledFlushDirtyElements = false;
            _dirtyElementsNeedsResorting = null;
            Timeline.finishSync();
        }
    }
}
```
_dirtyElements列表，先从深到浅排序，之后遍历，遍历的过程中执行rebuild方法，此时将所有标记为dirty的Element节点依次执行rebuild，preformRebuild，build，updateChild，update方法，执行界面更新。

当然了，这些build，update操作完成之后，后续会将需要绘制的RenderObject添加到需要layout的列表中，等待绘制渲染。

所有绘制完成之后将_dirtyElements列表清空，_inDirtyList标记位置为false。

### 3.2 提交给引擎绘制渲染
看看super.drawFrame()，这里就执行到了RendererBinding类中，定义如下：

```dart
void drawFrame() {
    pipelineOwner.flushLayout();
    pipelineOwner.flushCompositingBits();
    pipelineOwner.flushPaint();
    if (sendFramesToEngine) {
      renderView.compositeFrame(); // this sends the bits to the GPU
      pipelineOwner.flushSemantics(); // this also sends the semantics to the OS.
      _firstFrameSent = true;
    }
}
```
这里就是将最终需要绘制渲染的画面提交给引擎绘制的地方了，绘制完成之后就在界面显示更新后的视图了。

![微信公众号](https://ftp.bmp.ovh/imgs/2020/04/f472fa7e9c62a8c7.jpg)


