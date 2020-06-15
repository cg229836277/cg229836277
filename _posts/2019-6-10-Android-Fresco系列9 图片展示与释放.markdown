---
layout: post
title:  "Android-Fresco系列9 图片展示与释放"
date:   2019-6-10 9:53:25 +0800
categories: Android
---

先看流程图:

![](https://i.loli.net/2019/06/10/5cfe30e895b0775670.jpg)

## 一、回到起点
起点是AbstractProducerToDataSourceAdapter，因为ImagePipeline的submitFetchRequest最终调用了`CloseableProducerToDataSourceAdapter.create`方法，发起了整个请求图片到解码图片的流程，而CloseableProducerToDataSourceAdapter继承自AbstractProducerToDataSourceAdapter。
#### 1. AbstractProducerToDataSourceAdapter
在AbstractProducerToDataSourceAdapter的构造函数中定义了内部类，并实现了Consumer回调：
```java
//AbstractProducerToDataSourceAdapter
producer.produceResults(createConsumer(), settableProducerContext);
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

#### 2. onNewResultImpl
我们从前到后的分析都只考虑了乐观的情况，网络顺畅，图片没有问题，转码，变换，解码都是顺利的情况下，来到了最终的onNewResultImpl方法中：
```java
//AbstractProducerToDataSourceAdapter
protected void onNewResultImpl(@Nullable T result, int status) {
    if (super.setResult(result, isLast)) {
    }
}
//AbstractDataSource
protected boolean setResult(@Nullable T value, boolean isLast) {
    boolean result = setResultInternal(value, isLast);
    if (result) {
      notifyDataSubscribers();
    }
    return result;
}
```
#### 3. notifyDataSubscribers
接下来通过notifyDataSubscribers方法通知消息订阅者：
```java
//AbstractDataSource
private void notifyDataSubscribers() {
    final boolean isFailure = hasFailed();
    final boolean isCancellation = wasCancelled();
    for (Pair<DataSubscriber<T>, Executor> pair : mSubscribers) {
      notifyDataSubscriber(pair.first, pair.second, isFailure, isCancellation);
    }
}
//AbstractDataSource
private void notifyDataSubscriber(
  final DataSubscriber<T> dataSubscriber,
  final Executor executor,
  final boolean isFailure,
  final boolean isCancellation) {
    executor.execute(
        new Runnable() {
          @Override
          public void run() {
            if (isFailure) {
              dataSubscriber.onFailure(AbstractDataSource.this);
            } else if (isCancellation) {
              dataSubscriber.onCancellation(AbstractDataSource.this);
            } else {
              dataSubscriber.onNewResult(AbstractDataSource.this);
            }
          }
        });
}
```
是谁订阅了这个消息呢？

## 二、回到最终的消费者
回到AbstractDraweeController的submitRequest方法中：
```java
//AbstractDraweeController
protected void submitRequest() {
    final DataSubscriber<T> dataSubscriber =
        new BaseDataSubscriber<T>() {
          @Override
          public void onNewResultImpl(DataSource<T> dataSource) {
            T image = dataSource.getResult();
            if (image != null) {
              onNewResultInternal(
              id, dataSource, image, 
              progress, isFinished, 
              wasImmediate, hasMultipleResults);
            }
        }
    }
}

private void onNewResultInternal(){
    Drawable drawable = createDrawable(image);
    mSettableDraweeHierarchy.setImage(drawable, 1f, wasImmediate);
    if (previousImage != null && previousImage != image) {
        releaseImage(previousImage);
    }
}
```
显示的代码在之前的文章里面已经讲过就不分析了。

#### 1. 喜新厌旧，释放旧的对象
看看这个releaseImage，是AbstractDraweeController的虚函数，具体实现在子类PipelineDraweeController中：
```java
//PipelineDraweeController
protected void releaseImage(@Nullable             CloseableReference<CloseableImage> image) {
    CloseableReference.closeSafely(image);
}
```
最终调用的是SharedReference的deleteReference方法：
```java
public void deleteReference() {
    mResourceReleaser.release(deleted);
}
```
想想之前的DefaultDecoder里面返回解码之后的图片的时候第二个参数是BucketsBitmapPool类型,`CloseableReference.of(decodedBitmap, mBitmapPool)`,继承关系是：BucketsBitmapPool->BasePool->BasePool->Pool->ResourceReleaser，所以这里release调用到了BasePool这里：
```java
//BasePool
public void release(V value) {
    final int bucketedSize = getBucketedSizeForValue(value);
    final int sizeInBytes = getSizeInBytes(bucketedSize);
    final Bucket<V> bucket = getBucketIfPresent(bucketedSize);
    if (!mInUseValues.remove(value)) {
        free(value);
    } else {
        if (bucket == null ||
        bucket.isMaxLengthExceeded() ||
        isMaxSizeSoftCapExceeded() ||
        !isReusable(value)) {
            if (bucket != null) {
                bucket.decrementInUseCount();
                free(value);
                mUsed.decrement(sizeInBytes);
            } 
        } else {
            bucket.release(value);
            mFree.increment(sizeInBytes);
            mUsed.decrement(sizeInBytes);
        }
    }
}
//BucketsBitmapPool
@Override
protected void free(Bitmap value) {
    value.recycle();
}
```
1.当前正在使用的Set里面删除bitmap不成功的话，就直接释放掉
2.当前的SparseArray<Bucket<V>> mBuckets中有该Bucket的话，更新已使用和空闲空间大小，并将之前分配的空间添加到Bucket的mFreeList中。
3.bucket找不到，此时将bitmap释放掉，同时更新已使用空间大小。

#### 2. 释放总结
从这种Pool和Bucket这一套逻辑来看，充分利用了BitmapOptions的inBitmap参数配置，复用已经分配的空间大小，当这个bitmap被释放掉之后同时更新已使用和未使用空间大小，当然这个大小受到我们配置的参数限制。一方面使用inBitmap避免直接分配，另一方面又在获取复用的bitmap上充分利用已经分配的空间大小，这一点很赞。

## 三、销毁视图
#### 1. Detach
当视图变得不可见时，onVisibilityChange回调回来，就会在DraweeHolder中调用detachController方法：

```java
//DraweeHolder
private void detachController() {
    if (isControllerValid()) {
      mController.onDetach();
    }
}
//AbstractDraweeController
@Override
public void onDetach() {
    mDeferredReleaser.scheduleDeferredRelease(this);
}

//DeferredReleaser
public void scheduleDeferredRelease(Releasable releasable) {
    ensureOnUiThread();
    
    if (!mPendingReleasables.add(releasable)) {
      return;
    }
    // Posting to the UI queue is an O(n) operation, so we only do it once.
    // The one runnable does all the releases.
    if (mPendingReleasables.size() == 1) {
      mUiHandler.post(releaseRunnable);
    }
}

//DeferredReleaser
private final Runnable releaseRunnable = new Runnable() {
    @Override
    public void run() {
      ensureOnUiThread();
      for (Releasable releasable : mPendingReleasables) {
        releasable.release();
      }
      mPendingReleasables.clear();
    }
};
```
从AbstractDraweeController的onDetach方法可以看出来，此时需要将对象释放掉了。而且释放的Releasable对象是AbstractDraweeController自己，因为AbstractDraweeController实现了Releasable接口。

于是在DeferredReleaser中，必须确保在主线程中释放掉，因为涉及到UI的操作。看看AbstractDraweeController的release方法：
```java
//AbstractDraweeController
@Override
public void release() {
  if (mSettableDraweeHierarchy != null) {
    mSettableDraweeHierarchy.reset();
  }
  releaseFetch();
}
//PipelineDraweeController
private void releaseFetch() {
    if (mDataSource != null) {
        mDataSource.close();
        mDataSource = null;
    }
    if (mDrawable != null) {
        releaseDrawable(mDrawable);
    }
    mDrawable = null;
    if (mFetchedImage != null) {
        releaseImage(mFetchedImage);
        mFetchedImage = null;
    }
}
```
#### 2. release
release方法中的reset方法最后在GenericDraweeHierarchy中定义：
```java
//GenericDraweeHierarchy
@Override
public void reset() {
    //resetActualImages();
    mActualImageWrapper.setDrawable(new ColorDrawable(Color.TRANSPARENT));
    resetFade();
}
```
将我们设置图片的图层设置为透明，并将其他图层重置，再统一刷新。

releaseFetch方法中将mDataSource关闭，其实是AbstractDataSource类型：
```java
//AbstractDataSource
@Override
public boolean close() {
    mResult = null;
}
```

将mFetchedImage关闭，其实是CloseableReference<CloseableImage>类型，里面包裹的是CloseableStaticBitmap对象，
```java
//PipelineDraweeController
@Override
protected void releaseImage(@Nullable CloseableReference<CloseableImage> image) {
    CloseableReference.closeSafely(image);
}
//CloseableReference
public static void closeSafely(@Nullable CloseableReference<?> ref) {
    if (ref != null) {
        ref.close();
    }
}

//CloseableReference
@Override
public void close() {
    synchronized (this) {
        if (mIsClosed) {
            return;
        }
        mIsClosed = true;
    }
    mSharedReference.deleteReference();
}

//SharedReference
public void deleteReference() {
    if (decreaseRefCount() == 0) {
        T deleted;
        synchronized (this) {
            deleted = mValue;
            mValue = null;
        }
        mResourceReleaser.release(deleted);
        removeLiveReference(deleted);
    }
}

//BasePool
@Override
public void release(V value) {
//之前已叙述过
}

//SharedReference
private static void removeLiveReference(Object value) {
    synchronized (sLiveObjects) {
        Integer count = sLiveObjects.get(value);
        if (count == null) {
        // Uh oh.
        } else if (count == 1) {
            sLiveObjects.remove(value);
        } else {
            sLiveObjects.put(value, count - 1);
        }
    }
}
```
从上到下依次释放，该置空的置空，该remove的remove，避免内存泄漏。

## 四、总结
总结一下：
### 1. 内存缓存未编码数据

**CloseableReference<MemoryPooledByteBuffer>**

这里的MemoryPooledByteBuffer数据的生成顺序是：MemoryPooledByteBufferOutputStream执行newOutputStream方法获得MemoryPooledByteBufferOutputStream对象，然后执行toByteBuffer生成MemoryPooledByteBuffer。

而网络请求返回的数据被写在了MemoryPooledByteBufferOutputStream对象中，在写的过程中，这个Stream对象生成了内存块对象CloseableReference<MemoryChunk>，并持有他；

###  2. 内存缓存编码数据

**CloseableReference<CloseableImage>**

实际的类型是CloseableReference<CloseableStaticBitmap>,他们的继承关系是：CloseableStaticBitmap->CloseableBitmap->CloseableImage
### 3. 磁盘缓存

磁盘缓存的数据是将PooledByteBufferInputStream持有MemoryPooledByteBuffer对象，而这个对象持有NativeMemoryChunk内存块的数据，在读数据的时候，将内存中的数据读到缓存文件对应的CountingOutputStream，而且读写操作都在native层，nativeCopyToByteArray。

### 4. Producer

是内容生产者，同时生产者中定义了Consumer作为消费者

### 5. ImagePipeline/ImagePipelineFactory

是生产工具的提供者，同时也是生产流程的发起者

### 6. Controller（AbstractDraweeController/PipelineDraweeController）

将内容生产与Hierarchy链接起来

### 7. DraweeHolder

将DraweeView(GenericDraweeView/SimpleDraweeView)和Hierarchy以及Controller联系起来。

View中设置Hierarchy到DraweeHolder中，DraweeHolder持有Hierarchy，Hierarchy中管理Drawable，将多个Drawable分成不同的展示层级，同时将其对应到Controller中。

总结流程图：
![](https://i.loli.net/2019/06/10/5cfe30e84084775738.jpg)


