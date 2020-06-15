---
layout: post
title:  "Android-Fresco系列6 图片解码"
date:   2019-6-6 9:53:25 +0800
categories: Android
---

先看看流程图：
![](https://i.loli.net/2019/06/06/5cf8f773b5c6339933.jpg)

## 一、MultiplexProducer
从EncodedMemoryCacheProducer传递来的数据，来到了 MultiplexProducer.ForwardingConsumer中。

#### 1) 初始化
回去看看producer初始化的地方看看其中初始化顺序：
EncodedCacheKeyMultiplexProducer:MultiplexProducer->AddImageTransformMetaDataProducer->ResizeAndRotateProducer)

因为在ProducerSequenceFactory类的newEncodedCacheMultiplexToTranscodeSequence方法中，最后新建了一个EncodedCacheKeyMultiplexProducer对象，他继承自MultiplexProducer类，并且在其构造函数中初始化了一个Map:`Map<K, Multiplexer> mMultiplexers = new HashMap<>();`，而Multiplexer类有一个多路复用的逻辑，将相同的结果传递给多个消费者，并管理取消和持有最后的中间结果。

看看这个Map中保存了哪几个Consumer。要知道Consumer都定义在Producer中，直观的就是这几个Map中put了几个Producer：
```java
//ProducerSequenceFactory.getCommonNetworkFetchToEncodedMemorySequence
Producer<EncodedImage> inputProducer = newEncodedCacheMultiplexToTranscodeSequence();

mCommonNetworkFetchToEncodedMemorySequence = ProducerFactory.newAddImageTransformMetaDataProducer(inputProducer);

mCommonNetworkFetchToEncodedMemorySequence = mProducerFactory.newResizeAndRotateProducer(mCommonNetworkFetchToEncodedMemorySequence,);
```
第一行执行新建一个MultiplexProducer，第二行将producer封装到AddImageTransformMetaDataConsumer，第三行将上一步生成的mCommonNetworkFetchToEncodedMemorySequence封装到ResizeAndRotateProducer。

#### 2) produceResults
再来看看MultiplexProducer的produceResults方法：
```java
//MultiplexProducer
@Override
public void produceResults(Consumer<T> consumer, ProducerContext context) {
  do {
    createdNewMultiplexer = false;
    synchronized (this) {
      multiplexer = getExistingMultiplexer(key);
      if (multiplexer == null) {
        multiplexer = createAndPutNewMultiplexer(key);
        createdNewMultiplexer = true;
      }
    }
    //addNewConsumer may call consumer's onNewResult method immediately. For this reason
    // we release "this" lock. If multiplexer is removed from mMultiplexers in the meantime,
    // which is not very probable, then addNewConsumer will fail and we will be able to retry.
  } while (!multiplexer.addNewConsumer(consumer, context));
  if (createdNewMultiplexer) {
    multiplexer.startInputProducerIfHasAttachedConsumers();
  }
}
```
还是按照上面的步骤，第一步的时候multiplexer为空成立，执行createAndPutNewMultiplexer方法创建一个新的Multiplexer，而后将传递过来的consumer放到Pair<Consumer<T>, ProducerContext>对象中，并存到一个CopyOnWriteArraySet<Pair<Consumer<T>, ProducerContext>>的对象mConsumerContextPairs中。

第一个传递过来的consumer是AbstractProducerToDataSourceAdapter中的内部类BaseConsumer，因为在这个类的构造函数中通过createConsumer函数创建一个。

第二个传递过来的是AddImageTransformMetaDataProducer的Consumer，因为他在上面的第二步的时候被这个Producer包裹。

同时在startInputProducerIfHasAttachedConsumers方法中，内部创建了一个ForwardingConsumer，之后将其往下传递。
```java
//MultiplexProducer.Multiplexer
private void startInputProducerIfHasAttachedConsumers() {
    mForwardingConsumer = new ForwardingConsumer();
    mInputProducer.produceResults(
          forwardingConsumer,
          multiplexProducerContext);
}
```
produceResults往下执行就到了我们上一篇文章分析的EncodedMemoryCacheProducer中，然后EncodedMemoryCacheProducer处理完数据之后就到这里来了。

#### 3) Consumer
好了，这里开始要分析数据了，接收消息的是自己的亲儿子ForwardingConsumer，看看他是怎么处理的：
```java
//MultiplexProducer.Multiplexer.ForwardingConsumer
private class ForwardingConsumer extends BaseConsumer<T> {
  @Override
  protected void onNewResultImpl(T newResult, @Status int status) {
      Multiplexer.this.onNextResult(this, newResult, status);
}
```
原来这玩意不干活，通过遍历，将数据分发给已经保存到的consumer，从后往前发消息：
```java
public void onNextResult(){
    iterator = mConsumerContextPairs.iterator();
    while (iterator.hasNext()) {
        Pair<Consumer<T>, ProducerContext> pair = iterator.next();
        synchronized (pair) {
          pair.first.onNewResult(closeableObject, status);
        }
  }
}
```
就这样消息来到了AddImageTransformMetaDataProducer中。

