---
layout: post
title:  "Android OKHttp系列4-CacheInterceptor"
date:   2019-5-15 11:09:22 +0800
categories: Android
---


> 文章将会被同步至微信公众号：Android部落格
# 1、概述
> 处理缓存策略

流程图如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200331172320549.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTExOTg3NjQ=,size_16,color_FFFFFF,t_70#pic_center)

## 1.1 初始化Cache对象

在最开始初始化各个拦截器的时候，到CacheInterceptor对象创建的时候初始化了Cache。
```java
getResponseWithInterceptorChain();
interceptors.add(new CacheInterceptor(client.internalCache()));
```
在开始请求的时候需要初始化Cache，Cache的初始化如下：

```java
public Cache(File directory, long maxSize) {
    this(directory, maxSize, FileSystem.SYSTEM);
}

Cache(File directory, long maxSize, FileSystem fileSystem) {
    this.cache = DiskLruCache.create(fileSystem, directory, VERSION, ENTRY_COUNT, maxSize);
}
```
调用的时候只需要指定缓存目录和最大缓存空间即可。
此处缓存策略使用磁盘缓存DiskLruCache，与原生类不同的地方在于：
- sdk采用读Okio.source,写Okio.sink，原生使用StrictLineReader读，BufferedWriter写
- sdk在构造函数创建线程池，原生在全局创建，都是用于删除Entry

```java
//原生
private final ExecutorService executorService = new ThreadPoolExecutor(0, 1,
            60L, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>());
//okhttp
Executor executor = new ThreadPoolExecutor(0, 1, 60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<Runnable>(), Util.threadFactory("OkHttp DiskLruCache", true));
//okhttp创建了一个daemon线程
public static ThreadFactory threadFactory(final String name, final boolean daemon) {
    return new ThreadFactory() {
      @Override public Thread newThread(Runnable runnable) {
        Thread result = new Thread(runnable, name);
        result.setDaemon(daemon);
        return result;
      }
    };
}
```
- okhttp在DiskLruCache的内部函数Entry中新加了两个属性

```
final File[] cleanFiles;//表示可以读Entry列表
final File[] dirtyFiles;//被创建或被更新Entry列表
```
> DIRTY 行意味着缓存被创建或被更新。每一个成功写入DIRTY状态行后续一定跟着一个CLEAN或REMOVE状态，代表这条缓存可读或已被删除。DIRTY状态后续没有CLEAN或REMOVE状态意味着临时文件需要被删除。
>
> CLEAN 行意味着一个缓存已经被成功发布并且可以从缓存里面访问了。一个发布状态的行后续跟着当前缓存的大小，以空格分割，如果一个key对应多个value，就有多个数值了。
>
> READ 行意味着最近被访问了。
>
> REMOVE 行意味着当前缓存被删除了。

## 1.2 Cache策略

get和put方法体现了缓存的思想。

### 1.2.1 get

```java
public static String key(HttpUrl url) {
    return ByteString.encodeUtf8(url.toString()).md5().hex();
  }

  @Nullable Response get(Request request) {
    String key = key(request.url());
    DiskLruCache.Snapshot snapshot;
    Entry entry;
    try {
      snapshot = cache.get(key);
      if (snapshot == null) {
        return null;
      }
    } catch (IOException e) {
      // Give up because the cache cannot be read.
      return null;
    }

    try {
      entry = new Entry(snapshot.getSource(ENTRY_METADATA));
    } catch (IOException e) {
      Util.closeQuietly(snapshot);
      return null;
    }

    Response response = entry.response(snapshot);

    if (!entry.matches(request, response)) {
      Util.closeQuietly(response.body());
      return null;
    }

    return response;
  }
```

先看看key是什么，key是HttpUrl.Builder.toString以UTF-8格式进行md5加密之后转16进制。
然后从Cache里面获取一个Snapshot对象，这是真正读取数据的地方(从cleanFiles读取)，封装的是针对Entry的快照，构造函数如下：

```java
Snapshot(String key, long sequenceNumber, Source[] sources, long[] lengths) {
  this.key = key;
  this.sequenceNumber = sequenceNumber;
  this.sources = sources;
  this.lengths = lengths;
}
```
包含缓存key，序列号，未buffer数据，数据长度。

Entry是条目的意思，每一个缓存数据为一个条目，数据被封装在Enrty里面，接下来调用`entry = new Entry(snapshot.getSource(ENTRY_METADATA));`将未buffer的数据解析并封装到Entry对象，之后就可以将Entry封装到Response返回了：

