---
layout: post
title:  "flutter通信机制-EventChannel"
date:   2020-5-15 16:48:22 +0800
categories: flutter
---



流程图如下：

![EventChannel流程图](https://ftp.bmp.ovh/imgs/2020/05/ebf545d6db6753a6.jpg)

##  1、使用方式
当原生平台需要向dart发送消息时，需要用到EventChannel。

### 1.1 Android端注册
Android平台的注册方式：

```kotlin
class MainActivity : FlutterActivity(){
    val DATA_RESULT_CHANNEL = "com.yourname.yourname/typeData"
    
    override fun onCreate(savedInstanceState: Bundle?) {
        EventChannel(flutterView, DATA_RESULT_CHANNEL).setStreamHandler(object : EventChannel.StreamHandler {
            override fun onListen(arguments: Any?, events: EventChannel.EventSink?) {
                listenEvents = events!!
            }

            override fun onCancel(arguments: Any?) {
            }
        })
    }
    
    fun receiveMessage(data: Object) {
        if (data == null) {
            listenEvents.error("1", "data == null", null)
        } else {
            listenEvents.success(data: Object))
        }
    }
}
```

### 1.2 flutter端消费消息
flutter端的接收方式是：

```dart
class InteractUtil {
    static const EventChannel eventChannel =
      const EventChannel("com.yourname.yourname/typeData");
  List<InteractListener> listenerList;

  factory InteractUtil() => _getInstance();

  static InteractUtil get instance => _getInstance();
  static InteractUtil _instance;

  InteractUtil._internal() {
    listenerList = [];
    eventChannel.receiveBroadcastStream().listen(_onEvent, onError: _onError);
  }

  static InteractUtil _getInstance() {
    if (_instance == null) {
      _instance = new InteractUtil._internal();
    }
    return _instance;
  }

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
}

abstract class InteractListener {
  void onEvent(Object event);

  void onError(Object error);
}
```

## 2、Android端
### 2.1 EventChannel构造
看看EventChannel的构造函数：
> shell\platform\android\io\flutter\plugin\common\EventChannel.java

```java
public EventChannel(BinaryMessenger messenger, String name) {
    this(messenger, name, StandardMethodCodec.INSTANCE);
}

public EventChannel(BinaryMessenger messenger, String name, MethodCodec codec) {
    this.messenger = messenger;
    this.name = name;
    this.codec = codec;
}
```
与MethodChannel类似，这里在构造函数的参数里面构造了StandardMethodCodec和StandardMessageCodec。前者做方法名称和参数的编解码，后者做消息数据的编解码。

这里的messenger是FlutterView。

### 2.2 设置消息处理

通过setStreamHandler设置消息处理，并接收回调。
> shell\platform\android\io\flutter\plugin\common\EventChannel.java

```java
public void setStreamHandler(final StreamHandler handler) {
    messenger.setMessageHandler(name, handler == null ? null : new IncomingStreamRequestHandler(handler));
}
```
这里的handler就是Android MainActivity中定义的EventChannel.StreamHandler。

#### 2.2.1 IncomingStreamRequestHandler
IncomingStreamRequestHandler还是在EventChannel中，看看他的定义：

```java
private final class IncomingStreamRequestHandler implements BinaryMessageHandler {
    private final StreamHandler handler;
    private final AtomicReference<EventSink> activeSink = new AtomicReference<>(null);
    
    IncomingStreamRequestHandler(StreamHandler handler) {
      this.handler = handler;
    }
    
    @Override
    public void onMessage(ByteBuffer message, final BinaryReply reply) {
      final MethodCall call = codec.decodeMethodCall(message);
      if (call.method.equals("listen")) {
        onListen(call.arguments, reply);
      } else if (call.method.equals("cancel")) {
        onCancel(call.arguments, reply);
      } else {
        reply.reply(null);
      }
    }
    
    private void onListen(Object arguments, BinaryReply callback) {
        final EventSink eventSink = new EventSinkImplementation();
        handler.onCancel(null);
        handler.onListen(arguments, eventSink);
        callback.reply(codec.encodeSuccessEnvelope(null));
    }
    
    private void onCancel(Object arguments, BinaryReply callback) {
        final EventSink oldSink = activeSink.getAndSet(null);
        handler.onCancel(arguments);
        callback.reply(codec.encodeSuccessEnvelope(null));
    }
}
```
重载实现了onMessage方法，并在这个方法中根据方法名称的不同，分别调用onListen和onCancel方法。

onMessage是由dart端注册之后回调过来的，这个流程在dart端追踪。

根据dart端调用的方法，对应的调用到kotlin代码中的onListen或onCancel方法，这里以onListen为例跟踪代码。

### 2.3 onListen接收消息处理对象

在onListen方法中，初始化了EventSinkImplementation对象，同时将这个对象回调给Android的注册回调函数onListen，后续Android端的数据发送就依靠这个对象了。

#### 2.3.1 EventSinkImplementation
看看他的定义：

```java
private final class EventSinkImplementation implements EventSink {
  final AtomicBoolean hasEnded = new AtomicBoolean(false);

  @Override
  @UiThread
  public void success(Object event) {
    if (hasEnded.get() || activeSink.get() != this) {
      return;
    }
    EventChannel.this.messenger.send(name, codec.encodeSuccessEnvelope(event));
  }

  @Override
  @UiThread
  public void error(String errorCode, String errorMessage, Object errorDetails) {
    if (hasEnded.get() || activeSink.get() != this) {
      return;
    }
    EventChannel.this.messenger.send(
        name, codec.encodeErrorEnvelope(errorCode, errorMessage, errorDetails));
  }

  @Override
  @UiThread
  public void endOfStream() {
    if (hasEnded.getAndSet(true) || activeSink.get() != this) {
      return;
    }
    EventChannel.this.messenger.send(name, null);
  }
}
```
根据结果，kotlin中可以选择执行success或error或endOfStream函数，将对应的数据发送到dart端。

#### 2.3.2 success发送成功数据
以success为例，先经过codec对象编码，codec是StandardMethodCodec类型，看看encodeSuccessEnvelope方法是怎么编码的：
> shell\platform\android\io\flutter\plugin\common\StandardMethodCodec.java

```java
@Override
public ByteBuffer encodeSuccessEnvelope(Object result) {
    final ExposedByteArrayOutputStream stream = new ExposedByteArrayOutputStream();
    stream.write(0);
    messageCodec.writeValue(stream, result);
    final ByteBuffer buffer = ByteBuffer.allocateDirect(stream.size());
    buffer.put(stream.buffer(), 0, stream.size());
    return buffer;
}
```

ExposedByteArrayOutputStream继承自ByteArrayOutputStream，是一个ByteArray输出流，可以写入byte数据。

先写入成功标志位0，再写入数据。这里的messageCodec是StandardMessageCodec类型，看看怎么写数据的：
> shell\platform\android\io\flutter\plugin\common\StandardMessageCodec.java

```java
protected void writeValue(ByteArrayOutputStream stream, Object value) {
    if (value == null || value.equals(null)) {
      stream.write(NULL);
    } else if (value == Boolean.TRUE) {
      stream.write(TRUE);
    } else if (value == Boolean.FALSE) {
      stream.write(FALSE);
    } else if (value instanceof Number) {
      if (value instanceof Integer || value instanceof Short || value instanceof Byte) {
        stream.write(INT);
        writeInt(stream, ((Number) value).intValue());
      } else if (value instanceof Long) {
        stream.write(LONG);
        writeLong(stream, (long) value);
      } else if (value instanceof Float || value instanceof Double) {
        stream.write(DOUBLE);
        writeAlignment(stream, 8);
        writeDouble(stream, ((Number) value).doubleValue());
      } else if (value instanceof BigInteger) {
        stream.write(BIGINT);
        writeBytes(stream, ((BigInteger) value).toString(16).getBytes(UTF8));
      } else {
        throw new IllegalArgumentException("Unsupported Number type: " + value.getClass());
      }
    } else if (value instanceof String) {
      stream.write(STRING);
      writeBytes(stream, ((String) value).getBytes(UTF8));
    } else if (value instanceof byte[]) {
      stream.write(BYTE_ARRAY);
      writeBytes(stream, (byte[]) value);
    } else if (value instanceof int[]) {
      stream.write(INT_ARRAY);
      final int[] array = (int[]) value;
      writeSize(stream, array.length);
      writeAlignment(stream, 4);
      for (final int n : array) {
        writeInt(stream, n);
      }
    } else if (value instanceof long[]) {
      stream.write(LONG_ARRAY);
      final long[] array = (long[]) value;
      writeSize(stream, array.length);
      writeAlignment(stream, 8);
      for (final long n : array) {
        writeLong(stream, n);
      }
    } else if (value instanceof double[]) {
      stream.write(DOUBLE_ARRAY);
      final double[] array = (double[]) value;
      writeSize(stream, array.length);
      writeAlignment(stream, 8);
      for (final double d : array) {
        writeDouble(stream, d);
      }
    } else if (value instanceof List) {
      stream.write(LIST);
      final List<?> list = (List) value;
      writeSize(stream, list.size());
      for (final Object o : list) {
        writeValue(stream, o);
      }
    } else if (value instanceof Map) {
      stream.write(MAP);
      final Map<?, ?> map = (Map) value;
      writeSize(stream, map.size());
      for (final Entry<?, ?> entry : map.entrySet()) {
        writeValue(stream, entry.getKey());
        writeValue(stream, entry.getValue());
      }
    } else {
      throw new IllegalArgumentException("Unsupported value: " + value);
    }
}
```
编码方式就是先写数据长度，再写入具体数据。支持的数据类型如下：
```java
private static final byte NULL = 0;
private static final byte TRUE = 1;
private static final byte FALSE = 2;
private static final byte INT = 3;
private static final byte LONG = 4;
private static final byte BIGINT = 5;
private static final byte DOUBLE = 6;
private static final byte STRING = 7;
private static final byte BYTE_ARRAY = 8;
private static final byte INT_ARRAY = 9;
private static final byte LONG_ARRAY = 10;
private static final byte DOUBLE_ARRAY = 11;
private static final byte LIST = 12;
private static final byte MAP = 13;
```
> BigInteger 不可变的任意精度的整数。所有操作中，都以二进制补码形式表示 BigInteger。

支持bool,int,long,BigInteger,double,String,ByteArray,IntArray,LongArray,DoubleArray,List,Map。集合类型中的数据类型也必须是基本数据类型或其数组，以及String类型。

写int数据之前，先4字节对齐；写long或float或double类型之前，先8字节对齐。

String类型转换成byte[]数据再写入。

所有数据写入stream之后，通过allocateDirect方法为ByteBuffer分配stream大小的内存空间，并将stream中的数据写入ByteBuffer中。

### 2.4 send发送数据
#### 2.4.1 FlutterView
上一步生成的ByteBuffer数据在这里被send，messenger对象其实是FlutterView，看看send方法：

> shell\platform\android\io\flutter\view\FlutterView.java

```java
@Override
@UiThread
public void send(String channel, ByteBuffer message) {
    send(channel, message, null);
}

@Override
@UiThread
public void send(String channel, ByteBuffer message, BinaryReply callback) {
    if (!isAttached()) {
      Log.d(TAG, "FlutterView.send called on a detached view, channel=" + channel);
      return;
    }
    mNativeView.send(channel, message, callback);
}
```
#### 2.4.2 FlutterNativeView

mNativeView是FlutterNativeView类型，看看他里面的方法：

```java
@Override
@UiThread
public void send(String channel, ByteBuffer message) {
    dartExecutor.getBinaryMessenger().send(channel, message);
}
```
#### 2.4.3 DefaultBinaryMessenger

这里的getBinaryMessenger方法返回的是dartMessenger对象，对应DefaultBinaryMessenger类。是在DartExecutor构造函数里面初始化的：

> DartExecutor

```java
public DartExecutor(@NonNull FlutterJNI flutterJNI, @NonNull AssetManager assetManager) {
    this.flutterJNI = flutterJNI;
    this.assetManager = assetManager;
    this.dartMessenger = new DartMessenger(flutterJNI);
    dartMessenger.setMessageHandler("flutter/isolate", isolateChannelMessageHandler);
    this.binaryMessenger = new DefaultBinaryMessenger(dartMessenger);
}
```
看看DefaultBinaryMessenger里面的send方法：

> DefaultBinaryMessenger

```java
@Override
@UiThread
public void send(@NonNull String channel, @Nullable ByteBuffer message) {
  messenger.send(channel, message, null);
}
```

#### 2.4.4 DartMessenger

messenger其实是dartMessenger，对应DartMessenger类，看看里面的send方法：
> DartMessenger

```java
@Override
@UiThread
public void send(@NonNull String channel, @NonNull ByteBuffer message) {
    send(channel, message, null);
}

@Override
public void send(
  @NonNull String channel,
  @Nullable ByteBuffer message,
  @Nullable BinaryMessenger.BinaryReply callback) {
    int replyId = 0;
    if (message == null) {
      flutterJNI.dispatchEmptyPlatformMessage(channel, replyId);
    } else {
      flutterJNI.dispatchPlatformMessage(channel, message, message.position(), replyId);
    }
}
```

#### 2.4.5 FlutterJNI

这里的message不为空，对应调用dispatchPlatformMessage方法。是在FlutterJNI中调用到native层，看看这个方法：

```java
@UiThread
public void dispatchPlatformMessage(
  @NonNull String channel, @Nullable ByteBuffer message, int position, int responseId) {
    if (isAttached()) {
      nativeDispatchPlatformMessage(nativePlatformViewId, channel, message, position, responseId);
    } else {
    }
}

// Send a data-carrying platform message to Dart.
private native void nativeDispatchPlatformMessage(
  long nativePlatformViewId,
  @NonNull String channel,
  @Nullable ByteBuffer message,
  int position,
  int responseId);
```
nativeDispatchPlatformMessage调用到了native层，是在`shell\platform\android\platform_view_android_jni.cc`文件中，看看注册的地方：
> platform_view_android_jni.cc

```c
bool RegisterApi(JNIEnv* env) {
    static const JNINativeMethod flutter_jni_methods[] = {
        {
          .name = "nativeDispatchPlatformMessage",
          .signature = "(JLjava/lang/String;Ljava/nio/ByteBuffer;II)V",
          .fnPtr = reinterpret_cast<void*>(&DispatchPlatformMessage),
        },
    }
}
```
看看DispatchPlatformMessage方法的调用栈：

> platform_view_android_jni.cc

```c
static void DispatchPlatformMessage(JNIEnv* env,
                                    jobject jcaller,
                                    jlong shell_holder,
                                    jstring channel,
                                    jobject message,
                                    jint position,
                                    jint responseId) {
  ANDROID_SHELL_HOLDER->GetPlatformView()->DispatchPlatformMessage(
      env,                                         //
      fml::jni::JavaStringToString(env, channel),  //
      message,                                     //
      position,                                    //
      responseId                                   //
  );
}
```
> shell\platform\android\platform_view_android.cc

```c
void PlatformViewAndroid::DispatchPlatformMessage(JNIEnv* env,
                                                  std::string name,
                                                  jobject java_message_data,
                                                  jint java_message_position,
                                                  jint response_id) {
  uint8_t* message_data =
      static_cast<uint8_t*>(env->GetDirectBufferAddress(java_message_data));
  std::vector<uint8_t> message =
      std::vector<uint8_t>(message_data, message_data + java_message_position);

  fml::RefPtr<flutter::PlatformMessageResponse> response;
  if (response_id) {
    response = fml::MakeRefCounted<PlatformMessageResponseAndroid>(
        response_id, java_object_, task_runners_.GetPlatformTaskRunner());
  }

  PlatformView::DispatchPlatformMessage(
      fml::MakeRefCounted<flutter::PlatformMessage>(
          std::move(name), std::move(message), std::move(response)));
}
```
这一步将name，message封装到了PlatformMessage对象中。

> shell\common\platform_view.cc

```c
void PlatformView::DispatchPlatformMessage(
    fml::RefPtr<PlatformMessage> message) {
  delegate_.OnPlatformViewDispatchPlatformMessage(std::move(message));
}
```

> shell\common\shell.cc

```c
// |PlatformView::Delegate|
void Shell::OnPlatformViewDispatchPlatformMessage(
    fml::RefPtr<PlatformMessage> message) {
  FML_DCHECK(is_setup_);
  FML_DCHECK(task_runners_.GetPlatformTaskRunner()->RunsTasksOnCurrentThread());

  task_runners_.GetUITaskRunner()->PostTask(
      [engine = engine_->GetWeakPtr(), message = std::move(message)] {
        if (engine) {
          engine->DispatchPlatformMessage(std::move(message));
        }
      });
}
```

> shell\common\engine.cc

```c
void Engine::DispatchPlatformMessage(fml::RefPtr<PlatformMessage> message) {
  if (message->channel() == kLifecycleChannel) {
    if (HandleLifecyclePlatformMessage(message.get()))
      return;
  } else if (message->channel() == kLocalizationChannel) {
    if (HandleLocalizationPlatformMessage(message.get()))
      return;
  } else if (message->channel() == kSettingsChannel) {
    HandleSettingsPlatformMessage(message.get());
    return;
  }

  if (runtime_controller_->IsRootIsolateRunning() &&
      runtime_controller_->DispatchPlatformMessage(std::move(message))) {
    return;
  }

  // If there's no runtime_, we may still need to set the initial route.
  if (message->channel() == kNavigationChannel) {
    HandleNavigationPlatformMessage(std::move(message));
    return;
  }

  FML_DLOG(WARNING) << "Dropping platform message on channel: "
                    << message->channel();
}
```

在这里执行到```runtime_controller_->DispatchPlatformMessage```中，看看这个方法：
> runtime\runtime_controller.cc

```c
bool RuntimeController::DispatchPlatformMessage(
    fml::RefPtr<PlatformMessage> message) {
  if (auto* window = GetWindowIfAvailable()) {
    TRACE_EVENT1("flutter", "RuntimeController::DispatchPlatformMessage",
                 "mode", "basic");
    window->DispatchPlatformMessage(std::move(message));
    return true;
  }
  return false;
}
```

> lib\ui\window\window.cc

```c
void Window::DispatchPlatformMessage(fml::RefPtr<PlatformMessage> message) {
  std::shared_ptr<tonic::DartState> dart_state = library_.dart_state().lock();
  if (!dart_state) {
    FML_DLOG(WARNING)
        << "Dropping platform message for lack of DartState on channel: "
        << message->channel();
    return;
  }
  tonic::DartState::Scope scope(dart_state);
  Dart_Handle data_handle =
      (message->hasData()) ? ToByteData(message->data()) : Dart_Null();
  if (Dart_IsError(data_handle)) {
    FML_DLOG(WARNING)
        << "Dropping platform message because of a Dart error on channel: "
        << message->channel();
    return;
  }

  int response_id = 0;
  if (auto response = message->response()) {
    response_id = next_response_id_++;
    pending_responses_[response_id] = response;
  }

  tonic::LogIfError(
      tonic::DartInvokeField(library_.value(), "_dispatchPlatformMessage",
                             {tonic::ToDart(message->channel()), data_handle,
                              tonic::ToDart(response_id)}));
}
```
将message中的数据转换成Dart_Handle，并最终执行_dispatchPlatformMessage方法，同时传递channel name和数据。

## 3、flutter端
### 3.1 方法映射
_dispatchPlatformMessage对应的是hooks.dart文件中的_invoke3方法。

> lib\ui\hooks.dart

```dart
void _dispatchPlatformMessage(String name, ByteData data, int responseId) {
  if (name == ChannelBuffers.kControlChannelName) {
    try {
      channelBuffers.handleMessage(data);
    } catch (ex) {
      _printDebug('Message to "$name" caused exception $ex');
    } finally {
      window._respondToPlatformMessage(responseId, null);
    }
  } else if (window.onPlatformMessage != null) {
    _invoke3<String, ByteData, PlatformMessageResponseCallback>(
      window.onPlatformMessage,
      window._onPlatformMessageZone,
      name,
      data,
      (ByteData responseData) {
        window._respondToPlatformMessage(responseId, responseData);
      },
    );
  } else {
    channelBuffers.push(name, data, (ByteData responseData) {
      window._respondToPlatformMessage(responseId, responseData);
    });
  }
}
```
这里调用到了onPlatformMessage方法，携带的参数就是_dispatchPlatformMessage的name，data参数。

### 3.2 ServicesBinding初始化

这个onPlatformMessage是哪里定义的呢？记得在ServicesBinding的initInstances方法中，有定义这个方法：

> packages\flutter\lib\src\services\binding.dart\ServicesBinding

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
defaultBinaryMessenger就是_DefaultBinaryMessenger类型，而在onPlatformMessage被调用的时候，就执行到了里面的handlePlatformMessage方法。

### 3.3 _DefaultBinaryMessenger消息发送对象

看看方法体：
> packages\flutter\lib\src\services\binding.dart\\_DefaultBinaryMessenger

```dart
@override
Future<void> handlePlatformMessage(
String channel,
ByteData data,
ui.PlatformMessageResponseCallback callback,
) async {
    ByteData response;
    try {
      final MessageHandler handler = _handlers[channel];
      if (handler != null) {
        response = await handler(data);
      } else {
        ui.channelBuffers.push(channel, data, callback);
        callback = null;
      }
    } catch (exception, stack) {
    } finally {
      if (callback != null) {
        callback(response);
      }
    }
}
```
这里会执行到```ui.channelBuffers.push(channel, data, callback);```，看看是怎么讲数据push过去的：

> pkg\sky_engine\lib\ui\channel_buffers.dart

```dart
bool push(String channel, ByteData data, PlatformMessageResponseCallback callback) {
    _RingBuffer<_StoredMessage> queue = _messages[channel];
    if (queue == null) {
      queue = _makeRingBuffer(kDefaultBufferSize);
      _messages[channel] = queue;
    }
    final bool didOverflow = queue.push(_StoredMessage(data, callback));
    if (didOverflow) {
    }
    return didOverflow;
}
```
可以看到这里有一个消息队列，有消息过来就将channel对应的消息存起来放到队列中。

### 3.4 flutter注册消息接收

flutter注册的时候，会注册两个方法，一个是_onEvent，一个是_onError。

```dart
eventChannel.receiveBroadcastStream().listen(_onEvent, onError: _onError);
```
### 3.5 EventChannel构造
看看他的构造方法：
> packages\flutter\lib\src\services\platform_channel.dart

```dart
const EventChannel(this.name, [this.codec = const StandardMethodCodec(), BinaryMessenger binaryMessenger])
  : assert(name != null),
    assert(codec != null),
    _binaryMessenger = binaryMessenger;
```
codec是StandardMethodCodec类型，提供方法及其参数的编解码，binaryMessenger对象是上一章节中说的_DefaultBinaryMessenger类型，在ServicesBinding中定义及初始化。

#### 3.5.1 receiveBroadcastStream获取Stream对象
是EventChannel类的方法。

> packages\flutter\lib\src\services\platform_channel.dart

```dart
Stream<dynamic> receiveBroadcastStream([ dynamic arguments ]) {
    final MethodChannel methodChannel = MethodChannel(name, codec);
    StreamController<dynamic> controller;
    controller = StreamController<dynamic>.broadcast(onListen: () async {
    }, onCancel: () async {
    });
    return controller.stream;
}
```
这里先是使用EventChannel初始化时传递的name和生成的codec参数，构造了一个MethodChannel对象。

#### 3.5.2 StreamController控制数据流
接下来调用StreamController的broadcast方法，监听接收到的消息。
> pkg\sky_engine\lib\async\stream_controller.dart\StreamController

```dart
factory StreamController.broadcast(
  {void onListen(), void onCancel(), bool sync: false}) {
    return sync
        ? new _SyncBroadcastStreamController<T>(onListen, onCancel)
        : new _AsyncBroadcastStreamController<T>(onListen, onCancel);
}
```
可以看到上一步中，没有传sync参数，这里默认是false，也就是会返回一个_AsyncBroadcastStreamController对象。

#### 3.5.3 _AsyncBroadcastStreamController异步数据流处理

构造方法如下：
> pkg\sky_engine\lib\async\broadcast_stream_controller.dart

```dart
class _AsyncBroadcastStreamController<T> extends _BroadcastStreamController<T> {
  _AsyncBroadcastStreamController(void onListen(), void onCancel())
      : super(onListen, onCancel);
```
他继承自_BroadcastStreamController类，再看看super方法：
> pkg\sky_engine\lib\async\broadcast_stream_controller.dart

```dart
_BroadcastStreamController(this.onListen, this.onCancel)
  : _state = _STATE_INITIAL;
```

将两个回调方法给到自身定义的两个变量中，两个变量其实是回调方法。

在receiveBroadcastStream方法的最后会返回`controller.stream`，这个stream定义在_BroadcastStreamController中：

> pkg\sky_engine\lib\async\broadcast_stream_controller.dart

```dart
Stream<T> get stream => new _BroadcastStream<T>(this);
```

#### 3.5.4 _BroadcastStream构造
看看他的构造方法：
> pkg\sky_engine\lib\async\broadcast_stream_controller.dart

```dart
class _BroadcastStream<T> extends _ControllerStream<T> {
  _BroadcastStream(_StreamControllerLifecycle<T> controller)
      : super(controller);

  bool get isBroadcast => true;
}
```
这里的controller其实是_BroadcastStreamController类型，因为_BroadcastStreamController实现了_StreamControllerBase接口，而_StreamControllerBase接口继承了_StreamControllerLifecycle接口。

#### 3.5.5 _ControllerStream构造
_BroadcastStream继承自_ControllerStream，看看他的构造方法：
> pkg\sky_engine\lib\async\stream_controller.dart

```dart
class _ControllerStream<T> extends _StreamImpl<T> {
    _ControllerStream(this._controller);
}
```

#### 3.5.6 _StreamImpl构造
_ControllerStream继承自_StreamImpl方法，其定义的地方是：
> pkg\sky_engine\lib\async\stream_impl.dart

```dart
abstract class _StreamImpl<T> extends Stream<T> {
}
```

Stream中持有StreamController对象，继承关系先到这里。

### 3.6 listen监听
在我们自己的dart代码中执行完receiveBroadcastStream之后，就要执行listen方法了。

listen方法定义在_StreamImpl类中：
> pkg\sky_engine\lib\async\stream_impl.dart\\_StreamImpl

```dart
StreamSubscription<T> listen(void onData(T data),
  {Function onError, void onDone(), bool cancelOnError}) {
    cancelOnError = identical(true, cancelOnError);
    StreamSubscription<T> subscription =
        _createSubscription(onData, onError, onDone, cancelOnError);
    _onListen(subscription);
    return subscription;
}

// -------------------------------------------------------------------
/** Create a subscription object. Called by [subcribe]. */
StreamSubscription<T> _createSubscription(void onData(T data),
  Function onError, void onDone(), bool cancelOnError) {
    return new _BufferingStreamSubscription<T>(
        onData, onError, onDone, cancelOnError);
}
```
这里的onData方法对应注册时的_onEvent方法，第二个参数中的onError对应注册时的_onError方法。

但是在_StreamImpl的子类_ControllerStream中，也定义了这个_createSubscription方法：
> pkg\sky_engine\lib\async\stream_controller.dart

```dart
class _ControllerStream<T> extends _StreamImpl<T> {
  StreamSubscription<T> _createSubscription(void onData(T data),
          Function onError, void onDone(), bool cancelOnError) =>
      _controller._subscribe(onData, onError, onDone, cancelOnError);
}
```
该调用哪一个呢？在`https://dartpad.dev/`写个demo看看:

```dart
abstract class Test1{
  void listen(){
    print("class init");
    _test1();
    print("class init1");
  }
  
  void _test1(){
    print("_test1 1");
  }
}

class Demo1 extends Test1{
  void _test1(){
    print("_test1 2");
  }
}

class Demo2 extends Demo1{
}

void main() {
  Demo2().listen();
}
```
输出如下：
```
class init
_test1 2
class init1
```

应该是调用_ControllerStream的_createSubscription方法。_controller对应的是_AsyncBroadcastStreamController，实际_subscribe方法在其父类_BroadcastStreamController中定义：
> pkg\sky_engine\lib\async\broadcast_stream_controller.dart

```dart
StreamSubscription<T> _subscribe(void onData(T data), Function onError,
  void onDone(), bool cancelOnError) {
    if (isClosed) {
      onDone ??= _nullDoneHandler;
      return new _DoneStreamSubscription<T>(onDone);
    }
    StreamSubscription<T> subscription = new _BroadcastSubscription<T>(
        this, onData, onError, onDone, cancelOnError);
    _addListener(subscription);
    if (identical(_firstSubscription, _lastSubscription)) {
      // Only one listener, so it must be the first listener.
      _runGuarded(onListen);
    }
    return subscription;
}
```
_BroadcastSubscription的继承链条是：_BroadcastSubscription,_ControllerSubscription,_BufferingStreamSubscription.

看看_addListener方法：
> pkg\sky_engine\lib\async\broadcast_stream_controller.dart

```dart
void _addListener(_BroadcastSubscription<T> subscription) {
    assert(identical(subscription._next, subscription));
    subscription._eventState = (_state & _STATE_EVENT_ID);
    // Insert in linked list as last subscription.
    _BroadcastSubscription<T> oldLast = _lastSubscription;
    _lastSubscription = subscription;
    subscription._next = null;
    subscription._previous = oldLast;
    if (oldLast == null) {
      _firstSubscription = subscription;
    } else {
      oldLast._next = subscription;
    }
}
```
第一次添加监听，可以得到_firstSubscription和_lastSubscription相等，也就是执行_runGuarded方法，也就会回调到onListen方法。这个方法一开始对应的就是broadcast方法里面的onListen闭包函数。

### 3.7 回到dart注册
现在回到最开始注册回调的地方，执行EventChannel中receiveBroadcastStream里面的onListen回调方法：
> packages\flutter\lib\src\services\platform_channel.dart

```dart
onListen: () async {
      binaryMessenger.setMessageHandler(name, (ByteData reply) async {
        if (reply == null) {
          controller.close();
        } else {
          try {
            controller.add(codec.decodeEnvelope(reply));
          } on PlatformException catch (e) {
            controller.addError(e);
          }
        }
        return null;
      });
      try {
        await methodChannel.invokeMethod<void>('listen', arguments);
      } catch (exception, stack) {
      }
    }
```

### 3.7.1 设置消息处理
binaryMessenger是_DefaultBinaryMessenger类型，看看里面的setMessageHandler方法：
> packages\flutter\lib\src\services\binding.dart

```dart
@override
void setMessageHandler(String channel, MessageHandler handler) {
    if (handler == null)
      _handlers.remove(channel);
    else
      _handlers[channel] = handler;
    ui.channelBuffers.drain(channel, (ByteData data, ui.PlatformMessageResponseCallback callback) async {
      await handlePlatformMessage(channel, data, callback);
    });
}
```
MessageHandler就是onListen第二个参数里面的闭包块。先将这个MessageHandler放到_handlers Map中。然后执行drain方法：
> pkg\sky_engine\lib\ui\channel_buffers.dart

```dart
Future<void> drain(String channel, DrainChannelCallback callback) async {
    while (!_isEmpty(channel)) {
      final _StoredMessage message = _pop(channel);
      await callback(message.data, message.callback);
    }
}
```

看看_isEmpty方法：
```dart
bool _isEmpty(String channel) {
    final _RingBuffer<_StoredMessage> queue = _messages[channel];
    return (queue == null) ? true : queue.isEmpty;
}
```
先获取channel对应的消息队列，如果为空返回true，否则判断消息队列的队头是否等于队尾，相等则为true，否则为false。

如果消息队列中有消息，此时就回调callback方法，传递数据和message.callback参数。

callback方法对应的调用到_DefaultBinaryMessenger中的handlePlatformMessage方法：
> packages\flutter\lib\src\services\binding.dart

```dart
@override
Future<void> handlePlatformMessage(
String channel,
ByteData data,
ui.PlatformMessageResponseCallback callback,
) async {
    ByteData response;
    try {
      final MessageHandler handler = _handlers[channel];
      if (handler != null) {
        response = await handler(data);
      } else {
        ui.channelBuffers.push(channel, data, callback);
        callback = null;
      }
    } catch (exception, stack) {

    } finally {
      if (callback != null) {
        callback(response);
      }
    }
}
```

这里的handler不为空，于是调用到最开始binaryMessenger.setMessageHandler的第二个闭包函数中：
> packages\flutter\lib\src\services\platform_channel.dart

```dart
binaryMessenger.setMessageHandler(name, (ByteData reply) async {
    controller.add(codec.decodeEnvelope(reply));
}
```
先解码收到的数据，在StandardMethodCodec中解码：
> packages\flutter\lib\src\services\message_codecs.dart

```dart
dynamic decodeEnvelope(ByteData envelope) {
    if (buffer.getUint8() == 0)
        return messageCodec.readValue(buffer);
}
```
读取数据，然后返回，这里的数据类型为dynamic，需要我们自己清楚两端发送和接收的数据类型即可。

### 3.7.2 invokeMethod发消息给Android
这个方法执行的时机是dart端执行listen方法之后，就会回调到onListen的闭包函数中，然后通过MethodChannel执行一个channel名字为listen的方法，实际最终执行到了Android的IncomingStreamRequestHandler类中的onMessage方法中，具体流程可以参考之前的文章 **flutter通信机制-MethodChannel**。也就实现了Android端针对`EventChannel.EventSink`变量的初始化。

### 3.7.3 添加数据处理
然后执行controller的add方法：
> pkg\sky_engine\lib\async\broadcast_stream_controller.dart\\_BroadcastStreamController

```dart
void add(T data) {
    if (!_mayAddEvent) throw _addEventError();
    _sendData(data);
}
```
add方法定义在_AsyncBroadcastStreamController类中，看看_sendData方法:
```dart
void _sendData(T data) {
    for (_BroadcastSubscription<T> subscription = _firstSubscription;
        subscription != null;
        subscription = subscription._next) {
      subscription._addPending(new _DelayedData<T>(data));
    }
}
```

接下来调用_addPending方法，在_BufferingStreamSubscription类中：

> pkg\sky_engine\lib\async\broadcast_stream_controller.dart

```dart
void _addPending(_DelayedEvent event) {
    _StreamImplEvents<T> pending = _pending;
    if (_pending == null) {
      pending = _pending = new _StreamImplEvents<T>();
    }
    pending.add(event);
    if (!_hasPending) {
      _state |= _STATE_HAS_PENDING;
      if (!_isPaused) {
        _pending.schedule(this);
      }
    }
}
```

### 3.7.4 调度消息回调

看看_StreamImplEvents的schedule方法：
> pkg\sky_engine\lib\async\stream_impl.dart\\_PendingEvents

```dart
void schedule(_EventDispatch<T> dispatch) {
    if (isScheduled) return;
    assert(!isEmpty);
    if (_eventScheduled) {
      assert(_state == _STATE_CANCELED);
      _state = _STATE_SCHEDULED;
      return;
    }
    scheduleMicrotask(() {
      int oldState = _state;
      _state = _STATE_UNSCHEDULED;
      if (oldState == _STATE_CANCELED) return;
      handleNext(dispatch);
    });
    _state = _STATE_SCHEDULED;
}
```

handleNext方法执行的就是_StreamImplEvents中的方法，_StreamImplEvents继承自_PendingEvents类：
> pkg\sky_engine\lib\async\stream_impl.dart

```dart
void handleNext(_EventDispatch<T> dispatch) {
    assert(!isScheduled);
    _DelayedEvent event = firstPendingEvent;
    firstPendingEvent = event.next;
    if (firstPendingEvent == null) {
      lastPendingEvent = null;
    }
    event.perform(dispatch);
}
```
到这里执行perform方法，其对应的是_DelayedData类中的方法：
> pkg\sky_engine\lib\async\stream_impl.dart

```dart
void perform(_EventDispatch<T> dispatch) {
    dispatch._sendData(value);
}
```
dispatch对象在_BufferingStreamSubscription的_addPending方法中调用schedule的时候，指代的就是_BufferingStreamSubscription本身，因此_sendData调用会在_BufferingStreamSubscription中：
> pkg\sky_engine\lib\async\stream_impl.dart

```dart
void _sendData(T data) {
    bool wasInputPaused = _isInputPaused;
    _state |= _STATE_IN_CALLBACK;
    _zone.runUnaryGuarded(_onData, data);
    _state &= ~_STATE_IN_CALLBACK;
    _checkState(wasInputPaused);
}
```
```_zone.runUnaryGuarded(_onData, data)```最终就会调用到我们最初dart代码中定义的onEvent方法中了，也就是可以在```void _onEvent(Object event) {}```回调方法中处理data数据了。

## 4、scheduleMicrotask任务调度

在_PendingEvents的schedule方法中，会执行scheduleMicrotask方法，看看这个方法里面是怎么执行的：
> pkg\sky_engine\lib\async\schedule_microtask.dart

```dart
void scheduleMicrotask(void callback()) {
  _Zone currentZone = Zone.current;
  if (identical(_rootZone, currentZone)) {
    // No need to bind the callback. We know that the root's scheduleMicrotask
    // will be invoked in the root zone.
    _rootScheduleMicrotask(null, null, _rootZone, callback);
    return;
  }
  _ZoneFunction implementation = currentZone._scheduleMicrotask;
  if (identical(_rootZone, implementation.zone) &&
      _rootZone.inSameErrorZone(currentZone)) {
    _rootScheduleMicrotask(
        null, null, currentZone, currentZone.registerCallback(callback));
    return;
  }
  Zone.current.scheduleMicrotask(Zone.current.bindCallbackGuarded(callback));
}
```

### 4.1 Zone运行空间
Zone表示一个可以稳定异步调用的环境。代码总是在一个空间的上下文中执行，比如`Zone.current`。初始化main函数在`Zone.root`空间中执行，代码可以在不同的空间中执行，既可以通过`runZoned`创建一个新的空间，也可以通过`Zone.run`方法在一个已经存在的空间上下文中执行，比如通过`Zone.fork`创建的空间中。

异步回调方法总是在他们被调度的上下文空间中运行，两步即可实现：
1、注册回调方法，方法是registerCallback或registerUnaryCallback或registerBinaryCallback，这允许空间记录这一个回调方法，后续可能也会存在修改，比如返回另外一个回调方法。做注册操作时的空间预示着后续回调也运行在这个空间中。

2、在后续某个时间点，回调方法在对应的空间中运行。

为了方便，空间提供了bindCallback（bindUnaryCallback或bindBinaryCallback）方法来表示这种机制，最开始注册方法所在的空间，就是其包裹的回调方法被异步执行时所在的空间。

同样的，空间提供了bindCallbackGuarded（bindUnaryCallbackGuarded或bindBinaryCallbackGuarded）方法，应该在其中通过调用`Zone.runGuarded`去执行回调方法。

### 4.2 执行任务
这里的Zone.current其实是_RootZone，意味着跟main函数所在的空间相同。于是这里调用_rootScheduleMicrotask方法：

> pkg\sky_engine\lib\async\zone.dart

```dart
void _rootScheduleMicrotask(
    Zone self, ZoneDelegate parent, Zone zone, void f()) {
  if (!identical(_rootZone, zone)) {
    bool hasErrorHandler = !_rootZone.inSameErrorZone(zone);
    if (hasErrorHandler) {
      f = zone.bindCallbackGuarded(f);
    } else {
      f = zone.bindCallback(f);
    }
    // Use root zone as event zone if the function is already bound.
    zone = _rootZone;
  }
  _scheduleAsyncCallback(f);
}
```
_rootZone和zone是相等的，于是调用_scheduleAsyncCallback，将f回调函数异步调用：
> pkg\sky_engine\lib\async\schedule_microtask.dart

```dart
void _scheduleAsyncCallback(_AsyncCallback callback) {
  _AsyncCallbackEntry newEntry = new _AsyncCallbackEntry(callback);
  if (_nextCallback == null) {
    _nextCallback = _lastCallback = newEntry;
    if (!_isInCallbackLoop) {
      _AsyncRun._scheduleImmediate(_startMicrotaskLoop);
    }
  } else {
    _lastCallback.next = newEntry;
    _lastCallback = newEntry;
  }
}
```

接下来执行_startMicrotaskLoop方法：
> pkg\sky_engine\lib\async\schedule_microtask.dart

```dart
void _startMicrotaskLoop() {
  _isInCallbackLoop = true;
  try {
    // Moved to separate function because try-finally prevents
    // good optimization.
    _microtaskLoop();
  } finally {
    _lastPriorityCallback = null;
    _isInCallbackLoop = false;
    if (_nextCallback != null) {
      _AsyncRun._scheduleImmediate(_startMicrotaskLoop);
    }
  }
}
```
可以看到在finally的代码块中，又会异步的调用_startMicrotaskLoop，当_nextCallback不为空时，就可以一直调用_microtaskLoop方法了。这些调用并不会阻塞UI线程，因为当前是异步的，而异步执行的方法是_microtaskLoop：

```dart
void _microtaskLoop() {
  while (_nextCallback != null) {
    _lastPriorityCallback = null;
    _AsyncCallbackEntry entry = _nextCallback;
    _nextCallback = entry.next;
    if (_nextCallback == null) _lastCallback = null;
    (entry.callback)();
  }
}
```

_nextCallback就是_AsyncCallbackEntry封装的异步callback方法，执行回调之前将_nextCallback赋值为下一个回调方法。

callback就是我们在_scheduleAsyncCallback方法中封装过来的callback回调方法，这个回调方法就是_rootScheduleMicrotask中的f()，也就是上一章的scheduleMicrotask方法的第二个参数，最终回调到我们的onEvent方法了。

## 5、Android通知flutter

回到flutter中_DefaultBinaryMessenger的handlePlatformMessage方法中：

```dart
@override
Future<void> handlePlatformMessage(
String channel,
ByteData data,
ui.PlatformMessageResponseCallback callback,
) async {
    ByteData response;
    try {
      final MessageHandler handler = _handlers[channel];
      if (handler != null) {
        response = await handler(data);
      } else {
        ui.channelBuffers.push(channel, data, callback);
        callback = null;
      }
    } catch (exception, stack) {
    } finally {
      if (callback != null) {
        callback(response);
      }
    }
}
```

当没有通过`binaryMessenger.setMessageHandler`设置MessageHandler时，消息存在队列中，一旦注册之后，马上就将消息分发给注册者；当Map中存在channel对应的MessageHandler时，直接回调，也就是回到了setMessageHandler的闭包代码块中，重复执行3.7.2之后的流程。