## 三、AddImageTransformMetaDataProducer
#### 1) onNewResultImpl
这个Producer中的处理很简单：
```java
//AddImageTransformMetaDataProducer
@Override
protected void onNewResultImpl(EncodedImage newResult, @Status int status) {
      if (newResult == null) {
        getConsumer().onNewResult(null, status);
        return;
      }
      if (!EncodedImage.isMetaDataAvailable(newResult)) {
        newResult.parseMetaData();
      }
      getConsumer().onNewResult(newResult, status);
}
```
数据为空直接交给上一个消费者；如果图片数据的元数据不可用就解析，然后再交给上一个消费者。

那么上一个消费者是谁呢？看看他下一个初始化的是ResizeAndRotateProducer，因为这个producer包裹AddImageTransformMetaDataProducer对象。

## 四、 ResizeAndRotateProducer
#### 1) produceResults

在produceResults中定义了一个TransformingConsumer消费者，消费数据之前定义了JobScheduler：
```java
//ResizeAndRotateProducer
JobScheduler.JobRunnable job =
  new JobScheduler.JobRunnable() {
    @Override
    public void run(EncodedImage encodedImage, @Status int status) {
      doTransform(
          encodedImage,
          status,
          Preconditions.checkNotNull(
              mImageTranscoderFactory.createImageTranscoder(
                  encodedImage.getImageFormat(), mIsResizingEnabled)));
    }
};
mJobScheduler = new JobScheduler(mExecutor, job, MIN_TRANSFORM_INTERVAL_MS);
```
这个JobScheduler主要做转码操作。
转码工厂对象mImageTranscoderFactory的初始化是在ImagePipelineFactory中通过getImageTranscoderFactory定义的。不过mNativeCodeDisabled默认是false，看来这个对象对应的应该是MultiImageTranscoderFactory类：
```java
//ImagePipelineFactory
private ImageTranscoderFactory getImageTranscoderFactory() {
    if (mImageTranscoderFactory == null) {
      if (mConfig.getImageTranscoderFactory() == null
          && mConfig.getImageTranscoderType() == null
          && mConfig.getExperiments().isNativeCodeDisabled()) {
        mImageTranscoderFactory =
            new SimpleImageTranscoderFactory(mConfig.getExperiments().getMaxBitmapSize());
      } else {
        mImageTranscoderFactory =
            new MultiImageTranscoderFactory(
             mConfig.getExperiments().getMaxBitmapSize(),
             mConfig.getExperiments().getUseDownsamplingRatioForResizing(),
             mConfig.getImageTranscoderFactory(),
             mConfig.getImageTranscoderType());
      }
    }
    return mImageTranscoderFactory;
}
```

#### 2) onNewResultImpl
精简一下代码，只贴一些关键的代码：
```java
//ResizeAndRotateProducer
@Override
protected void onNewResultImpl(@Nullable EncodedImage newResult, @Status int status) {
    ImageFormat imageFormat = newResult.getImageFormat();
      TriState shouldTransform =
        shouldTransform(mProducerContext.getImageRequest(),newResult,Preconditions.checkNotNull(
        mImageTranscoderFactory.createImageTranscoder(imageFormat, mIsResizingEnabled)));
      if(shouldTransform != "YES"){
          forwardNewResult(newResult, status, imageFormat);
      } else {
          if (isLast ||
          mProducerContext.isIntermediateResultExpected()/*false*/) {
            mJobScheduler.scheduleJob();
        }
    }
}
```
##### 1. shouldTransform

判断是否应该转码：
```java
//ResizeAndRotateProducer
private static TriState shouldTransform(
  ImageRequest request,
  EncodedImage encodedImage,
  ImageTranscoder imageTranscoder) {
    if (encodedImage == null || encodedImage.getImageFormat() == ImageFormat.UNKNOWN) {
      return TriState.UNSET;
    }
    
    if (!imageTranscoder.canTranscode(encodedImage.getImageFormat())) {
      return TriState.NO;
    }
    
    return TriState.valueOf(
        shouldRotate(request.getRotationOptions(), encodedImage)
            || imageTranscoder.canResize(
                encodedImage, request.getRotationOptions(), request.getResizeOptions()));
}
```
传递进来的参数imageTranscoder来自`mImageTranscoderFactory.createImageTranscoder`，对应的是MultiImageTranscoderFactory的createImageTranscoder方法：
```java
//MultiImageTranscoderFactory
@Override
public ImageTranscoder createImageTranscoder(ImageFormat imageFormat, boolean isResizingEnabled) {
    ImageTranscoder imageTranscoder = getNativeImageTranscoder(imageFormat, isResizingEnabled);
    return imageTranscoder == null
        ? getSimpleImageTranscoder(imageFormat, isResizingEnabled)
        : imageTranscoder;
}
```
因为MultiImageTranscoderFactory初始化的时候，mPrimaryImageTranscoderFactory和mImageTranscoderType参数默认为null，所以最终调用的是getNativeImageTranscoder方法获取到了转码处理类。

