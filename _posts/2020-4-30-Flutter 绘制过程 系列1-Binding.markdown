---
layout: post
title:  "Flutter 绘制过程 系列1-Binding"
date:   2020-4-30 11:09:22 +0800
categories: flutter
---

## 1、Widget
StatelessWidget和StatefulWidget都继承自Widget。

Widget作为虚类，定义了`Element createElement()`方法，给继承者实现，返回Element对象。

具体到StatelessWidget，实现createElement，返回StatelessElement对象。StatelessElement继承自ComponentElement。

具体到StatefulWidget，实现createElement，返回StatefulElement对象。StatefulElement继承自ComponentElement。

ComponentElement继承自Element类。

在执行createElement方法时，都会把Widget传递给Element对象，因此Element持有一个Widget对象。

## 2、Element
在Element树的特定位置，Element代表一个Widget实例。

多个Element形成了一颗Element树，大多数Element有一个独一无二的child，但是那些继承自RenderObjectElement的Element,就可以有多个child.

通过调用Widget.createElement方法创建一个Element，通过调用Element的mount方法，将一个新的Element添加到一个父Element的属性为slot的位置。

Element类的mount方法，负责填充所有的子Widget，并在必要时调用attachRenderObject方法，将关联的渲染对象（render objects）附着到渲染树，此时Element被标记为“active”，然后在屏幕上显示。

## 3、RenderObject
渲染树中的一个对象，通过RenderObjectToWidgetAdapter将其和Element联系起来。渲染树的根是RenderView，他有一个唯一的child，是RenderBox类型。在绘制阶段，将RenderObject生成对应的Layer tree，再将其生成Scene，交给GPU绘制。

**总结**

三者之间的基本关系就是：Element持有Widget对象，在Element的mount阶段，通过Widget对象创建RenderObject对象，这个对象被Element持有。所以Element持有Widget和RenderObject对象。

Element管理着Widget生命周期，在生命周期不同阶段，处理RenderObject不同的渲染绘制任务。

## 4、启动

首先从main.dart的main方法开始运行:

```dart
void main() {
  runApp(MyApp());
}
```

runApp方法是在一个binding.dart文件里面：

> packages\flutter\lib\src\widgets\binding.dart

```dart
void runApp(Widget app) {
  WidgetsFlutterBinding.ensureInitialized()
    ..scheduleAttachRootWidget(app)
    ..scheduleWarmUpFrame();
}
```
### 4.1 WidgetsFlutterBinding

> packages\flutter\lib\src\widgets\binding.dart

WidgetsFlutterBinding类调用静态初始化方法，执行初始化操作：

```dart
class WidgetsFlutterBinding extends BindingBase with GestureBinding, ServicesBinding, SchedulerBinding, PaintingBinding, SemanticsBinding, RendererBinding, WidgetsBinding {
  static WidgetsBinding ensureInitialized() {
    if (WidgetsBinding.instance == null)
      WidgetsFlutterBinding();
    return WidgetsBinding.instance;
  }
}
```

with关键字类似于java里面的implement关键字，可以初始化及调用with后面类的方法。如果对于这个关键字比较陌生的话，可以写下面的demo代码验证一下，看看初始化顺序是怎么样的：

```dart
abstract class Test1{
  Test1(){
    print("parent class");
    init();
  }

  void init(){
    print("class init");
  }
}

mixin Demo1 on Test1{
  void init(){
    print("demo1 init");
    super.init();
    print("demo1 init done");
  }
}

mixin Demo2 on Test1,Demo1 {
  void init(){
    print("demo2 init");
    super.init();
    print("demo2 init done");
  }
}

class Test2 extends Test1 with Demo1,Demo2{
  Test2(){
    print("child class");
  }
  
  static void test(){
    Test2();
  }
}

void main() {
  Test2.test();
}
```

> parent class
> 
> demo2 init
> 
> demo1 init
> 
> class init
> 
> demo1 init done
> 
> demo2 init done
> 
> child class

透过demo我们可以知道，调用顺序是先调用父类构造函数，然后with后面的类，从后往前调用，但是因为在每个binding类的initInstances方法中，都先调用了super.initInstances方法，所以实际上先执行前面的binding类代码。先看看BindingBase的构造函数：
> packages\flutter\lib\src\foundation\binding.dart

