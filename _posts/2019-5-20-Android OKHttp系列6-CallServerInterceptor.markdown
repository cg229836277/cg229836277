---
layout: post
title:  "Android OKHttp系列6-CallServerInterceptor"
date:   2019-5-20 11:09:22 +0800
categories: Android
---

> 文章将会被同步至微信公众号：Android部落格

## 1、概述

> 开始写入request body数据，并读取服务端返回的数据


上一篇文章说到有两个Http协议兼容处理请求，因此有两个流程图，基本是相似的处理流程：
- Http1Codec

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200331153011854.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTExOTg3NjQ=,size_16,color_FFFFFF,t_70#pic_center)

- Http2Codec

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020033115302738.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTExOTg3NjQ=,size_16,color_FFFFFF,t_70#pic_center)

### 1.1 Http1Codec

#### 1.1.1 写请求头

```java
@Override public void writeRequestHeaders(Request request) throws IOException {
    String requestLine = RequestLine.get(
        request, streamAllocation.connection().route().proxy().type());
    writeRequest(request.headers(), requestLine);
}
```

- 请求行（Request Line）

分为三个部分：请求方法、请求地址和协议及版本，以CRLF(rn)结束。

HTTP/1.1 定义的请求方法有8种：GET、POST、PUT、DELETE、PATCH、HEAD、OPTIONS、TRACE,最常的两种GET和POST。

看看这个get方法就知道了：

```java
public static String get(Request request, Proxy.Type proxyType) {
    StringBuilder result = new StringBuilder();
    result.append(request.method());
    result.append(' ');
    
    if (includeAuthorityInRequestLine(request, proxyType)) {
      result.append(request.url());
    } else {
      result.append(requestPath(request.url()));
    }
    
    result.append(" HTTP/1.1");
    return result.toString();
}
```
先写请求方法，在写请求地址信息，对于http请求，直接写地址，对于https请求要写入地址相关参数，然后写入协议版本。

> 例如：https://www.idataapi.cn/?rec=baidu_2&renqun_youhua=752394
>
> 写入的就是：Post www.idataapi.cn/?rec=baidu_2&renqun_youhua=752394 HTTP/1.1

接下来开始写头数据：