这个类是：
```java
//NativeImageTranscoderFactory
@Nullable
private ImageTranscoder getNativeImageTranscoder(
  ImageFormat imageFormat, boolean isResizingEnabled) {
    return NativeImageTranscoderFactory.getNativeImageTranscoderFactory(
        mMaxBitmapSize, mUseDownSamplingRatio)
        .createImageTranscoder(imageFormat, isResizingEnabled);
}

//NativeImageTranscoderFactory
public static ImageTranscoderFactory getNativeImageTranscoderFactory(){
    ImageTranscoderFactory imageTranscoderFactory = (ImageTranscoderFactory)
      Class.forName("com.facebook.imagepipeline.nativecode.NativeJpegTranscoderFactory")
          .getConstructor(Integer.TYPE, Boolean.TYPE)
          .newInstance(maxBitmapSize, useDownSamplingRatio);
} 
//NativeJpegTranscoderFactory
public ImageTranscoder createImageTranscoder(ImageFormat imageFormat, boolean isResizingEnabled) {
    if (imageFormat != DefaultImageFormats.JPEG) {
      return null;
    }
    return new NativeJpegTranscoder(isResizingEnabled, mMaxBitmapSize, mUseDownSamplingRatio);
}
```

##### 2. canTranscode

终于找到了最终转码的类NativeJpegTranscoder，在这里做一些是否能够转码的判断canTranscode：
```java
//NativeJpegTranscoder
public boolean canTranscode(ImageFormat imageFormat) {
    return imageFormat == DefaultImageFormats.JPEG;
}
```
只有JPEG格式才转码。

##### 3. shouldRotate
两个条件：1，当图片本身旋转角度不等于0且用户设置的允许在原图片上做旋转操作；2，图片方向上下，左右，左下，右下翻转。
```java
(JpegTranscoderUtils.getRotationAngle(rotationOptions, encodedImage) != 0 ||

INVERTED_EXIF_ORIENTATIONS.contains(encodedImage.getExifOrientation())
```

##### 4. canResize
```java
imageTranscoder.canResize(
    encodedImage, 
    request.getRotationOptions(), 
    request.getResizeOptions()
)
```
imageTranscoder对应的是NativeJpegTranscoder，通过canResize判断是否需要调整：
```java
//NativeJpegTranscoder
public boolean canResize(){
    return JpegTranscoderUtils.getSoftwareNumerator(
    rotationOptions, resizeOptions, encodedImage, mResizingEnabled)
    < JpegTranscoderUtils.SCALE_DENOMINATOR;
}
```
SCALE_DENOMINATOR是调整分母，等于8(*为什么是8？估计与编码数据位有关系，此处不深究*)，getSoftwareNumerator函数是获取分子，当分子小于8的时候就需要调整。
```java
//JpegTranscoderUtils
public static int getSoftwareNumerator(){
    final int rotationAngle = getRotationAngle(rotationOptions, encodedImage);
    int exifOrientation = ExifInterface.ORIENTATION_UNDEFINED;
    if (INVERTED_EXIF_ORIENTATIONS.contains(encodedImage.getExifOrientation())) {
      exifOrientation = getForceRotatedInvertedExifOrientation(rotationOptions, encodedImage);
    }

    final boolean swapDimensions =
        rotationAngle == 90
            || rotationAngle == 270
            || exifOrientation == ExifInterface.ORIENTATION_TRANSPOSE
            || exifOrientation == ExifInterface.ORIENTATION_TRANSVERSE;
    final int widthAfterRotation =
        swapDimensions ? encodedImage.getHeight() : encodedImage.getWidth();
    final int heightAfterRotation =
        swapDimensions ? encodedImage.getWidth() : encodedImage.getHeight();

    float ratio = determineResizeRatio(resizeOptions, widthAfterRotation, heightAfterRotation);
    int numerator = roundNumerator(ratio, resizeOptions.roundUpFraction);
    if (numerator > SCALE_DENOMINATOR) {
      return SCALE_DENOMINATOR;
    }
    return (numerator < 1) ? 1 : numerator;
}
```
- 1.先计算旋转角度，图片本身已经旋转的角度加上用户设置的旋转角度