```dart
BindingBase() {
    initInstances();
}
```
到这里开始执行with从前到后binding的initInstances方法。

### 4.2 GestureBinding
绑定手势子系统，当一个点按下事件从window传递过来之后，被其拦截，由GestureBinding决定在哪一个节点生效。

```dart
@override
void initInstances() {
    super.initInstances();
    _instance = this;
    window.onPointerDataPacket = _handlePointerDataPacket;
}
```
获取window窗口的onPointerDataPacket方法。

### 4.3 ServicesBinding
监听系统消息，并通过BinaryMessenger转发。

```dart
@override
void initInstances() {
    super.initInstances();
    _instance = this;
    _defaultBinaryMessenger = createBinaryMessenger();
    window
      ..onPlatformMessage = defaultBinaryMessenger.handlePlatformMessage;
    initLicenses();
    SystemChannels.system.setMessageHandler(handleSystemMessage);
}
```
通过createBinaryMessenger方法创建了一个默认的消息传递者。

### 4.4 SchedulerBinding
调度管理。调度分为几个阶段，分别是：
- 没有frame需要处理的时候，处于Idle。
- transientCallbacks，暂时的回调，比如更新RenderObject状态到animate。
- midFrameMicrotasks，中间帧微服务，比如正在处理暂时回调任务的时候。
- persistentCallbacks，持续性回调，比如layer在创建，布局，绘制阶段。
- postFrameCallbacks，清理，并准备下一帧。

```dart
@override
void initInstances() {
    super.initInstances();
    _instance = this;
    SystemChannels.lifecycle.setMessageHandler(_handleLifecycleMessage);
    readInitialLifecycleStateFromNativeWindow();
    
    if (!kReleaseMode) {
      int frameNumber = 0;
      addTimingsCallback((List<FrameTiming> timings) {
        for (FrameTiming frameTiming in timings) {
          frameNumber += 1;
          _profileFramePostEvent(frameNumber, frameTiming);
        }
      });
    }
}
```

### 4.5 PaintingBinding
绑定了绘制库。

hook了缓存清理逻辑，用于清理图片缓存。

```dart
@override
void initInstances() {
    super.initInstances();
    _instance = this;
    _imageCache = createImageCache();
    if (shaderWarmUp != null) {
      shaderWarmUp.execute();
    }
}
```
初始化方法中创建了图片缓存对象，设置默认图片缓存大小。

shaderWarmUp其实是一个着色器预处理。在正式运行app之前，先生成一个着色器ShaderWarmUp，绘制一个场景(Scene)到里面，并缓存起来。当app里面有复杂的场景需要着色时，可以减少动画或交互时的卡顿。

着色预热操作是在GPU线程里面同步操作的，意味着，app第一帧的渲染必须等到该操作完成之后才能继续。

### 4.6 SemanticsBinding
Semantics是语义的意思。这个类将语义层（semantics layer）与flutter engine联系起来。

```dart
@override
void initInstances() {
    super.initInstances();
    _instance = this;
    _accessibilityFeatures = window.accessibilityFeatures;
}
```
### 4.7 RendererBinding
> packages\flutter\lib\src\rendering\binding.dart

将渲染树和Flutter engine联系起来。

```dart
@override
void initInstances() {
    super.initInstances();
    _instance = this;
    _pipelineOwner = PipelineOwner(
      onNeedVisualUpdate: ensureVisualUpdate,
      onSemanticsOwnerCreated: _handleSemanticsOwnerCreated,
      onSemanticsOwnerDisposed: _handleSemanticsOwnerDisposed,
    );
    window
      ..onMetricsChanged = handleMetricsChanged
      ..onTextScaleFactorChanged = handleTextScaleFactorChanged
      ..onPlatformBrightnessChanged = handlePlatformBrightnessChanged
      ..onSemanticsEnabledChanged = _handleSemanticsEnabledChanged
      ..onSemanticsAction = _handleSemanticsAction;
    initRenderView();
    _handleSemanticsEnabledChanged();
    assert(renderView != null);
    addPersistentFrameCallback(_handlePersistentFrameCallback);
    initMouseTracker();
}
```
看看几个关键的类。
#### 4.7.1 PipelineOwner
> packages\flutter\lib\src\rendering\object.dart

PipelineOwner管理着渲染流程。

