---
layout: post
title:  "Android OKHttp系列3-BridgeInterceptor"
date:   2019-5-14 11:09:22 +0800
categories: Android
---


> 文章将会被同步至微信公众号：Android部落格

# 1、概述
>用来整理请求和响应的数据

流程图如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200331173116403.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTExOTg3NjQ=,size_16,color_FFFFFF,t_70#pic_center)

## 1.1 Request.Builder

请求Header被封装在这个Builder类中，针对不同的情况填写不同的Header值。Http Header请求字段列表如下：

 | Header | 解释 | 示例 |
 | ----- | :--------  | :--------- | :------   |
 |Accept	 |指定客户端能够接收的内容类型	 |Accept: text/plain, text/html |
 |Accept-Charset	 |浏览器可以接受的字符编码集。 |	Accept-Charset: iso-8859-5 |
 |Accept-Encoding |	指定浏览器可以支持的web服务器返回内容压缩编码类型。 |	Accept-Encoding: compress, gzip |
 |Accept-Language	 |浏览器可接受的语言 |	Accept-Language: en,zh |
 |Accept-Ranges	 |可以请求网页实体的一个或者多个子范围字段	 |Accept-Ranges: bytes |
 |Authorization	 |HTTP授权的授权证书 |	Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ== |
 |Cache-Control	 |指定请求和响应遵循的缓存机制 |	Cache-Control: no-cache |
 |Connection |	表示是否需要持久连接。（HTTP 1.1默认进行持久连接） |	Connection: close |
 |Cookie	 |HTTP请求发送时，会把保存在该请求域名下的所有cookie值一起发送给web服务器。 |	Cookie: $Version=1; Skin=new; |
 |Content-Length |	请求的内容长度 |	Content-Length: 348 |
 |Content-Type	 |请求的与实体对应的MIME信息 |	Content-Type: application/x-www-form-urlencoded |
 |Date	 |请求发送的日期和时间	 |Date: Tue, 15 Nov 2010 08:12:31 GMT |
 |Expect	 |请求的特定的服务器行为 |	Expect: 100-continue |
 |From	 |发出请求的用户的Email	 |From: user@email.com |
 |Host	 |指定请求的服务器的域名和端口号 |	Host: www.baidu.com |
 |If-Match	 |只有请求内容与实体相匹配才有效 |	If-Match: “737060cd8c284d8af7ad3082f209582d” |
 |If-Modified-Since |	如果请求的部分在指定时间之后被修改则请求成功，未被修改则返回304代码	 |If-Modified-Since: Sat, 29 Oct 2010 19:43:31 GMT |
 |If-None-Match |	如果内容未改变返回304代码，参数为服务器先前发送的Etag，与服务器回应的Etag比较判断是否改变 |	If-None-Match: “737060cd8c284d8af7ad3082f209582d” |
 |If-Range |	如果实体未改变，服务器发送客户端丢失的部分，否则发送整个实体。参数也为Etag |	If-Range: “737060cd8c284d8af7ad3082f209582d” |
 |If-Unmodified-Since |	只在实体在指定时间之后未被修改才请求成功 |	If-Unmodified-Since: Sat, 29 Oct 2010 19:43:31 GMT |
 |Max-Forwards |	限制信息通过代理和网关传送的时间 |	Max-Forwards: 10 |
 |Pragma |	用来包含实现特定的指令	 |Pragma: no-cache |
 |Proxy-Authorization |	连接到代理的授权证书 |	Proxy-Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ== |
 |Range	 |只请求实体的一部分，指定范围 |	Range: bytes=500-999 |
 |Referer	 |先前网页的地址，当前请求网页紧随其后,即来路 |	Referer: http://www.baidu.com |
 |TE	 |客户端愿意接受的传输编码，并通知服务器接受接受尾加头信息 |	TE: trailers,deflate;q=0.5 |
 |Upgrade |	向服务器指定某种传输协议以便服务器进行转换（如果支持） |	Upgrade: HTTP/2.0, SHTTP/1.3, IRC/6.9, RTA/x11 |
 |User-Agent |	User-Agent的内容包含发出请求的用户信息 |	User-Agent: Mozilla/5.0 (Linux; X11) |
 |Via |	通知中间网关或代理服务器地址，通信协议	 |Via: 1.0 fred, 1.1 nowhere.com (Apache/1.1) |
 |Warning	 |关于消息实体的警告信息 |	Warn: 199 Miscellaneous warning |
 