- 2.获取翻转角度，结合用户设置的旋转角度和图片本身的翻转角度计算

- 3.结合旋转角度和翻转角度判断是否需要交换分辨率，就是宽度和高度互换

- 4.计算宽高的调整比率：
```java
public static float determineResizeRatio(ResizeOptions resizeOptions, int width, int height) {
    if (resizeOptions == null) {
      return 1.0f;
    }
    
    final float widthRatio = ((float) resizeOptions.width) / width;
    final float heightRatio = ((float) resizeOptions.height) / height;
    float ratio = Math.max(widthRatio, heightRatio);
    
    if (width * ratio > resizeOptions.maxBitmapSize) {
      ratio = resizeOptions.maxBitmapSize / width;
    }
    if (height * ratio > resizeOptions.maxBitmapSize) {
      ratio = resizeOptions.maxBitmapSize / height;
    }
    return ratio;
}
```
将用户设置的宽高与当前已经获取的宽高分别相除取最大，然后将限制的最大裁剪图片大小（默认2048）与生成的裁剪宽高对比，大于的话，取比值返回。

- 5.通过roundNumerator方法向上取整
```java
public static int roundNumerator(float maxRatio, float roundUpFraction) {
    return (int) (roundUpFraction + maxRatio * SCALE_DENOMINATOR);
}
```
roundUpFraction默认等于2.0f/3，然后将上一步取到的比率 * 8 加上默认取整分数最后强转成int成为分子。

- 6.最终判断分子小于1返回1，大于1返回分子的值，然后用这个分子值判断是否小于8，小于8的话就要调整图片了。

回到TransformingConsumer的shouldTransform方法中，只要图片格式是JPEG，需要调整分辨率或旋转图片就需要转码了。

##### 5. 转码

如果需要转码调整角度或宽高的话，就需要执行TransformingConsumer类构造函数中的JobScheduler了,其中定义了一个转码函数doTransform：
```java
//TransformingConsumer
private void doTransform(EncodedImage encodedImage, 
        @Status int status, ImageTranscoder imageTranscoder) {
    ImageRequest imageRequest = mProducerContext.getImageRequest();
    PooledByteBufferOutputStream outputStream = mPooledByteBufferFactory.newOutputStream();
    EncodedImage ret;
    ImageTranscodeResult result = imageTranscoder.transcode(
        encodedImage, outputStream,
        imageRequest.getRotationOptions(),
        imageRequest.getResizeOptions(), null, DEFAULT_JPEG_QUALITY
    );
    CloseableReference<PooledByteBuffer> ref = CloseableReference.of(outputStream.toByteBuffer());
    ret = new EncodedImage(ref);
    ret.setImageFormat(JPEG);
    ret.parseMetaData();
    getConsumer().onNewResult(ret, status);
}
```
转码的时候传入了原编码图片，图片的stream数据流，旋转参数，宽高调整参数。

transcode在NativeJpegTranscoder类中定义：
```java
//NativeJpegTranscoder
@Override
public ImageTranscodeResult transcode(
final EncodedImage encodedImage,
final OutputStream outputStream,
@Nullable RotationOptions rotationOptions,
@Nullable final ResizeOptions resizeOptions,
@Nullable ImageFormat outputFormat,
@Nullable Integer quality)
throws IOException {
    final int downsampleRatio = DownsampleUtil.determineSampleSize(
    rotationOptions, resizeOptions, encodedImage, mMaxBitmapSize);
    final int softwareNumerator = JpegTranscoderUtils.getSoftwareNumerator(
    rotationOptions, resizeOptions, encodedImage, mResizingEnabled);
    final int downsampleNumerator = JpegTranscoderUtils.calculateDownsampleNumerator(downsampleRatio);
    final int numerator;
    if (mUseDownsamplingRatio) {
        numerator = downsampleNumerator;
    } else {
        numerator = softwareNumerator;
    }
    InputStream is = encodedImage.getInputStream();
    if (INVERTED_EXIF_ORIENTATIONS.contains(encodedImage.getExifOrientation())) {
        // Use exif orientation to rotate since we can't use the rotation angle for
        // inverted exif orientations
        final int exifOrientation = JpegTranscoderUtils.getForceRotatedInvertedExifOrientation(rotationOptions, encodedImage);
        transcodeJpegWithExifOrientation(is, outputStream, exifOrientation, numerator, quality);
    } else {
        // Use actual rotation angle in degrees to rotate
        final int rotationAngle = JpegTranscoderUtils.getRotationAngle(rotationOptions, encodedImage);
        transcodeJpeg(is, outputStream, rotationAngle, numerator, quality);
    }
}
```
上面这一部分的代码最终在jni中实现了JPEG图片的旋转和伸缩变换，