PipelineOwner对外提供接口用于驱动渲染流程。并存储着那些请求访问渲染流程中各个阶段的状态。可以通过以下步骤来刷新渲染流程：
- flushLayout 

更新任何需要重新计算布局的渲染对象。在这个阶段渲染对象的大小和布局都被计算好了。在此阶段，渲染对象的绘制和合成状态被置成dirty。
- flushCompositingBits 

渲染对象的复合bit被设置成dirty状态之后会在这个阶段更新，并且每一个渲染对象都可以知道自己的子对象是否需要被复合。这个信息在绘制阶段执行裁剪的时候会起到作用。
- flushPaint 

访问那些需要被绘制的渲染对象。在这个阶段，这些渲染对象有机会记录绘制命令到PictureLayer，以及组建其他复合Layer。
- flushSemantics 

最后，如果启用了语义，该阶段将编译渲染对象的语义。 该语义信息由辅助技术可改善渲染树的可访问性。

#### 4.7.2 RenderView
接下来会执行initRenderView方法，创建一个RenderView对象，他继承自RenderObject类，是整个渲染树的根。

```dart
void initRenderView() {
    renderView = RenderView(configuration: createViewConfiguration(), window: window);
    renderView.prepareInitialFrame();
}
```

接下来RenderView的对象还调用了prepareInitialFrame方法，如下：
> packages\flutter\lib\src\rendering\view.dart

```dart
void prepareInitialFrame() {
    scheduleInitialLayout();
    scheduleInitialPaint(_updateMatricesAndCreateNewRootLayer());
}
```
- scheduleInitialLayout方法：

> packages\flutter\lib\src\rendering\object.dart

```dart
void scheduleInitialLayout() {
    _relayoutBoundary = this;
    owner._nodesNeedingLayout.add(this);
}
```
这个方法属于RenderObject类，由此我们看到RenderView实际调用的是父类的方法，而这个owner就是我们在initInstances方法中初始化的PipelineOwner类的对象。

_nodesNeedingLayout是一个RenderObject对象的List，全局已经初始化了，在这里先将RenderView对象添加进去，作为根。

- _updateMatricesAndCreateNewRootLayer方法

在调用scheduleInitialPaint方法之前先调用_updateMatricesAndCreateNewRootLayer新建一个Layer对象:
> packages\flutter\lib\src\rendering\view.dart

```dart
Layer _updateMatricesAndCreateNewRootLayer() {
    _rootTransform = configuration.toMatrix();
    final ContainerLayer rootLayer = TransformLayer(transform: _rootTransform);
    rootLayer.attach(this);
    return rootLayer;
}
```
ContainerLayer其实是一个复合的Layer，可以包含很多子Layer。另外Layer继承自AbstractNode。

- scheduleInitialPaint方法：
> packages\flutter\lib\src\rendering\object.dart

```dart
void scheduleInitialPaint(ContainerLayer rootLayer) {
    _layer = rootLayer;
    owner._nodesNeedingPaint.add(this);
}
```
_nodesNeedingPaint是一个RenderObject对象的List，全局已经初始化了，在这里先将一个RenderView对象添加进去，另外可以看到RenderObject持有一个Layer对象。

#### 4.7.3 addPersistentFrameCallback方法

回到RendererBinding的initInstances方法中，addPersistentFrameCallback方法添加回调，回调方法是_handlePersistentFrameCallback。

addPersistentFrameCallback方法则是把这个回调放到List中存起来，在SchedulerBinding的_persistentCallbacks阶段集中调用：

```dart
final List<FrameCallback> _persistentCallbacks = <FrameCallback>[];

void addPersistentFrameCallback(FrameCallback callback) {
    _persistentCallbacks.add(callback);
}
```
_handlePersistentFrameCallback方法里面调用的就是drawFrame方法。

### 4.8 WidgetsBinding
> packages\flutter\lib\src\widgets\binding.dart

将widget layer与flutter engine融合在一起。

initInstances方法：
> packages\flutter\lib\src\widgets\binding.dart

```dart
void initInstances() {
    super.initInstances();
    _buildOwner = BuildOwner();
    buildOwner.onBuildScheduled = _handleBuildScheduled;
}
```
BuildOwner类管理着widget的框架，比如跟踪哪些widget需要重建，在全局处理应用于widget树的任务，比如开发者在调试时启动热加载，就会组织那些渲染树中不活跃的列表元素，并触发重组指令。
