---
layout: post
title:  "Android-Fresco系列3 Producer"
date:   2019-6-5 9:53:25 +0800
categories: Android
---

先看流程图：
![](https://i.loli.net/2019/06/05/5cf733964fcfc63925.jpg)

## 一、发起请求
记得在AbstractDraweeController的submitRequest方法中先获取一个DataSource对象(getDataSource())，然后定义了一个DataSubscriber对象，是以内部类的方式初始化的，最后调用mDataSource.subscribe().

#### 1) ControllerBuilder
```java
//PipelineDraweeController
@Override
protected DataSource<CloseableReference<CloseableImage>> getDataSource() {
    DataSource<CloseableReference<CloseableImage>> result = mDataSourceSupplier.get();
    return result;
}

```
mDataSourceSupplier对象是在PipelineDraweeControllerBuilder的obtainController方法中，会执行一个initialize方法，第一个参数是obtainDataSourceSupplier，这个方法最终调用的是getDataSourceSupplierForRequest方法，在AbstractDraweeControllerBuilder类中，这个getDataSourceSupplierForRequest方法之前有讲到过：
```java
return new Supplier<DataSource<IMAGE>>() {
  @Override
  public DataSource<IMAGE> get() {
    return getDataSourceForRequest(
        controller, controllerId, imageRequest, callerContext, cacheLevel);
  }

  @Override
  public String toString() {
    return Objects.toStringHelper(this).add("request", imageRequest.toString()).toString();
  }
};
```
getDataSource方法中的get调用的就是这里的方法(getDataSourceForRequest)：
```java
//PipelineDraweeControllerBuilder
@Override
protected DataSource<CloseableReference<CloseableImage>> getDataSourceForRequest(
  DraweeController controller,
  String controllerId,
  ImageRequest imageRequest,
  Object callerContext,
  AbstractDraweeControllerBuilder.CacheLevel cacheLevel) {
    return mImagePipeline.fetchDecodedImage(
        imageRequest,
        callerContext,
        convertCacheLevelToRequestLevel(cacheLevel),
        getRequestListener(controller));
}
```
cacheLevel是CacheLevel.FULL_FETCH。
- fetchDecodedImage

看看fetchDecodedImage方法：
```java
//ImagePipeline
public DataSource<CloseableReference<CloseableImage>> fetchDecodedImage(
  ImageRequest imageRequest,
  Object callerContext,
  ImageRequest.RequestLevel lowestPermittedRequestLevelOnSubmit,
  @Nullable RequestListener requestListener) {
    try {
        Producer<CloseableReference<CloseableImage>> producerSequence = mProducerSequenceFactory.getDecodedImageProducerSequence(imageRequest);
        return submitFetchRequest(producerSequence, imageRequest, lowestPermittedRequestLevelOnSubmit, callerContext, requestListener);
    } catch (Exception exception) {
        return DataSources.immediateFailedDataSource(exception);
    }
}
```
- getDecodedImageProducerSequence

先看看getDecodedImageProducerSequence方法：
```java
//ProducerSequenceFactory
public Producer<CloseableReference<CloseableImage>> getDecodedImageProducerSequence(ImageRequest imageRequest) {
    Producer<CloseableReference<CloseableImage>> pipelineSequence =
        getBasicDecodedImageSequence(imageRequest);
    
    if (imageRequest.getPostprocessor() != null) {
      pipelineSequence = getPostprocessorSequence(pipelineSequence);
    }
    
    if (mUseBitmapPrepareToDraw) {//Bitmap.prepareToDraw
      pipelineSequence = getBitmapPrepareSequence(pipelineSequence);
    }
    return pipelineSequence;
}
```
- getBasicDecodedImageSequence

getBasicDecodedImageSequence方法的目的是根据不同的请求类型新建不同的内容生产者对象：
```java
//ProducerSequenceFactory
private Producer<CloseableReference<CloseableImage>> getBasicDecodedImageSequence(ImageRequest imageRequest) {
    try {
      Uri uri = imageRequest.getSourceUri();
      switch (imageRequest.getSourceUriType()) {
        case SOURCE_TYPE_NETWORK:
          return getNetworkFetchSequence();
        case SOURCE_TYPE_LOCAL_VIDEO_FILE:
          return getLocalVideoFileFetchSequence();
        case SOURCE_TYPE_LOCAL_IMAGE_FILE:
          return getLocalImageFileFetchSequence();
        case SOURCE_TYPE_LOCAL_CONTENT:
          if (MediaUtils.isVideo(mContentResolver.getType(uri))) {
            return getLocalVideoFileFetchSequence();
          }
          return getLocalContentUriFetchSequence();
        case SOURCE_TYPE_LOCAL_ASSET:
          return getLocalAssetFetchSequence();
        case SOURCE_TYPE_LOCAL_RESOURCE:
          return getLocalResourceFetchSequence();
        case SOURCE_TYPE_QUALIFIED_RESOURCE:
          return getQualifiedResourceFetchSequence();
        case SOURCE_TYPE_DATA:
          return getDataFetchSequence();
        default:
          throw new IllegalArgumentException(
              "Unsupported uri scheme! Uri is: " + getShortenedUriString(uri));
      }
    } finally {
    }
}
```

#### 2) ProducerSequenceFactory
以SOURCE_TYPE_NETWORK为例，获取的是网络请求内容的生产对象：
```java
//ProducerSequenceFactory
private synchronized Producer<CloseableReference<CloseableImage>> getNetworkFetchSequence() {
    if (mNetworkFetchSequence == null) {
      mNetworkFetchSequence =
          newBitmapCacheGetToDecodeSequence(getCommonNetworkFetchToEncodedMemorySequence());
    }
    return mNetworkFetchSequence;
}

private synchronized Producer<EncodedImage> getCommonNetworkFetchToEncodedMemorySequence() {
    if (mCommonNetworkFetchToEncodedMemorySequence == null) {
        Producer<EncodedImage> inputProducer = newEncodedCacheMultiplexToTranscodeSequence(mProducerFactory.newNetworkFetchProducer(mNetworkFetcher));
        mCommonNetworkFetchToEncodedMemorySequence = ProducerFactory.newAddImageTransformMetaDataProducer(inputProducer);
        mCommonNetworkFetchToEncodedMemorySequence = mProducerFactory.newResizeAndRotateProducer( mCommonNetworkFetchToEncodedMemorySequence, mResizeAndRotateEnabledForNetwork && !mDownsampleEnabled,mImageTranscoderFactory);
    }
    return mCommonNetworkFetchToEncodedMemorySequence;
}

private Producer<EncodedImage> newEncodedCacheMultiplexToTranscodeSequence(
      Producer<EncodedImage> inputProducer) {
    if (WebpSupportStatus.sIsWebpSupportRequired &&
        (!mWebpSupportEnabled || WebpSupportStatus.sWebpBitmapFactory == null)) {
      inputProducer = mProducerFactory.newWebpTranscodeProducer(inputProducer);
    }
    if (mDiskCacheEnabled) {
      inputProducer = newDiskCacheSequence(inputProducer);
    }
    EncodedMemoryCacheProducer encodedMemoryCacheProducer = mProducerFactory.newEncodedMemoryCacheProducer(inputProducer);
    return mProducerFactory.newEncodedCacheKeyMultiplexProducer(encodedMemoryCacheProducer);
}

private Producer<EncodedImage> newDiskCacheSequence(Producer<EncodedImage> inputProducer) {
    Producer<EncodedImage> cacheWriteProducer;
    if (mPartialImageCachingEnabled) {
      Producer<EncodedImage> partialDiskCacheProducer =
          mProducerFactory.newPartialDiskCacheProducer(inputProducer);
      cacheWriteProducer = mProducerFactory.newDiskCacheWriteProducer(partialDiskCacheProducer);
    } else {
      cacheWriteProducer = mProducerFactory.newDiskCacheWriteProducer(inputProducer);
    }
    DiskCacheReadProducer result = mProducerFactory.newDiskCacheReadProducer(cacheWriteProducer);
    return result;
}
```
上面的代码比较长，从getCommonNetworkFetchToEncodedMemorySequence方法开始罗列出了newEncodedCacheMultiplexToTranscodeSequence，newDiskCacheSequence方法，从这几个方法最终产生了一个Producer<EncodedImage>对象。在这三个方法里面总共产生了以下Producer:

- 网络请求

NetworkFetchProducer(implements Producer<EncodedImage>)
- Webp格式处理

WebpTranscodeProducer(implements Producer<EncodedImage>)
- 部分缓存与请求的数据结合成一整个返回处理

PartialDiskCacheProducer(implements Producer<EncodedImage>)
- 磁盘缓存写处理

DiskCacheWriteProducer(implements Producer<EncodedImage>)
- 磁盘缓存读处理

DiskCacheReadProducer(implements Producer<EncodedImage>)
- 已编码内存缓存处理

EncodedMemoryCacheProducer(implements Producer<EncodedImage>)
- 编码缓存多路复用处理

EncodedCacheKeyMultiplexProducer(extends MultiplexProducer<Pair<CacheKey, ImageRequest.RequestLevel>, EncodedImage>)
- 裁剪旋转处理

ResizeAndRotateProducer(implements Producer<EncodedImage>)

将上述所有的图片处理封装到一个Producer<EncodedImage>对象中，然后传递给newBitmapCacheGetToDecodeSequence方法：
```java
private Producer<CloseableReference<CloseableImage>> newBitmapCacheGetToDecodeSequence(
  Producer<EncodedImage> inputProducer) {
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("ProducerSequenceFactory#newBitmapCacheGetToDecodeSequence");
    }
    DecodeProducer decodeProducer = mProducerFactory.newDecodeProducer(inputProducer);
    Producer<CloseableReference<CloseableImage>> result =
        newBitmapCacheGetToBitmapCacheSequence(decodeProducer);
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
    return result;
}
```
先新建一个解码对象DecodeProducer，他实现了Producer<CloseableReference<CloseableImage>>接口。然后执行newBitmapCacheGetToBitmapCacheSequence方法将解码对象封装起来：
```java
private Producer<CloseableReference<CloseableImage>> newBitmapCacheGetToBitmapCacheSequence(Producer<CloseableReference<CloseableImage>> inputProducer) {
    BitmapMemoryCacheProducer bitmapMemoryCacheProducer = mProducerFactory.newBitmapMemoryCacheProducer(inputProducer);
    BitmapMemoryCacheKeyMultiplexProducer bitmapKeyMultiplexProducer = mProducerFactory.newBitmapMemoryCacheKeyMultiplexProducer(bitmapMemoryCacheProducer);
    ThreadHandoffProducer<CloseableReference<CloseableImage>> threadHandoffProducer = mProducerFactory.newBackgroundThreadHandoffProducer( bitmapKeyMultiplexProducer, mThreadHandoffProducerQueue);
    return mProducerFactory.newBitmapMemoryCacheGetProducer(threadHandoffProducer);
}
```
一层一层叠加的封装，让代码看起来云里雾里，但是只要记住一个，就是Producer结尾的类，一定实现了produceResults方法，在这个方法里面干活。

AbstractDraweeController类的submitRequest方法中调用的getDataSource方法引申出了Producer类的初始化，然后接下来就是要依赖这些类干活了。

- submitFetchRequest

从ImagePipeline类的fetchDecodedImage方法中调用的第一个方法getDecodedImageProducerSequence回来，接下来调用submitFetchRequest方法：
```java
//ImagePipeline
private <T> DataSource<CloseableReference<T>> submitFetchRequest(Producer<CloseableReference<T>> producerSequence, ImageRequest imageRequest, ImageRequest.RequestLevel lowestPermittedRequestLevelOnSubmit, Object callerContext, @Nullable RequestListener requestListener) {
    ImageRequest.RequestLevel lowestPermittedRequestLevel = ImageRequest.RequestLevel.getMax(imageRequest.getLowestPermittedRequestLevel(), lowestPermittedRequestLevelOnSubmit);
    SettableProducerContext settableProducerContext = new SettableProducerContext(imageRequest, generateUniqueFutureId(), finalRequestListener, callerContext, lowestPermittedRequestLevel, /* isPrefetch */ false, imageRequest.getProgressiveRenderingEnabled() || !UriUtil.isNetworkUri(imageRequest.getSourceUri()), imageRequest.getPriority());
    return CloseableProducerToDataSourceAdapter.create(producerSequence, settableProducerContext, finalRequestListener);
}
```
先构造了一个SettableProducerContext对象，这个对象封装了请求的上下文，CloseableProducerToDataSourceAdapter.create方法则是最终返回：
```java
//CloseableProducerToDataSourceAdapter
public static <T> DataSource<CloseableReference<T>> create(Producer<CloseableReference<T>> producer, SettableProducerContext settableProducerContext, RequestListener listener) {
    CloseableProducerToDataSourceAdapter<T> result = new CloseableProducerToDataSourceAdapter<T>(producer, settableProducerContext, listener);
}
//CloseableProducerToDataSourceAdapter
private CloseableProducerToDataSourceAdapter(
      Producer<CloseableReference<T>> producer,
      SettableProducerContext settableProducerContext,
      RequestListener listener) {
    super(producer, settableProducerContext, listener);
}
//AbstractProducerToDataSourceAdapter
protected AbstractProducerToDataSourceAdapter(
  Producer<T> producer,
  SettableProducerContext settableProducerContext,
  RequestListener requestListener) {
  producer.produceResults(createConsumer(), settableProducerContext);
}
```
可以看到最终调用到了AbstractProducerToDataSourceAdapter的构造函数，这个类继承自AbstractDataSource。在createConsumer()方法中我们可以看到订阅者：
```java
private Consumer<T> createConsumer() {
    return new BaseConsumer<T>() {
      @Override
      protected void onNewResultImpl(@Nullable T newResult, @Status int status) {
        AbstractProducerToDataSourceAdapter.this.onNewResultImpl(newResult, status);
      }

      @Override
      protected void onFailureImpl(Throwable throwable) {
        AbstractProducerToDataSourceAdapter.this.onFailureImpl(throwable);
      }

      @Override
      protected void onCancellationImpl() {
        AbstractProducerToDataSourceAdapter.this.onCancellationImpl();
      }

      @Override
      protected void onProgressUpdateImpl(float progress) {
        AbstractProducerToDataSourceAdapter.this.setProgress(progress);
      }
    };
}
```
每一个订阅的消息返回对应着不同的处理逻辑，都在AbstractProducerToDataSourceAdapter类中定义着。

#### 3) Producer创建
接下来执行producer.produceResults方法。producer是从ImagePipeline的fetchDecodedImage方法中生成的。这些producer统一实现了Producer接口：
```java
public interface Producer<T> {

  /**
   * Start producing results for given context. Provided consumer is notified whenever progress is
   * made (new value is ready or error occurs).
   * @param consumer
   * @param context
   */
  void produceResults(Consumer<T> consumer, ProducerContext context);
}
```
先看看从前到后生成的producer有哪些：

(NetworkFetchProducer->

(WebpTranscodeProducer -> PartialDiskCacheProducer -> DiskCacheWriteProducer -> DiskCacheReadProducer -> EncodedMemoryCacheProducer) -> 

EncodedCacheKeyMultiplexProducer:MultiplexProducer -> AddImageTransformMetaDataProducer -> ResizeAndRotateProducer) -> 

DecodeProducer -> BitmapMemoryCacheProducer -> BitmapMemoryCacheKeyMultiplexProducer:MultiplexProducer -> ThreadHandoffProducer -> BitmapMemoryCacheGetProducer:BitmapMemoryCacheProducer = mNetworkFetchSequence

前面的producer作为后面的inputProducer参数传入，然后封装，依次向后。

再来看看AbstractProducerToDataSourceAdapter类构造函数中的producer.produceResults方法，一石激起千层浪，看看这些层层封装的Producer调用顺序：
BitmapMemoryCacheProducer
ThreadHandoffProducer
BitmapMemoryCacheProducer
DecodeProducer
ResizeAndRotateProducer
AddImageTransformMetaDataProducer
EncodedMemoryCacheProducer
DiskCacheReadProducer
DiskCacheWriteProducer
NetworkFetchProducer

最后调用NetworkFetchProducer在线下载数据，数据下载完成之后，响应的顺序刚好相反，对应的producer实现的produceResults方法的第一个参数是Consumer<EncodedImage>，用来通知上一层的消费者，因为consumer对象是从上一个调用者传递过来的。

## 二、开始请求
- NetworkFetchProducer

先看看NetworkFetchProducer在线下载图片。NetworkFetchProducer封装了一个mNetworkFetcher对象，这个对象是在ImagePipelineConfig构造的时候创建的，这个对象对应的是HttpUrlConnectionNetworkFetcher类，如果用户默认不指定的话。
- produceResults

看看produceResult方法：
```java
@Override
public void produceResults(Consumer<EncodedImage> consumer, ProducerContext context) {
    final FetchState fetchState = mNetworkFetcher.createFetchState(consumer, context);
    mNetworkFetcher.fetch(fetchState, new NetworkFetcher.Callback() {
          @Override
          public void onResponse(InputStream response, int responseLength) throws IOException {
              NetworkFetchProducer.this.onResponse(fetchState, response, responseLength);
          }
          @Override
          public void onFailure(Throwable throwable) {
            NetworkFetchProducer.this.onFailure(fetchState, throwable);
          }

          @Override
          public void onCancellation() {
            NetworkFetchProducer.this.onCancellation(fetchState);
          }
    });
}
```
先封装了一个HttpUrlConnectionNetworkFetchState对象，他继承自FetchState，封装了consumer和请求上下文context，以及中间状态的时间点。

onResponse是请求的返回，也是在HttpUrlConnectionNetworkFetcher调用fetch有结果返回之后回调到这里的：
```java
@Override
public void fetch(final HttpUrlConnectionNetworkFetchState fetchState, final Callback callback) {
    fetchState.submitTime = mMonotonicClock.now();
    final Future<?> future = mExecutorService.submit(
        new Runnable() {
          @Override
          public void run() {
            fetchSync(fetchState, callback);
          }
    });
}

@VisibleForTesting
void fetchSync(HttpUrlConnectionNetworkFetchState fetchState, Callback callback) {
    HttpURLConnection connection = null;
    InputStream is = null;
    try {
      connection = downloadFrom(fetchState.getUri(), MAX_REDIRECTS);
      if (connection != null) {
        is = connection.getInputStream();
        callback.onResponse(is, -1);
      }
    } catch (IOException e) {
      callback.onFailure(e);
    } finally {
    }
}
```
一个典型的HttpURLConnection请求下载，使用固定线程池，线程数量为3，这里不再展开分析。从callback.onResponse回调到produceResults的onResponse方法，然后处理：
```java
protected void onResponse(FetchState fetchState, InputStream responseData, int responseContentLength) throws IOException {
    final PooledByteBufferOutputStream pooledOutputStream;
    if (responseContentLength > 0) {
      pooledOutputStream = mPooledByteBufferFactory.newOutputStream(responseContentLength);
    } else {
      pooledOutputStream = mPooledByteBufferFactory.newOutputStream();
    }
    final byte[] ioArray = mByteArrayPool.get(READ_SIZE);
    try {
      int length;
      while ((length = responseData.read(ioArray)) >= 0) {
        if (length > 0) {
          pooledOutputStream.write(ioArray, 0, length);
          maybeHandleIntermediateResult(pooledOutputStream, fetchState);
          float progress = calculateProgress(pooledOutputStream.size(), responseContentLength);
          fetchState.getConsumer().onProgressUpdate(progress);
        }
      }
      mNetworkFetcher.onFetchCompletion(fetchState, pooledOutputStream.size());
      handleFinalResult(pooledOutputStream, fetchState);
    } finally {
      mByteArrayPool.release(ioArray);
      pooledOutputStream.close();
    }
}
```
在while循环中读取数据，在此过程中计算进度，并把进度向上传递给上一层的消费者，这个消费者是DiskCacheWriteProducer。对于是否使用进度，由用户自己决定，不设置的话默认为false，这个开关创建是在ImagePipelineConfig中：
```java
private boolean mProgressiveRenderingEnabled = false;
private static DefaultImageRequestConfig
  sDefaultImageRequestConfig = new DefaultImageRequestConfig();

//ImageRequestBuilder
public ImageRequestBuilder setProgressiveRenderingEnabled(boolean enabled) {
    mProgressiveRenderingEnabled = enabled;
    return this;
}
```
看看下载完成之后调用的方法handleFinalResult：
```java
protected void handleFinalResult(
  PooledByteBufferOutputStream pooledOutputStream, FetchState fetchState) {
    notifyConsumer(
        pooledOutputStream,
        Consumer.IS_LAST | fetchState.getOnNewResultStatusFlags(),
        fetchState.getResponseBytesRange(),
        fetchState.getConsumer());
}
protected static void notifyConsumer(
  PooledByteBufferOutputStream pooledOutputStream,
  @Consumer.Status int status,
  @Nullable BytesRange responseBytesRange,
  Consumer<EncodedImage> consumer) {
    CloseableReference<PooledByteBuffer> result =
        CloseableReference.of(pooledOutputStream.toByteBuffer());
    EncodedImage encodedImage = null;
    try {
      encodedImage = new EncodedImage(result);
      encodedImage.setBytesRange(responseBytesRange);
      encodedImage.parseMetaData();
      consumer.onNewResult(encodedImage, status);
    } finally {
      EncodedImage.closeSafely(encodedImage);
      CloseableReference.closeSafely(result);
    }
}
```
CloseableReference.of方法将下载的stream转成bytebuffer封装成一个CloseableReference对象，这个方法的主要操作是new一个CloseableReference对象，对象中再new一个SharedReference对象，在这个类的构造函数中将bytebuffer存储到一个map中，`private static final Map<Object, Integer> sLiveObjects = new IdentityHashMap<>();`，key是bytebuffer值，value是针对这个对象的计数。

parseMetaData看方法名字就知道，读取图片流的元数据，详情如下:
```java
public void parseMetaData() {
    final ImageFormat imageFormat = ImageFormatChecker.getImageFormat_WrapIOException(getInputStream());
    mImageFormat = imageFormat;
    // BitmapUtil.decodeDimensions has a bug where it will return 100x100 for some WebPs even though
    // those are not its actual dimensions
    final Pair<Integer, Integer> dimensions;
    if (DefaultImageFormats.isWebpFormat(imageFormat)) {
      dimensions = readWebPImageSize();
    } else {
      dimensions = readImageMetaData().getDimensions();
    }
    if (imageFormat == DefaultImageFormats.JPEG && mRotationAngle == UNKNOWN_ROTATION_ANGLE) {
      // Load the JPEG rotation angle only if we have the dimensions
      if (dimensions != null) {
        mExifOrientation = JfifUtil.getOrientation(getInputStream());
        mRotationAngle = JfifUtil.getAutoRotateAngleFromOrientation(mExifOrientation);
      }
    } else if (imageFormat == DefaultImageFormats.HEIF
        && mRotationAngle == UNKNOWN_ROTATION_ANGLE) {
      mExifOrientation = HeifExifUtil.getOrientation(getInputStream());
      mRotationAngle = JfifUtil.getAutoRotateAngleFromOrientation(mExifOrientation);
    } else {
      mRotationAngle = 0;
    }
}
```
getImageFormat_WrapIOException方法读取格式，传入的inputStream数据：
```java
//ImageFormatChecker
public static ImageFormat getImageFormat(final InputStream is) throws IOException {
    return getInstance().determineImageFormat(is);
}

private ImageFormatChecker() {
    updateMaxHeaderLength();
}

private void updateMaxHeaderLength() {
    mMaxHeaderLength = mDefaultFormatChecker.getHeaderSize();
}
//DefaultImageFormatChecker
final int MAX_HEADER_LENGTH =
      Ints.max(
          //https://developers.google.com/speed/webp/docs/riff_container    
          //WebP格式，谷歌（google）开发的一种旨在加快图片加载速度的图片格式。图片压缩体积大约只有JPEG的2/3，并能节省大量的服务器宽带资源和数据空间。
          //VP8X，无损
          EXTENDED_WEBP_HEADER_LENGTH,
          //RIFF，有损
          SIMPLE_WEBP_HEADER_LENGTH,
          JPEG_HEADER_LENGTH,
          PNG_HEADER_LENGTH,
          GIF_HEADER_LENGTH,
          //BMP（全称Bitmap）是Windows操作系统中的标准图像文件格式，可以分成两类：设备有向量相关位图（DDB）和设备无向量相关位图（DIB），使用非常广。
          BMP_HEADER_LENGTH,
          //ico是Icon file的缩写，是Windows的图标文件格式的一种。图标文件可以存储单个图案、多尺寸、多色板的图标文件。一个图标实际上是多张不同格式的图片的集合体，并且还包含了一定的透明区域。
          ICO_HEADER_LENGTH,
          //高效率图像格式（High Efficiency Image Format ，HEIF）,它所生成的图像文件相对较小，且图像质量也高于较早的 JPEG 标准。HEIF 这种新的图像格式基于高效视频压缩格式（也称为 HEVC 或 H.265），它通过使用更先进的压缩算法来实现图片的压缩存储。
          HEIF_HEADER_LENGTH);
//DefaultImageFormatChecker
@Override
public int getHeaderSize() {
    return MAX_HEADER_LENGTH;
}
  
public ImageFormat determineImageFormat(final InputStream is) throws IOException {
    Preconditions.checkNotNull(is);
    final byte[] imageHeaderBytes = new byte[mMaxHeaderLength];
    final int headerSize = readHeaderFromStream(mMaxHeaderLength, is, imageHeaderBytes);
    
    ImageFormat format = mDefaultFormatChecker.determineFormat(imageHeaderBytes, headerSize);
    if (format != null && format != ImageFormat.UNKNOWN) {
      return format;
    }
    return ImageFormat.UNKNOWN;
}
```
先获取header的大小，然后通过determineFormat方法获取格式：
```java
//DefaultImageFormatChecker
@Override
public final ImageFormat determineFormat(byte[] headerBytes, int headerSize) {
    if (WebpSupportStatus.isWebpHeader(headerBytes, 0, headerSize)) {
      return getWebpFormat(headerBytes, headerSize);
    }
    
    if (isJpegHeader(headerBytes, headerSize)) {
      return DefaultImageFormats.JPEG;
    }
    
    if (isPngHeader(headerBytes, headerSize)) {
      return DefaultImageFormats.PNG;
    }
    
    if (isGifHeader(headerBytes, headerSize)) {
      return DefaultImageFormats.GIF;
    }
    
    if (isBmpHeader(headerBytes, headerSize)) {
      return DefaultImageFormats.BMP;
    }
    
    if (isIcoHeader(headerBytes, headerSize)) {
      return DefaultImageFormats.ICO;
    }
    
    if (isHeifHeader(headerBytes, headerSize)) {
      return DefaultImageFormats.HEIF;
    }
    
    return ImageFormat.UNKNOWN;
}
```
通过读取图片头部信息去判断。每一张图片的生成都要按照约定的协议，相关协议如下：
- Webp

> 图片来源https://developers.google.com/speed/webp/docs/riff_container

头部信息长度大于等于20个字节，并且头部有ASCII字符RIFF和文件大小以及ASCII字符WEBP。
![https://developers.google.com/speed/webp/docs/riff_container](https://i.loli.net/2019/05/28/5cece217aae9f36043.png)

- JPEG
jpeg格式是也是分段存储相关信息，头信息的segment key是SOI，码值固定是十六进制的FF,D8，而且下一个segment是以FF开头的。所以代码里面的定义是：
```java
private static final byte[] JPEG_HEADER = new byte[] {(byte) 0xFF, (byte)0xD8, (byte)0xFF};
```
见下图：
> 图表来源：https://en.wikipedia.org/wiki/JPEG_File_Interchange_Format

![](https://i.loli.net/2019/05/28/5cece331878a359795.png)

- PNG(Portable Network Graphics)

```java
private static final byte[] PNG_HEADER = new byte[] {
  (byte) 0x89,
  'P', 'N', 'G',
  (byte) 0x0D, (byte) 0x0A, (byte) 0x1A, (byte) 0x0A};
```
最开始高位0x89做数据位识别，50,4E,47是PNG字符，最后连续三个低位的16进制值固定。

见下图：
> 图片来源https://en.wikipedia.org/wiki/Portable_Network_Graphics

![](https://i.loli.net/2019/05/28/5cece4e03ab9b61056.png)

- GIF
格式信息主要在开始六个字节，直接上图：
> 数据详见http://www.onicos.com/staff/iz/formats/gif.html

![](https://i.loli.net/2019/05/28/5cece6cc5b86d82367.png)
代码配置如下：
```java
private static final byte[] GIF_HEADER_87A = ImageFormatCheckerUtils.asciiBytes("GIF87a");
private static final byte[] GIF_HEADER_89A = ImageFormatCheckerUtils.asciiBytes("GIF89a");
```

- BMP
格式信息在头部最开始的两个字节，用来存放BM字符:
> 图片来源http://www.onicos.com/staff/iz/formats/bmp.html

![](https://i.loli.net/2019/05/28/5cece8015afdc87578.png)

```java
private static final byte[] BMP_HEADER = ImageFormatCheckerUtils.asciiBytes("BM");
```

- ICO
格式信息藏在头部的六个字节中，相关定义如下：
> 图片来源https://en.wikipedia.org/wiki/ICO_(file_format)

![](https://i.loli.net/2019/05/28/5cece8c85f93d33196.png)

第一个字节必须是0，第二个字节是1或2，第三个字节定义了图片的数量，代码中定义如下：
```java
//0x00000100
private static final byte[] ICO_HEADER =
  new byte[] {(byte) 0x00, (byte) 0x00, (byte) 0x01, (byte) 0x00};
```

- HEIF
以ftpy字符开始，结构见下图：
> 图片来源https://nokiatech.github.io/heif/technical.html

![](https://nokiatech.github.io/heif/img/isobmff.png)
代码定义如下：
```java
private static final String HEIF_HEADER_PREFIX = "ftyp";

private static final String[] HEIF_HEADER_SUFFIXES = {
"heic", "heix", "hevc", "hevx", "mif1", "msf1"
};
private static final int HEIF_HEADER_LENGTH =
  ImageFormatCheckerUtils.asciiBytes(HEIF_HEADER_PREFIX + HEIF_HEADER_SUFFIXES[0]).length;
```
通过上面的一通分析，可以知道各种图片的封装协议，以及通过各个 参考网站知道数据存储在哪一块。

接下来的操作就是读取图片的分辨率，然后是JPEG和HEIF格式的话，获取图片的角度（横向或竖向），以及旋转角度。

至此生成一个编码后的图片对象的任务算是完成了，主要是将CloseableReference对象封装到EncodedImage对象中然后往上传递数据。