java端定义的方法是transcodeJpegWithExifOrientation或nativeTranscodeJpeg。

代码执行路径是在：
`fresco\native-imagetranscoder\src\main\jni\native-imagetranscoder\JpegTranscoder.cpp的JpegTranscoder_transcodeJpegWithExifOrientation和JpegTranscoder_transcodeJpeg`。

详细的变换过程此处不详细解读。
> 可以参考:
>
> https://blog.csdn.net/abcjennifer/article/details/8074492
>
> https://blog.csdn.net/abcjennifer/article/details/8074492

回到doTransform方法，我们传递进去转码的outputStream被处理完成之后，获取bytebuffer数据然后就被重新封装进CloseableReference对象，再封装到EncodedImage里面去，重新读取原数据信息，并接着执行onNewResult数据向上传递。

## 五、DecodeProducer
数据接下来传递到了这个消费者，同样的看看produceResults和Consumer吧。

#### 1) produceResults
```java
//DecodeProducer
@Override public void produceResults(){
    ProgressiveDecoder progressiveDecoder;
    if (!UriUtil.isNetworkUri(imageRequest.getSourceUri())) {
        progressiveDecoder =
            new LocalImagesProgressiveDecoder(
                consumer, producerContext, mDecodeCancellationEnabled, mMaxBitmapSize);
      } else {
        ProgressiveJpegParser jpegParser = new ProgressiveJpegParser(mByteArrayPool);
        progressiveDecoder =
            new NetworkImagesProgressiveDecoder(
                consumer,
                producerContext,
                jpegParser,
                mProgressiveJpegConfig,
                mDecodeCancellationEnabled,
                mMaxBitmapSize);
      }
  }
```
针对在线图片和本地图片分为两个Consumer，NetworkImagesProgressiveDecoder和LocalImagesProgressiveDecoder，因为这两个类继承自ProgressiveDecoder，ProgressiveDecoder类继承自DelegatingConsumer。

##### 1. ProgressiveDecoder

NetworkImagesProgressiveDecoder和LocalImagesProgressiveDecoder这两个类中干的活不是太多，主要在父类ProgressiveDecoder中：
```java
//ProgressiveDecoder
public ProgressiveDecoder(){
    JobRunnable job = new JobRunnable() {
        @Override
        public void run(EncodedImage encodedImage, @Status int status) {
            if (!UriUtil.isNetworkUri(request.getSourceUri())) {
                //重新计算采样大小
                encodedImage.setSampleSize(
                    DownsampleUtil.determineSampleSize(
                    request.getRotationOptions(),
                    request.getResizeOptions(),
                    encodedImage,
                    maxBitmapSize)
                );
            }
            doDecode(encodedImage, status);
        }
    }
    mJobScheduler = new JobScheduler(mExecutor, job, mImageDecodeOptions.minDecodeIntervalMs);
}
```
同样的，在构造函数中定义了一个JobScheduler，在数据返回到这里的时候执行解码。

#### 2) onNewResultImpl

接收数据的地方：
```java
//ProgressiveDecoder
@Override
public void onNewResultImpl(EncodedImage newResult, @Status int status) {
final boolean isPlaceholder = statusHasFlag(status, IS_PLACEHOLDER);
    if (isLast || isPlaceholder || mProducerContext.isIntermediateResultExpected()) {
      mJobScheduler.scheduleJob();
    }
}
```
执行scheduleJob方法，开始执行在构造函数中定义的JobScheduler。

#### 3) doDecode

开始解码。
```java
//ProgressiveDecoder
private void doDecode(EncodedImage encodedImage, @Status int status) {
    CloseableImage image = mImageDecoder.decode(encodedImage, length, quality, mImageDecodeOptions);
    handleResult(image, status);
}
```
看看mImageDecoder是怎么初始化的。

