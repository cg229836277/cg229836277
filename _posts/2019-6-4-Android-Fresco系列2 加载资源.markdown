---
layout: post
title:  "Android-Fresco系列2 加载资源"
date:   2019-6-4 9:53:25 +0800
categories: Android
---

流程图如下：
![](https://i.loli.net/2019/06/04/5cf665898a67728487.jpg)

## 一、SimpleDraweeView加载图片
```java
val draweeView = findViewById<SimpleDraweeView>(R.id.my_image_view)
draweeView.setImageURI("http://ww1.sinaimg.cn/large/610dc034ly1fjaxhky81vj20u00u0ta1.jpg")
```
通过setImageURI函数设置图片加载，参数是Uri参数或图片地址。

#### 1) setImageURI
```java
//SimpleDraweeView
public void setImageURI(Uri uri, @Nullable Object callerContext) {
    DraweeController controller =
        mControllerBuilder
            .setCallerContext(callerContext)
            .setUri(uri)
            .setOldController(getController())
            .build();
    setController(controller);
}
```
从这个方法开始了图片加载之旅。比较重要的是两个类PipelineDraweeControllerBuilder，AbstractDraweeControllerBuilder，前者继承自后者。
#### 2) PipelineDraweeControllerBuilder
- callerContext参数默认为null。
- mControllerBuilder对象在SimpleDraweeView的init方法中初始化。
- setUri在PipelineDraweeControllerBuilder类中
```java
//PipelineDraweeControllerBuilder
@Override
public PipelineDraweeControllerBuilder setUri(@Nullable Uri uri) {
    if (uri == null) {
      return super.setImageRequest(null);
    }
    ImageRequest imageRequest = ImageRequestBuilder.newBuilderWithSource(uri)
        .setRotationOptions(RotationOptions.autoRotateAtRenderTime())
        .build();
    return super.setImageRequest(imageRequest);
}
```
#### 3) ImageRequest
ImageRequestBuilder类构建了一个ImageRequest对象，build方法是：
```java
//ImageRequestBuilder
public ImageRequest build() {
    validate();
    return new ImageRequest(this);
}
```
validate用于检查图片地址。

ImageRequest构造函数主要初始化了以下参数：
```java
mCacheChoice//缓存选择
mSourceUri//地址
mSourceUriType//地址类型，网络，本地，asset，resource
mProgressiveRenderingEnabled//进度渲染是否允许
mLocalThumbnailPreviewsEnabled//本地略缩图预览是否允许
mImageDecodeOptions//图片解码选项
mResizeOptions//裁剪
mRotationOptions//旋转
mIsDiskCacheEnabled//磁盘缓存
mIsMemoryCacheEnabled//内存缓存
...
```
#### 4) AbstractDraweeControllerBuilder
最后还要调用一下父类AbstractDraweeControllerBuilder的setImageRequest方法，初始化以下mImageRequest对象：
```java
//AbstractDraweeControllerBuilder
public BUILDER setImageRequest(REQUEST imageRequest) {
    mImageRequest = imageRequest;
    return getThis();
}
```
- setOldController

getController方法先获取一下前一个Controller，然后设置到AbstractDraweeControllerBuilder类中。

- build

定义在AbstractDraweeControllerBuilder类中：
```java
//AbstractDraweeControllerBuilder
@Override
public AbstractDraweeController build() {
    return buildController();
}
//AbstractDraweeControllerBuilder
protected AbstractDraweeController buildController() {
    AbstractDraweeController controller = obtainController();
    return controller;
}

```

#### 5) PipelineDraweeControllerBuilder
- obtainController

obtainController方法定义在PipelineDraweeControllerBuilder类中：
```java
//PipelineDraweeControllerBuilder
@Override
protected PipelineDraweeController obtainController() {
    try {
      DraweeController oldController = getOldController();
      PipelineDraweeController controller;
      final String controllerId = generateUniqueControllerId();
      if (oldController instanceof PipelineDraweeController) {
        controller = (PipelineDraweeController) oldController;
      } else {
        controller = mPipelineDraweeControllerFactory.newController();
      }
      controller.initialize(
          obtainDataSourceSupplier(controller, controllerId),
          controllerId,
          getCacheKey(),
          getCallerContext(),
          mCustomDrawableFactories,
          mImageOriginListener);
      return controller;
    } finally {
    }
}
```
设置Controller的时候，先判断前一个controller是否存在，存在的话获取旧的，并强转成PipelineDraweeController类型，否则新建一个。

#### 6) PipelineDraweeControllerFactory
- newController

一般情况下，第一次加载都是新建：
```java
//PipelineDraweeControllerFactory
public PipelineDraweeController newController() {
    PipelineDraweeController controller =
        internalCreateController(
            mResources,
            mDeferredReleaser,
            mAnimatedDrawableFactory,
            mUiThreadExecutor,
            mMemoryCache,
            mDrawableFactories);
    return controller;
}

protected PipelineDraweeController internalCreateController() {
    return new PipelineDraweeController(
        resources,
        deferredReleaser,
        animatedDrawableFactory,
        uiThreadExecutor,
        memoryCache,
        drawableFactories);
}
```

最终返回一个PipelineDraweeController对象。然后执行对象的initialize方法，在执行这个方法之前，里面有一个参数是通过obtainDataSourceSupplier方法获取的,另外一个是getCacheKey，先看看这两个方法再(这个方法涉及的流程比较长，删除部分不关心的代码)：

- obtainDataSourceSupplier
```java
//AbstractDraweeControllerBuilder
protected Supplier<DataSource<IMAGE>> obtainDataSourceSupplier(
  final DraweeController controller, final String controllerId) {
    Supplier<DataSource<IMAGE>> supplier = null;
    
    // final image supplier;
    if (mImageRequest != null) {
      supplier = getDataSourceSupplierForRequest(controller, controllerId, mImageRequest);
    } else if (mMultiImageRequests != null) {
      supplier = getFirstAvailableDataSourceSupplier(
              controller, controllerId, mMultiImageRequests, mTryCacheOnlyFirst);
    }
    return supplier;
}
```
即使走getFirstAvailableDataSourceSupplier分支，也会调用getDataSourceSupplierForRequest方法，看看这个方法：
- getDataSourceSupplierForRequest
```java
//AbstractDraweeControllerBuilder
/** Creates a data source supplier for the given image request. */
protected Supplier<DataSource<IMAGE>> getDataSourceSupplierForRequest(
  final DraweeController controller,
  final String controllerId,
  final REQUEST imageRequest,
  final CacheLevel cacheLevel) {
    final Object callerContext = getCallerContext();
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
}
```
到这里可以缓一缓，因为返回了一个Supplier<DataSource>对象，而且新建这个对象的时候在实现get方法的时候返回了一个getDataSourceForRequest方法，这个方法在PipelineDraweeControllerBuilder类中定义，先不看。

- getCacheKey
获取缓存key。
```java
//PipelineDraweeControllerBuilder
private @Nullable CacheKey getCacheKey() {
    final ImageRequest imageRequest = getImageRequest();
    final CacheKeyFactory cacheKeyFactory = mImagePipeline.getCacheKeyFactory();
    CacheKey cacheKey = null;
    if (cacheKeyFactory != null && imageRequest != null) {
      if (imageRequest.getPostprocessor() != null) {
        cacheKey = cacheKeyFactory.getPostprocessedBitmapCacheKey(
            imageRequest,
            getCallerContext());
      } else {
        cacheKey = cacheKeyFactory.getBitmapCacheKey(
            imageRequest,
            getCallerContext());
      }
    }
    return cacheKey;
}
```
cacheKeyFactory的初始化是在ImagePipelineConfig的构造函数中：
```java
//ImagePipelineConfig
mCacheKeyFactory = builder.mCacheKeyFactory == null
        ? DefaultCacheKeyFactory.getInstance()
        : builder.mCacheKeyFactory;
```
看看DefaultCacheKeyFactory类中怎么返回缓存key的：
```java
//DefaultCacheKeyFactory
@Override
public CacheKey getBitmapCacheKey(ImageRequest request, Object callerContext) {
    return new BitmapMemoryCacheKey(
        getCacheKeySourceUri(request.getSourceUri()).toString(),
        request.getResizeOptions(),
        request.getRotationOptions(),
        request.getImageDecodeOptions(),
        null,
        null,
        callerContext);
}
```
其实是返回一个BitmapMemoryCacheKey对象，这个对象继承自CacheKey。

回到obtainController方法中调用的initialize方法，再看看：
```java
//PipelineDraweeController
public void initialize(
  Supplier<DataSource<CloseableReference<CloseableImage>>> dataSourceSupplier,
  String id,
  CacheKey cacheKey,
  Object callerContext,
  @Nullable ImmutableList<DrawableFactory> customDrawableFactories,
  @Nullable ImageOriginListener imageOriginListener) {
    super.initialize(id, callerContext);
    mDataSourceSupplier = dataSourceSupplier;
    mCacheKey = cacheKey;
}
```
主要初始化了几个对象。

获取controller对象完毕之后，就开始执行setController方法：
- setController
```java
//DraweeView
/** Sets the controller. */
public void setController(@Nullable DraweeController draweeController) {
    mDraweeHolder.setController(draweeController);
    super.setImageDrawable(mDraweeHolder.getTopLevelDrawable());
}
```
DraweeHolder中定义了setController方法：
```java
//DraweeHolder
public void setController(@Nullable DraweeController draweeController) {
    boolean wasAttached = mIsControllerAttached;
    if (wasAttached) {
      detachController();
    }
    
    // Clear the old controller
    if (isControllerValid()) {
      mController.setHierarchy(null);
    }
    mController = draweeController;
    if (mController != null) {
      mController.setHierarchy(mHierarchy);
    } else {
    }
    
    if (wasAttached) {
      attachController();
    }
}
```

## 二、视图可见性发生变化
可以看到此时wasAttached为false，这样一来detachController和attachController方法都无法调用了，调用栈流程到这里仿佛中断了。

#### 1) GenericDraweeView
记得在GenericDraweeView在构造函数中会执行inflateHierarchy方法：
```java
//GenericDraweeView
protected void inflateHierarchy(Context context, @Nullable AttributeSet attrs) {
    GenericDraweeHierarchyBuilder builder =
        GenericDraweeHierarchyInflater.inflateBuilder(context, attrs);
    setAspectRatio(builder.getDesiredAspectRatio());
    setHierarchy(builder.build());
}
```

#### 2) DraweeHolder
重新看看setHierarchy，最终调用到DraweeHolder的setHierarchy方法：
```java
//DraweeHolder
public void setHierarchy(DH hierarchy) {
    mEventTracker.recordEvent(Event.ON_SET_HIERARCHY);
    final boolean isControllerValid = isControllerValid();

    setVisibilityCallback(null);
    mHierarchy = Preconditions.checkNotNull(hierarchy);
    Drawable drawable = mHierarchy.getTopLevelDrawable();
    onVisibilityChange(drawable == null || drawable.isVisible());
    setVisibilityCallback(this);

    if (isControllerValid) {
      mController.setHierarchy(hierarchy);
    }
}
```
这里有一个setVisibilityCallback方法，设置了visiable监听：
```java
//DraweeHolder
private void setVisibilityCallback(@Nullable VisibilityCallback visibilityCallback) {
    Drawable drawable = getTopLevelDrawable();
    if (drawable instanceof VisibilityAwareDrawable) {
      ((VisibilityAwareDrawable) drawable).setVisibilityCallback(visibilityCallback);
    }
}
```
getTopLevelDrawable方法获取的是mHierarchy对象的Drawable，而mHierarchy方法对应GenericDraweeHierarchy类，这个类里面的getTopLevelDrawable方法对应的是RootDrawable，它继承自ForwardingDrawable类，它又继承自Android原生Drawable。所以实际上是设置了Drawable可见性监听。当收到可见性变化时，先在ForwardingDrawable接收到回调信号：
```java
//ForwardingDrawable
@Override
public boolean setVisible(boolean visible, boolean restart) {
    final boolean superResult = super.setVisible(visible, restart);
    if (mCurrentDelegate == null) {
      return superResult;
    }

    return mCurrentDelegate.setVisible(visible, restart);
 }
```
mCurrentDelegate是RootDrawable：
```java
//RootDrawable
@Override
public boolean setVisible(boolean visible, boolean restart) {
    if (mVisibilityCallback != null) {
      mVisibilityCallback.onVisibilityChange(visible);
    }
    return super.setVisible(visible, restart);
}
```
终于看到回调方法的地方了，刚才说到DraweeHolder设置了监听，那么在DraweeHolder中看看回调逻辑：
```java
//DraweeHolder
@Override
public void onVisibilityChange(boolean isVisible) {
    mIsVisible = isVisible;
    attachOrDetachController();
}
//DraweeHolder
private void attachOrDetachController() {
    if (mIsHolderAttached && mIsVisible) {
      attachController();
    } else {
      detachController();
    }
}
```
如果holder没有附属到view上时，将其连接上，否则剥离掉。
先看看attachController函数：
- attachController
```java
if (mController != null && mController.getHierarchy() != null) {
  mController.onAttach();
}
```
开始在Controller类中执行onAttach方法，这个方法实现在AbstractDraweeController类中，定义在DraweeController接口中。
```java
//AbstractDraweeController
@Override public void onAttach() {
  if (!mIsRequestSubmitted) {
      submitRequest();
    }
}
```
核心方法就两行代码，提交刚才的请求：
```java
//AbstractDraweeController
protected void submitRequest() {
    final T closeableImage = getCachedImage();
    if (closeableImage != null) {
        onImageLoadedFromCacheImmediately(mId, closeableImage);
        onNewResultInternal(mId, mDataSource, closeableImage, 1.0f, true, true, true);
        return;
    }
    mSettableDraweeHierarchy.setProgress(0, true);
    mIsRequestSubmitted = true;
    mHasFetchFailed = false;
    mDataSource = getDataSource();
    final String id = mId;
    final boolean wasImmediate = mDataSource.hasResult();
    final DataSubscriber<T> dataSubscriber =
    
    new BaseDataSubscriber<T>() {
          @Override
          public void onNewResultImpl(DataSource<T> dataSource) {
            // isFinished must be obtained before image, otherwise we might set intermediate result
            // as final image.
            boolean isFinished = dataSource.isFinished();
            boolean hasMultipleResults = dataSource.hasMultipleResults();
            float progress = dataSource.getProgress();
            T image = dataSource.getResult();
            if (image != null) {
              onNewResultInternal(
                  id, dataSource, image, progress, isFinished, wasImmediate, hasMultipleResults);
            } else if (isFinished) {
              onFailureInternal(id, dataSource, new NullPointerException(), /* isFinished */ true);
            }
          }

          @Override
          public void onFailureImpl(DataSource<T> dataSource) {
            onFailureInternal(id, dataSource, dataSource.getFailureCause(), /* isFinished */ true);
          }

          @Override
          public void onProgressUpdate(DataSource<T> dataSource) {
            boolean isFinished = dataSource.isFinished();
            float progress = dataSource.getProgress();
            onProgressUpdateInternal(id, dataSource, progress, isFinished);
          }
        };
    mDataSource.subscribe(dataSubscriber, mUiThreadImmediateExecutor);
}
```
这个方法里面是核心的代码，分为两个部分，前一个部分是找本地缓存，有缓存的话，直接取缓存数据展示然后返回，否则就去请求并加载。

#### 3) 取内存缓存
先看取缓存,getCachedImage:
```java
//PipelineDraweeController
  @Override
  protected @Nullable CloseableReference<CloseableImage> getCachedImage() {
    try {
      if (mMemoryCache == null || mCacheKey == null) {
        return null;
      }
      // We get the CacheKey
      CloseableReference<CloseableImage> closeableImage = mMemoryCache.get(mCacheKey);
      if (closeableImage != null && !closeableImage.get().getQualityInfo().isOfFullQuality()) {
        closeableImage.close();
        return null;
      }
      return closeableImage;
    } finally {
    }
}
```
从内存缓存中取数据，是CloseableImage类型的，取到缓存并符合质量条件就返回，对于不符合质量条件的关闭closeableImage并返回null，关闭的过程实际是在一个SharedReference对象中将CloseableImage对象的引用计数删除或计数减一。
后续单独讲缓存策略。

获取到缓存之后，执行onImageLoadedFromCacheImmediately通知消息注册者，从内存取到缓存数据了。

#### 4) 显示内存缓存图片
onNewResultInternal才是真正加载图片的地方：
```java
//AbstractDraweeController
private void onNewResultInternal(
      String id,
      DataSource<T> dataSource,
      @Nullable T image,
      float progress,
      boolean isFinished,
      boolean wasImmediate,
      boolean deliverTempResult) {
    try {
        Drawable drawable;
        try {
            drawable = createDrawable(image);
        } catch (Exception exception) {
        }
        try {
            // set the new image
            if (isFinished) {
                mDataSource = null;
               mSettableDraweeHierarchy.setImage(drawable, 1f, wasImmediate);
            }
        } finally {
            releaseDrawable(previousDrawable);
        }
    } finally {
    }
}
```
先看看createDrawable的代码：
```java
//PipelineDraweeController
@Override
protected Drawable createDrawable(CloseableReference<CloseableImage> image) {
    drawable = mDefaultDrawableFactory.createDrawable(closeableImage);
    if (drawable != null) {
        return drawable;
    }
}
//DefaultDrawableFactory
@Override
@Nullable
public Drawable createDrawable(CloseableImage closeableImage) {
    try {
        if (closeableImage instanceof CloseableStaticBitmap) {
            CloseableStaticBitmap closeableStaticBitmap = (CloseableStaticBitmap) closeableImage;
            Drawable bitmapDrawable = new BitmapDrawable(mResources, closeableStaticBitmap.getUnderlyingBitmap());
            if (!hasTransformableRotationAngle(closeableStaticBitmap)
                && !hasTransformableExifOrientation(closeableStaticBitmap)) {
              // Return the bitmap drawable directly as there's nothing to transform in it
                return bitmapDrawable;
            }
        }
    } finally {
    }
}
```
在createDrawable方法中，将获取到的缓存数据强转成CloseableStaticBitmap对象并执行getUnderlyingBitmap方法返回，实际就是就是一个Bitmap对象，再将其封装到BitmapDrawable对象，然后展示。

setImage最终调用在GenericDraweeHierarchy中：
```java
//AbstractDraweeController
mSettableDraweeHierarchy.setImage(drawable, 1f, wasImmediate);

//GenericDraweeHierarchy
@Override
public void setImage(Drawable drawable, float progress, boolean immediate) {
    drawable = WrappingUtils.maybeApplyLeafRounding(drawable, mRoundingParams, mResources);
    drawable.mutate();
    mActualImageWrapper.setDrawable(drawable);
    mFadeDrawable.beginBatchMode();
    fadeOutBranches();
    fadeInLayer(ACTUAL_IMAGE_INDEX);
    setProgress(progress);
    if (immediate) {
      mFadeDrawable.finishTransitionImmediately();
    }
    mFadeDrawable.endBatchMode();
}
```
先看看有没有圆角参数需要设置，然后设置真正的位图，再渐变，设置进度，最后将刷新参数减一并统一刷新视图。

从attachController方法到现在只是讲了从缓存读取数据并展示，下面将以在线图片为例看看怎么加载图片的。