## 1.2 RequestBody

```java
RequestBody body = userRequest.body();
if (body != null) {
  MediaType contentType = body.contentType();
  if (contentType != null) {
    requestBuilder.header("Content-Type", contentType.toString());
  }

  long contentLength = body.contentLength();
  if (contentLength != -1) {
    requestBuilder.header("Content-Length", Long.toString(contentLength));
    requestBuilder.removeHeader("Transfer-Encoding");
  } else {
    requestBuilder.header("Transfer-Encoding", "chunked");
    requestBuilder.removeHeader("Content-Length");
  }
}
```

`RequestBody`用于在请求过程中传递数据，一般用于Post请求。
> 同时了解一下Get和Post请求的区别：
HTTP规定，当执行GET请求的时候，设置请求方法为GET，而且要求把传送的数据放在url中以方便记录。

> 如果是POST请求，设置请求方法为POST，并把数据放在request body中。当然，可以在GET请求的时候添加request body数据，也可以在POST的时候将数据放在url中，但是似乎这两种操作都没有必要。

> 针对浏览器（发起http请求）和服务器（接受http请求），虽然理论上，可以在url中无限加参数，但是请求端封装数据和服务器解析数据有很大成本的，会限制单次运输量来控制风险，数据量太大对浏览器和服务器都是很大负担。业界不成文的规定是，（大多数）浏览器通常都会限制url长度在2K个字节，而（大多数）服务器最多处理64K大小的url。超过的部分，恕不处理。如果你用GET服务，在request body偷偷藏了数据，不同服务器的处理方式也是不同的，有些服务器会解析数据，有些服务器直接忽略，所以，虽然GET可以带request body，也不能保证一定能被接收到。

如果`RequestBody`不为null的话，则开始往请求Header里面添加一些数据：
- **Content-Type** 内容类型，详见 https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Type
- **Content-Length** 请求消息正文的长度。如果长度大于0，则添加`Content-Length`字段，同时删除`Transfer-Encoding`字段，否则做相反的操作。

接下来是常规流程，分别处理以下请求头字段：
- **Host** 请求头指明了服务器的域名（对于虚拟主机来说），以及（可选的）服务器监听的TCP端口号。如果没有给定端口号，会自动使用被请求服务的默认端口：

```java
public static int defaultPort(String scheme) {
    if (scheme.equals("http")) {
      return 80;
    } else if (scheme.equals("https")) {
      return 443;
    } else {
      return -1;
    }
}
```
- **Connection** 头（header） 决定当前的事务完成后，是否会关闭网络连接。如果该值是“keep-alive”，网络连接就是持久的，不会关闭，使得对同一个服务器的请求可以继续在该连接上完成。如果不设置，sdk默认设置为`Keep-Alive`.
- **Accept-Encoding** 请求头 Accept-Encoding 会将客户端能够理解的内容编码方式——通常是某种压缩算法——进行通知。通过内容协商的方式，服务端会选择一个客户端提议的方式，使用并在响应报文首部 Content-Encoding 中通知客户端该选择。
- **Range** 是一个请求首部，告知服务器返回文件的哪一部分。在一个  Range 首部中，可以一次性请求多个部分，服务器会以 multipart 文件的形式将其返回。如果服务器返回的是范围响应，需要使用 206 Partial Content 状态码。

假如所请求的范围不合法，那么服务器会返回  416 Range Not Satisfiable 状态码，表示客户端错误。服务器允许忽略  Range  首部，从而返回整个文件，状态码用 200 。

请求的时候如果不设置这两个字段，则sdk会默认处理采用gzip压缩传输：

```java
boolean transparentGzip = false;
if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
  transparentGzip = true;
  requestBuilder.header("Accept-Encoding", "gzip");
}
```
- **Cookie** 是一个请求首部，其中含有先前由服务器通过`Set-Cookie`首部投放并存储到客户端的 HTTP cookies。