这个对象是在ImagePipelineFactory类的getImageDecoder函数中初始化的：
```java
//ImagePipelineFactory
private ImageDecoder getImageDecoder() {
    final AnimatedFactory animatedFactory = getAnimatedFactory();

    ImageDecoder gifDecoder = null;
    ImageDecoder webPDecoder = null;
    
    if (animatedFactory != null) {
        gifDecoder = animatedFactory.getGifDecoder(mConfig.getBitmapConfig());
        webPDecoder = animatedFactory.getWebPDecoder(mConfig.getBitmapConfig());
    }
    mImageDecoder = new DefaultImageDecoder(gifDecoder, webPDecoder, getPlatformDecoder());
    return mImageDecoder;
}
```
可见mImageDecoder对象对应的是DefaultImageDecoder类，在构造的时候，第三个参数getPlatformDecoder用于选择平台的解码类：
```java
//ImagePipelineFactory
public PlatformDecoder getPlatformDecoder() {
    if (mPlatformDecoder == null) {
      mPlatformDecoder =
          PlatformDecoderFactory.buildPlatformDecoder(
              mConfig.getPoolFactory(), mConfig.getExperiments().isGingerbreadDecoderEnabled());
    }
    return mPlatformDecoder;
}
```
直接看buildPlatformDecoder方法吧，看看不同的Android版本选择了不同的解码类：
```java
//PlatformDecoderFactory
public static PlatformDecoder buildPlatformDecoder(
  PoolFactory poolFactory, boolean gingerbreadDecoderEnabled) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
      int maxNumThreads = poolFactory.getFlexByteArrayPoolMaxNumThreads();
      return new OreoDecoder(
          poolFactory.getBitmapPool(), maxNumThreads, new Pools.SynchronizedPool<>(maxNumThreads));
    } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
      int maxNumThreads = poolFactory.getFlexByteArrayPoolMaxNumThreads();
      return new ArtDecoder(
          poolFactory.getBitmapPool(), maxNumThreads, new Pools.SynchronizedPool<>(maxNumThreads));
    } else {
      if (gingerbreadDecoderEnabled && Build.VERSION.SDK_INT < Build.VERSION_CODES.KITKAT) {
        return new GingerbreadPurgeableDecoder();
      } else {
        return new KitKatPurgeableDecoder(poolFactory.getFlexByteArrayPool());
      }
    }
}
```
当前我的测试机器是Android O，就以OreoDecoder为例往下走。

最后一个参数maxNumThreads的值等于：`public static final int DEFAULT_MAX_NUM_THREADS = Runtime.getRuntime().availableProcessors();`
就是可用的处理器个数。

好了，回到decode方法，已经知道mImageDecoder对象对应的是DefaultImageDecoder类，那么decode方法定义在DefaultImageDecoder类中：
```java
//DefaultImageDecoder
@Override
public CloseableImage decode(
  final EncodedImage encodedImage,
  final int length,
  final QualityInfo qualityInfo,
  final ImageDecodeOptions options) {
    return mDefaultDecoder.decode(encodedImage, length, qualityInfo, options);
}
```
mDefaultDecoder在这个类中定义：
```java
//DefaultImageDecoder
private final ImageDecoder mDefaultDecoder =
  new ImageDecoder() {
    @Override
    public CloseableImage decode(
        EncodedImage encodedImage,
        int length,
        QualityInfo qualityInfo,
        ImageDecodeOptions options) {
      ImageFormat imageFormat = encodedImage.getImageFormat();
      if (imageFormat == DefaultImageFormats.JPEG) {
        return decodeJpeg(encodedImage, length, qualityInfo, options);
      } else if (imageFormat == DefaultImageFormats.GIF) {
        return decodeGif(encodedImage, length, qualityInfo, options);
      } else if (imageFormat == DefaultImageFormats.WEBP_ANIMATED) {
        return decodeAnimatedWebp(encodedImage, length, qualityInfo, options);
      } else if (imageFormat == ImageFormat.UNKNOWN) {
        throw new DecodeException("unknown image format", encodedImage);
      }
      return decodeStaticImage(encodedImage, options);
    }
};
```

#### 4) 解码JPEG
以JPEG为例，执行decodeJpeg方法：
```java
//DefaultImageDecoder
public CloseableStaticBitmap decodeJpeg(
  final EncodedImage encodedImage,
  int length,
  QualityInfo qualityInfo,
  ImageDecodeOptions options) {
    CloseableReference<Bitmap> bitmapReference =
        mPlatformDecoder.decodeJPEGFromEncodedImageWithColorSpace(
            encodedImage, options.bitmapConfig, null, length, options.colorSpace);
    try {
      maybeApplyTransformation(options.bitmapTransformation, bitmapReference);
      return new CloseableStaticBitmap(
          bitmapReference,
          qualityInfo,
          encodedImage.getRotationAngle(),
          encodedImage.getExifOrientation());
    } finally {
      bitmapReference.close();
    }
}
```
此处mPlatformDecoder对应的就是Android O版本的解码类OreoDecoder，在OreoDecoder类中没有decodeJPEGFromEncodedImageWithColorSpace方法，其定义在父类DefaultDecoder中：
```java
 //DefaultDecoder
 @Override
public CloseableReference<Bitmap> decodeJPEGFromEncodedImageWithColorSpace(
    EncodedImage encodedImage,
    Bitmap.Config bitmapConfig,
    @Nullable Rect regionToDecode,
    int length, @Nullable final ColorSpace colorSpace) {
    boolean isJpegComplete = encodedImage.isCompleteAt(length);
    final BitmapFactory.Options options = getDecodeOptionsForStream(encodedImage, bitmapConfig);
    InputStream jpegDataStream = encodedImage.getInputStream();
    return decodeFromStream(jpegDataStream, options, regionToDecode, colorSpace);
}
```
##### 1.读取jpeg的尾部数据判断数据是否完整