```java
public void writeRequest(Headers headers, String requestLine) throws IOException {
    if (state != STATE_IDLE) throw new IllegalStateException("state: " + state);
    sink.writeUtf8(requestLine).writeUtf8("\r\n");
    for (int i = 0, size = headers.size(); i < size; i++) {
      sink.writeUtf8(headers.name(i))
          .writeUtf8(": ")
          .writeUtf8(headers.value(i))
          .writeUtf8("\r\n");
    }
    sink.writeUtf8("\r\n");
    state = STATE_OPEN_REQUEST_BODY;
}
```
看图说话：
![](https://ftp.bmp.ovh/imgs/2020/05/b814c57fb5af3ace.png)

#### 1.1.2 写请求体

如果是非GET和HEAD请求，并且body数据不为空的情况下，就可以写request body数据了：

- 100-continue

通知接收方客户端要发送一个体积可能很大的消息体，期望收到状态码为100 (Continue)  的临时回复。

服务器开始检查请求消息头，可能会返回一个状态码为 100 (Continue) 的回复来告知客户端继续发送消息体，也可能会返回一个状态码为417 (Expectation Failed) 的回复来告知对方要求不能得到满足。

```java
if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
    httpCodec.flushRequest();
    realChain.eventListener().responseHeadersStart(realChain.call());
    responseBuilder = httpCodec.readResponseHeaders(true);
}
```
- 第一步就是刷新请求，将数据写入并发送到服务端；

- 第二步开始读取返回的头数据，看看服务端是否允许继续发送数据。

```java
if (expectContinue && statusLine.code == HTTP_CONTINUE) {
        return null;
} 
```
如果服务端返回100，意味着客户端可以继续发送body数据了：

```java
long contentLength = request.body().contentLength();
CountingSink requestBodyOut =
    new CountingSink(httpCodec.createRequestBody(request, contentLength));
BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);

request.body().writeTo(bufferedRequestBody);
bufferedRequestBody.close();
```
```java
@Override public void finishRequest() throws IOException {
    sink.flush();//数据写入
}
```
#### 1.1.3 读返回数据
- 读返回Headers

正常逻辑下，如果客户端发送100-continue，服务端返回100，此时返回构造Buidler为空，此时就是公共流程（意思是不管有没有请求body）:

```java
if (responseBuilder == null) {
  responseBuilder = httpCodec.readResponseHeaders(false);
}
```
```java
Response.Builder responseBuilder = new Response.Builder()
  .protocol(statusLine.protocol)
  .code(statusLine.code)
  .message(statusLine.message)
  .headers(readHeaders());
```
数据从上一个拦截器（ConnectInterceptor）初始化的source里面读取。

```java
source = Okio.buffer(Okio.source(rawSocket));
sink = Okio.buffer(Okio.sink(rawSocket));
```

- 封装Response

读取Header返回之后封装一个Response对象出来：

```java
Response response = responseBuilder
    .request(request)
    .handshake(streamAllocation.connection().handshake())
    .sentRequestAtMillis(sentRequestMillis)
    .receivedResponseAtMillis(System.currentTimeMillis())
    .build();
```

这里处理了一个例外的情况，就是客户端Header没有100-continue字段，但是服务端返回100，此时sdk仍然把字段里面的值读取出来。

```java
int code = response.code();
if (code == 100) {
  // server sent a 100-continue even though we did not request one.
  // try again to read the actual response
  responseBuilder = httpCodec.readResponseHeaders(false);

  response = responseBuilder
          .request(request)
          .handshake(streamAllocation.connection().handshake())
          .sentRequestAtMillis(sentRequestMillis)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();

  code = response.code();
}
```

接下来处理Response body：

```java
if (forWebSocket && code == 101) {
  // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
  response = response.newBuilder()
      .body(Util.EMPTY_RESPONSE)
      .build();
} else {
  response = response.newBuilder()
      .body(httpCodec.openResponseBody(response))
      .build();
}
```

第一种情况是websocket协议请求并且表示服务器应客户端升级协议的请求（Upgrade请求头）正在进行协议切换。
服务器会发送一个Upgrade响应头来表示其正在切换过去的协议。
此时直接封装一个空的body并构建。

> WebSockets 是一种先进的技术。它可以在用户的浏览器和服务器之间打开交互式通信会话。使用此API，您可以向服务器发送消息并接收事件驱动的响应，而无需通过轮询服务器的方式以获得响应。

第二种就是正常情况了，读取body数据：

- 没有body数据

创建一个`newFixedLengthSource`对象，长度为0，此时释放`Http1Codec`和`RealConnection`对象。

将source对象封装到`RealResponseBody`类

- body数据被分块传输

```java
if ("chunked".equalsIgnoreCase(response.header("Transfer-Encoding")))
```

将请求url封装到`newChunkedSource`对象；
将source对象封装到`RealResponseBody`类

- body数据以固定长度传输

将数据长度封装到`newFixedLengthSource`对象；
将source对象封装到`RealResponseBody`类

- 返回的body数据长度未知

创建`UnknownLengthSource`对象；
将source对象封装到`RealResponseBody`类

body数据在哪里读取呢？在`BridgeInterceptor`中：

```java
if (transparentGzip
    && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
    && HttpHeaders.hasBody(networkResponse)) {
  GzipSource responseBody = new GzipSource(networkResponse.body().source());
  Headers strippedHeaders = networkResponse.headers().newBuilder()
      .removeAll("Content-Encoding")
      .removeAll("Content-Length")
      .build();
  responseBuilder.headers(strippedHeaders);
  String contentType = networkResponse.header("Content-Type");
  responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));
}
```

### 1.2 处理返回异常

- 服务端连接关闭

```java
if ("close".equalsIgnoreCase(response.request().header("Connection"))
    || "close".equalsIgnoreCase(response.header("Connection"))) {
  streamAllocation.noNewStreams();
}
```
将连接标记为没有新的数据流，避免再次分配数据流

- 数据长度与返回不匹配

```java
if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
  throw new ProtocolException(
      "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
}
```

HTTP协议中 204 No Content 成功状态响应码表示目前请求成功，但客户端不需要更新其现有页面。204 响应默认是可以被缓存的。在响应中需要包含头信息 ETag。

在 HTTP 协议中，响应状态码 205 Reset Content 用来通知客户端重置文档视图，比如清空表单内容、重置 canvas 状态或者刷新用户界面。

这两种情况下就抛异常返回了。

### 1.3 Http2Codec
流程上基本与Http1Codec一致，只是部分方法实现不一样。
#### 1.3.1 写请求头

将key,value封装到Header列表中,然后将header数据封装到`Http2Stream`并写入：

```java
List<Header> requestHeaders = http2HeadersList(request);
stream = connection.newStream(requestHeaders, hasRequestBody);
```
在`newStream`方法中比较关键的两步是：

```java
stream = new Http2Stream(streamId, this, outFinished, inFinished, requestHeaders);
writer.synStream(outFinished, streamId, associatedStreamId, requestHeaders);
```
创建数据流完毕之后开始写入数据并等待服务端返回了。

#### 1.3.2 读取返回数据
- 读取返回Header数据

```java
@Override public Response.Builder readResponseHeaders(boolean expectContinue) throws IOException {
    List<Header> headers = stream.takeResponseHeaders();
    Response.Builder responseBuilder = readHttp2HeadersList(headers, protocol);
    if (expectContinue && Internal.instance.code(responseBuilder) == HTTP_CONTINUE) {
      return null;
    }
    return responseBuilder;
}
```
在`takeResponseHeaders()`方法中，有个无限循环等着Header数据被返回：

```java
readTimeout.enter();
try {
  while (responseHeaders == null && errorCode == null) {
    waitForIo();
  }
} finally {
  readTimeout.exitAndThrowIfTimedOut();
}
```
那么这个`responseHeaders`在哪里初始化呢？
记得在`ConnectInterceptor`中说过，`startHttp2`方法中，连接开始之后就有一个While循环等着读取数据，数据返回之后将数据回调到`Http2Connection`的内部类`ReaderRunnable`中处理，`ReaderRunnable`封装了一个读取类`Http2Reader`，在这个类中分别复写了`data`,`headers`等方法。

```java
@Override public void headers(boolean inFinished, int streamId, int associatedStreamId,
    List<Header> headerBlock) {
      Http2Stream stream;
      synchronized (Http2Connection.this) {
        stream = getStream(streamId);
        if (stream == null) {
            final Http2Stream newStream = new Http2Stream(streamId, Http2Connection.this,
                  false, inFinished, headerBlock);
        }
    }
    stream.receiveHeaders(headerBlock);
}
```
这个stream对象就是上一步中初始化的。

到这里就可以读取到Headers数据了。

接下来把Header数据封装到`Response.Builder`对象，如果是100-continue返回就直接返回null。

对于客户端请求头字段包含100-continue，而服务端恰好返回100,此时要填充请求Header body数据再次发送请求。

接着再次读取服务端返回的Header的数据。

```java
responseBuilder = httpCodec.readResponseHeaders(false);
```
#### 1.3.3 读取返回数据
封装返回的数据比较简单：

```java
@Override public ResponseBody openResponseBody(Response response) throws IOException {
    streamAllocation.eventListener.responseBodyStart(streamAllocation.call);
    String contentType = response.header("Content-Type");
    long contentLength = HttpHeaders.contentLength(response);
    Source source = new StreamFinishingSource(stream.getSource());
    return new RealResponseBody(contentType, contentLength, Okio.buffer(source));
}
```
`stream.getSource()`数据来源是：

在`Http2Reader`的`nextFrame`方法中，会有数据返回

```java
readData(handler, length, flags, streamId);
handler.data(inFinished, streamId, source, length);
```
回调到`Http2Connection`：

```java
dataStream.receiveData(source, length);
```
到`Http2Stream`：

```java
void receiveData(BufferedSource in, int length) throws IOException {
    assert (!Thread.holdsLock(Http2Stream.this));
    this.source.receive(in, length);
}
```
上面的整个数据回调流程在`ConnectInterceptor`中有详细说明。

异常处理与Http2Codec一致。数据在`BridgeInterceptor`中处理。

微信公众号二维码：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200331153056115.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTExOTg3NjQ=,size_16,color_FFFFFF,t_70#pic_center)
