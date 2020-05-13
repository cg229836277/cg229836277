---
layout: post
title:  "Android OKHttp系列7-HttpURLConnection"
date:   2019-5-23 11:09:22 +0800
categories: Android
---


> 文章将会被同步至微信公众号：Android部落格
> 
>**文章末尾有系列流程图高清图片地址**

## 1、Android Http请求
一个典型的请求方式是：

```kotlin
private fun getContent(url: String): String? {
    var connection: HttpURLConnection? = null
    var info: String? = null
    try {
        connection = URL(url).openConnection() as HttpURLConnection
        connection.requestMethod = "GET"
        connection.useCaches = true
        connection.connectTimeout = 30000
        connection.readTimeout = 30000
        connection.connect()
        val responseCode = connection.responseCode
        if (responseCode == HttpURLConnection.HTTP_OK) {
            var inputStream = connection.inputStream
            var output = ByteArrayOutputStream()
            try {
                val buffer = ByteArray(4096)
                //inputStream.read(buffer)
                if (inputStream == null) {
                    return null
                }
                var n = inputStream.read(buffer)
                while (-1 != n) {
                    output.write(buffer, 0, n)
                    n = inputStream.read(buffer)
                }
                output.flush()
                info = output.toString("UTF-8")
            } finally {
                inputStream.close()
                inputStream = null
                output.close()
            }
        } else {
            Log.e(TAG, "postContent response: $responseCode")
        }
    } catch (e: Exception) {
        Log.e(TAG, "Exception " + e.message)
    } finally {
        connection?.disconnect()
    }
    return info
}
```
接下来看看请求背后发生了什么。