##### 2.获取解码参数，getDecodeOptionsForStream：
```java
//DefaultDecoder
private static BitmapFactory.Options getDecodeOptionsForStream(
  EncodedImage encodedImage, Bitmap.Config bitmapConfig) {
    final BitmapFactory.Options options = new BitmapFactory.Options();
    // Sample size should ONLY be different than 1 when downsampling is enabled in the pipeline
    options.inSampleSize = encodedImage.getSampleSize();
    options.inJustDecodeBounds = true;
    // fill outWidth and outHeight
    BitmapFactory.decodeStream(encodedImage.getInputStream(), null, options);
    if (options.outWidth == -1 || options.outHeight == -1) {
      throw new IllegalArgumentException();
    }
    
    options.inJustDecodeBounds = false;
    options.inDither = true;
    options.inPreferredConfig = bitmapConfig;
    options.inMutable = true;
    
    return options;
}
```
> https://developer.android.com/reference/android/graphics/BitmapFactory.Options

- inSampleSize

采样大小，根据设定的宽高和图片的宽高综合，获取方法在DownsampleUtil.determineSampleSize中。
> If set to a value > 1, requests the decoder to subsample the original image, returning a smaller image to save memory. The sample size is the number of pixels in either dimension that correspond to a single pixel in the decoded bitmap. For example, inSampleSize == 4 returns an image that is 1/4 the width/height of the original, and 1/16 the number of pixels. Any value <= 1 is treated the same as 1. Note: the decoder uses a final value based on powers of 2, any other value will be rounded down to the nearest power of 2.
- inJustDecodeBounds

不为像素数据分配内存，但是可以查询Bitmap的数据
- inDither

是否抖动处理
> https://blog.csdn.net/love_xsq/article/details/50387260

- inPreferredConfig

颜色模式选择，如果设置的话，就优先按照设置的来，否则按照图片已有的颜色模式设置。sdk默认设置的是ARGB_8888。
- inMutable

返回的bitmap是否可变，设置成可变之后，可以针对返回的Bitmap做一些修改。
> If set, decode methods will always return a mutable Bitmap instead of an immutable one. This can be used for instance to programmatically apply effects to a Bitmap loaded through BitmapFactory.
>
> Can not be set simultaneously with inPreferredConfig = Bitmap.Config.HARDWARE, because hardware bitmaps are always immutable.

#### 5) decodeFromStream

开始解码了：
```java
//DefaultDecoder
private CloseableReference<Bitmap> decodeFromStream(
  InputStream inputStream,
  BitmapFactory.Options options,
  @Nullable Rect regionToDecode,
  @Nullable final ColorSpace colorSpace) {
    final int sizeInBytes = getBitmapSize(targetWidth, targetHeight, options);
    @Nullable Bitmap bitmapToReuse = null;
    bitmapToReuse = mBitmapPool.get(sizeInBytes);
    options.inBitmap = bitmapToReuse;
    Bitmap decodedBitmap = null;
    ByteBuffer byteBuffer = mDecodeBuffers.acquire();
    if (byteBuffer == null) {
        byteBuffer = ByteBuffer.allocate(DECODE_BUFFER_SIZE);
    }
    if (regionToDecode != null && bitmapToReuse != null) {
        BitmapRegionDecoder bitmapRegionDecoder = null;
        bitmapToReuse.reconfigure(targetWidth, targetHeight, options.inPreferredConfig);
        bitmapRegionDecoder = BitmapRegionDecoder.newInstance(inputStream, true);
        decodedBitmap = bitmapRegionDecoder.decodeRegion(regionToDecode, options);
    } 
    if (decodedBitmap == null) {
        decodedBitmap = BitmapFactory.decodeStream(inputStream, null, options);
    }
    return CloseableReference.of(decodedBitmap, mBitmapPool);
}
```
##### 1.获取图片所有像素点的byte总数：宽 x 高 x config位（ARGB = 4位）
##### 2.通过byte大小获取分配一个可复用的Bitmap

