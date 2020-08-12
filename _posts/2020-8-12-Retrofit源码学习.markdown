---
layout: post
title:  "Retrofit源码学习"
date:   2020-8-12 18:58:25 +0800
categories: Android
---

> 文章将会同步到个人微信公众号：Android部落格

## 1 基本使用

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_retrofit);
    new Thread(new Runnable() {
        @Override
        public void run() {
            try {
                Retrofit retrofit = new Retrofit.Builder()
                        .baseUrl("https://api.github.com/")
                        .addConverterFactory(GsonConverterFactory.create())
                        .build();
                GithubService service = retrofit.create(GithubService.class);
                Call<ResponseBody> repos = service.listRepos("cg229836277");
                Response<ResponseBody> response = repos.execute();
                ResponseBody responseBody = response.body();
                String message = responseBody.string();
                MyLog.d(TAG, "message:" + message);

                Call<UserInfoBean> userRepos = service.getUserInfo("cg229836277");
                Response<UserInfoBean> userResponse = userRepos.execute();
                UserInfoBean userInfoBean = userResponse.body();
                String userName = userInfoBean.getName();
                MyLog.d(TAG, "userName:" + userName);
            } catch (Exception | Error e) {
                e.printStackTrace();
            }
        }
    }).start();
}
```

run方法中包裹的代码以空格为界，分为两部分，上半部分是返回的结果不解析，直接返回ResponseBody，拿到body string之后做其他处理；下半部分获取用户相关信息，由Gson解析序列化成UserInfoBean对象。

所有的接口定义在`GithubService`类中，如下：

```java
public interface GithubService {
    @GET("users/{user}/repos")
    Call<ResponseBody> listRepos(@Path("user") String user);

    @GET("users/{user}")
    Call<UserInfoBean> getUserInfo(@Path("user") String user);
}
```

## 2 源码查看

### 2.1 Retrofit.Builder
首先需要构建Retrofit，通过Retrofit.Builder类构建。

看看构建方法里面的几个方法。

#### 2.1.1 client
```java
public Builder client(OkHttpClient client) {
    return callFactory(Objects.requireNonNull(client, "client == null"));
}