```java
public Response response(DiskLruCache.Snapshot snapshot) {
  String contentType = responseHeaders.get("Content-Type");
  String contentLength = responseHeaders.get("Content-Length");
  Request cacheRequest = new Request.Builder()
      .url(url)
      .method(requestMethod, null)
      .headers(varyHeaders)
      .build();
  return new Response.Builder()
      .request(cacheRequest)
      .protocol(protocol)
      .code(code)
      .message(message)
      .headers(responseHeaders)
      .body(new CacheResponseBody(snapshot, contentType, contentLength))
      .handshake(handshake)
      .sentRequestAtMillis(sentRequestMillis)
      .receivedResponseAtMillis(receivedResponseMillis)
      .build();
}
```
另外每次读数据完毕之后写一个READ到journal文件。

### 1.2.2 put
put方法代码不多，但是涉及到DiskLruCache很多逻辑：
```java
Entry entry = new Entry(response);
DiskLruCache.Editor editor = null;
try {
  editor = cache.edit(key(response.request().url()));
  if (editor == null) {
    return null;
  }
  entry.writeTo(editor);
  return new CacheRequestImpl(editor);
} catch (IOException e) {
  abortQuietly(editor);
  return null;
}
```
- Cache.edit()
先做初始化操作，如果journalFile存在的话，先将数据读取出来，在读取过程中判断lruEntries（LinkedHashMap<String, Entry>）对象中是否存在当前url的Entry数据，不存在的话，将Entry数据存入;journalFile文件不存在，就先创建journalFileTmp文件，再写入数据，然后将journalFileTmp修改为journalFile。

```java
Entry entry = lruEntries.get(key);
if (entry == null) {
  entry = new Entry(key);
  lruEntries.put(key, entry);
}
```
写入的数据是什么呢？是一个DIRTY跟key（url）。

```java
journalWriter.writeUtf8(DIRTY).writeByte(' ').writeUtf8(key).writeByte('\n');
```
整个edit方法的核心目的是创建一个Entry，将Entry放到lruEntries中，并返回一个Editor对象，Editor封装了entry。

- entry.writeTo(editor)

数据部分要开始写了，

```java
public void writeTo(DiskLruCache.Editor editor) throws IOException {
  BufferedSink sink = Okio.buffer(editor.newSink(ENTRY_METADATA));
  sink.writeUtf8(url);
  sink.close();
}
```
`editor.newSink`方法返回一个写对象sink，sink对应一个dirtyFile文件写入，如下：
```java
File dirtyFile = entry.dirtyFiles[index];
Sink sink;
try {
  sink = fileSystem.sink(dirtyFile);
} catch (FileNotFoundException e) {
  return Okio.blackhole();
}
```
可见数据是写在dirtyFile文件中。

## 1.3 Http的缓存
第一步是结合Request和已经缓存的Response制定`CacheStrategy`。

如果缓存的`cacheResponse`对象存在的话，需要重点关注服务器返回的几个Header字段：

- Date 是一个通用首部，其中包含了报文创建的日期和时间
- Expires 响应头包含日期/时间， 即在此时候之后，响应过期
- Last-Modified 是一个响应首部，其中包含源头服务器认定的资源做出修改的日期及时间
- ETag HTTP响应头是资源的特定版本的标识符。这可以让缓存更高效，并节省带宽，因为如果内容没有改变，Web服务器不需要发送完整的响应。而如果内容发生了变化，使用ETag有助于防止资源的同时更新相互覆盖（“空中碰撞”）
- Age 消息头里包含消息对象在缓存代理中存贮的时长，以秒为单位