mBitmapPool的初始化是：
```java
//PlatformDecoderFactory
return new OreoDecoder(
  poolFactory.getBitmapPool(), maxNumThreads, new Pools.SynchronizedPool<>(maxNumThreads));
```
这里的BitmapPool默认是BucketsBitmapPool,继承自BasePool类。看看get方法，定义在BasePool中:
```java
//BasePool
 public V get(int size) {
    int bucketedSize = getBucketedSize(size);
    Bucket<V> bucket = getBucket(bucketedSize);
    if (bucket != null) {
        // find an existing value that we can reuse
        V value = getValue(bucket);
        if (value != null) {
            bucketedSize = getBucketedSizeForValue(value);
            sizeInBytes = getSizeInBytes(bucketedSize);
            mUsed.increment(sizeInBytes);
            mFree.decrement(sizeInBytes);
            return value;
        }
    }
    sizeInBytes = getSizeInBytes(bucketedSize);
    mUsed.increment(sizeInBytes);
    V value = alloc(bucketedSize);
    mInUseValues.add(value)
    trimToSoftCap();
    return value;
}

//BasePool
synchronized Bucket<V> getBucket(int bucketedSize) {
    // get an existing bucket
    Bucket<V> bucket = mBuckets.get(bucketedSize);
    if (bucket != null || !mAllowNewBuckets) {
      return bucket;
    }
    
    Bucket<V> newBucket = newBucket(bucketedSize);
    mBuckets.put(bucketedSize, newBucket);
    return newBucket;
}

//BasePool
Bucket<V> newBucket(int bucketedSize) {
    return new Bucket<V>(
        /*itemSize*/ getSizeInBytes(bucketedSize),
        /*maxLength*/ Integer.MAX_VALUE,
        /*inUseLength*/ 0,
        mPoolParams.fixBucketsReinitialization);
}
```
贴了三个函数，粗略讲一下这三个函数干了啥事情：
- Bucket是一个系统已分配对象的管理类，里面封装了对象的大小（itemSize），已分配对象被释放后的队列(freeList)等参数，泛型类型是Bitmap
- mBuckets是一个以Bucket的大小为key，Buckey为value的SparseArray。同时在BasePool中有两个Counter，用于计算空闲和已使用的空间大小：
```java
mFree = new Counter();
mUsed = new Counter();
```
当我们从 mBuckets里面找到一个相同大小的Bucket，就将mUsed加上这个Bucket大小，同时将mFree减去这个大小。

每次分配之前要调用canAllocate检查我们已经分配的大小是否超过限制，是的话就要调用trimToSize调整大小。

另外每次分配完了之后还要调用trimToSoftCap方法检查已经分配的值，超过的话也要调用trimToSize调整。

- 从上面的代码可以看出，来了一个复用分配请求的时候，先从mBuckets里面去找有没有相同大小的Bucket，有的话，就直接返回Bucket的value，否则的话新建一个alloc：
```java
//BucketsBitmapPool
@Override
protected Bitmap alloc(int size) {
    return Bitmap.createBitmap(
        1,
        (int) Math.ceil(size / (double) BitmapUtil.RGB_565_BYTES_PER_PIXEL),
        Bitmap.Config.RGB_565);
}
```
宽度为1，高度为像素位大小 / 2,像素位参数设置为RGB_565。

##### 3.将bitmapToReuse设置到可复用选项
`options.inBitmap = bitmapToReuse;`。

##### 4.如果设置的编码区域不为空（默认为null），就按照设定区域编码，否则的话执行BitmapFactory.decodeStream生成一个decodedBitmap。

##### 5.返回`CloseableReference.of(decodedBitmap, mBitmapPool)`。

解析到这里仿佛已经忘了开始的地方，在DefaultImageDecoder的decodeJpeg方法中，我们返回了解码后的数据给到CloseableReference<Bitmap> bitmapReference对象，然后将这个对象封装到了CloseableStaticBitmap对象中。

```java
//DefaultImageDecoder:decodeJpeg
return new CloseableStaticBitmap(
          bitmapReference,
          qualityInfo,
          encodedImage.getRotationAngle(),
          encodedImage.getExifOrientation());
```

接下来这个数据会返回到DecodeProducer.ProgressiveDecoder的doDecode方法中，在调用decode方法之后，我们实际获取到了CloseableStaticBitmap对象，然后执行`handleResult(image, status);`:
```java
//ProgressiveDecoder
private void handleResult(final CloseableImage decodedImage, final @Status int status) {
    CloseableReference<CloseableImage> decodedImageRef = CloseableReference.of(decodedImage);
    getConsumer().onNewResult(decodedImageRef, status);
}
```
好了，数据接着往上传递给下一个消费者。