public Builder callFactory(okhttp3.Call.Factory factory) {
    this.callFactory = Objects.requireNonNull(factory, "factory == null");
    return this;
}
```

这里将`OkHttpClient`设置为`callFactory`，因为OkHttpClient集成自Call.Factory，所以可以直接传递：

```java
open class OkHttpClient internal constructor(
  builder: Builder
) : Cloneable, Call.Factory, WebSocket.Factory {
```

#### 2.1.2 baseUrl
用于设置请求网址，比如`https://api.github.com/`是请求的基础url，后面还可以加很多请求query参数。

这个方法最终调用到：
```java
public Builder baseUrl(HttpUrl baseUrl) {
  Objects.requireNonNull(baseUrl, "baseUrl == null");
  List<String> pathSegments = baseUrl.pathSegments();
  if (!"".equals(pathSegments.get(pathSegments.size() - 1))) {
    throw new IllegalArgumentException("baseUrl must end in /: " + baseUrl);
  }
  this.baseUrl = baseUrl;
  return this;
}
```
在调用之前还调用了HttpUrl的get方法，将传递进来的URL或String地址，转成HttpUrl类型：
```java
private HttpUrl(Builder builder) {
    this.scheme = builder.scheme;
    this.username = percentDecode(builder.encodedUsername, false);
    this.password = percentDecode(builder.encodedPassword, false);
    this.host = builder.host;
    this.port = builder.effectivePort();
    this.pathSegments = percentDecode(builder.encodedPathSegments, false);
    this.queryNamesAndValues = builder.encodedQueryNamesAndValues != null
        ? percentDecode(builder.encodedQueryNamesAndValues, true)
        : null;
    this.fragment = builder.encodedFragment != null
        ? percentDecode(builder.encodedFragment, false)
        : null;
    this.url = builder.toString();
}
```
get方法将你传入的url先做规范化处理，比如url开始和结束位置有空格等其他非法字符等，处理完毕之后，接着获取请求方式，http或https，请求host,端口等信息。

#### 2.1.3 addConverterFactory

这个方法中，可以添加请求数据或返回数据的序列化类。比如开头的实例中，需要将返回的用户信息序列化成UserInfoBean类，可以通过这个方法添加GsonConverterFactory对象。

#### 2.1.4 addCallAdapterFactory

添加请求处理类，比如RxJavaCallAdapterFactory，可以通过将他的对象传递到这个方法，引入对RxJava的支持。

#### 2.1.5 build

```java
public Retrofit build() {
  if (baseUrl == null) {
    throw new IllegalStateException("Base URL required.");
  }

  okhttp3.Call.Factory callFactory = this.callFactory;
  if (callFactory == null) {
    callFactory = new OkHttpClient();
  }

  Executor callbackExecutor = this.callbackExecutor;
  if (callbackExecutor == null) {
    callbackExecutor = platform.defaultCallbackExecutor();
  }

  // Make a defensive copy of the adapters and add the default Call adapter.
  List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
  callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));

  // Make a defensive copy of the converters.
  List<Converter.Factory> converterFactories =
      new ArrayList<>(
          1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());

  // Add the built-in converter factory first. This prevents overriding its behavior but also
  // ensures correct behavior when using converters that consume all types.
  converterFactories.add(new BuiltInConverters());
  converterFactories.addAll(this.converterFactories);
  converterFactories.addAll(platform.defaultConverterFactories());

  return new Retrofit(
      callFactory,
      baseUrl,
      unmodifiableList(converterFactories),
      unmodifiableList(callAdapterFactories),
      callbackExecutor,
      validateEagerly);
}
```

- baseUrl，上层不设置baseUrl，会直接报错；
- callFactory，上层不设置，默认设置OkHttpClient，如果自己自定义的话，可以添加一些自定的interceptor，这个是okhttp的一大特色；
- callAdapterFactories，将上层添加的请求工厂类添加进来，另外还有平台自己默认的一个请求工厂类，在Platform类中定义，如下：

```java
List<? extends CallAdapter.Factory> defaultCallAdapterFactories(
  @Nullable Executor callbackExecutor) {
    DefaultCallAdapterFactory executorFactory = new DefaultCallAdapterFactory(callbackExecutor);
    return hasJava8Types
        ? asList(CompletableFutureCallAdapterFactory.INSTANCE, executorFactory)
        : singletonList(executorFactory);
}
```

这里hasJava8Types 在sdk版本大于等于24时，为true，这里将CompletableFutureCallAdapterFactory和DefaultCallAdapterFactory一起放到一个List中然后返回，这里DefaultCallAdapterFactory持有传递过来的callbackExecutor对象。

- converterFactories

用于存放请求参数或结果参数的序列化工具，初始化如下：

```java
// Make a defensive copy of the converters.
List<Converter.Factory> converterFactories =
  new ArrayList<>(
      1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());

// Add the built-in converter factory first. This prevents overriding its behavior but also
// ensures correct behavior when using converters that consume all types.
converterFactories.add(new BuiltInConverters());
converterFactories.addAll(this.converterFactories);
converterFactories.addAll(platform.defaultConverterFactories());

//retrofit\src\main\java\retrofit2\Platform.java
List<? extends Converter.Factory> defaultConverterFactories() {
    return hasJava8Types ? singletonList(OptionalConverterFactory.INSTANCE) : emptyList();
}
```
`OptionalConverterFactory`只有在Android sdk 版本大于等于24才添加。

`BuiltInConverters`为默认添加的转换类。

**总结**

最终build方法返回一个Retrofit对象，封装了上述参数。

```java
return new Retrofit(
          callFactory,
          baseUrl,
          unmodifiableList(converterFactories),
          unmodifiableList(callAdapterFactories),
          callbackExecutor,
          validateEagerly);
```

### 2.2 Retrofit.create

还是贴代码，然后分步骤分析:

```java
public <T> T create(final Class<T> service) {
    validateServiceInterface(service);
    return (T)
        Proxy.newProxyInstance(...);
}
```
上边create方法的参数里面，Class对象service对应文章开头的GithubService类的对象。

#### 2.2.1 validateServiceInterface
用于检查传递过来的形参service是否合法，第一，必须是interface类型；第二，interface类型的类，不能限定类型，比如GithubService<String>。以上两种情况下只要不满足就会抛出异常。

```java
private void validateServiceInterface(Class<?> service) {
    if (!service.isInterface()) {
      throw new IllegalArgumentException("API declarations must be interfaces.");
    }
    
    if (service.getTypeParameters().length != 0) {
        throw new IllegalArgumentException(message.toString());
    }
}
```

这个方法还涉及一个validateEagerly参数，在Retrofit.Builder中通过validateEagerly方法设置，如果设置为true之后，在这个方法中，就会接着解析interface里面的方法：

```java
if (validateEagerly) {
  Platform platform = Platform.get();
  for (Method method : service.getDeclaredMethods()) {
    if (!platform.isDefaultMethod(method) && !Modifier.isStatic(method.getModifiers())) {
      loadServiceMethod(method);
    }
  }
}
```
`isDefaultMethod`表示当前方法是public且非abstract修饰的实例方法，也就是一个非static方法，有方法体，在interface里面定义。其实说的是java 8的新特性，在interface中，可以添加default修饰符到一个方法，这个方法可以有实现，可以避免当在interface中添加一个方法，所有实现都必须改动的问题。

> A default method is a public non-abstract instance method, that is, a non-static method with a body, declared in an interface type.


validateEagerly为true时，会获取到service interface下所有定义的方法，并判断所有方法的类型，如果满足非static并且是default方法，就会调用loadServiceMethod方法，这个方法下一节分析。

### 2.3 调用interface中的方法

以文章开头demo中GithubService中的方法为例，通过`GithubService service = retrofit.create(GithubService.class);`获取了GithubService对象，接下来调用里面定义的API方法，具体调用在Retrofit.create方法中定义的内部类中实现：

```java
return (T)
    Proxy.newProxyInstance(
        service.getClassLoader(),
        new Class<?>[] {service},
        new InvocationHandler() {
          private final Platform platform = Platform.get();
          private final Object[] emptyArgs = new Object[0];

          @Override
          public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            args = args != null ? args : emptyArgs;
            return platform.isDefaultMethod(method)
                ? platform.invokeDefaultMethod(method, service, proxy, args)
                : loadServiceMethod(method).invoke(args);
        }
    });
```
在InvocationHandler类中，调用最后在invoke方法里面：
- proxy，Retrofit的代理对象；
- method，GithubService中定义的方法；
- args，方法中定义的参数，比如`Call<ResponseBody> listRepos(@Path("user") String user);`的参数就是user。

### 2.4 ServiceMethod和HttpServiceMethod

接下来判断是否是default修饰的方法，如果是，就直接调用，否则通过loadServiceMethod方法调用。
> ServiceMethod

```java
static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);

    Type returnType = method.getGenericReturnType();
    if (Utils.hasUnresolvableType(returnType)) {}
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
}
```
通过这个方法可以拆分出很多细节，包括注解解析，返回类型解析等。最终这些属性被封装到ServiceMethod类中。

```java
//Retrofit
ServiceMethod<?> loadServiceMethod(Method method) {
    ServiceMethod result = ServiceMethod.parseAnnotations(this, method);
}
//ServiceMethod
static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);
    
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
}
//RequestFactory
static RequestFactory parseAnnotations(Retrofit retrofit, Method method) {
    return new Builder(retrofit, method).build();
}

RequestFactory build() {
    for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
    }
    
    int parameterCount = parameterAnnotationsArray.length;
    parameterHandlers = new ParameterHandler<?>[parameterCount];
    for (int p = 0, lastParameter = parameterCount - 1; p < parameterCount; p++) {
        parameterHandlers[p] =
            parseParameter(p, parameterTypes[p], parameterAnnotationsArray[p], p == lastParameter);
    }
    
    return new RequestFactory(this);
}
```
上边的代码引申出了RequestFactory，这个类主要处理请求相关的参数。

#### 2.4.1 RequestFactory

- parseAnnotations

方法中通过建造者模式，构建了请求相关的诸多参数，这里只分析其中比较核心的参数。

- parseMethodAnnotation

通过方法可以知道，这个方法主要解析方法的注解：
```java
private void parseMethodAnnotation(Annotation annotation) {
  if (annotation instanceof DELETE) {
    parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
  } else if (annotation instanceof GET) {
    parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
  } else if (annotation instanceof HEAD) {
    parseHttpMethodAndPath("HEAD", ((HEAD) annotation).value(), false);
  } else if (annotation instanceof PATCH) {
    parseHttpMethodAndPath("PATCH", ((PATCH) annotation).value(), true);
  } else if (annotation instanceof POST) {
    parseHttpMethodAndPath("POST", ((POST) annotation).value(), true);
  } else if (annotation instanceof PUT) {
    parseHttpMethodAndPath("PUT", ((PUT) annotation).value(), true);
  } else if (annotation instanceof OPTIONS) {
    parseHttpMethodAndPath("OPTIONS", ((OPTIONS) annotation).value(), false);
  } else if (annotation instanceof HTTP) {
    HTTP http = (HTTP) annotation;
    parseHttpMethodAndPath(http.method(), http.path(), http.hasBody());
  } else if (annotation instanceof retrofit2.http.Headers) {
    String[] headersToParse = ((retrofit2.http.Headers) annotation).value();
    if (headersToParse.length == 0) {
      throw methodError(method, "@Headers annotation is empty.");
    }
    headers = parseHeaders(headersToParse);
  } else if (annotation instanceof Multipart) {
    if (isFormEncoded) {
      throw methodError(method, "Only one encoding annotation is allowed.");
    }
    isMultipart = true;
  } else if (annotation instanceof FormUrlEncoded) {
    if (isMultipart) {
      throw methodError(method, "Only one encoding annotation is allowed.");
    }
    isFormEncoded = true;
  }
}
```
**请求方法判断**

if-else判断，从DELETE到OPTIONS都属于http的请求方法，参见：[https://www.runoob.com/http/http-methods.html](https://www.runoob.com/http/http-methods.html)。

**请求方法注解参数**

`@GET("users/{user}/repos")`

以上述GET请求方法为例，parseHttpMethodAndPath方法解析出了`users/{user}/repos`这个地址，以及地址中的user参数。

```java
private void parseHttpMethodAndPath(String httpMethod, String value, boolean hasBody) {
    this.httpMethod = httpMethod;
    this.hasBody = hasBody;
    this.relativeUrl = value;
    this.relativeUrlParamNames = parsePathParameters(value);
}
```
relativeUrl对应地址，relativeUrlParamNames对应地址中的动态参数。

**注解以HTTP开头**

表示可以自定义请求，示例如下：
```java
@HTTP(method = "GET", path = "users/{user}", hasBody = false)
Call<UserInfoBean> getUserInfo1(@Path("user") String user);
```

**注解以Headers开头**

在http请求中添加请求头，如果一个项目中多个请求API都有相同的header字段，比如每个请求头中都有设备型号字段，设备的Android版本字段等，可以统一在OKHttpClient中添加Interceptor，而Interceptor中通过添加Request.Builder设置。这个OKHttpClient对象通过`Retrofit.Builder`中的client方法设置：

```java
public Builder client(OkHttpClient client) {}
```

**注解包含Multipart**

常见的 POST 数据提交的方式。使用表单上传文件时，必须让 <form> 表单的 enctype 等于 multipart/form-data。

请求头必须包含一个特殊的头信息：Content-Type，且其值必须是multipart/form-data，同时还要规定一个内容分隔符，用于分隔请求体中的多个POST内容。具体的头信息如下：
`Content-Type: multipart/form-data; boundary=${bound}`。

> [https://imququ.com/post/four-ways-to-post-data-in-http.html](https://imququ.com/post/four-ways-to-post-data-in-http.html)

示例如下：
```
POST http://www.example.com HTTP/1.1
Content-Type:multipart/form-data; boundary=----WebKitFormBoundaryrGKCBY7qhFd3TrwA

------WebKitFormBoundaryrGKCBY7qhFd3TrwA
Content-Disposition: form-data; name="text"

title
------WebKitFormBoundaryrGKCBY7qhFd3TrwA
Content-Disposition: form-data; name="file"; filename="chrome.png"
Content-Type: image/png

PNG ... content of chrome.png ...
------WebKitFormBoundaryrGKCBY7qhFd3TrwA--
```
主要用于上传文件。

**注解包含FormUrlEncoded**

其实是Content-Type:application/x-www-form-urlencoded，是post提交数据的一种方式，请求参数采用类似get的参数拼接方式。

> *application/x-www-form-urlencoded*
> 
> - post的默认请求
> 
> - 需要把对象参数序列化为字符串参数
> 
> - 参数采用类似get的参数拼接方式
> 
> - 使用URIencode转码方式，转码会增加体积，适合短字节
> 
> - 请求参数放在请求体里
> 
> - 不在地址栏显示参数，安全性高

> *multipart/form-data*
> 
> - 不转码，适合传输长字节(如文件)
> 
> - 请求参数放在请求体里
> 
> - 不在地址栏显示参数，安全性高

- parseParameter

解析注解地址中的参数。

```java
private @Nullable ParameterHandler<?> parseParameter(
        int p, Type parameterType, @Nullable Annotation[] annotations, boolean allowContinuation) {
    ParameterHandler<?> result = null;
    if (annotations != null) {
        for (Annotation annotation : annotations) {
            ParameterHandler<?> annotationAction =
            parseParameterAnnotation(p, parameterType, annotations, annotation);
            result = annotationAction;
        }
    }
    return result;
}

private ParameterHandler<?> parseParameterAnnotation(
        int p, Type type, Annotation[] annotations, Annotation annotation) {
    if (annotation instanceof Url) {
        return new ParameterHandler.RelativeUrl(method, p);
    } else if (annotation instanceof Path) {
        Converter<?, String> converter = retrofit.stringConverter(type, annotations);
        return new ParameterHandler.Path<>(method, p, name, converter, path.encoded());
    } else if (annotation instanceof Query) {
        Converter<?, String> converter = retrofit.stringConverter(iterableType, annotations);
        return new ParameterHandler.Query<>(name, converter, encoded).iterable();
    }
    ....
}
```
这里根据注解不同类型，返回不同类型的ParameterHandler，这个类的作用会在后面生成Request的时候，将API传递的参数值args替换到注解的地址或请求参数中。

例如ParameterHandler.Path会将注解中的{user}替换为cg229836277。

#### 2.4.2 HttpServiceMethod.parseAnnotations

ServiceMethod的parseAnnotations方法中，最后调用了HttpServiceMethod.parseAnnotations方法返回，同时也作为loadServiceMethod的结果返回。同时也是demo中GithubService调用API返回的结果，是Call类型。

- parseAnnotations

```java
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
  Retrofit retrofit, Method method, RequestFactory requestFactory) {
    Type adapterType = method.getGenericReturnType();
    CallAdapter<ResponseT, ReturnT> callAdapter =
        createCallAdapter(retrofit, method, adapterType, annotations);
    Type responseType = callAdapter.responseType();
    Converter<ResponseBody, ResponseT> responseConverter =
        createResponseConverter(retrofit, method, responseType);

    okhttp3.Call.Factory callFactory = retrofit.callFactory;
    return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
}
```
最中返回一个CallAdapted的对象。

- createCallAdapter

创建了请求CallAdapter，从Retrofit的callAdapterFactories列表中取：

```java
private static <ResponseT, ReturnT> CallAdapter<ResponseT, ReturnT> createCallAdapter(
  Retrofit retrofit, Method method, Type returnType, Annotation[] annotations) {
    try {
      //noinspection unchecked
      return (CallAdapter<ResponseT, ReturnT>) retrofit.callAdapter(returnType, annotations);
    } catch (RuntimeException e) { 
    }
}
```
> Retrofit

```java
public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
}

public CallAdapter<?, ?> nextCallAdapter(
  @Nullable CallAdapter.Factory skipPast, Type returnType, Annotation[] annotations) {
    int start = callAdapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
      CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
}
```

Retrofit的callAdapterFactories中最开始添加的child是：
> Platform

```java
List<? extends CallAdapter.Factory> defaultCallAdapterFactories(
  @Nullable Executor callbackExecutor) {
    DefaultCallAdapterFactory executorFactory = new DefaultCallAdapterFactory(callbackExecutor);
    return hasJava8Types
        ? asList(CompletableFutureCallAdapterFactory.INSTANCE, executorFactory)
        : singletonList(executorFactory);
}
```
CompletableFutureCallAdapterFactory和DefaultCallAdapterFactory被添加进来。

根据返回的泛型数据类型以及注解，最终返回的是DefaultCallAdapterFactory。然后调用get方法，返回CallAdapter的对象。

```java
public @Nullable CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit){
    return new CallAdapter<Object, Call<?>>() {
        @Override
        public Type responseType() {
            return responseType;
        }
        
        @Override
        public Call<Object> adapt(Call<Object> call) {
            return executor == null ? call : new ExecutorCallbackCall<>(executor, call);
        }
    };
}
```
这里的executor就是通过Retrofit.Builder中的callbackExecutor方法设置的。

- createResponseConverter

> HttpServiceMethod

```java
private static <ResponseT> Converter<ResponseBody, ResponseT> createResponseConverter(
  Retrofit retrofit, Method method, Type responseType) {
    return retrofit.responseBodyConverter(responseType, annotations);
}
```

同样的从Retrofit.Builder的build方法中初始化的几个Converter中确定：

> Retrofit.Builder

```java
List<Converter.Factory> converterFactories =
      new ArrayList<>(
          1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());

  // Add the built-in converter factory first. This prevents overriding its behavior but also
  // ensures correct behavior when using converters that consume all types.
  converterFactories.add(new BuiltInConverters());
  converterFactories.addAll(this.converterFactories);
  converterFactories.addAll(platform.defaultConverterFactories());
```

> Retrofit

```java
public <T> Converter<ResponseBody, T> nextResponseBodyConverter(
      @Nullable Converter.Factory skipPast, Type type, Annotation[] annotations) {
    int start = converterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = converterFactories.size(); i < count; i++) {
      Converter<ResponseBody, ?> converter =
          converterFactories.get(i).responseBodyConverter(type, annotations, this);
      if (converter != null) {
        //noinspection unchecked
        return (Converter<ResponseBody, T>) converter;
      }
    }
}
```

这里选择BuiltInConverters，调用其responseBodyConverter方法，实际返回的是BufferingResponseBodyConverter。

如果我们需要序列化的话，选择的就是在Retrofit构建时通过addConverterFactory传递进来的GsonConverterFactory，这里实际返回的是GsonResponseBodyConverter。

- callFactory

在Retrofit.Builder的build方法中，callFactory不设置的话，默认新建了一个OkHttpClient。

**总结**

HttpServiceMethod的parseAnnotations方法最后，将上述几个参数封装到CallAdapted返回。

#### 2.4.3 HttpServiceMethod.invoke
这个方法中先将上一节生成的几个对象封装到Call，然后调用CallAdapted的adapt方法返回。

> HttpServiceMethod

```java
@Override
final @Nullable ReturnT invoke(Object[] args) {
    Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
    return adapt(call, args);
}
```
args中就是调用API方法传递过来的。

> CallAdapted

```java
@Override
protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
  return callAdapter.adapt(call);
}
```
上一节分析过，callAdapter实际是DefaultCallAdapterFactory中get方法返回的CallAdapter，这里调用adapt方法返回：

```java
@Override
public Call<Object> adapt(Call<Object> call) {
    return executor == null ? call : new ExecutorCallbackCall<>(executor, call);
}
```

分两种情况，依据是是否通过Retrofit.Bulider的callbackExecutor设置executor：
- 不设置的话，为空，返回OkHttpCall；
- 设置之后，返回ExecutorCallbackCall，同时封装了executor和OkHttpCall。

> ExecutorCallbackCall

```java
ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
  this.callbackExecutor = callbackExecutor;
  this.delegate = delegate;
}
```
这里的callbackExecutor就是Retrofit构建的时候传入的线程池对象。在下一步执行请求的时候可能会用到。

### 2.5 Call.execute

此处会在上一步返回的ExecutorCallbackCall中执行：

> ExecutorCallbackCall

```java
public Response<T> execute() throws IOException {
  return delegate.execute();
}
```
delegate指向OkHttpCall，看看他的execute方法：

> OkHttpCall

```java
@Override
public Response<T> execute() throws IOException {
    okhttp3.Call call;
    
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;
    
      call = getRawCall();
    }
    
    if (canceled) {
      call.cancel();
    }
    
    return parseResponse(call.execute());
}

private okhttp3.Call getRawCall() throws IOException {
    return rawCall = createRawCall();
}

private okhttp3.Call createRawCall() throws IOException {
    okhttp3.Call call = callFactory.newCall(requestFactory.create(args));
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
}
```

- RequestFactory.create

> RequestFactory

```java
okhttp3.Request create(Object[] args) throws IOException {
    ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;
    RequestBuilder requestBuilder =
        new RequestBuilder(
            httpMethod,
            baseUrl,
            relativeUrl,
            headers,
            contentType,
            hasBody,
            isFormEncoded,
            isMultipart);
            
    List<Object> argumentList = new ArrayList<>(argumentCount);
    for (int p = 0; p < argumentCount; p++) {
      argumentList.add(args[p]);
      handlers[p].apply(requestBuilder, args[p]);
    }
    
    return requestBuilder.get().tag(Invocation.class, new Invocation(method, argumentList)).build();
}
```
通过API上边的注解获取了请求相关的属性信息，这里返回okhttp3.Request对象。

方法中的for循环，根据API调用时传递过来的参数，透过ParameterHandler对应到注解变量中，示例如下：

```java
@GET("users/{user}")
Call<UserInfoBean> getUserInfo(@Path("user") String user);
```

Path类型的注解最终调用到ParameterHandler.Path类中的apply方法，最终拼接的逻辑在RequestBuilder的addPathParam方法中：

> RequestBuilder

```java
void addPathParam(String name, String value, boolean encoded) {
    String replacement = canonicalizeForPath(value, encoded);
    String newRelativeUrl = relativeUrl.replace("{" + name + "}", replacement);
    if (PATH_TRAVERSAL.matcher(newRelativeUrl).matches()) {
      throw new IllegalArgumentException(
          "@Path parameters shouldn't perform path traversal ('.' or '..'): " + value);
    }
    relativeUrl = newRelativeUrl;
}
```

- 生成okhttp Request

首先通过RequestBuilder的get方法，将baseUrl和relativeUrl拼接起来，作为请求Url；

然后通过不同的contentType将对应的参数填充到RequestBody；

再将url，header，method，body封装到RequestBuilder返回；

最后调用RequestBuilder的build方法，将上述信息封装到okhttp3.Request中。

- newCall

callFactory对应OkHttpClient，这里调用newCall之后，返回okhttp3.Call，然后调用execute方法返回结果。

### 2.6 结果解析
从OkHttpCall的execute方法最后一行开始：

```java
@Override
public Response<T> execute() throws IOException {
    return parseResponse(call.execute());
}
```
parseResponse方法做具体的解析工作：

```java
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();
    rawResponse =
        rawResponse
        .newBuilder()
        .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
        .build();
    int code = rawResponse.code();
    ...
    ExceptionCatchingResponseBody catchingBody = new ExceptionCatchingResponseBody(rawBody);
    T body = responseConverter.convert(catchingBody);
    return Response.success(body, rawResponse);
}
```

- 第一步获取Response.body，然后将Response中的code,headers等重新封装到Response中。
- code小于200或大于300统一处理为报错；code等于204或205处理为成功，但是没有内容。
- 处理成功的内容，需要根据最初Retrofit.Builder构建的时候传递的Converter将结果中的body序列化成API返回Call泛型限定的类型。

responseConverter通过构建OkHttpCall时传递过来。根据前面的分析，HttpServiceMethod.invoke构建了OkHttpCall，传递过来的responseConverter实际是BufferingResponseBodyConverter或自定义的转换类型。

针对返回Response<ResponseBody>类型的数据，使用BufferingResponseBodyConverter，将body封装到返回：

> BufferingResponseBodyConverter.convert
>
> Utils.buffer

```java
static ResponseBody buffer(final ResponseBody body) throws IOException {
    Buffer buffer = new Buffer();
    body.source().readAll(buffer);
    return ResponseBody.create(body.contentType(), body.contentLength(), buffer);
}
```

针对返回Response<T>类型的数据，使用GsonResponseBodyConverter，将body序列化成T返回：

```java
public T convert(ResponseBody value) throws IOException {
    JsonReader jsonReader = gson.newJsonReader(value.charStream());
    try {
      T result = adapter.read(jsonReader);
      return result;
    } finally {
      value.close();
    }
}
```

最后通过Response.success方法将返回的原始okhttp3.Response及body返回给调用者。

#### 2.6.2 enqueue方法调用
另外根据OkHttpCall类提供的enqueue方法，可以换一种请求方式：

```java
Call<UserInfoBean> userRepos = service.getUserInfo1("cg229836277");
userRepos.enqueue(new Callback<UserInfoBean>() {
    @Override
    public void onResponse(Call<UserInfoBean> call, Response<UserInfoBean> response) {
        UserInfoBean userInfoBean = response.body();
        String userName = userInfoBean.getName();
        Log.d(TAG, "userName:" + userName);
    }

    @Override
    public void onFailure(Call<UserInfoBean> call, Throwable t) {
        Log.d(TAG, "userName error:" + t);
    }
});
```

在Retrofit构建时调用callbackExecutor传递自定义的工作线程，可以直接在UI线程中调用，因为okhttp的enqueue方法会在自定义的工作线程中请求，而execute方法则不会。如果设置了线程池，会在DefaultCallAdapterFactory.ExecutorCallbackCall中调用：

```java
@Override
public void enqueue(final Callback<T> callback) {
  Objects.requireNonNull(callback, "callback == null");

  delegate.enqueue(
      new Callback<T>() {
        @Override
        public void onResponse(Call<T> call, final Response<T> response) {
          callbackExecutor.execute(
              () -> {
                if (delegate.isCanceled()) {
                  // Emulate OkHttp's behavior of throwing/delivering an IOException on
                  // cancellation.
                  callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
                } else {
                  callback.onResponse(ExecutorCallbackCall.this, response);
                }
              });
        }

        @Override
        public void onFailure(Call<T> call, final Throwable t) {
          callbackExecutor.execute(() -> callback.onFailure(ExecutorCallbackCall.this, t));
        }
      });
}
```

## 3 总结

Retrofit总体看来就是使用了注解，简化了各种请求参数的填写。底层请求还是用的OkHttp。

另外上层提供了请求和结果处理的接口，使得这部分处理具有很大的灵活性，比如请求的时候可以通过addCallAdapterFactory方法添加RxJava。

另外需要注意的是，源码里面选择CallAdapter和Converter的方式，核心是下边两行代码：

```java
//获取请求
CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
//获取转换
Converter<ResponseBody, ?> converter =
          converterFactories.get(i).responseBodyConverter(type, annotations, this);
```
get方法在各自的转换类中实现，如果与方法的返回类型，及类型的泛型向匹配就返回对应的请求或转换类。