---
layout: post
title:  "Android-Fresco系列1 初始化"
date:   2019-6-4 9:53:25 +0800
categories: Android
---

先看流程图：
![](https://i.loli.net/2019/06/04/5cf621638b46913598.jpg)

## 一、开始使用
- 在工程的app目录下的build.gradle添加引用：
```groovy
implementation 'com.facebook.fresco:fresco:1.12.0'
```
- Application类中的onCreate方法中添加初始化：
```java
Fresco.initialize(this)
```

- layout xml文件中添加控件定义
```xml
<com.facebook.drawee.view.SimpleDraweeView
        android:id="@+id/my_image_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        fresco:placeholderImage="@drawable/ic_launcher_background"
```

- Activity中添加图片展示
```java
val uri = Uri.parse("http://ww1.sinaimg.cn/large/610dc034ly1fjaxhky81vj20u00u0ta1.jpg")
val draweeView = findViewById<SimpleDraweeView>(R.id.my_image_view)
draweeView.setImageURI(uri)
```

## 二、初始化
#### 1、Fresco
从Fresco.initialize开始说起。
```java
public static void initialize(Context context) {
initialize(context, null, null);
}

public static void initialize(
  Context context,
  @Nullable ImagePipelineConfig imagePipelineConfig,
  @Nullable DraweeConfig draweeConfig) {
    try {
      SoLoader.init(context, 0);
    } catch (IOException e) {
    }
    // we should always use the application context to avoid memory leaks
    context = context.getApplicationContext();
    if (imagePipelineConfig == null) {
      ImagePipelineFactory.initialize(context);
    } else {
      ImagePipelineFactory.initialize(imagePipelineConfig);
    }
    initializeDrawee(context, draweeConfig);
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
}
```
从初始化代码引申出两个配置类ImagePipelineConfig和DraweeConfig。

#### 2、ImagePipelineFactory
> ImagePipeline工厂类

##### 1) ImagePipelineConfig
这个类为ImagePipeline配置参数，是核心的类。
配置的参数主要有：Bitmap参数，内存缓存相关参数，磁盘缓存相关参数等。
不设置这个参数的时候sdk会默认初始化：
```java
//ImagePipelineFactory:initialize
public static synchronized void initialize(Context context) {
    initialize(ImagePipelineConfig.newBuilder(context).build());
}
//ImagePipelineConfig.Builder
public ImagePipelineConfig build() {
  return new ImagePipelineConfig(this);
}
//ImagePipelineConfig
private ImagePipelineConfig(Builder builder) {
    DefaultBitmapMemoryCacheParamsSupplier//内存缓存
    BitmapMemoryCacheTrimStrategy//内存缓存清理策略
    CacheKeyFactory//缓存数据key，默认DefaultCacheKeyFactory
    FileCacheFactory//文件缓存，默认DiskStorageCacheFactory(DynamicDefaultDiskStorageFactory)
    MemoryCacheParams//内存缓存配置，默认DefaultEncodedMemoryCacheParamsSupplier
    ImageDecoder//图片解码，创建一个CloseableImage对象
    ImageTranscoderFactory//图片变换，resize，如果设置resizeable为true才有效
    DiskCacheConfig//磁盘缓存配置，不设置的话，sdk默认配置一个
    MemoryTrimmableRegistry//注册系统内存剩余容量通知
    NetworkFetcher//在线图片下载类，默认初始化为HttpUrlConnectionNetworkFetcher，使用HttpUrlConnection下载
    PlatformBitmapFactory//依据Bitmap参数生成优化后的bitmao
    PoolFactory//参数配置池，定义一些对象池
    ProgressiveJpegConfig//jpeg配置进度
    ImageDecoderConfig//图片解码配置
    ExecutorSupplier//不同场景下的线程池提供者，默认配置DefaultExecutorSupplier
    ...
}
```
在此会创建一个ImagePipelineConfig类，包含各种图片的配置信息。
接下来初始化一个ImagePipelineFactory类：
```java
//ImagePipelineFactory
public static synchronized void initialize(ImagePipelineConfig imagePipelineConfig) {
    sInstance = new ImagePipelineFactory(imagePipelineConfig);
}

public ImagePipelineFactory(ImagePipelineConfig config) {
    mConfig = Preconditions.checkNotNull(config);
    mThreadHandoffProducerQueue =
        new ThreadHandoffProducerQueue(
            config.getExecutorSupplier().forLightweightBackgroundTasks());
}
```
config.getExecutorSupplier()是DefaultExecutorSupplier对象，这个ThreadHandoffProducerQueue类用来管理线程池队列，管理队列的开始停止，执行线程池。

#### 3、DraweeConfig
```java
//Fresco
private static void initializeDrawee(Context context, @Nullable DraweeConfig draweeConfig) {
    sDraweeControllerBuilderSupplier =
        new PipelineDraweeControllerBuilderSupplier(context, draweeConfig);
    SimpleDraweeView.initialize(sDraweeControllerBuilderSupplier);
}
```
##### 1) PipelineDraweeControllerBuilderSupplier
在这个类中初始化了ImagePipeline,并剔除部分不关心的代码：
```java
public PipelineDraweeControllerBuilderSupplier(
      Context context,
      ImagePipelineFactory imagePipelineFactory,
      Set<ControllerListener> boundControllerListeners,
      @Nullable DraweeConfig draweeConfig) {
    mImagePipeline = imagePipelineFactory.getImagePipeline();

    if (draweeConfig != null && draweeConfig.getPipelineDraweeControllerFactory() != null) {
      mPipelineDraweeControllerFactory = draweeConfig.getPipelineDraweeControllerFactory();
    } else {
      mPipelineDraweeControllerFactory = new PipelineDraweeControllerFactory();
    }
    mPipelineDraweeControllerFactory.init();
}
```
##### 2) ImagePipelineFactory
初始化ImagePipeline的过程在ImagePipelineFactory类中：
```java
public getImagePipeline() {
    if (mImagePipeline == null) {
      mImagePipeline =
          new ImagePipeline(
              getProducerSequenceFactory(),
              mConfig.getRequestListeners(),
              mConfig.getIsPrefetchEnabledSupplier(),
              getBitmapMemoryCache(),
              getEncodedMemoryCache(),
              getMainBufferedDiskCache(),
              getSmallImageBufferedDiskCache(),
              mConfig.getCacheKeyFactory(),
              mThreadHandoffProducerQueue,
              Suppliers.of(false),
              mConfig.getExperiments().isLazyDataSource(),
              mConfig.getCallerContextVerifier());
    }
    return mImagePipeline;
}
```
##### 3) PipelineDraweeControllerFactory
接下来初始化一个PipelineDraweeControllerFactory类，并调用其init方法：
```java
mPipelineDraweeControllerFactory = new PipelineDraweeControllerFactory();
mPipelineDraweeControllerFactory.init(Resources,DeferredReleaser,DrawableFactory,UiThreadImmediateExecutorService,MemoryCache,DrawableFactory);
```
##### 4) ImagePipeline
在getImagePipeline函数的参数中初始化了几个类，分别是：
- ProducerSequenceFactory//(getProducerSequenceFactory)
- ProducerFactory//(getProducerSequenceFactory 方法中初始化ProducerSequenceFactory类获取该对象)
- ContentResolver(初始化ProducerSequenceFactory类获取该对象)
- ProducerFactor(在初始化ProducerSequenceFactory类时调用getProducerFactory方法)
- BitmapMemoryCacheFactory(InstrumentedMemoryCache<CountingMemoryCache>)
- EncodedMemoryCacheFactory(InstrumentedMemoryCache<CountingMemoryCache>)
- BufferedDiskCache(mMainBufferedDiskCache<DiskCacheConfig>)
- BufferedDiskCache(mSmallImageFileCache<DiskCacheConfig>)

这个类的初始化牵涉到很多工作类的初始化，Producer是生产者，主要生产编解码类，缓存处理类，Bitmap生产类等。

#### 3、SimpleDraweeView
这个类在初始化的时候只初始化了一个对象：
```java
SimpleDraweeView.initialize(sDraweeControllerBuilderSupplier);
public static void initialize(
  Supplier<? extends AbstractDraweeControllerBuilder> draweeControllerBuilderSupplier) {
    sDraweecontrollerbuildersupplier = draweeControllerBuilderSupplier;
}
```

## 三、SimpleDraweeView初始化
#### 1) 构造函数
当在xml中定义了SimpleDraweeView，在Activity的onCreate方法中开始加载图片的时候，首先会初始化这个View控件:
```java
//SimpleDraweeView
@TargetApi(Build.VERSION_CODES.LOLLIPOP)
public SimpleDraweeView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
    super(context, attrs, defStyleAttr, defStyleRes);
    init(context, attrs);
}
```
#### 2) super

先调用父类构造函数，父类的继承关系是GenericDraweeView -> DraweeView -> ImageView.
- DraweeView

在构造函数中，先调用父类ImageView的构造函数，再执行自己的init函数：
```java
//DraweeView
private void init(Context context) {
    mDraweeHolder = DraweeHolder.create(null, context);
}
//DraweeHolder
public static <DH extends DraweeHierarchy> DraweeHolder<DH> create(
  @Nullable DH hierarchy,
  Context context) {
    DraweeHolder<DH> holder = new DraweeHolder<DH>(hierarchy);
    holder.registerWithContext(context);
    return holder;
}
```
#### 3) GenericDraweeView

在构造函数中，先调用父类DraweeView的构造函数，再运行自己的inflateHierarchy方法：
```java
//GenericDraweeView
protected void inflateHierarchy(Context context, @Nullable AttributeSet attrs) {
    GenericDraweeHierarchyBuilder builder =
        GenericDraweeHierarchyInflater.inflateBuilder(context, attrs);
    setAspectRatio(builder.getDesiredAspectRatio());
    setHierarchy(builder.build());
}

//GenericDraweeHierarchyInflater
public static GenericDraweeHierarchyBuilder inflateBuilder(
  Context context, @Nullable AttributeSet attrs) {
    Resources resources = context.getResources();
    GenericDraweeHierarchyBuilder builder = new GenericDraweeHierarchyBuilder(resources);
    builder = updateBuilder(builder, context, attrs);
    return builder;
}
```
创建一个GenericDraweeHierarchyBuilder对象，然后执行updateBuilder方法，在这个方法中，读取所有xml文件中设置的参数，并将参数统一封装到GenericDraweeHierarchyBuilder对象中。

#### 4) GenericDraweeHierarchy
> Hierarchy意思是等级制度，阶层的意思

接下来执行builder.build()生成一个GenericDraweeHierarchy对象：
```java
public GenericDraweeHierarchy build() {
    validate();
    return new GenericDraweeHierarchy(this);
}
```
GenericDraweeHierarchy的继承关系是：GenericDraweeHierarchy -> SettableDraweeHierarchy -> DraweeHierarchy。

GenericDraweeHierarchy构造函数中做了以下几件事情：
- RoundingParams 圆形图片参数
- ForwardingDrawable 图片Drawable加载状态回调
- layers，Drawable数组类型，从下到上定义了六层：
```java
private static final int BACKGROUND_IMAGE_INDEX = 0;//背景
private static final int PLACEHOLDER_IMAGE_INDEX = 1;//占位
private static final int ACTUAL_IMAGE_INDEX = 2;//实际展示图
private static final int PROGRESS_BAR_IMAGE_INDEX = 3;//进度
private static final int RETRY_IMAGE_INDEX = 4;//重试
private static final int FAILURE_IMAGE_INDEX = 5;//失败
private static final int OVERLAY_IMAGES_INDEX = 6;//图片覆盖
```
- FadeDrawable 渐变Drawable，构造函数参数是layers
- RootDrawable 根Drawable，可以回调Drawable visiable变化，在Drawable不可见时不调用绘制方法

接下来执行setHierarchy将DraweeHierarchy对象设置到父类DraweeView中：
```java
//DraweeView
public void setHierarchy(DH hierarchy) {
    mDraweeHolder.setHierarchy(hierarchy);
    super.setImageDrawable(mDraweeHolder.getTopLevelDrawable());
}
```
mDraweeHolder对象在DraweeView构造函数中初始化。
#### 5) init
> SimpleDraweeView中

init方法中判断当前是否是编辑模式，`isInEditMode`，例如在Android Studio中设计布局的时候，View的这个方法可以返回true，此时可以自定义一个画布，避免开发的过程中无法实时预览到自定义控件的轮廓。

另外还读取了在视图xml文件中对SimpleDraweeView设置的一些参数。

同时如果不是编辑模式的时候，初始化了一个：
```java
mControllerBuilder = sDraweecontrollerbuildersupplier.get();
```
这个sDraweecontrollerbuildersupplier对象对应的是PipelineDraweeControllerBuilderSupplier类，get方法如下：
```java
@Override
public PipelineDraweeControllerBuilder get() {
    PipelineDraweeControllerBuilder pipelineDraweeControllerBuilder =
        new PipelineDraweeControllerBuilder(
            mContext, mPipelineDraweeControllerFactory, mImagePipeline, mBoundControllerListeners);
    return pipelineDraweeControllerBuilder.setPerfDataListener(mDefaultImagePerfDataListener);
}
```
这里新建了一个PipelineDraweeControllerBuilder类，封装了ImagePipeline和PipelineDraweeControllerFactory对象。这个类继承自AbstractDraweeControllerBuilder，该类实现了SimpleDraweeControllerBuilder接口。

此时初始化控件完毕，接下来开始设置图片资源并开始加载。