先看流程图：
![](https://ftp.bmp.ovh/imgs/2020/05/d826812169e32460.jpg)

### 1.1 URL

> libcore\ojluni\src\main\java\java\net

- 构造
new URL(url)最终调用到：

```java
public URL(URL context, String spec, URLStreamHandler handler) throws MalformedURLException {
    if (handler == null &&
    (handler = getURLStreamHandler(protocol)) == null) {
        throw new MalformedURLException("unknown protocol: "+protocol);
    }
    this.handler = handler;
    handler.parseURL(this, spec, start, limit);
}
```
- getURLStreamHandler

```java
static URLStreamHandler getURLStreamHandler(String protocol) {
    URLStreamHandler handler = handlers.get(protocol);
    if (handler == null) {
        handler = createBuiltinHandler(protocol);
    }
    if (handler != null) {
       handlers.put(protocol, handler);
    }
}
```
- createBuiltinHandler

```java
private static URLStreamHandler createBuiltinHandler(String protocol)
        throws ClassNotFoundException, InstantiationException, IllegalAccessException {
    URLStreamHandler handler = null;
    if (protocol.equals("file")) {
        handler = new sun.net.www.protocol.file.Handler();
    } else if (protocol.equals("ftp")) {
        handler = new sun.net.www.protocol.ftp.Handler();
    } else if (protocol.equals("jar")) {
        handler = new sun.net.www.protocol.jar.Handler();
    } else if (protocol.equals("http")) {
        handler = (URLStreamHandler)Class.
            forName("com.android.okhttp.HttpHandler").newInstance();
    } else if (protocol.equals("https")) {
        handler = (URLStreamHandler)Class.
               forName("com.android.okhttp.HttpsHandler").newInstance();
    }
    return handler;
}
```
这里可以知道，Http和Https协议调用的是okhttp库，不过Android有进行定制改造。

- parseURL（URLStreamHandler）

URL构造函数最后调用到这个方法，在URLStreamHandler类中处理，因为HttpsHandler，继承自HttpHandler，而HttpHandler继承自URLStreamHandler。

```java
protected void parseURL(URL u, String spec, int start, int limit) {
    setURL(u, protocol, host, port, authority, userInfo, path, query, ref);
}

protected void setURL(URL u, String protocol, String host, int port,String authority, String userInfo, String path,
String query, String ref) {
// ensure that no one can reset the protocol on a given URL.
    u.set(u.getProtocol(), host, port, authority, userInfo, path, query, ref);
}
```
对应设置了协议，host，端口，验证信息，验证信息中的用户信息，文件路径，query数据，引用。

此时URL初始化完毕。

### 1.2 openConnection(HttpHandler)
> 以Https为例追踪代码。

```java
public URLConnection openConnection() throws java.io.IOException {
    return handler.openConnection(this);
}
```
handler对象对应HttpsHandler类，而这个类里面没有这个方法，因为HttpsHandler继承自HttpHandler，在看看HttpHandler中看：

```java
@Override protected URLConnection openConnection(URL url) throws IOException {
    return newOkUrlFactory(null /* proxy */).open(url);
}
```
- newOkUrlFactory

```java
protected OkUrlFactory newOkUrlFactory(Proxy proxy) {
    OkUrlFactory okUrlFactory = createHttpOkUrlFactory(proxy);
    // For HttpURLConnections created through java.net.URL Android uses a connection pool that
    // is aware when the default network changes so that pooled connections are not re-used when
    // the default network changes.
    okUrlFactory.client().setConnectionPool(configAwareConnectionPool.get());
    return okUrlFactory;
}
```
- createHttpOkUrlFactory

```java
public static OkUrlFactory createHttpOkUrlFactory(Proxy proxy) {
    OkHttpClient client = new OkHttpClient();

    // Set the default (same protocol) redirect behavior. The default can be overridden for
    // each instance using HttpURLConnection.setInstanceFollowRedirects().
    client.setFollowRedirects(HttpURLConnection.getFollowRedirects());

    // Do not permit http -> https and https -> http redirects.
    client.setFollowSslRedirects(false);

    // Permit cleartext traffic only (this is a handler for HTTP, not for HTTPS).
    client.setConnectionSpecs(CLEARTEXT_ONLY);

    // OkHttp requires that we explicitly set the response cache.
    OkUrlFactory okUrlFactory = new OkUrlFactory(client);

    // Use the installed NetworkSecurityPolicy to determine which requests are permitted over
    // http.
    OkUrlFactories.setUrlFilter(okUrlFactory, CLEARTEXT_FILTER);

    ResponseCache responseCache = ResponseCache.getDefault();
    if (responseCache != null) {
        AndroidInternal.setResponseCache(okUrlFactory, responseCache);
    }
    return okUrlFactory;
}
```
看到熟悉的OkHttpClient了。

这个方法中主要设置了一些参数，包含连接，读写超时时间，重定向，连接请求加密or明文，设置代理，初始化OkUrlFactory类，用来初始化一个地址管理类，另外还设置了缓存。

- 请求加密 or not

```java
OkUrlFactories.setUrlFilter(okUrlFactory, CLEARTEXT_FILTER);

private static final CleartextURLFilter CLEARTEXT_FILTER = new CleartextURLFilter();

private static final class CleartextURLFilter implements URLFilter {
    @Override
    public void checkURLPermitted(URL url) throws IOException {
        String host = url.getHost();
        if (!NetworkSecurityPolicy.getInstance().isCleartextTrafficPermitted(host)) {
            throw new IOException("Cleartext HTTP traffic to " + host + " not permitted");
        }
    }
}
```
isCleartextTrafficPermitted方法最终的调用是在
ApplicationConfig类中：

```java
public boolean isCleartextTrafficPermitted(String hostname) {
    return getConfigForHostname(hostname).isCleartextTrafficPermitted();
}
```

在getConfigForHostname调用过程中，先调用ensureInitialized方法，确保相关变量初始化，我们发现关键的变量是mConfigSource，他是一个ConfigSource接口。

```java
private void ensureInitialized() {
    synchronized(mLock) {
        if (mInitialized) {
            return;
        }
        mConfigs = mConfigSource.getPerDomainConfigs();
        mDefaultConfig = mConfigSource.getDefaultConfig();
        mConfigSource = null;
        mTrustManager = new RootTrustManager(this);
        mInitialized = true;
    }
}
```
这个ConfigSource是在哪里初始化的呢？是在NetworkSecurityPolicy类中初始化的：

```java
public static ApplicationConfig getApplicationConfigForPackage(Context context,
        String packageName) throws PackageManager.NameNotFoundException {
    Context appContext = context.createPackageContext(packageName, 0);
    ManifestConfigSource source = new ManifestConfigSource(appContext);
    return new ApplicationConfig(source);
}
```

看到ManifestConfigSource就放心了，这个跟Android工程中的AndroidManifest.xml应该有点关系，实际上确实有关系。
在这个类中会初始化一个XmlConfigSource类，看到xml就知道要开始解析相关配置了。
这个文件配置来自于AndroidManifest.xml中配置的节点:

```xml
android:networkSecurityConfig="@xml/security_config"
```
PackageParser类源码定义如下：

```java
ai.networkSecurityConfigRes = 
sa.getResourceId(com.android.internal.R.styleable.AndroidManifestApplication_networkSecurityConfig,0);
```
配置完了之后开始解析，在解析之前要先确保初始化，在XmlConfigSource类中：

```java
private void ensureInitialized() {
    parseNetworkSecurityConfig(parser);
}

private void parseNetworkSecurityConfig(XmlResourceParser parser) throws IOException, XmlPullParserException, ParserException {
    XmlUtils.beginDocument(parser, "network-security-config");
    int outerDepth = parser.getDepth();
    while (XmlUtils.nextElementWithin(parser, outerDepth)) {
        if ("base-config".equals(parser.getName())) {
           baseConfigBuilder = parseConfigEntry(parser, seenDomains, null, CONFIG_BASE).get(0).first;
        }else if ("domain-config".equals(parser.getName())) {
            builders.addAll(parseConfigEntry(parser, seenDomains, baseConfigBuilder, CONFIG_DOMAIN));
       } else if ("debug-overrides".equals(parser.getName())) {
        }
    }
}
```
大概知道xml文件怎么配置了：

```java
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="true" />
</network-security-config>
```
因为在parseConfigEntry方法中有代码：

```java
for (int i = 0; i < parser.getAttributeCount(); i++) {
    String name = parser.getAttributeName(i);
    if ("hstsEnforced".equals(name)) {
        builder.setHstsEnforced(parser.getAttributeBooleanValue(i, NetworkSecurityConfig.DEFAULT_HSTS_ENFORCED));
    } else if ("cleartextTrafficPermitted".equals(name)) {
        builder.setCleartextTrafficPermitted(parser.getAttributeBooleanValue(i,NetworkSecurityConfig.DEFAULT_CLEARTEXT_TRAFFIC_PERMITTED));
    }
}
```
上面写了这么多源码追踪核心目的是为了解决怎么明文传输数据。当然如果不设置的话，就不能明文传递，默认的代码是：

```java
boolean usesCleartextTraffic = (mApplicationInfo.flags & ApplicationInfo.FLAG_USES_CLEARTEXT_TRAFFIC) != 0 && mApplicationInfo.targetSandboxVersion < 2;
source = new DefaultConfigSource(usesCleartextTraffic, mApplicationInfo);
```
### 1.3 OkUrlFactory
将刚才初始化的OkHttpClient类封装到OkUrlFactory中，下一步执行open方法。
- open

```java
public HttpURLConnection open(URL url) {
    return open(url, client.getProxy());
}

HttpURLConnection open(URL url, Proxy proxy) {
    String protocol = url.getProtocol();
    OkHttpClient copy = client.copyWithDefaults();
    copy.setProxy(proxy);
    
    if (protocol.equals("http")) return new HttpURLConnectionImpl(url, copy, urlFilter);
    if (protocol.equals("https")) return new HttpsURLConnectionImpl(url, copy, urlFilter);
    throw new IllegalArgumentException("Unexpected protocol: " + protocol);
}
```
在初始化OkHttpClient类的时候，proxy默认为空，然后根据请求地址中的协议头确定不同的类处理请求。
对于http请求使用HttpURLConnectionImpl；对于https请求使用HttpsURLConnectionImpl。

看看继承关系：

HttpURLConnectionImpl：HttpURLConnection：URLConnection：

HttpsURLConnectionImpl：DelegatingHttpsURLConnection(HttpURLConnectionImpl)：HttpsURLConnection：HttpURLConnection：URLConnection

后续以HttpsURLConnectionImpl为例说明。

### 1.4 HttpsURLConnectionImpl
open方法中最重要的一步就是构造一个HttpsURLConnectionImpl类，把我们初始化的url,OkHttpClient,URLFilter对象封装到这个类中，看来这个类是干活的主体。

```java
public HttpsURLConnectionImpl(URL url, OkHttpClient client, URLFilter filter) {
    this(new HttpURLConnectionImpl(url, client, filter));
}
```
- connect

获取到HttpsURLConnection之后，开始连接服务端，好吧，这个方法不在HttpURLConnectionImpl类中定义，只能找找父类了(DelegatingHttpsURLConnection中)：

```java
@Override public void connect() throws IOException {
    connected = true;
    delegate.connect();
}
```
这个delegate对象是HttpURLConnectionImpl类：

```java
public HttpsURLConnectionImpl(URL url, OkHttpClient client, URLFilter filter) {
    this(new HttpURLConnectionImpl(url, client, filter));
}

public HttpsURLConnectionImpl(HttpURLConnectionImpl delegate) {
    super(delegate);
    this.delegate = delegate;
}
```
好了，真正的连接在中HttpURLConnectionImpl类中：

```java
@Override public final void connect() throws IOException {
    initHttpEngine();
    boolean success;
    do {
      success = execute(false);
    } while (!success);
}
```
- initHttpEngine

初始化http引擎：
```java
private void initHttpEngine() throws IOException {
    connected = true;
    try {
      if (doOutput) {
        if (method.equals("GET")) {
          // they are requesting a stream to write to. This implies a POST method
          method = "POST";
        } else if (!HttpMethod.permitsRequestBody(method)) {
          throw new ProtocolException(method + " does not support writing");
        }
      }
      // If the user set content length to zero, we know there will not be a request body.
      httpEngine = newHttpEngine(method, null, null, null);
    } catch (IOException e) {
    }
}
```
一个URL连接既可以做输入也可以做输出操作，当doOutPut设置为true时，意味着将要往连接里面写数据了。如果此时请求方法是GET，将会被强制修改为POST。

```java
private HttpEngine newHttpEngine(String method, StreamAllocation streamAllocation,RetryableSink requestBody, Response priorResponse) throws MalformedURLException, UnknownHostException {
    URL url = getURL();
    HttpUrl httpUrl = Internal.instance.getHttpUrlChecked(url.toString());
    Request.Builder builder = new Request.Builder()
        .url(httpUrl)
        .method(method, placeholderBody);
    Headers headers = requestHeaders.build();
    for (int i = 0, size = headers.size(); i < size; i++) {
      builder.addHeader(headers.name(i), headers.value(i));
    }
    Request request = builder.build();

    // If we're currently not using caches, make sure the engine's client doesn't have one.
    OkHttpClient engineClient = client;
    return new HttpEngine(engineClient, request, bufferRequestBody, true, false, streamAllocation,
        requestBody, priorResponse);
}
```
只截取比较重要的代码片段，将URL封装到HttpUrl，创建Request对象，同时将Header数据写入，然后构建一个HttpEngine对象返回。在构建的时候如果StreamAllocation对象为空，就初始化：

```java
this.streamAllocation = streamAllocation != null
    ? streamAllocation
    : new StreamAllocation(client.getConnectionPool(), createAddress(client, request));
```
streamAllocation初始化的地方之前已经分析过。

- execute

开始执行请求，流程相当于RetryAndFollowUpInterceptor和CacheInterceptor。

```java
private boolean execute(boolean readResponse) throws IOException {
    boolean releaseConnection = true;
    if (urlFilter != null) {
      urlFilter.checkURLPermitted(httpEngine.getRequest().url());
    }
    try {
      httpEngine.sendRequest();
      if (readResponse) {
        httpEngine.readResponse();
      }
      releaseConnection = false;

      return true;
    } catch (RequestException e) {
    } catch (RouteException e) {
      // The attempt to connect via a route failed. The request will not have been sent.
      HttpEngine retryEngine = httpEngine.recover(e);
      if (retryEngine != null) {
        releaseConnection = false;
        httpEngine = retryEngine;
        return false;
      }
    } catch (IOException e) {
      // An attempt to communicate with a server failed. The request may have been sent.
      HttpEngine retryEngine = httpEngine.recover(e);
      if (retryEngine != null) {
        releaseConnection = false;
        httpEngine = retryEngine;
        return false;
      }
    } finally {
      // We're throwing an unchecked exception. Release any resources.
      if (releaseConnection) {
        StreamAllocation streamAllocation = httpEngine.close();
        streamAllocation.release();
      }
    }
  }
```
- checkURLPermitted

用于检查当前请求是否可以明文传递，这个方法在HttpHandler里面有初始化。

- sendRequest

开始在HttpEngine类里面发送请求了

### 1.5 HttpEngine
从sendRequest方法开始,集合了BridgeInterceptor和ConnectInterceptor.

- sendRequest

```java
public void sendRequest() throws RequestException, RouteException, IOException {
    Request request = networkRequest(userRequest);
    InternalCache responseCache = Internal.instance.internalCache(client);
    Response cacheCandidate = responseCache != null
        ? responseCache.get(request)
        : null;

    long now = System.currentTimeMillis();
    cacheStrategy = new CacheStrategy.Factory(now, request, cacheCandidate).get();
    networkRequest = cacheStrategy.networkRequest;
    cacheResponse = cacheStrategy.cacheResponse;

    if (networkRequest != null) {
      httpStream = connect();
      httpStream.setHttpEngine(this);
      if (callerWritesRequestBody && permitsRequestBody(networkRequest) && requestBodyOut == null) {
        long contentLength = OkHeaders.contentLength(request);
        if (bufferRequestBody) {
          if (contentLength != -1) {
            // Buffer a request body of a known length.
            httpStream.writeRequestHeaders(networkRequest);
            requestBodyOut = new RetryableSink((int) contentLength);
          } else {
            // Buffer a request body of an unknown length. Don't write request
            // headers until the entire body is ready; otherwise we can't set the
            // Content-Length header correctly.
            requestBodyOut = new RetryableSink();
          }
        } else {
          httpStream.writeRequestHeaders(networkRequest);
          requestBodyOut = httpStream.createRequestBody(networkRequest, contentLength);
        }
      }
    } else {
      if (cacheResponse != null) {
        // We have a valid cached response. Promote it to the user response immediately.
        this.userResponse = cacheResponse.newBuilder()
            .request(userRequest)
            .priorResponse(stripBody(priorResponse))
            .cacheResponse(stripBody(cacheResponse))
            .build();
      } else {
        // We're forbidden from using the network, and the cache is insufficient.
        this.userResponse = new Response.Builder()
            .request(userRequest)
            .priorResponse(stripBody(priorResponse))
            .protocol(Protocol.HTTP_1_1)
            .code(504)
            .message("Unsatisfiable Request (only-if-cached)")
            .body(EMPTY_BODY)
            .build();
      }

      userResponse = unzip(userResponse);
    }
  }
```
这个方法比较重要，下面详细解读。

- networkRequest

填充一些用户请求的时候可能没有初始化的请求字段：
Host，Connection，Accept-Encoding，Cookie，Cookie2，User-Agent

- connect

开始连接，并返回基于协议版本的HttpStream，这个方法调用的前提是缓存无效或不允许缓存的情况下开始连接请求，否则的话，如果缓存有效就使用缓存。

```java
httpStream = connect();
//HttpEngine
private HttpStream connect() throws RouteException, RequestException, IOException {
    boolean doExtensiveHealthChecks = !networkRequest.method().equals("GET");
    return streamAllocation.newStream(client.getConnectTimeout(),client.getReadTimeout(), client.getWriteTimeout(),
        client.getRetryOnConnectionFailure(), doExtensiveHealthChecks);
}
```

- StreamAllocation

```java
public HttpStream newStream(int connectTimeout, int readTimeout, int writeTimeout,
      boolean connectionRetryEnabled, boolean doExtensiveHealthChecks)
      throws RouteException, IOException {
    try {
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, connectionRetryEnabled, doExtensiveHealthChecks);

      HttpStream resultStream;
      if (resultConnection.framedConnection != null) {
        resultStream = new Http2xStream(this, resultConnection.framedConnection);
      } else {
        resultStream = new Http1xStream(this, resultConnection.source, resultConnection.sink);
      }

      synchronized (connectionPool) {
        resultConnection.streamCount++;
        stream = resultStream;
        return resultStream;
      }
    } catch (IOException e) {
      throw new RouteException(e);
    }
  }
  
private RealConnection findHealthyConnection(
    while (true) {
        RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout, connectionRetryEnabled);
        /* Returns true if this connection is ready to host new streams. */
        if (candidate.isHealthy(doExtensiveHealthChecks)) {
            return candidate;
        }
    }
}

private RealConnection findConnection(){
    RealConnection newConnection = new RealConnection(route);
    acquire(newConnection);
    newConnection.connect(connectTimeout, readTimeout, writeTimeout, address.getConnectionSpecs(),connectionRetryEnabled);
    routeDatabase().connected(newConnection.getRoute());
    return newConnection;
}
```

- RealConnection

```java
public void connect(){
    while (protocol == null) {
        try {
            rawSocket = proxy.type() == Proxy.Type.DIRECT || proxy.type() == Proxy.Type.HTTP
                ? address.getSocketFactory().createSocket()
                : new Socket(proxy);
            connectSocket(connectTimeout, readTimeout, writeTimeout, connectionSpecSelector);
      } catch (IOException e) {
      }
  }
}

private void connectSocket(){
    Platform.get().connectSocket(rawSocket, route.getSocketAddress(), connectTimeout);
    source = Okio.buffer(Okio.source(rawSocket));
    sink = Okio.buffer(Okio.sink(rawSocket));
    if (route.getAddress().getSslSocketFactory() != null) {
      connectTls(readTimeout, writeTimeout, connectionSpecSelector);
    } else {
      protocol = Protocol.HTTP_1_1;
      socket = rawSocket;
    }
    if (protocol == Protocol.SPDY_3 || protocol == Protocol.HTTP_2) {
      socket.setSoTimeout(0); // Framed connection timeouts are set per-stream.
      FramedConnection framedConnection = new FramedConnection.Builder(true)
          .socket(socket, route.getAddress().url().host(), source, sink)
          .protocol(protocol)
          .build();
      framedConnection.sendConnectionPreface();
      // Only assign the framed connection once the preface has been sent successfully.
      this.framedConnection = framedConnection;
    }
}
```
看到SPDY了，SPDY是Http2的前身，是google定义的一套协议，HTTP/2的关键功能主要来自SPDY技术，换言之，SPDY的成果被采纳而最终演变为HTTP/2。

- FramedConnection

构造函数

```java
private FramedConnection(Builder builder){
    if (protocol == Protocol.HTTP_2) {
      variant = new Http2();
      // Like newSingleThreadExecutor, except lazy creates the thread.
      pushExecutor = new ThreadPoolExecutor(0, 1, 60, TimeUnit.SECONDS,
          new LinkedBlockingQueue<Runnable>(),
          Util.threadFactory(String.format("OkHttp %s Push Observer", hostName), true));
    } else if (protocol == Protocol.SPDY_3) {
      variant = new Spdy3();
    } else {
      throw new AssertionError(protocol);
    }
    socket = builder.socket;
    frameWriter = variant.newWriter(builder.sink, client);
    readerRunnable = new Reader(variant.newReader(builder.source, client));
    new Thread(readerRunnable).start(); //
}
```
读取线程定义在variant对象中，这个对象分属两个类：

- Http2

```java
@Override public FrameReader newReader(BufferedSource source, boolean client) {
    return new Reader(source, 4096, client);
}

static final class Reader implements FrameReader {
    final Hpack.Reader hpackReader;
    Reader(BufferedSource source, int headerTableSize, boolean client) {
}

@Override public void readConnectionPreface() throws IOException {
  ByteString connectionPreface = source.readByteString(CONNECTION_PREFACE.size());
}

@Override public boolean nextFrame(Handler handler) throws IOException {
  try {
    source.require(9); // Frame header size
  } catch (IOException e) {
    return false; // This might be a normal socket close.
  }
  byte type = (byte) (source.readByte() & 0xff);
  switch (type) {
    case TYPE_DATA:
      readData(handler, length, flags, streamId);
      break;
    case TYPE_HEADERS:
      readHeaders(handler, length, flags, streamId);
      break;
    case TYPE_PRIORITY:
      readPriority(handler, length, flags, streamId);
      break;
    case TYPE_RST_STREAM:
      readRstStream(handler, length, flags, streamId);
      break;
    case TYPE_SETTINGS:
      readSettings(handler, length, flags, streamId);
      break;
    case TYPE_PUSH_PROMISE:
      readPushPromise(handler, length, flags, streamId);
      break;
    case TYPE_PING:
      readPing(handler, length, flags, streamId);
      break;
    case TYPE_GOAWAY:
      readGoAway(handler, length, flags, streamId);
      break;
    case TYPE_WINDOW_UPDATE:
      readWindowUpdate(handler, length, flags, streamId);
      break;
    default:
      // Implementations MUST discard frames that have unknown or unsupported types.
      source.skip(length);
  }
  return true;
}
```
- Spdy3

```java
static final class Reader implements FrameReader {
    Reader(BufferedSource source, boolean client) {
      this.source = source;
      this.headerBlockReader = new NameValueBlockReader(this.source);
      this.client = client;
    }

    @Override public void readConnectionPreface() {
    }

    /**
     * Send the next frame to {@code handler}. Returns true unless there are no
     * more frames on the stream.
     */
    @Override public boolean nextFrame(Handler handler) throws IOException {
      int w1;
      try {
        w1 = source.readInt();
      } catch (IOException e) {
        return false; // This might be a normal socket close.
      }

      boolean control = (w1 & 0x80000000) != 0;
      if (control) {
        int type = (w1 & 0xffff);
        switch (type) {
          case TYPE_SYN_STREAM:
            readSynStream(handler, flags, length);
            return true;

          case TYPE_SYN_REPLY:
            readSynReply(handler, flags, length);
            return true;

          case TYPE_RST_STREAM:
            readRstStream(handler, flags, length);
            return true;

          case TYPE_SETTINGS:
            readSettings(handler, flags, length);
            return true;

          case TYPE_PING:
            readPing(handler, flags, length);
            return true;

          case TYPE_GOAWAY:
            readGoAway(handler, flags, length);
            return true;

          case TYPE_HEADERS:
            readHeaders(handler, flags, length);
            return true;

          case TYPE_WINDOW_UPDATE:
            readWindowUpdate(handler, flags, length);
            return true;

          default:
            source.skip(length);
            return true;
        }
      } else {
        int streamId = w1 & 0x7fffffff;
        boolean inFinished = (flags & FLAG_FIN) != 0;
        handler.data(inFinished, streamId, source, length);
        return true;
      }
    }
}
```

后边就是启动读取线程了，这个Reader是个内部类，在FramedConnection类中，

```java
class Reader extends NamedRunnable implements FrameReader.Handler {
    final FrameReader frameReader;

    private Reader(FrameReader frameReader) {
      super("OkHttp %s", hostName);
      this.frameReader = frameReader;
    }

    @Override protected void execute() {
      try {
        if (!client) {
          frameReader.readConnectionPreface();
        }
        while (frameReader.nextFrame(this)) {
        }
      } catch (IOException e) {
      } finally {
      
      }
    }
}
```
同样的，对于服务端要尝试读取前言数据，对于客户端需要通过nextFrame调用子类Http2和SPDY3的nextFrame方法读取数据。

- 读取数据后回调

数据读取完毕之后，从子类Http2和SPDY3中通过FrameReader.Handler类回调回来，到FramedConnection的Reader类中处理数据，因为Reader是实现了FrameReader.Handler接口。

```java
 @Override public void data(){
    dataStream.receiveData(source, length);
 }
 @Override public void headers(){
    stream.receiveHeaders(headerBlock, headersMode);
 }
```

从HttpEngine的sendRequest和connect穿越回来，确认了HttpStream的子类，有两个，Http2xStream封装了SPDY3和Http2的请求逻辑，Http1xStream封装了Http1的请求逻辑。继续回到接下来的流程。

### 1.6 回到HttpEngine的sendRequest方法。

#### 1.6.1 sendRequest写请求body

对于请求类中包含请求body数据的，就开始写入请求头和请求body

```java
//Http2xStream
@Override public void writeRequestHeaders(Request request) throws IOException {
    List<Header> requestHeaders = framedConnection.getProtocol() == Protocol.HTTP_2
        ? http2HeadersList(request)
        : spdy3HeadersList(request);
    stream = framedConnection.newStream(requestHeaders, permitsRequestBody, hasResponseBody);
 }
//Http1xStream
@Override public void writeRequestHeaders(Request request) throws IOException {
    httpEngine.writingRequestHeaders();
    String requestLine = RequestLine.get(
        request, httpEngine.getConnection().getRoute().getProxy().type());
    writeRequest(request.headers(), requestLine);
}
```
#### 1.6.2 不需要请求连接（connect）但有缓存
这种情况下，获取缓存的响应数据，并封装到Response，然后unzip解压。

到此HttpEngine的sendRequest和connect就完成了。主要完成了http不同协议对应不同的处理类（Http2，SPDY3），不同的处理类封装到不同的流操作类（Http2xStream，Http1xStream），这个流操作类处理了数据的读写。

### 1.7 HttpURLConnectionImpl
回到HttpURLConnectionImpl方法，上面走的请求连接流程来自于HttpURLConnectionImpl的connect函数：

```java
@Override public final void connect() throws IOException {
    initHttpEngine();
    boolean success;
    do {
      success = execute(false);//readResponse？
    } while (!success);
}
```
此时不需要读取Response，尽管可能我们已经读取到了响应数据，但是也可能存在异常情况需要重试。

在execute执行过程中，如果出现异常，需要重新初始化一个HttpEngine对象，用于后续重试：

```java
private boolean execute(boolean readResponse) throws IOException {
try {
      httpEngine.sendRequest();
      return true;
    } catch (RequestException e) {
      throw toThrow;
    } catch (RouteException e) {
      // The attempt to connect via a route failed. The request will not have been sent.
      HttpEngine retryEngine = httpEngine.recover(e);
      if (retryEngine != null) {
        httpEngine = retryEngine;
        return false;
      }
      throw toThrow;
    } catch (IOException e) {
      // An attempt to communicate with a server failed. The request may have been sent.
      HttpEngine retryEngine = httpEngine.recover(e);
      if (retryEngine != null) {
        httpEngine = retryEngine;
        return false;
      }
      throw e;
    } finally {
      // We're throwing an unchecked exception. Release any resources.
      if (releaseConnection) {
        StreamAllocation streamAllocation = httpEngine.close();
        streamAllocation.release();
      }
    }
}
```
如果当前路由请求失败，且存在其他路由可以选择，就是用其他路由重新请求。recover函数的核心工作就是去判断有没有其他的路由。

如果请求成功返回true，connect函数的工作就完成了。

### 1.8 ResponseCode

- 读取请求返回码

```kotlin
//getResponseCode
val responseCode = connection.responseCode
```
- HttpURLConnection getResponseCode

因为HttpsURLConnectionImpl的代理delegate类设置的是HttpURLConnectionImpl，所以实际调用在该类中：

```java
@Override public final int getResponseCode() throws IOException {
    return getResponse().getResponse().code();
}
```

先获取Response类：

```java
private HttpEngine getResponse() throws IOException {
    initHttpEngine();

    if (httpEngine.hasResponse()) {
      return httpEngine;
    }

    while (true) {
      if (!execute(true)) {
        continue;
      }

      Response response = httpEngine.getResponse();
      Request followUp = httpEngine.followUpRequest();

      if (followUp == null) {
        httpEngine.releaseStreamAllocation();
        return httpEngine;
      }

      if (++followUpCount > HttpEngine.MAX_FOLLOW_UPS) {
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }
      ....//省略重试逻辑判断

      httpEngine = newHttpEngine(followUp.method(), streamAllocation, (RetryableSink) requestBody,
          response);
    }
}
```
上面这一段包含了两个逻辑，继续执行execute，没有连接服务器的话，就继续连接，并读取返回的数据封装成Response，然后接下来判断是否需要重试，也就是RetryAndFollowUpInterceptor的工作。

execute方法的参数是是否需要读取readResponse，此处是true：

```java
private boolean execute(boolean readResponse) throws IOException {
    try {
      httpEngine.sendRequest();
      if (readResponse) {
        httpEngine.readResponse();
      }
      return true;
    } catch (RequestException e) {
    }
}
```
在connect方法中，已经执行过了execute(false)方法了，在sendRequest方法中已经初始化了CacheStrategy对象，这里再次执行到这里的时候直接返回了。

下一步就是直接读取readResponse了，因为connect方法的时候，这些数据都已经存储在source对象里面了。

```java
//HttpEngine
public void readResponse() throws IOException {
    networkResponse = readNetworkResponse();
    userResponse = networkResponse.newBuilder()
    .request(userRequest)
    .priorResponse(stripBody(priorResponse))
    .cacheResponse(stripBody(cacheResponse))
    .networkResponse(stripBody(networkResponse))
    .build();

    if (hasBody(userResponse)) {
      maybeCache();
      userResponse = unzip(cacheWritingResponse(storeRequest, userResponse));
    }
}
```
- readNetworkResponse

到了这一步把headers和body封装到Response类中：

```java
//HttpEngine
private Response readNetworkResponse() throws IOException {
    httpStream.finishRequest();

    Response networkResponse = httpStream.readResponseHeaders()
        .request(networkRequest)
        .handshake(streamAllocation.connection().getHandshake())
        .header(OkHeaders.SENT_MILLIS, Long.toString(sentRequestMillis))
        .header(OkHeaders.RECEIVED_MILLIS, Long.toString(System.currentTimeMillis()))
        .build();

    if (!forWebSocket) {
      networkResponse = networkResponse.newBuilder()
          .body(httpStream.openResponseBody(networkResponse))
          .build();
    }

    if ("close".equalsIgnoreCase(networkResponse.request().header("Connection"))
        || "close".equalsIgnoreCase(networkResponse.header("Connection"))) {
      streamAllocation.noNewStreams();
    }

    return networkResponse;
}
```

- 1.一开始，将sink写数据通道关闭，预示着要开始读数据了；

- 2.httpStream对应我们根据不同协议初始化的Http1xStream和Http2xStream类，在不同的类中分别读取headers和body，这个不再赘述;

- 3.最后判断如果客户端或服务端关闭了连接，就将数据流的标记置为true，表示没有新的数据了。

在readResponse方法的最后，对于有响应body返回的，如果要缓存的话，就缓存，同时把source的数据解压，因为sdk里面如果客户端不设置传输方式的话，默认设置了压缩传输。

接下来就是顺理成章的获取userResponse.code()方法返回响应码了。

### 1.9 getInputStream
获取返回的数据。

```java
//HttpURLConnectionImpl
 @Override public final InputStream getInputStream() throws IOException {
    HttpEngine response = getResponse();

    // if the requested file does not exist, throw an exception formerly the
    // Error page from the server was returned if the requested file was
    // text/html this has changed to return FileNotFoundException for all
    // file types
    if (getResponseCode() >= HTTP_BAD_REQUEST) {
      throw new FileNotFoundException(url.toString());
    }

    return response.getResponse().body().byteStream();
 }
```
body数据原本是
在HttpEngine类的unzip方法中，可以看到responseBody被封装到RealResponseBody对象中。

```java
return response.newBuilder()
        .headers(strippedHeaders)
        .body(new RealResponseBody(strippedHeaders, Okio.buffer(responseBody)))
        .build();
```
RealResponseBody继承自ResponseBody，看看byteStream的逻辑。

- ResponseBody

```java
public final InputStream byteStream() throws IOException {
    return source().inputStream();
}

public abstract BufferedSource source() throws IOException;
```

这个source的来源是Okio.buffer(responseBody)，

```java
public static BufferedSource buffer(Source source) {
    if (source == null) throw new IllegalArgumentException("source == null");
    return new RealBufferedSource(source);
}
```
RealBufferedSource中inputStream方法是：

```java
@Override public InputStream inputStream() {
    return new InputStream() {
      @Override public int read() throws IOException {
        return buffer.readByte() & 0xff;
      }

      @Override public int read(byte[] data, int offset, int byteCount) throws IOException {
        return buffer.read(data, offset, byteCount);
      }

      @Override public int available() throws IOException {
        return (int) Math.min(buffer.size, Integer.MAX_VALUE);
      }

      @Override public void close() throws IOException {
        RealBufferedSource.this.close();
      }

      @Override public String toString() {
        return RealBufferedSource.this + ".inputStream()";
      }
    };
  }
```
封装了读的方法，便于我们接下来读取数据：

```java
var n = inputStream.read(buffer)
```
至此分析完毕。

OKHttp系列流程图地址：

- OKHttp-Android源码
https://i.loli.net/2019/05/23/5ce609bcea39b44613.jpg

- OKHttp-总览
https://i.loli.net/2019/05/23/5ce60c6b33b1f90332.jpg
https://i.loli.net/2019/05/23/5ce60c6b3151011143.jpg
https://i.loli.net/2019/05/23/5ce60c6bd889c63055.jpg

- OKHttp-RetryAndFollowUpInterceptor
https://i.loli.net/2019/05/23/5ce60c6b635ac99646.jpg

- OKHttp-CacheInterceptor
https://i.loli.net/2019/05/23/5ce60c6c0127f22737.jpg

- OKHttp-BridgeInterceptor
https://i.loli.net/2019/05/23/5ce60c6b7b37c15134.jpg

- OKHttp-ConnectInterceptor
https://i.loli.net/2019/05/23/5ce60c6c5b7f019352.jpg

- OKHttp-CallServerInterceptor
https://i.loli.net/2019/05/23/5ce60c6c03eb171712.jpg
https://i.loli.net/2019/05/23/5ce60c6bf2ba942282.jpg

微信公众号二维码：
![](https://ftp.bmp.ovh/imgs/2020/04/f472fa7e9c62a8c7.jpg)

