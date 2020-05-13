---
layout: post
title:  "Flutter 通信机制-MethodChannel"
date:   2020-5-9 11:09:22 +0800
categories: flutter
---

> 微信公众号：Android部落格

流程图如下：
![流程图](https://ftp.bmp.ovh/imgs/2020/05/47a564f0485c6973.jpg)

## 1、发送和接收

### 1.1 flutter端发送消息方式是：

```dart
class InteractUtil {
    static const platform = const MethodChannel("com.yourname.yourname/method");
      
    factory InteractUtil() => _getInstance();
    
    static InteractUtil get instance => _getInstance();
    static InteractUtil _instance;
    
    InteractUtil._internal();
  
    static InteractUtil _getInstance() {
        if (_instance == null) {
        _instance = new InteractUtil._internal();
        }
        return _instance;
    }
    
    Future<String> customMethodName(
    String parameter1, String parameter2) async {
        Future<String> result;
        try {
            print("customMethodName $parameter1 and $parameter2");
            result = await platform.invokeMethod("customMethodName", {
                "parameter1": parameter1,
                "parameter2": parameter2,
            });
        } on PlatformException catch (e) {
            print(e.toString());
        }
        return result;
    }
}
```

### 1.2 Android端接收消息的方式是：

```kotlin
class MainActivity : FlutterActivity() {
    val METHOD_CHANNEL = "com.yourname.yourname/method"
    override fun onCreate(savedInstanceState: Bundle?) {
        FlutterMain.startInitialization(this)
        super.onCreate(savedInstanceState)
        GeneratedPluginRegistrant.registerWith(this)

        MethodChannel(flutterView, METHOD_CHANNEL).setMethodCallHandler { p0, p1 ->
        Log.d(TAG, "method is ${p0.method}")
            when (p0.method) {
                "customMethodName" -> {
                    val parameter1 = p0.argument<String>("parameter1") as String
                    val parameter2 = p0.argument<String>("parameter2") as String
                }
            }
        }
    }
}
```
> p0是MethodCall类型，p1是Result类型

两端的MethodChannel name必须一致，另外flutter端调用invokeMethod 的方法名称要在Android或iOS端有定义。

## 2、flutter端
### 2.1 MethodChannel初始化
MethodChannel构造函数如下：

```dart
const MethodChannel(this.name, [this.codec = const StandardMethodCodec(), BinaryMessenger binaryMessenger ])
    : assert(name != null),assert(codec != null),
    _binaryMessenger = binaryMessenger;
```
先初始化了一个StandardMethodCodec对象，看看构造函数，如下：

```dart
const StandardMethodCodec([this.messageCodec = const StandardMessageCodec()]);
```
> StandardMethodCodec实现了MethodCodec接口

构造函数中又初始化了一个StandardMessageCodec对象，StandardMessageCodec实现了MessageCodec接口。

### 2.2 _binaryMessenger初始化
MethodChannel初始化成员变量的时候，会初始化_binaryMessenger参数，也就是set，看看定义：

```dart
BinaryMessenger get binaryMessenger => _binaryMessenger ?? defaultBinaryMessenger; // ignore: deprecated_member_use_from_same_package
final BinaryMessenger _binaryMessenger;
```
此处的binaryMessenger是null，但是下次当需要get的时候会走defaultBinaryMessenger参数，这个参数来自哪里呢？在`src\services\binary_messenger.dart`文件中定义了get方法：

```dart
BinaryMessenger get defaultBinaryMessenger {
  return ServicesBinding.instance.defaultBinaryMessenger;
}
```
看看ServicesBinding的defaultBinaryMessenger怎么初始化的：
> packages\flutter\lib\src\services\binding.dart\ServicesBinding

```dart
mixin ServicesBinding on BindingBase {
    @override
    void initInstances() {
        _instance = this;
        _defaultBinaryMessenger = createBinaryMessenger();
        window.onPlatformMessage = defaultBinaryMessenger.handlePlatformMessage;
        SystemChannels.system.setMessageHandler(handleSystemMessage);
    }
    BinaryMessenger get defaultBinaryMessenger =>  _defaultBinaryMessenger;
    BinaryMessenger _defaultBinaryMessenger;
}
```
同一个文件中看看createBinaryMessenger方法：

```dart
BinaryMessenger createBinaryMessenger() {
    return const _DefaultBinaryMessenger._();
}
```
创建了_DefaultBinaryMessenger对象，其继承自BinaryMessenger。

### 2.3 window.onPlatformMessage
将defaultBinaryMessenger.handlePlatformMessage方法设置给window.onPlatformMessage。也就是_DefaultBinaryMessenger的handlePlatformMessage方法。

看看onPlatformMessage的定义：
> lib\ui\window.dart\Window

```dart
PlatformMessageCallback get onPlatformMessage => _onPlatformMessage;
PlatformMessageCallback _onPlatformMessage;
Zone _onPlatformMessageZone;
set onPlatformMessage(PlatformMessageCallback callback) {
    _onPlatformMessage = callback;
    _onPlatformMessageZone = Zone.current;
}
```
这里的callback就是_DefaultBinaryMessenger的handlePlatformMessage方法。

看看这个方法的定义：
> packages\flutter\lib\src\services\binding.dart\\_DefaultBinaryMessenger

```dart
@override
Future<void> handlePlatformMessage(String channel,ByteData data,ui.PlatformMessageResponseCallback callback,) async {
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
onPlatformMessage接收到特定平台的消息时，就会调用到handlePlatformMessage方法。window.onPlatformMessage是一个全局的变量，在很多消息传递的地方都可以调用。

### 2.4 setMessageHandler
handleSystemMessage在ServicesBinding中，可以被其他Binding重载，如下:

```dart
Future<void> handleSystemMessage(Object systemMessage) async { }
```
看看system变量：
> packages\flutter\lib\src\services\system_channels.dart\SystemChannels

```dart
static const BasicMessageChannel<dynamic> system = BasicMessageChannel<dynamic>(
  'flutter/system',
  JSONMessageCodec(),
);
```
通过JSON来编解码消息。JSONMessageCodec实现了MessageCodec接口。

### 2.5 MethodChannel.invokeMethod
我们自己的dart调用invokeMethod方法发送消息，如下：

```dart
Future<T> invokeMethod<T>(String method, [ dynamic arguments ]) async {
    assert(method != null);
    final ByteData result = await binaryMessenger.send(name,codec.encodeMethodCall(MethodCall(method, arguments)),);
    if (result == null) {
      throw MissingPluginException('No implementation found for method $method on channel $name');
    }
    final T typedResult = codec.decodeEnvelope(result);
    return typedResult;
}
```
> 其实还有invokeListMethod和invokeMapMethod方法，最终都是执行到invokeMethod方法。

binaryMessenger对应的类是_DefaultBinaryMessenger，codec对应的是StandardMethodCodec类。

#### 2.5.1 数据编码
此处先将方法和参数通过MethodCall封装，然后在StandardMethodCodec中编码，看看具体怎么编码的：
> StandardMethodCodec

```dart
@override
ByteData encodeMethodCall(MethodCall call) {
    final WriteBuffer buffer = WriteBuffer();
    messageCodec.writeValue(buffer, call.method);
    messageCodec.writeValue(buffer, call.arguments);
    return buffer.done();
}
```
看看WriteBuffer的构造函数：
> packages\flutter\lib\src\foundation\serialization.dart\WriteBuffer

```dart
WriteBuffer() {
    _buffer = Uint8Buffer();
    _eightBytes = ByteData(8);
    _eightBytesAsList = _eightBytes.buffer.asUint8List();
}
```
WriteBuffer构造函数中初始化的Uint8Buffer类型，可以看做是一个List<int>，而ByteData可以看做是一个byte数组，长度为8。

messageCodec是一个StandardMessageCodec类型，看看他的writeValue方法：

```dart
void writeValue(WriteBuffer buffer, dynamic value) {
    writeSize(buffer, value.length);
    buffer.putxxx(_valueFloat64);
}
```
根据value的不同类型，put到不同的集合中。如果value是int类型，就放到_buffer中；如果是float类型，占8位，放到_eightBytes中；而我们的method是String类型，将其转换为utf-8之后，转成byte数组放到_buffer中。对于value占位大于1的，先写占用长度，再写占用值。不同的长度对应不同的数字，如下：

```dart
static const int _valueNull = 0;
static const int _valueTrue = 1;
static const int _valueFalse = 2;
static const int _valueInt32 = 3;
static const int _valueInt64 = 4;
static const int _valueLargeInt = 5;
static const int _valueFloat64 = 6;
static const int _valueString = 7;
static const int _valueUint8List = 8;
static const int _valueInt32List = 9;
static const int _valueInt64List = 10;
static const int _valueFloat64List = 11;
static const int _valueList = 12;
static const int _valueMap = 13;
```

看done方法：
> packages\flutter\lib\src\foundation\serialization.dart\WriteBuffer

```dart
ByteData done() {
    final ByteData result = _buffer.buffer.asByteData(0, _buffer.lengthInBytes);
    _buffer = null;
    return result;
}
```
其实是将刚才write进去的数据转成一个byte数组返回，这个数组的代表就是ByteData对象。

### 2.6 发送消息（binaryMessenger.send）
上面提到binaryMessenger是_DefaultBinaryMessenger类型，看看里面的send方法：

```dart
@override
Future<ByteData> send(String channel, ByteData message) {
    final MessageHandler handler = _mockHandlers[channel];
    if (handler != null)
      return handler(message);
    return _sendPlatformMessage(channel, message);
}
```
这里的_mockHandlers是Map<String, MessageHandler>类型，而MessageHandler的类型却是：

```dart
typedef MessageHandler = Future<ByteData> Function(ByteData message);
```
一个函数。所以send方法最开始从这个Map中查找channel名称对应的函数是否存在，存在的话，直接调用，而不用执行后面的方法了。明显的，这里需要调用_sendPlatformMessage方法：

```dart
Future<ByteData> _sendPlatformMessage(String channel, ByteData message) {
    final Completer<ByteData> completer = Completer<ByteData>();
    ui.window.sendPlatformMessage(channel, message, (ByteData reply) {
        completer.complete(reply);
    });
    return completer.future;
}
```
看看ui.window的sendPlatformMessage方法：
> ui\window.dart\Window

```dart
void sendPlatformMessage(String name,ByteData data,PlatformMessageResponseCallback callback) {
    final String error = _sendPlatformMessage(name, _zonedPlatformMessageResponseCallback(callback), data);
}
String _sendPlatformMessage(String name,
                          PlatformMessageResponseCallback callback,
                          ByteData data) native 'Window_sendPlatformMessage';
```
最终调用到了native的Window_sendPlatformMessage方法：
> ui\window\window.cc\Window

```c
void Window::RegisterNatives(tonic::DartLibraryNatives* natives) {
    {"Window_sendPlatformMessage", _SendPlatformMessage, 4, true},
}

void _SendPlatformMessage(Dart_NativeArguments args) {
  tonic::DartCallStatic(&SendPlatformMessage, args);
}

Dart_Handle SendPlatformMessage(Dart_Handle window,const std::string& name,Dart_Handle callback,Dart_Handle data_handle) {
  UIDartState* dart_state = UIDartState::Current();

  if (!dart_state->window()) {
    return tonic::ToDart(
        "Platform messages can only be sent from the main isolate");
  }

  fml::RefPtr<PlatformMessageResponse> response;
  if (!Dart_IsNull(callback)) {
    response = fml::MakeRefCounted<PlatformMessageResponseDart>(
        tonic::DartPersistentValue(dart_state, callback),
        dart_state->GetTaskRunners().GetUITaskRunner());
  }
  if (Dart_IsNull(data_handle)) {
    dart_state->window()->client()->HandlePlatformMessage(
        fml::MakeRefCounted<PlatformMessage>(name, response));
  } else {
    tonic::DartByteData data(data_handle);
    const uint8_t* buffer = static_cast<const uint8_t*>(data.data());
    dart_state->window()->client()->HandlePlatformMessage(fml::MakeRefCounted<PlatformMessage>(name, std::vector<uint8_t>(buffer, buffer + data.length_in_bytes()),response));
  }

  return Dart_Null();
}
```
DartCallStatic方法将args转换成了dart可处理的变量类型。

#### 2.6.1 预定义返回

这里response对应的是PlatformMessageResponseDart类型，看看这个方法：
> ui\window\platform_message_response_dart.cc\PlatformMessageResponseDart

```c
mMessageResponseDart::PlatformMessageResponseDart(
    tonic::DartPersistentValue callback,
    fml::RefPtr<fml::TaskRunner> ui_task_runner)
    : callback_(std::move(callback)),
      ui_task_runner_(std::move(ui_task_runner)) {}

void PlatformMessageResponseDart::Complete(std::unique_ptr<fml::Mapping> data) {
  if (callback_.is_empty())
    return;
  FML_DCHECK(!is_complete_);
  is_complete_ = true;
  ui_task_runner_->PostTask(fml::MakeCopyable(
      [callback = std::move(callback_), data = std::move(data)]() mutable {
        std::shared_ptr<tonic::DartState> dart_state = callback.dart_state().lock();
        if (!dart_state)
          return;
        tonic::DartState::Scope scope(dart_state);

        Dart_Handle byte_buffer = WrapByteData(std::move(data));
        tonic::DartInvoke(callback.Release(), {byte_buffer});
      }));
}
```
调用完成时会回调到这里,最终会调用callback对应的闭包。

### 2.6.3 处理消息（HandlePlatformMessage）

接下来执行HandlePlatformMessage，看看一系列调用堆栈：
> runtime\runtime_controller.cc

```c
// |WindowClient|
void RuntimeController::HandlePlatformMessage(fml::RefPtr<PlatformMessage> message) {
  client_.HandlePlatformMessage(std::move(message));
}
```
> shell\common\engine.cc

```c
void Engine::HandlePlatformMessage(fml::RefPtr<PlatformMessage> message) {
  if (message->channel() == kAssetChannel) {
    HandleAssetPlatformMessage(std::move(message));
  } else {
    delegate_.OnEngineHandlePlatformMessage(std::move(message));
  }
}
```
> shell\common\shell.cc

```c
// |Engine::Delegate|
void Shell::OnEngineHandlePlatformMessage(fml::RefPtr<PlatformMessage> message) {
    if (message->channel() == kSkiaChannel) {
        HandleEngineSkiaMessage(std::move(message));
        return;
    }

    task_runners_.GetPlatformTaskRunner()->PostTask([view = platform_view_->GetWeakPtr(), message = std::move(message)]() {
        if (view) {
            view->HandlePlatformMessage(std::move(message));
        }
    });
}
```
view对应类，以Android平台为例：
> shell\platform\android\platform_view_android.cc

```c
// |PlatformView|
void PlatformViewAndroid::HandlePlatformMessage(fml::RefPtr<flutter::PlatformMessage> message) {
  JNIEnv* env = fml::jni::AttachCurrentThread();
  fml::jni::ScopedJavaLocalRef<jobject> view = java_object_.get(env);
  if (view.is_null())
    return;

  int response_id = 0;
  if (auto response = message->response()) {
    response_id = next_response_id_++;
    pending_responses_[response_id] = response;
  }
  auto java_channel = fml::jni::StringToJavaString(env, message->channel());
  if (message->hasData()) {
    fml::jni::ScopedJavaLocalRef<jbyteArray> message_array(env, env->NewByteArray(message->data().size()));
    env->SetByteArrayRegion(message_array.obj(), 0, message->data().size(), reinterpret_cast<const jbyte*>(message->data().data()));
    message = nullptr;

    // This call can re-enter in InvokePlatformMessageXxxResponseCallback.
    FlutterViewHandlePlatformMessage(env, view.obj(), java_channel.obj(),message_array.obj(), response_id);
  } else {
    message = nullptr;
    
    // This call can re-enter in InvokePlatformMessageXxxResponseCallback.
    FlutterViewHandlePlatformMessage(env, view.obj(), java_channel.obj(),nullptr, response_id);
  }
}
```
接下来调用FlutterViewHandlePlatformMessage方法，这个方法对应的是Java jni注册的方法，看看方法体：
> shell\platform\android\platform_view_android_jni.cc

```c
void FlutterViewHandlePlatformMessage(JNIEnv* env, jobject obj,jstring channel,jobject message,jint responseId) {
  env->CallVoidMethod(obj, g_handle_platform_message_method, channel, message,responseId);
  FML_CHECK(CheckException(env));
}
```
看看g_handle_platform_message_method注册的地方：
> shell\platform\android\platform_view_android_jni.cc

```c
g_handle_platform_message_method = env->GetMethodID(g_flutter_jni_class->obj(), "handlePlatformMessage","(Ljava/lang/String;[BI)V");
```

### 2.6.4 到系统平台处理消息

对应Java中的handlePlatformMessage方法，参数是String，byte数组，int。看看handlePlatformMessage方法：
> shell\platform\android\io\flutter\embedding\engine\FlutterJNI.java

```java
private void handlePlatformMessage(
  @NonNull final String channel, byte[] message, final int replyId) {
    if (platformMessageHandler != null) {
      platformMessageHandler.handleMessageFromDart(channel, message, replyId);
    }
}
```
在FlutterJNI类中，通过setPlatformMessageHandler方法设置platformMessageHandler对象，看看是哪里设置的。还记得FlutterNativeView构造函数中，最后调用了attach方法，如下：
> shell\platform\android\io\flutter\view\FlutterNativeView.java

```java
private void attach(FlutterNativeView view, boolean isBackgroundView) {
    mFlutterJNI.attachToNative(isBackgroundView);
    dartExecutor.onAttachedToJNI();
}
```
看看onAttachedToJNI方法：
> shell\platform\android\io\flutter\embedding\engine\dart\DartExecutor.java

```java
public void onAttachedToJNI() {
    flutterJNI.setPlatformMessageHandler(dartMessenger);
}
```

这里的dartMessenger对应的是DartMessenger类，所以此处的platformMessageHandler对应的是DartMessenger对象。另外在dartMessenger初始化的时候，会调用setMessageHandler方法：
> shell\platform\android\io\flutter\embedding\engine\dart\DartExecutor.java

```java
private final BinaryMessenger.BinaryMessageHandler isolateChannelMessageHandler =
  new BinaryMessenger.BinaryMessageHandler() {
    @Override
    public void onMessage(ByteBuffer message, final BinaryReply callback) {
      isolateServiceId = StringCodec.INSTANCE.decodeMessage(message);
      if (isolateServiceIdListener != null) {
        isolateServiceIdListener.onIsolateServiceIdAvailable(isolateServiceId);
      }
    }
};

public DartExecutor(@NonNull FlutterJNI flutterJNI, @NonNull AssetManager assetManager) {
    this.flutterJNI = flutterJNI;
    this.assetManager = assetManager;
    this.dartMessenger = new DartMessenger(flutterJNI);
    dartMessenger.setMessageHandler("flutter/isolate", isolateChannelMessageHandler);
    this.binaryMessenger = new DefaultBinaryMessenger(dartMessenger);
}
```
这里的isolateChannelMessageHandler对应的是BinaryMessenger.BinaryMessageHandler类，channel name是flutter/isolate。在DartMessenger中，通过setMessageHandler方法将channel name和MessageHandler放到了messageHandlers这个Map对象中。如下：

```java
@Override
public void setMessageHandler(@NonNull String channel, @Nullable BinaryMessenger.BinaryMessageHandler handler) {
    if (handler == null) {
      Log.v(TAG, "Removing handler for channel '" + channel + "'");
      messageHandlers.remove(channel);
    } else {
      Log.v(TAG, "Setting handler for channel '" + channel + "'");
      messageHandlers.put(channel, handler);
    }
}
```

回到开始调用handleMessageFromDart的地方，这个方法在DartMessenger中有定义：
> shell\platform\android\io\flutter\embedding\engine\dart\DartMessenger.java

```java
@Override
public void handleMessageFromDart(
  @NonNull final String channel, @Nullable byte[] message, final int replyId) {
    Log.v(TAG, "Received message from Dart over channel '" + channel + "'");
    BinaryMessenger.BinaryMessageHandler handler = messageHandlers.get(channel);
    if (handler != null) {
      try {
        Log.v(TAG, "Deferring to registered handler to process message.");
        final ByteBuffer buffer = (message == null ? null : ByteBuffer.wrap(message));
        handler.onMessage(buffer, new Reply(flutterJNI, replyId));
      } catch (Exception ex) {
        Log.e(TAG, "Uncaught exception in binary message listener", ex);
        flutterJNI.invokePlatformMessageEmptyResponseCallback(replyId);
      }
    } else {
      Log.v(TAG, "No registered handler for message. Responding to Dart with empty reply message.");
      flutterJNI.invokePlatformMessageEmptyResponseCallback(replyId);
    }
}
```
从messageHandlers这个Map中取出channel name对应的MessageHandler，然后回调到他的onMessage方法中。

## 3、Android端
### 3.1 MethodChannel
Android端最开始注册接收的代码如下：

```kotlin
MethodChannel(flutterView, METHOD_CHANNEL).setMethodCallHandler {
}
```
flutterView其实是FlutterActivityDelegate中的FlutterView flutterView对象，而FlutterView实现了BinaryMessenger接口。

看看MethodChannel构造函数：
> MethodChannel

```java
public MethodChannel(BinaryMessenger messenger, String name) {
    this(messenger, name, StandardMethodCodec.INSTANCE);
}

public MethodChannel(BinaryMessenger messenger, String name, MethodCodec codec) {
    this.messenger = messenger;
    this.name = name;
    this.codec = codec;
}
```
这里的messenger就是FlutterView类型。

### 3.2 MethodCallHandler
接下来看看MethodChannel的setMethodCallHandler方法：
> shell\platform\android\io\flutter\plugin\common\MethodChannel.java

```java
public void setMethodCallHandler(final @Nullable MethodCallHandler handler) {
    messenger.setMessageHandler(name,handler == null ? null : new IncomingMethodCallHandler(handler));
}
```
到这里，我们的handler其实是在我们的MainActivity中定义了的，只不过lamada表达式将类名隐藏，只显示了闭包接收的参数。完整的形式是：
```kotlin
MethodChannel(flutterView, METHOD_CHANNEL).setMethodCallHandler(MethodChannel.MethodCallHandler { p0, p1 ->
}
```
#### 3.2.1 IncomingMethodCallHandler
最终通过IncomingMethodCallHandler将我们的MethodCallHandler初始化给IncomingMethodCallHandler类的handler对象，如下：

```java
private final class IncomingMethodCallHandler implements BinaryMessageHandler {
    private final MethodCallHandler handler;

    IncomingMethodCallHandler(MethodCallHandler handler) {
      this.handler = handler;
    }
    public void onMessage(ByteBuffer message, final BinaryReply reply) {
      
    }
}
```
这里的codec在MethodChannel构造的时候默认为StandardMethodCodec实例。

看看messenger所属的FlutterView中的setMessageHandler函数：
> FlutterView

```java
public void setMessageHandler(String channel, BinaryMessageHandler handler) {
    mNativeView.setMessageHandler(channel, handler);
}
```
这里的handler就是IncomingMethodCallHandler,看看setMessageHandler的setMessageHandler方法：
> FlutterNativeView

```java
public void setMessageHandler(String channel, BinaryMessageHandler handler) {
    dartExecutor.getBinaryMessenger().setMessageHandler(channel, handler);
}
```

### 3.3 DefaultBinaryMessenger
又回到dartExecutor这里了，getBinaryMessenger()方法对应的是DefaultBinaryMessenger，看看他的setMessageHandler方法：

```java
public void setMessageHandler(@NonNull String channel, @Nullable BinaryMessenger.BinaryMessageHandler handler) {
  messenger.setMessageHandler(channel, handler);
}
```
DefaultBinaryMessenger构造的时候，有初始化一个messenger对象，他对应的是DartMessenger，就可以发现最终是向他所属的Map中添加channel对应的BinaryMessenger.BinaryMessageHandler。

```java
messageHandlers.put(channel, handler);
```

这里终于将dart发送过来的消息和java层的注册对接起来了。

### 3.4 MethodCallHandler中处理消息
handleMessageFromDart中通过handler执行的onMessage方法，即IncomingMethodCallHandler类中的onMessage方法，通过StandardMethodCodec解码之后得到MethodCall对象，然后回调给MainActivity里面MethodCallHandler的onMethodCall方法。

> IncomingMethodCallHandler直接处理dart来的数据，并分发

```java
public void onMessage(ByteBuffer message, final BinaryReply reply) {
    final MethodCall call = codec.decodeMethodCall(message);
    handler.onMethodCall(call,new Result() {
        @Override
        public void success(Object result) {
            reply.reply(codec.encodeSuccessEnvelope(result));
        }
        
        @Override
        public void error(String errorCode, String errorMessage, Object errorDetails) {
            reply.reply(codec.encodeErrorEnvelope(errorCode, errorMessage, errorDetails));
        }
    });
}
```

MainActicity注册Handler之后接收数据，处理数据，完成之后返回

```kotlin
MethodChannel(flutterView, METHOD_CHANNEL).setMethodCallHandler(MethodChannel.MethodCallHandler { p0, p1 ->
    MyLog.d(TAG, "method is ${p0.method}")
    when (p0.method) {
        "method_name" -> {
            p1.success(data)
        }
    }
}
```

## 4、返回
Android端处理完dart的消息之后，可以通过Result对象将需要返回的数据返回，包括成功，失败，未完成。

reply其实是在handleMessageFromDart方法中初始化的，```handler.onMessage(buffer, new Reply(flutterJNI, replyId))```。

### 4.1 返回成功或失败
成功的话，会通过StandardMethodCodec的encodeSuccessEnvelope将数据编码：

```java
reply.reply(codec.encodeSuccessEnvelope(result));

public ByteBuffer encodeSuccessEnvelope(Object result) {
    final ExposedByteArrayOutputStream stream = new ExposedByteArrayOutputStream();
    stream.write(0);
    messageCodec.writeValue(stream, result);
    final ByteBuffer buffer = ByteBuffer.allocateDirect(stream.size());
    buffer.put(stream.buffer(), 0, stream.size());
    return buffer;
}
```
先写进去一个成功的标志位，messageCodec是StandardMethodCodec类型，将要返回的数据写进一个byte数组中，然后放到ByteBuffer中，返回。

失败的话，会有失败原因：

```java
reply.reply(codec.encodeErrorEnvelope(errorCode, errorMessage, errorDetails));

@Override
public ByteBuffer encodeErrorEnvelope(
  String errorCode, String errorMessage, Object errorDetails) {
    final ExposedByteArrayOutputStream stream = new ExposedByteArrayOutputStream();
    stream.write(1);
    messageCodec.writeValue(stream, errorCode);
    messageCodec.writeValue(stream, errorMessage);
    messageCodec.writeValue(stream, errorDetails);
    final ByteBuffer buffer = ByteBuffer.allocateDirect(stream.size());
    buffer.put(stream.buffer(), 0, stream.size());
    return buffer;
}
```
编码失败数据时，与成功唯一的差别是标志位不一样。

### 4.2 Reply返回数据
看看Reply的具体返回方式：
#### 4.2.1 Reply

```java
@Override
public void reply(@Nullable ByteBuffer reply) {
  if (done.getAndSet(true)) {
    throw new IllegalStateException("Reply already submitted");
  }
  if (reply == null) {
    flutterJNI.invokePlatformMessageEmptyResponseCallback(replyId);
  } else {
    flutterJNI.invokePlatformMessageResponseCallback(replyId, reply, reply.position());
  }
}
```
我们这里的reply不为空，执行invokePlatformMessageResponseCallback方法：
> shell\platform\android\io\flutter\embedding\engine\FlutterJNI.java

```java
public void invokePlatformMessageResponseCallback(int responseId, @Nullable ByteBuffer message, int position) {
    if (isAttached()) {
      nativeInvokePlatformMessageResponseCallback(nativePlatformViewId, responseId, message, position);
    }
}

private native void nativeInvokePlatformMessageResponseCallback(long nativePlatformViewId, int responseId, @Nullable ByteBuffer message, int position);
```
这里的nativePlatformViewId其实是我们在应用启动时，初始化引擎返回给我们的一个long类型的id。

### 4.3 引擎中处理数据

nativeInvokePlatformMessageResponseCallback对应注册的native方法是InvokePlatformMessageResponseCallback：
> shell\platform\android\platform_view_android_jni.cc

```c
bool RegisterApi(JNIEnv* env) {
  static const JNINativeMethod flutter_jni_methods[] = {
    {
      .name = "nativeInvokePlatformMessageResponseCallback",
      .signature = "(JILjava/nio/ByteBuffer;I)V",
      .fnPtr = reinterpret_cast<void*>(&InvokePlatformMessageResponseCallback),
    },
  }
}
```
具体看看InvokePlatformMessageResponseCallback的调用栈：
> shell\platform\android\platform_view_android_jni.cc

```c
static void InvokePlatformMessageResponseCallback(JNIEnv* env,jobject jcaller,jlong shell_holder,jint responseId,jobject message,jint position) {
    ANDROID_SHELL_HOLDER->GetPlatformView()->InvokePlatformMessageResponseCallback(env,responseId,message,position);
}
```
> shell\platform\android\platform_view_android.cc

```c
void PlatformViewAndroid::InvokePlatformMessageResponseCallback(JNIEnv* env,jint response_id,jobject java_response_data,jint java_response_position) {
    if (!response_id)
        return;
    auto it = pending_responses_.find(response_id);
    if (it == pending_responses_.end())
        return;
    uint8_t* response_data = static_cast<uint8_t*>(env->GetDirectBufferAddress(java_response_data));
    std::vector<uint8_t> response = std::vector<uint8_t>(response_data, response_data + java_response_position);
    auto message_response = std::move(it->second);
    pending_responses_.erase(it);
    message_response->Complete(std::make_unique<fml::DataMapping>(std::move(response)));
}
```

### 4.3 平台返回数据回到dart

这里的Complete对应的是在PlatformMessageResponseDart中，flutter发送消息的过程中有定义，这个方法中对应的callback方法在dart send方法的时候就有定义，这里再写一遍：

```dart
Future<ByteData> _sendPlatformMessage(String channel, ByteData message) {
    ui.window.sendPlatformMessage(channel, message, (ByteData reply) {
        completer.complete(reply);
    });
    return completer.future;
}
```
最终执行到这个闭包了。Completer是一个工作流的状态管理类，最开始的时候新建一个对象，接下来执行future任务，任务完成后执行complete表示执行完成。

执行完成的时候，在MethodChannel的invokeMethod方法中就接收到了返回数据：
> flutter\lib\src\services\platform_channel.dart

```dart
Future<T> invokeMethod<T>(String method, [ dynamic arguments ]) async {
    final ByteData result = await binaryMessenger.send(name,codec.encodeMethodCall(MethodCall(method, arguments)),);
    if (result == null) {
        throw MissingPluginException('No implementation found for method $method on channel $name');
    }
    final T typedResult = codec.decodeEnvelope(result);
    return typedResult;
}
```
这里的codec是StandardMethodCodec类型，将返回的数据解码就可以返回了。最终也在我们自己的dart代码中接收到返回数据了。

> 微信公众号二维码

![微信公众号](https://ftp.bmp.ovh/imgs/2020/04/f472fa7e9c62a8c7.jpg)