这个首部可能会被完全移除，例如在浏览器的隐私设置里面设置为禁用cookie。sdk中如果请求的时候不设置，也不做任何处理，默认为`CookieJar.NO_COOKIES`。

示例：
```
Cookie: <cookie-list>
Cookie: name=value
Cookie: name=value; name2=value2; name3=value3
```
- **User-Agent** 首部包含了一个特征字符串，用来让网络协议的对端来识别发起请求的用户代理软件的应用类型、操作系统、软件开发商以及版本号。请求的时候如果不设置，sdk默认填充。

```java
if (userRequest.header("User-Agent") == null) {
    requestBuilder.header("User-Agent", Version.userAgent());
}
public static String userAgent() {
    return "okhttp/${project.version}";
}
```
到此请求头的字段已经整理完毕，可以交给下一个拦截器干活了，然后等着数据返回，返回之后开始处理返回的Response。请求头示例如下：
![请求头结构](https://ftp.bmp.ovh/imgs/2020/05/524eaf2068cfa9f3.png)

## 1.3 Response

Response的Header字段列表如下：

| Header | 解释 | 示例 |
| ----- | :--------  | :--------- | :------   |
| Accept-Ranges |	表明服务器是否支持指定范围请求及哪种类型的分段请求 |	Accept-Ranges: bytes |
|Age	 |从原始服务器到代理缓存形成的估算时间（以秒计，非负） |	Age: 12 |
|Allow	 |对某网络资源的有效的请求行为，不允许则返回405 |	Allow: GET, HEAD |
|Cache-Control	 |告诉所有的缓存机制是否可以缓存及哪种类型 |	Cache-Control: no-cache |
|Content-Encoding	 |web服务器支持的返回内容压缩编码类型。 |Content-Encoding: gzip |
|Content-Language	 |响应体的语言 |	Content-Language: en,zh |
|Content-Length |	响应体的长度 |	Content-Length: 348 |
|Content-Location |	请求资源可替代的备用的另一地址 |	Content-Location: /index.htm |
|Content-MD5	 |返回资源的MD5校验值 |	Content-MD5: Q2hlY2sgSW50ZWdyaXR5IQ== |
|Content-Range	 |在整个返回体中本部分的字节位置 |	Content-Range: bytes 21010-47021/47022 |
|Content-Type	 |返回内容的MIME类型 |	Content-Type: text/html; charset=utf-8 |
|Date	 |原始服务器消息发出的时间 |	Date: Tue, 15 Nov 2010 08:12:31 GMT |
|ETag	 |请求变量的实体标签的当前值 |	ETag: “737060cd8c284d8af7ad3082f209582d” |
|Expires	 |响应过期的日期和时间	 |Expires: Thu, 01 Dec 2010 16:00:00 GMT |
|Last-Modified	 |请求资源的最后修改时间 |	Last-Modified: Tue, 15 Nov 2010 12:45:26 GMT |
|Location	 |用来重定向接收方到非请求URL的位置来完成请求或标识新的资源	 |Location: http://www.baidu.com |
|Pragma	 |包括实现特定的指令，它可应用到响应链上的任何接收方 |	Pragma: no-cache |
|Proxy-Authenticate |	它指出认证方案和可应用到代理的该URL上的参数 |	Proxy-Authenticate: Basic |
|refresh |应用于重定向或一个新的资源被创造，在5秒之后重定向（由网景提出，被大部分浏览器支持）| Refresh: 5; url= http://www.baidu.com |
|Retry-After |如果实体暂时不可取，通知客户端在指定时间之后再次尝试 | Retry-After: 120 |
|Server |	web服务器软件名称 |	Server: Apache/1.3.27 (Unix) (Red-Hat/Linux) |
|Set-Cookie |设置Http Cookie |	Set-Cookie: UserID=JohnDoe; Max-Age=3600; Version=1 |
|Trailer	 |指出头域在分块传输编码的尾部存在 |	Trailer: Max-Forwards |
|Transfer-Encoding |	文件传输编码 |	Transfer-Encoding:chunked |
|Vary	 |告诉下游代理是使用缓存响应还是从原始服务器请求 |	Vary: * |
|Via |	告知代理客户端响应是通过哪里发送的	 |Via: 1.0 fred, 1.1 nowhere.com (Apache/1.1) |
|Warning |	警告实体可能存在的问题 |	Warning: 199 Miscellaneous warning |
|WWW-Authenticate|表明客户端请求实体应该使用的授权方案 |WWW-Authenticate: Basic |

- **receiveHeaders**

返回数据伊始，调用这个方法，在`HttpHeaders`类中，主要目的是保存cookie。

- **Response.Builder** 开始创建一个Builder对象，并封装返回的数据。

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
- **Content-Encoding** 是一个实体消息首部，用于对特定媒体类型的数据进行压缩。

当这个首部出现的时候，它的值表示消息主体进行了何种方式的内容编码转换。这个消息首部用来告知客户端应该怎样解码才能获取在 Content-Type 中标示的媒体类型内容。
此处sdk处理的是gzip压缩并且返回数据存在body的情况，首先使用`GzipSource`解压缩。对于`networkResponse.body().source()`这一行代码看起来比较难以理解，其实是在接收到content内容之后，okio将其写到一个自定义的`Buffer`,然后传递回来，详细创建过程如下：

```java
public static ResponseBody create(@Nullable MediaType contentType, String content) {
    Charset charset = UTF_8;
    if (contentType != null) {
      charset = contentType.charset();
      if (charset == null) {
        charset = UTF_8;
        contentType = MediaType.parse(contentType + "; charset=utf-8");
      }
    }
    Buffer buffer = new Buffer().writeString(content, charset);
    return create(contentType, buffer.size(), buffer);
}

/** Returns a new response body that transmits {@code content}. */
public static ResponseBody create(final @Nullable MediaType contentType, byte[] content) {
    Buffer buffer = new Buffer().write(content);
    return create(contentType, content.length, buffer);
}

/** Returns a new response body that transmits {@code content}. */
public static ResponseBody create(@Nullable MediaType contentType, ByteString content) {
    Buffer buffer = new Buffer().write(content);
    return create(contentType, content.size(), buffer);
}

/** Returns a new response body that transmits {@code content}. */
public static ResponseBody create(final @Nullable MediaType contentType,
      final long contentLength, final BufferedSource content) {
    if (content == null) throw new NullPointerException("source == null");
    return new ResponseBody() {
      @Override public @Nullable MediaType contentType() {
        return contentType;
      }

      @Override public long contentLength() {
        return contentLength;
      }

      @Override public BufferedSource source() {
        return content;
      }
    };
 }
```

- **Response.Headers**

接下来处理响应的头信息，在gzip压缩的情况下，删除`Content-Encoding`和`Content-Length`字段，重新构建一个Headers,然后将处理过的`Header`封装到`responseBuilder`.

- **RealResponseBody**

从命名可以看出，是实际的响应数据body封装类，将响应的body，类型，长度默认为-1封装到一个RealResponseBody对象,然后封装到`responseBuilder`。

```java
RealResponseBody(contentType, -1L, Okio.buffer(responseBody))
```

最终执行`responseBuilder.build()`方法返回一个Response对象。再看看一个响应的头示例：

![响应头结构](https://ftp.bmp.ovh/imgs/2020/05/630b01d3d83f26cf.png)

对于Response body部分，略加说明：
响应信息最后一部分是body，但并不是所有的响应都会带有一个body，比如响应码201或204就不会。

Response body可以分为三种策略返回：
- 单个资源的body, 连续的并知道长度的文件, 定义在以下两个头字段中：`Content-Type`和`Content-Length`。
- 单个资源的body, 连续的但不知道长度的文件, 以分段的形式被编码发送，以`Transfer-Encoding`字段被设置为`chunked`体现。
- 多个资源的body, 以多段body组成，每一部分包含了不同的信息，不过相对比较少见。

微信公众号：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200331173229291.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTExOTg3NjQ=,size_16,color_FFFFFF,t_70#pic_center)