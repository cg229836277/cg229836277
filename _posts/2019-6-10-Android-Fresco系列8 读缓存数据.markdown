---
layout: post
title:  "Android-Fresco系列8 读缓存数据"
date:   2019-6-10 9:53:25 +0800
categories: Android
---


看流程图：
![](https://i.loli.net/2019/06/10/5cfe0ec59886361590.jpg)

## 一、读取解码内存缓存

#### 1. BitmapMemoryCacheProducer
之前加载图片资源的时候有说到过，从缓存取数据，讲的是从内存取，在AbstractDraweeController的submitRequest方法中，先从缓存取数据，getCachedImage方法：
```java
//PipelineDraweeController
@Override
protected @Nullable CloseableReference<CloseableImage> getCachedImage() {
    CloseableReference<CloseableImage> closeableImage = mMemoryCache.get(mCacheKey);
    if (closeableImage != null && !closeableImage.get().getQualityInfo().isOfFullQuality()) {
        closeableImage.close();
        return null;
    }
    return closeableImage;
}
```
mMemoryCache对应的是ImagePipelineFactory中的`InstrumentedMemoryCache<CacheKey, CloseableImage>`,通过getBitmapMemoryCache方法初始化：
```java
public InstrumentedMemoryCache<CacheKey, CloseableImage> getBitmapMemoryCache() {
    if (mBitmapMemoryCache == null) {
      mBitmapMemoryCache =
          BitmapMemoryCacheFactory.get(
              getBitmapCountingMemoryCache(),
              mConfig.getImageCacheStatsTracker());
    }
    return mBitmapMemoryCache;
}
```
而在InstrumentedMemoryCache类中，get方法是:
```java
@Override
public CloseableReference<V> get(K key) {
    CloseableReference<V> result = mDelegate.get(key);
}
```
mDelegate对应的getBitmapCountingMemoryCache方法初始化：
```java
public CountingMemoryCache<CacheKey, CloseableImage>
getBitmapCountingMemoryCache() {
    if (mBitmapCountingMemoryCache == null) {
      mBitmapCountingMemoryCache =
          BitmapCountingMemoryCacheFactory.get(
              mConfig.getBitmapMemoryCacheParamsSupplier(),
              mConfig.getMemoryTrimmableRegistry(),
              mConfig.getBitmapMemoryCacheTrimStrategy());
    }
    return mBitmapCountingMemoryCache;
}
```

#### 2. CountingMemoryCache
mDelegate对应的是CountingMemoryCache类，看看这个类里面的get方法：
```java
//CountingMemoryCache
@Nullable
public CloseableReference<V> get(final K key) {
    Preconditions.checkNotNull(key);
    Entry<K, V> oldExclusive;
    CloseableReference<V> clientRef = null;
    synchronized (this) {
      oldExclusive = mExclusiveEntries.remove(key);
      Entry<K, V> entry = mCachedEntries.get(key);
      if (entry != null) {
        clientRef = newClientReference(entry);
      }
    }
    maybeNotifyExclusiveEntryRemoval(oldExclusive);
    maybeUpdateCacheParams();
    maybeEvictEntries();
    return clientRef;
}
//CountingMemoryCache
private synchronized CloseableReference<V> newClientReference(final Entry<K, V> entry) {
    increaseClientCount(entry);
    return CloseableReference.of(
        entry.valueRef.get(),
        new ResourceReleaser<V>() {
          @Override
          public void release(V unused) {
            releaseClientReference(entry);
          }
        });
}
```
又回到了之前分析的内存缓存的类CountingMemoryCache中了，这个类掌控了内存缓存的写入和获取，以及缓存的清理。

在取缓存的时候先从等待清理的缓存列表中删除，然后从正式缓存列表中去读，如果读取到了缓存（实际是entry对象），就通过newClientReference封装，然后返回。返回之前先检查内存限制参数是否需要更新，接下来查看是否需要清理内存。

在DefaultImageDecoder的decodeJpeg方法中，我们返回了解码后的数据给到CloseableReference<Bitmap> bitmapReference对象，然后将这个对象封装到了CloseableStaticBitmap对象中。

```java
return new CloseableStaticBitmap(
          bitmapReference,
          qualityInfo,
          encodedImage.getRotationAngle(),
          encodedImage.getExifOrientation());
```
实际我们在内存中缓存的解码之后的数据是CloseableStaticBitmap类型的，而PipelineDraweeController中getCachedImage函数返回的就是这个类型的数据。

## 二、读取编码内存缓存
#### 1. BitmapMemoryCacheProducer
先执行这个类中的produceResults取编码内存缓存。
##### 1) produceResults
```java
@Override public void produceResults(final Consumer<EncodedImage> consumer, final ProducerContext producerContext) {
    final CacheKey cacheKey = mCacheKeyFactory.getEncodedCacheKey(imageRequest, producerContext.getCallerContext());
    if (cachedReference != null) {
        EncodedImage cachedEncodedImage = new EncodedImage(cachedReference);
        consumer.onNewResult(cachedEncodedImage, Consumer.IS_LAST);
        return;
    }
}

```
先去查找编码内存缓存，存在数据的话，就将Consumer的状态置为IS_LAST，然后向后传递数据，其实这个状态位一旦置为IS_LAST，预示着后续的Producer不用干活了，但是这个Producer之前的Producer中的Consumer则要开始干活了。

#### 2. 图片处理
具体干什么活呢？之前文章提到过，编码内存缓存的数据是CloseableReference<MemoryPooledByteBuffer>类型的，还要经过伸缩变换，图片翻转，转码等操作，将数据解码成CloseableReference<CloseableStaticBitmap>，然后向上传递给控件展示，此处的流程就是文章 **图片解码** 里面的业务逻辑了，这里不再重复。

## 三、读取磁盘缓存
在加载图片之前，先从内存取，取不到的话再从磁盘取，看看从磁盘取数据的过程。

#### 1. DiskCacheReadProducer
##### 1) produceResults
在这个方法中定义了读取磁盘缓存的逻辑：
```java
//DiskCacheReadProducer
public void produceResults(){
    final Task<EncodedImage> diskLookupTask = mDefaultBufferedDiskCache.get(cacheKey, isCancelled);
    final Continuation<EncodedImage, Void> continuation =
        onFinishDiskReads(consumer, producerContext);
        diskLookupTask.continueWith(continuation);
}
```

##### 2) BufferedDiskCache
mDefaultBufferedDiskCache对应的是BufferedDiskCache类，看看里面的get方法：
```java
//BufferedDiskCache
public Task<EncodedImage> get(CacheKey key, AtomicBoolean isCancelled) {
    try {
      final EncodedImage pinnedImage = mStagingArea.get(key);
      if (pinnedImage != null) {
        return foundPinnedImage(key, pinnedImage);
      }
      return getAsync(key, isCancelled);
    } finally {
    }
}
```
先从mStagingArea里面取，他维护了一个Map：
`private Map<CacheKey, EncodedImage> mMap;`,如果取到数据了执行foundPinnedImage函数再返回，否则的话要通过getAsync方法获取。分别看看这两个方法：
- foundPinnedImage
```java
//BufferedDiskCache
private Task<EncodedImage> foundPinnedImage(CacheKey key, EncodedImage pinnedImage) {
    mImageCacheStatsTracker.onStagingAreaHit(key);
    return Task.forResult(pinnedImage);
}
```
这里用到了外部sdk，返回一个Task对象，实际是在一个工作线程中返回数据，并回调中间过程包含了结果。

- getAsync
```java
//BufferedDiskCache
private Task<EncodedImage> getAsync(){
    return Task.call(
        new Callable<EncodedImage>() {
        @Override
        public @Nullable EncodedImage call() {
            EncodedImage result = mStagingArea.get(key);
            if (result != null) {
            } else {
                final PooledByteBuffer buffer = readFromDiskCache(key);
                CloseableReference<PooledByteBuffer> ref = CloseableReference.of(buffer);
                EncodedImage result = new EncodedImage(ref);
            }
            return result;
        }
    });
}
```
这里还是先从mStagingArea里面取，，取不到再去调用readFromDiskCache函数从磁盘读取：
```java
private @Nullable PooledByteBuffer readFromDiskCache(final CacheKey key){
    final BinaryResource diskCacheResource = mFileCache.getResource(key);
    final InputStream is = diskCacheResource.openStream();
    PooledByteBuffer byteBuffer = mPooledByteBufferFactory.newByteBuffer(is, (int) diskCacheResource.size());
    return byteBuffer;
}
```
mFileCache对象对应的是DiskStorageCache类，看看getResource方法:
```java
public @Nullable BinaryResource getResource(final CacheKey key) {
    BinaryResource resource = null;
    List<String> resourceIds = CacheKeyUtil.getResourceIds(key);
    for (int i = 0; i < resourceIds.size(); i++) {
        resourceId = resourceIds.get(i);
        resource = mStorage.getResource(resourceId, key);
        if (resource != null) {
            break;
        }
    }
    return resource;
}
```
mStorage对应的是DynamicDefaultDiskStorage类，看看getResource函数：
```java
//DynamicDefaultDiskStorage
@Override
public BinaryResource getResource(String resourceId, Object debugInfo) throws IOException {
    return get().getResource(resourceId, debugInfo);
}
```
get函数返回一个DefaultDiskStorage类型的delegate对象，然后在这里调用getResource方法:
```java
//DefaultDiskStorage
@Override
public @Nullable BinaryResource getResource(String resourceId, Object debugInfo) {
    final File file = getContentFileFor(resourceId);
    if (file.exists()) {
      file.setLastModified(mClock.now());
      return FileBinaryResource.createOrNull(file);
    }
    return null;
}

//DefaultDiskStorage
File getContentFileFor(String resourceId) {
    return new File(getFilename(resourceId));
}

//DefaultDiskStorage
private String getFilename(String resourceId) {
    FileInfo fileInfo = new FileInfo(FileType.CONTENT, resourceId);
    String path = getSubdirectoryPath(fileInfo.resourceId);
    return fileInfo.toPath(path);
}
```
上面三个函数取到了缓存文件，然后将其封装到FileBinaryResource，他实现了BinaryResource接口。

回到readFromDiskCache方法中，将BinaryResource包裹的file打开输入流InputStream，然后执行`mPooledByteBufferFactory.newByteBuffer(InputStream)`,生成PooledByteBuffer对象返回。mPooledByteBufferFactory在ImagePipelineFactory的getMainBufferedDiskCache函数中初始化的：
```java
//ImagePipelineFactory
mConfig.getPoolFactory().getPooledByteBufferFactory(mConfig.getMemoryChunkType()),
```
最终返回的是MemoryPooledByteBufferFactory对象，跟读取内存缓存时分配的对象是一样的：
```java
//MemoryPooledByteBufferFactory
@Override
public MemoryPooledByteBuffer newByteBuffer(InputStream inputStream, int initialCapacity) {
    MemoryPooledByteBufferOutputStream outputStream =
        new MemoryPooledByteBufferOutputStream(mPool, initialCapacity);
    try {
      return newByteBuf(inputStream, outputStream);
    } finally {
      outputStream.close();
    }
}
```
先分配一个outputStream对象，探后执行newByteBuf函数，将inputStream数据copy到outputStream中：
```java
//MemoryPooledByteBufferFactory
MemoryPooledByteBuffer newByteBuf(
  InputStream inputStream, MemoryPooledByteBufferOutputStream outputStream) {
    mPooledByteStreams.copy(inputStream, outputStream);
    return outputStream.toByteBuffer();
}
```
mPooledByteStreams对应的是PooledByteStreams类：
```java
//PooledByteStreams
public long copy(final InputStream from, final OutputStream to) throws IOException {
    long count = 0;
    byte[] tmp = mByteArrayPool.get(mTempBufSize);
    
    try {
      while (true) {
        int read = from.read(tmp, 0, mTempBufSize);
        if (read == -1) {
          return count;
        }
        to.write(tmp, 0, read);
        count += read;
      }
    } finally {
      mByteArrayPool.release(tmp);
    }
}
```
最后toByteBuffer对应的就是MemoryPooledByteBuffer对象了。

刚才讲了从磁盘读取，读取到了就返回到getAsync方法中，将这个bytebuffer数据封装到EncodeImage中：

- continueWith

get方法返回的Task，通过continueWith方法提交continuation这个任务，任务就是从磁盘读取数据。
```java
//DiskCacheReadProducer
final Continuation<EncodedImage, Void> continuation =
    onFinishDiskReads(consumer, producerContext);
diskLookupTask.continueWith(continuation);
```
- onFinishDiskReads

这个方法用于监听数据返回，在Task任务执行完毕之后就回调到then方法中了，通过getResult
```java
//DiskCacheReadProducer
private Continuation<EncodedImage, Void> onFinishDiskReads(){
    return new Continuation<EncodedImage, Void>() {
        @Override
        public Void then(Task<EncodedImage> task){
        EncodedImage cachedReference = task.getResult();
            if (cachedReference != null) {
                consumer.onNewResult(cachedReference, Consumer.IS_LAST);
            }
        }
    };
}
```
到这里从磁盘读取的数据如果不为空，就继续往下传递，不过将消费状态修改为完结，后续其他的Consumer就不用继续干活了，回调到之前的Consumer则要处理图片，跟编码图片的处理流程是一样的。