> ETag字段比较有趣(https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=zh-cn)
> - 服务器使用 ETag HTTP 标头传递验证令牌。
> - 验证令牌可实现高效的资源更新检查：资源未发生变化时不会传送任何数据。
>
> 假定在首次提取资源 120 秒后，浏览器又对该资源发起了新的请求。 首先，浏览器会检查本地缓存并找到之前的响应。 遗憾的是，该响应现已过期，浏览器无法使用。 此时，浏览器可以直接发出新的请求并获取新的完整响应。 不过，这样做效率较低，因为如果资源未发生变化，那么下载与缓存中已有的完全相同的信息就毫无道理可言！
> 
> 这正是验证令牌（在 ETag 标头中指定）旨在解决的问题。 服务器生成并返回的随机令牌通常是文件内容的哈希值或某个其他指纹。 客户端不需要了解指纹是如何生成的，只需在下一次请求时将其发送至服务器。 如果指纹仍然相同，则表示资源未发生变化，您就可以跳过下载。
> ![在这里插入图片描述](https://ftp.bmp.ovh/imgs/2020/05/23c6647c56611493.png)
>
> 在上例中，客户端自动在“If-None-Match” HTTP 请求标头内提供 ETag 令牌。 服务器根据当前资源核对令牌。 如果它未发生变化，服务器将返回“304 Not Modified”响应，告知浏览器缓存中的响应未发生变化，可以再延用 120 秒。 请注意，您不必再次下载响应，这节约了时间和带宽。
>
>
> 作为网络开发者，您如何利用高效的重新验证？浏览器会替我们完成所有工作： 它会自动检测之前是否指定了验证令牌，它会将验证令牌追加到发出的请求上，并且它会根据从服务器接收的响应在必要时更新缓存时间戳。 我们唯一要做的就是确保服务器提供必要的 ETag 令牌。 检查您的服务器文档中有无必要的配置标记。

看看sdk中是怎么处理的：

先获取缓存中的上述列出的字段值：

```java
Headers headers = cacheResponse.headers();
for (int i = 0, size = headers.size(); i < size; i++) {
  String fieldName = headers.name(i);
  String value = headers.value(i);
  if ("Date".equalsIgnoreCase(fieldName)) {
    servedDate = HttpDate.parse(value);
    servedDateString = value;
  } else if ("Expires".equalsIgnoreCase(fieldName)) {
    expires = HttpDate.parse(value);
  } else if ("Last-Modified".equalsIgnoreCase(fieldName)) {
    lastModified = HttpDate.parse(value);
    lastModifiedString = value;
  } else if ("ETag".equalsIgnoreCase(fieldName)) {
    etag = value;
  } else if ("Age".equalsIgnoreCase(fieldName)) {
    ageSeconds = HttpHeaders.parseSeconds(value, -1);
  }
}
```
然后对应的处理是：

```java
String conditionName;
String conditionValue;
if (etag != null) {
    conditionName = "If-None-Match";
    conditionValue = etag;
} else if (lastModified != null) {
    conditionName = "If-Modified-Since";
    conditionValue = lastModifiedString;
} else if (servedDate != null) {
    conditionName = "If-Modified-Since";
    conditionValue = servedDateString;
} else {
    return new CacheStrategy(request, null); // No condition! Make a regular request.
}

Headers.Builder conditionalRequestHeaders = request.headers().newBuilder();
Internal.instance.addLenient(conditionalRequestHeaders, conditionName, conditionValue);

Request conditionalRequest = request.newBuilder()
  .headers(conditionalRequestHeaders.build())
  .build();
return new CacheStrategy(conditionalRequest, cacheResponse);
```

请求字段中加入`If-None-Match`，然后填入etag发送给服务器，用于判断数据是否发生改变了，如果服务器返回“304 Not Modified”，那就可以取缓存的数据了，节省了资源。

- If-Modified-Since
是一个条件式请求首部，服务器只在所请求的资源在给定的日期时间之后对内容进行过修改的情况下才会将资源返回，状态码为 200  。如果请求的资源从那时起未经修改，那么返回一个不带有消息主体的  304  响应，而在 Last-Modified 首部中会带有上次修改时间。 不同于  If-Unmodified-Since, If-Modified-Since 只可以用在 GET 或 HEAD 请求中。当与 If-None-Match 一同出现时，它（If-Modified-Since）会被忽略掉，除非服务器不支持 If-None-Match。

另外针对过期时间和缓存存储时长，会计算当前缓存是否过期来判断是否需要重新请求。

当然Request也有Cache策略，当请求的时候可以设置缓存相关的字段到Header中：
- Cache-Control 通用（Request和Response）消息头字段，被用于在http 请求和响应中，通过指定指令来实现缓存机制。缓存指令是单向的, 这意味着在请求设置的指令，在响应中不一定包含相同的指令。
- Pragma 是一个在 HTTP/1.0 中规定的通用首部，这个首部的效果依赖于不同的实现，所以在“请求-响应”链中可能会有不同的效果。它用来向后兼容只支持 HTTP/1.0 协议的缓存服务器，那时候 HTTP/1.1 协议中的 Cache-Control 还没有出来。
对于这两个字段需要考虑它的值，sdk处理如下：

```java
if ("no-cache".equalsIgnoreCase(directive)) {
  noCache = true;
} else if ("no-store".equalsIgnoreCase(directive)) {
  noStore = true;
} else if ("max-age".equalsIgnoreCase(directive)) {
  maxAgeSeconds = HttpHeaders.parseSeconds(parameter, -1);
} else if ("s-maxage".equalsIgnoreCase(directive)) {
  sMaxAgeSeconds = HttpHeaders.parseSeconds(parameter, -1);
} else if ("private".equalsIgnoreCase(directive)) {
  isPrivate = true;
} else if ("public".equalsIgnoreCase(directive)) {
  isPublic = true;
} else if ("must-revalidate".equalsIgnoreCase(directive)) {
  mustRevalidate = true;
} else if ("max-stale".equalsIgnoreCase(directive)) {
  maxStaleSeconds = HttpHeaders.parseSeconds(parameter, Integer.MAX_VALUE);
} else if ("min-fresh".equalsIgnoreCase(directive)) {
  minFreshSeconds = HttpHeaders.parseSeconds(parameter, -1);
} else if ("only-if-cached".equalsIgnoreCase(directive)) {
  onlyIfCached = true;
} else if ("no-transform".equalsIgnoreCase(directive)) {
  noTransform = true;
} else if ("immutable".equalsIgnoreCase(directive)) {
  immutable = true;
}
```
> 以上各个字段详解见：https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Cache-Control

通过以上字段可以得到Request和Response对象，接下来差异化处理
- 两个对象都没有获取到

```java
return new Response.Builder()
  .request(chain.request())
  .protocol(Protocol.HTTP_1_1)
  .code(504)//网关或者代理服务超时
  .message("Unsatisfiable Request (only-if-cached)")
  .body(Util.EMPTY_RESPONSE)
  .sentRequestAtMillis(-1L)
  .receivedResponseAtMillis(System.currentTimeMillis())
  .build();
```
- Request没获取但是缓存Response存在

```java
if (networkRequest == null) {
  return cacheResponse.newBuilder()
      .cacheResponse(stripBody(cacheResponse))
      .build();
}
```
意味着不需要请求，直接返回缓存的Response。
接下来执行`networkResponse = chain.proceed(networkRequest);`等着其他拦截器干完活返回吧。

## 1.4 处理服务器返回
- 缓存Response不为空，并且服务器返回码显示数据未修改(304 HTTP_NOT_MODIFIED)

```java
if (cacheResponse != null) {
  if (networkResponse.code() == HTTP_NOT_MODIFIED) {
    Response response = cacheResponse.newBuilder()
        .headers(combine(cacheResponse.headers(), networkResponse.headers()))
        .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
        .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();
    networkResponse.body().close();

    // Update the cache after combining headers but before stripping the
    // Content-Encoding header (as performed by initContentStream()).
    cache.trackConditionalCacheHit();
    cache.update(cacheResponse, response);
    return response;
  } else {
    closeQuietly(cacheResponse.body());
  }
}
```
这种情况省心省事省力，直接填充对应的数据，更新Header，将networkResponse和cacheResponse的body剥离掉，更新缓存，然后返回重新构建的Response。

networkResponse和cacheResponse的body数据剥离掉了，后面的数据是不是没有body数据了？并不是，因为networkResponse和cacheResponse只是Response的两个参数，实际上Response有一个ResponseBody属性存储body数据。

- 请求返回的Reponse Header中有body并允许缓存

将最新返回的Response put到缓存，同时将body数据写入
Response，这里提供了read方法。

```java
private Response cacheWritingResponse(final CacheRequest cacheRequest, Response response) throws IOException {
// Some apps return a null body; for compatibility we treat that like a null cache request.
if (cacheRequest == null) return response;
Sink cacheBodyUnbuffered = cacheRequest.body();
if (cacheBodyUnbuffered == null) return response;

final BufferedSource source = response.body().source();
final BufferedSink cacheBody = Okio.buffer(cacheBodyUnbuffered);

Source cacheWritingSource = new Source() {
  boolean cacheRequestClosed;

  @Override public long read(Buffer sink, long byteCount) throws IOException {
    long bytesRead;
    try {
      bytesRead = source.read(sink, byteCount);
    } catch (IOException e) {
      throw e;
    }

    if (bytesRead == -1) {
      if (!cacheRequestClosed) {
        cacheRequestClosed = true;
        cacheBody.close(); // The cache response is complete!
      }
      return -1;
    }

    sink.copyTo(cacheBody.buffer(), sink.size() - bytesRead, bytesRead);
    cacheBody.emitCompleteSegments();
    return bytesRead;
  }

  @Override public Timeout timeout() {
    return source.timeout();
  }

  @Override public void close() throws IOException {

  }
};

String contentType = response.header("Content-Type");
long contentLength = response.body().contentLength();
return response.newBuilder()
    .body(new RealResponseBody(contentType, contentLength, Okio.buffer(cacheWritingSource)))
    .build();
}
```
然后更新缓存，将旧的缓存数据删掉。

微信公众号：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200331173015327.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTExOTg3NjQ=,size_16,color_FFFFFF,t_70#pic_center)