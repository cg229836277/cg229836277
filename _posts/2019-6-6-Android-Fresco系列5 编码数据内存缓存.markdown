---
layout: post
title:  "Android-Fresco系列5 编码数据内存缓存"
date:   2019-6-6 9:53:25 +0800
categories: Android
---

流程图如下：
![](https://i.loli.net/2019/06/06/5cf8c4bfa819c62191.jpg)

## 一、EncodedMemoryCacheProducer
#### 1) 数据来源
- 从返回的数据流读取数据

网络请求返回InputStream，按照常规思维，从这个stream里面读取数据到byte[]再保存就行了，但是sdk里面的处理更好。

在NetworkFetchProducer中有数据返回之后，开始新建一个返回数据大小的输出数据流：
```java
//HttpUrlConnectionNetworkFetcher
void fetchSync(HttpUrlConnectionNetworkFetchState fetchState, Callback callback) {
    callback.onResponse(is, -1);
}
```
从这里可以看出每次返回的内容长度都为-1，看看在接收回调的地方怎么处理：
```java
//NetworkFetchProducer
PooledByteBufferOutputStream pooledOutputStream = mPooledByteBufferFactory.newOutputStream();
```
mPooledByteBufferFactory对应的是MemoryPooledByteBufferFactory，看看是怎么新建的：
```java
//MemoryPooledByteBufferFactory
@Override
public MemoryPooledByteBufferOutputStream newOutputStream(int initialCapacity) {
    return new MemoryPooledByteBufferOutputStream(mPool, initialCapacity);
}
```
此时的initialCapacity是1024kb（PoolFactory:getMemoryChunkPool），定义在DefaultNativeMemoryChunkPoolParams中,再看看MemoryPooledByteBufferOutputStream的构造函数：
```java
public MemoryPooledByteBufferOutputStream(MemoryChunkPool pool, int initialCapacity) {
    super();
    mBufRef = CloseableReference.of(mPool.get(initialCapacity), mPool);
}
```
这个mPool对象对应的就是NativeMemoryChunkPool对象。继承关系是：NativeMemoryChunkPool->MemoryChunkPool->BasePool，看看get方法：
```java
//BasePool
public V get(int size) {
    Bucket<V> bucket = getBucket(bucketedSize);
    if (bucket != null) {
        V value = getValue(bucket);
        return value;
    } else {
        V value = alloc(bucketedSize);
        return value;
    }
}
```
#### 2) Bucket

回去看看BasePool类中的getBucket方法。看继承关系，getBucket方法应该是在MemoryChunkPool类中调用。看看getBucket:
```java
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
- Bucket是一个系统已分配对象的管理类，里面封装了对象的大小（itemSize），已分配对象被释放后的队列(freeList)等参数，泛型类型是MemoryChunk
- mBuckets是一个以Bucket的大小为key，Bucket<V>为value的SparseArray。同时在BasePool中有两个Counter，用于计算空闲和已使用的空间大小：

```java
SparseArray mBuckets = new SparseArray<Bucket<V>>();
mFree = new Counter();
mUsed = new Counter();
```
当我们从 mBuckets里面找到一个相同大小的Bucket，就将mUsed加上这个Bucket大小，同时将mFree减去这个大小。

每次分配之前要调用canAllocate检查我们已经分配的大小是否超过限制，是的话就要调用trimToSize调整大小。

另外每次分配完了之后还要调用trimToSoftCap方法检查已经分配的值，超过的话也要调用trimToSize调整。

- 从上面的代码可以看出，来了一个复用分配请求的时候，先从mBuckets里面去找有没有相同大小的Bucket，有的话，就直接返回Bucket的value，否则的话新建一个alloc，
看看alloc方法：
```java
//NativeMemoryChunkPool
@Override
protected NativeMemoryChunk alloc(int bucketedSize) {
    return new NativeMemoryChunk(bucketedSize);
}
//NativeMemoryChunk
public NativeMemoryChunk(final int size) {
    mNativePtr = nativeAllocate(mSize);
}
```
在NativeMemoryChunk.c中，可以看到是怎么分配一块内存区域的：
```c
static jlong NativeMemoryChunk_nativeAllocate(
    JNIEnv* env,
    jclass clzz,
    jint size) {
  UNUSED(clzz);
  void* pointer = malloc(size);
  if (!pointer) {
    (*env)->ThrowNew(env, jRuntimeException_class, "could not allocate memory");
    return 0;
  }
  return PTR_TO_JLONG(pointer);
}
```
malloc分配一块指定大小的内存，并返回指向这块内存开始地址的指针，这块内存在native heap中。

pooledOutputStream对应的是MemoryPooledByteBufferOutputStream类，封装的是CloseableReference<NativeMemoryChunk>内存块。

#### 2) 读取数据
接下来看读取数据：
```java
protected void onResponse(){
final byte[] ioArray = mByteArrayPool.get(READ_SIZE);
    int length;
    while ((length = responseData.read(ioArray)) >= 0) {
        if (length > 0) {
            pooledOutputStream.write(ioArray, 0, length);
        }
    }
}
```
这里mByteArrayPool对应的是PoolFactory中的getSmallByteArrayPool函数：
```java
public ByteArrayPool getSmallByteArrayPool() {
    if (mSmallByteArrayPool == null) {
      mSmallByteArrayPool = new GenericByteArrayPool(
          mConfig.getMemoryTrimmableRegistry(),
          mConfig.getSmallByteArrayPoolParams(),
          mConfig.getSmallByteArrayPoolStatsTracker());
    }
    return mSmallByteArrayPool;
}
```
在GenericByteArrayPool中默认初始化了一个mBucketSizes的int类型数组，在getSmallByteArrayPoolParams方法中，设置了这个数组的默认大小是16kb。另外GenericByteArrayPool继承自BasePool。

为什么这么做呢？因为请求回来的数据要分多次读取，每次读取都意味着内存块的分配，此时将已经分配的内存块放到SparseArray<Bucket<V>> mBuckets中存起来，当同样大小的内存块被释放掉后，后续新的分配请求来临时，直接返回之前分配好的就行，不用重复分配，避免内存抖动。

mByteArrayPool.get就是在开始请求内存分配了。

接下来我们将请求下来的数据保存在pooledOutputStream对象中，在NetworkFetchProducer封装EncodeImage之前要将stream数据流转成bytebuffer：
```java
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

//MemoryPooledByteBufferOutputStream
@Override
public MemoryPooledByteBuffer toByteBuffer() {
    ensureValid();
    return new MemoryPooledByteBuffer(mBufRef, mCount);
}

//MemoryPooledByteBuffer
public MemoryPooledByteBuffer(CloseableReference<MemoryChunk> bufRef, int size) {
    mBufRef = bufRef.clone();
    mSize = size;
}
```
通过上面的逻辑可以看出，CloseableReference包裹的是MemoryPooledByteBuffer类型的对象，而MemoryPooledByteBuffer中持有一个CloseableReference<MemoryChunk>的引用，而MemoryChunk就是分配的内存块。

NetworkFetchProducer中我们已经把在线获取到的数据流封装到CloseableReference对象，并把这个对象封装到EncodedImage对象，然后上传给消息订阅者DiskCacheWriteProducer，在磁盘缓存处理类中，我们把这个数据流写到缓存目录的文件中。

在DiskCacheWriteProducer中处理完成之后，继续向上传递EncodedImage对象给下一个消息订阅者，他在EncodedMemoryCacheProducer类中,记住此时存储的对象是未解码的数据对应的是MemoryPooledByteBuffer数据。

## 二、开始缓存
#### 1) EncodedMemoryCacheConsumer

`EncodedMemoryCacheConsumer`是一个`EncodedMemoryCacheProducer`的内部类，对应的是一个消费者。在看具体消费之前先看看是怎么订阅消息的。

- produceResults

一贯的，需要实现接口的方法：
```java
//EncodedMemoryCacheConsumer
@Override
public void produceResults(final Consumer<EncodedImage> consumer, final ProducerContext producerContext) {
    final String requestId = producerContext.getId();
    final ImageRequest imageRequest = producerContext.getImageRequest();
    final CacheKey cacheKey = mCacheKeyFactory.getEncodedCacheKey(imageRequest, producerContext.getCallerContext());

    CloseableReference<PooledByteBuffer> cachedReference = mMemoryCache.get(cacheKey);
    if (cachedReference != null) {
        EncodedImage cachedEncodedImage = new EncodedImage(cachedReference);
        consumer.onProgressUpdate(1f);
        consumer.onNewResult(cachedEncodedImage, Consumer.IS_LAST);
        return;
    } else {
        final boolean isMemoryCacheEnabled =
        producerContext.getImageRequest().isMemoryCacheEnabled();
        Consumer consumerOfInputProducer = new EncodedMemoryCacheConsumer(consumer, mMemoryCache, cacheKey, isMemoryCacheEnabled);
        mInputProducer.produceResults(consumerOfInputProducer, producerContext);
    }
}
```

订阅流程其实比较简单，先判断当前内存里面有没有对应key的value值，有的话，直接返回给上一个订阅者，否则的话，订阅消息，等着数据返回。

从DiskCacheWriteProducer向上传递的数据直接到EncodedMemoryCacheConsumer的onNewResultImpl方法中：
```java
@Override
public void onNewResultImpl(EncodedImage newResult, @Status int status) {
    CloseableReference<PooledByteBuffer> ref = newResult.getByteBufferRef();
    CloseableReference<PooledByteBuffer> cachedResult = null;
    if (mIsMemoryCacheEnabled) {
        cachedResult = mMemoryCache.cache(mRequestedCacheKey, ref);
    }
    if (cachedResult != null) {
        EncodedImage cachedEncodedImage;
        try {
            cachedEncodedImage = new EncodedImage(cachedResult);
            cachedEncodedImage.copyMetaDataFrom(newResult);
        } finally {
            CloseableReference.closeSafely(cachedResult);
        }
        try {
            getConsumer().onProgressUpdate(1f);
            getConsumer().onNewResult(cachedEncodedImage, status);
            return;
        } finally {
            EncodedImage.closeSafely(cachedEncodedImage);
        }    
    }
    getConsumer().onNewResult(newResult, status);
}
```
mMemoryCache的初始化是在ImagePipelineFactory类中：
```java
public InstrumentedMemoryCache<CacheKey, PooledByteBuffer> getEncodedMemoryCache() {
    if (mEncodedMemoryCache == null) {
      mEncodedMemoryCache = EncodedMemoryCacheFactory.get(getEncodedCountingMemoryCache(), mConfig.getImageCacheStatsTracker());
    }
    return mEncodedMemoryCache;
}
```
分解一下初始化的步骤：
##### 1. getEncodedCountingMemoryCache
先获取内存缓存计算：
```java
public CountingMemoryCache<CacheKey, PooledByteBuffer> getEncodedCountingMemoryCache() {
    if (mEncodedCountingMemoryCache == null) {
      mEncodedCountingMemoryCache =
          EncodedCountingMemoryCacheFactory.get(
              mConfig.getEncodedMemoryCacheParamsSupplier(), mConfig.getMemoryTrimmableRegistry());
    }
    return mEncodedCountingMemoryCache;
}
```
先获取内存缓存的参数提供者,这个在ImagePipelineConfig类的构造函数中提供：
```java
//ImagePipelineConfig
mEncodedMemoryCacheParamsSupplier = builder.mEncodedMemoryCacheParamsSupplier == null
        ? new DefaultEncodedMemoryCacheParamsSupplier()
        : builder.mEncodedMemoryCacheParamsSupplier;
```
如果调用者不提供的话，默认是构造一个DefaultEncodedMemoryCacheParamsSupplier对象,涉及到内存缓存的参数配置，这个比较重要，代码全部贴一下:
```java
//DefaultEncodedMemoryCacheParamsSupplier
public class DefaultEncodedMemoryCacheParamsSupplier implements Supplier<MemoryCacheParams> {

  // We want memory cache to be bound only by its memory consumption
  private static final int MAX_CACHE_ENTRIES = Integer.MAX_VALUE;
  private static final int MAX_EVICTION_QUEUE_ENTRIES = MAX_CACHE_ENTRIES;
  private static final long PARAMS_CHECK_INTERVAL_MS = TimeUnit.MINUTES.toMillis(5);

  @Override
  public MemoryCacheParams get() {
    final int maxCacheSize = getMaxCacheSize();
    final int maxCacheEntrySize = maxCacheSize / 8;
    return new MemoryCacheParams(
        maxCacheSize,
        MAX_CACHE_ENTRIES,
        maxCacheSize,
        MAX_EVICTION_QUEUE_ENTRIES,
        maxCacheEntrySize,
        PARAMS_CHECK_INTERVAL_MS);
  }

  private int getMaxCacheSize() {
    final int maxMemory = (int) Math.min(Runtime.getRuntime().maxMemory(), Integer.MAX_VALUE);
    if (maxMemory < 16 * ByteConstants.MB) {
      return 1 * ByteConstants.MB;
    } else if (maxMemory < 32 * ByteConstants.MB) {
      return 2 * ByteConstants.MB;
    } else {
      return 4 * ByteConstants.MB;
    }
  }
}
```
通过执行getMaxCacheSize方法获取最大内存缓存大小,先获取当前应用运行时被分配的最大内存大小，与int类型的最大值（2147483647约为2G）两者取最小的一个。

取到之后再来判断，小于16M，为1M；小于32M，为2M；其他为4M。可见内存缓存设置的比较小。

MemoryCacheParams就是内存缓存的配置了，每个参数的含义及数值如下：
- int maxCacheSize,最大缓存大小

这个已经知道了，通过getMaxCacheSize方法获取到了；
- int maxCacheEntries,最大缓存节点数

这个值等于Integer.MAX_VALUE（2147483647）；
- int maxEvictionQueueSize,最大删除队列大小

这个值等于maxCacheSize；
- int maxEvictionQueueEntries,最大删除节点队列大小

这个值等于Integer.MAX_VALUE（2147483647）；
- int maxCacheEntrySize,最大缓存节点大小

这个值等于maxCacheSize / 8；
- long paramsCheckIntervalMs,参数检查间隔时间

5分钟。

##### 2. getImageCacheStatsTracker

先获取内存缓存的参数提供者,这个在ImagePipelineConfig类的构造函数中提供,用户不设置的话，默认是
`NoOpMemoryTrimmableRegistry.getInstance()`。

##### 3. EncodedCountingMemoryCacheFactory.get

上面通过getEncodedCountingMemoryCache和getImageCacheStatsTracker方法获取到了内存缓存计数和内存调整通知注册。

看看这个方法是怎么封装这两个对象的：
```java
//EncodedCountingMemoryCacheFactory
public static CountingMemoryCache<CacheKey, PooledByteBuffer> get(
  Supplier<MemoryCacheParams> encodedMemoryCacheParamsSupplier,
  MemoryTrimmableRegistry memoryTrimmableRegistry) {
    ValueDescriptor<PooledByteBuffer> valueDescriptor =
        new ValueDescriptor<PooledByteBuffer>() {
          @Override
          public int getSizeInBytes(PooledByteBuffer value) {
            return value.size();
        }
    };
    
    CountingMemoryCache.CacheTrimStrategy trimStrategy = new NativeMemoryCacheTrimStrategy();
    
    CountingMemoryCache<CacheKey, PooledByteBuffer> countingCache =
        new CountingMemoryCache<>(valueDescriptor, trimStrategy, encodedMemoryCacheParamsSupplier);
    
    memoryTrimmableRegistry.registerMemoryTrimmable(countingCache);
    
    return countingCache;
}
```
- ValueDescriptor

用来读取图片数据的大小
- NativeMemoryCacheTrimStrategy

这个类中定义了内存缓存变动的策略：
```java
//NativeMemoryCacheTrimStrategy
@Override
public double getTrimRatio(MemoryTrimType trimType) {
    switch (trimType) {
      case OnCloseToDalvikHeapLimit:
        // Resources cached on native heap do not consume Dalvik heap, so no trimming here.
        return 0;
      case OnAppBackgrounded:
      case OnSystemMemoryCriticallyLowWhileAppInForeground:
      case OnSystemLowMemoryWhileAppInForeground:
      case OnSystemLowMemoryWhileAppInBackground:
        return 1;
      default:
        FLog.wtf(TAG, "unknown trim type: %s", trimType);
        return 0;
    }
}
```

- CountingMemoryCache

这个对象通过CountingMemoryCache构造函数，封装了内容大小描述，内存缓存参数配置，以及内存变动的策略。
最后将这个对象注册到内存变动中并返回CountingMemoryCache对象。

##### 4. getImageCacheStatsTracker

接着获取图片缓存状态追踪，这个对象对应的是mImageCacheStatsTracker，是在ImagePipelineConfig的构造函数中初始化的，默认是NoOpImageCacheStatsTracker，直接执行单例方法获取对象，`NoOpImageCacheStatsTracker.getInstance()`。

##### 5. EncodedMemoryCacheFactory.get
以上步骤生成了不同的处理逻辑对象之后，通过get方法生成一个InstrumentedMemoryCache对象mEncodedMemoryCache：
```java
//InstrumentedMemoryCache
public static InstrumentedMemoryCache<CacheKey, PooledByteBuffer> get(
  final CountingMemoryCache<CacheKey, PooledByteBuffer> encodedCountingMemoryCache,
  final ImageCacheStatsTracker imageCacheStatsTracker) {

    imageCacheStatsTracker.registerEncodedMemoryCache(encodedCountingMemoryCache);
    
    MemoryCacheTracker memoryCacheTracker = new MemoryCacheTracker<CacheKey>() {
      @Override
      public void onCacheHit(CacheKey cacheKey) {
        imageCacheStatsTracker.onMemoryCacheHit(cacheKey);
      }
    
      @Override
      public void onCacheMiss() {
        imageCacheStatsTracker.onMemoryCacheMiss();
      }
    
      @Override
      public void onCachePut() {
        imageCacheStatsTracker.onMemoryCachePut();
      }
    };
    
    return new InstrumentedMemoryCache<>(encodedCountingMemoryCache, memoryCacheTracker);
}
```
将CountingMemoryCache对象注册到图片缓存状态追踪。

新建一个MemoryCacheTracker对象，封装到InstrumentedMemoryCache，这个对象回调了缓存状态，包含命中状态，保存等。每一个状态执行imageCacheStatsTracker(NoOpImageCacheStatsTracker)的方法。

最终构造一个InstrumentedMemoryCache对象返回，构造函数如下：
```java
//InstrumentedMemoryCache
public InstrumentedMemoryCache(MemoryCache<K, V> delegate, MemoryCacheTracker tracker) {
    mDelegate = delegate;
    mTracker = tracker;
}
```
这个delegate对应的是CountingMemoryCache类，这个类比较重要。

#### 2) 处理数据
从EncodedMemoryCacheProducer的mMemoryCache对象初始化过程一直到现在终于理清了这个对象包含哪些东西，接下来看看onNewResultImpl中的mMemoryCache.cache方法，这个方法就开始尝试往内存里面缓存图片的bytebuffer数据了。

通过上面的分析我们知道mMemoryCache对应的是InstrumentedMemoryCache类，这个类里面定义了cache方法：
```java
//InstrumentedMemoryCache
@Override
public CloseableReference<V> cache(K key, CloseableReference<V> value) {
    mTracker.onCachePut();
    return mDelegate.cache(key, value);
}
```
刚才说到过mDelegate对应的是CountingMemoryCache类，看看这个类里面的cache方法：
```java
public @Nullable CloseableReference<V> cache(
  final K key, final CloseableReference<V> valueRef, final EntryStateObserver<K> observer) {
    maybeUpdateCacheParams();
    
    Entry<K, V> oldExclusive;
    CloseableReference<V> oldRefToClose = null;
    CloseableReference<V> clientRef = null;
    synchronized (this) {
      // remove the old item (if any) as it is stale now
      oldExclusive = mExclusiveEntries.remove(key);
      Entry<K, V> oldEntry = mCachedEntries.remove(key);
      if (oldEntry != null) {
        makeOrphan(oldEntry);
        oldRefToClose = referenceToClose(oldEntry);
      }
    
      if (canCacheNewValue(valueRef.get())) {
        Entry<K, V> newEntry = Entry.of(key, valueRef, observer);
        mCachedEntries.put(key, newEntry);
        clientRef = newClientReference(newEntry);
      }
    }
    CloseableReference.closeSafely(oldRefToClose);
    maybeNotifyExclusiveEntryRemoval(oldExclusive);
    
    maybeEvictEntries();
    return clientRef;
}
```
##### 1.参数来源
cache方法中key来自:
```java
final CacheKey cacheKey = mCacheKeyFactory.getEncodedCacheKey(
    imageRequest,
    producerContext.getCallerContext()
);`
```
实际是将uri封装到一个SimpleCacheKey中。

calue来自EncodedImage对象的mPooledByteBufferRef变量，这个变量对应的是CloseableReference<PooledByteBuffer>类，其实这个CloseableReference并不神秘：
```java
private CloseableReference(T t, ResourceReleaser<T> resourceReleaser) {
    mSharedReference = new SharedReference<T>(t, resourceReleaser);
}
```
t就是我们真正的数据。

##### 2.是否需要更新参数

如果最后一次更新缓存的时间与当前时间相比超过5分钟，就取一下最新的内存缓存参数，在DefaultEncodedMemoryCacheParamsSupplier类中。

##### 3.开始缓存数据
CountingMemoryCache构造函数中初始化了两个Map：
```java
//CountingMemoryCache
mExclusiveEntries = new CountingLruMap<>(wrapValueDescriptor(valueDescriptor));
mCachedEntries = new CountingLruMap<>(wrapValueDescriptor(valueDescriptor));
```
一个是等着清理删除的缓存数据，一个是缓存的数据。CountingLruMap实际维护的是一个LinkedHashMap对象。

开始缓存之前先把这两个map里面已经存在的数据删除掉，并把这个数据置为孤立（orphan）状态，然后put新的数据：
```java
//CountingMemoryCache
if (canCacheNewValue(valueRef.get())) {
    Entry<K, V> newEntry = Entry.of(key, valueRef, observer);
    mCachedEntries.put(key, newEntry);
    clientRef = newClientReference(newEntry);
}
```
先判断是否能缓存新数据，因为我们设置了最大缓存大小，和最大缓存数的限制：
```java
//CountingMemoryCache
private synchronized boolean canCacheNewValue(V value) {
    int newValueSize = mValueDescriptor.getSizeInBytes(value);
    return (newValueSize <= mMemoryCacheParams.maxCacheEntrySize) &&
        (getInUseCount() <= mMemoryCacheParams.maxCacheEntries - 1) &&
        (getInUseSizeInBytes() <= mMemoryCacheParams.maxCacheSize - newValueSize);
}
```
实际就是统计mCachedEntries和mExclusiveEntries里面的数据大小。

- 第一个判断条件是单个缓存数据的大小限制；
- 第二个判断条件是mCachedEntries数据个数 - mExclusiveEntries数据个数小于最大缓存数量限制；
- 第三个判断条件是mCachedEntries数据大小 - mExclusiveEntries数据大小小于最大缓存大小 - 当前要缓存数据大小小于最大缓存大小的限制。

这个三个条件都满足的条件下，才可以put数据进去，put之前先将数据封装到一个Entry对象：
```java
//CountingMemoryCache
static <K, V> Entry<K, V> of(
    final K key,
    final CloseableReference<V> valueRef,
    final @Nullable EntryStateObserver<K> observer) {
  return new Entry<>(key, valueRef, observer);
}
```
一个Entry对象封装了数据，key，orphan状态，clientCount计数。

mCachedEntries.put(key, newEntry);方法就是存缓存的地方了：
```java
//CountingLruMap
public synchronized V put(K key, V value) {
    // We do remove and insert instead of just replace, in order to cause a structural change
    // to the map, as we always want the latest inserted element to be last in the queue.
    V oldValue = mMap.remove(key);
    mSizeInBytes -= getValueSizeInBytes(oldValue);
    mMap.put(key, value);
    mSizeInBytes += getValueSizeInBytes(value);
    return oldValue;
}
```
put方法里面的操作就是把数据方法放到map里面去，然后算出占用空间大小。

最后就是将执行newClientReference方法了：
```java
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
这个方法中先将Entry的clientCount变量加1，然后将entry中的数据封装到一个新的CloseableReference对象中，另外这个地方还定义了释放的回调，可以看看这个回调方法release(CloseableReference中执行close方法会回调到这里)：
```java
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
//CloseableReference
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
```
这里的mResourceReleaser就是我们初始化时传递过去ResourceReleaser对象了。

在回调的release方法里面执行releaseClientReference方法释放掉当前缓存的数据：
```java
//CountingMemoryCache
private void releaseClientReference(final Entry<K, V> entry) {
    boolean isExclusiveAdded;
    CloseableReference<V> oldRefToClose;
    synchronized (this) {
      decreaseClientCount(entry);
      isExclusiveAdded = maybeAddToExclusives(entry);
      oldRefToClose = referenceToClose(entry);
    }
    CloseableReference.closeSafely(oldRefToClose);
    maybeNotifyExclusiveEntryInsertion(isExclusiveAdded ? entry : null);
    maybeUpdateCacheParams();
    maybeEvictEntries();
}
```
- 将entry的clientCount计数减一
- 对于非orphan且clientCount=0的entry加入mExclusiveEntries
- 对于orphan且clientCount=0的entry关闭其封装的CloseableReference数据
- 更新内存缓存的参数配置
- 依据最新的内存缓存参数，开始删除一些符合条件的数据。看看maybeEvictEntries函数：
```java
//CountingMemoryCache
private void maybeEvictEntries() {
    ArrayList<Entry<K, V>> oldEntries;
    synchronized (this) {
      int maxCount = Math.min(
          mMemoryCacheParams.maxEvictionQueueEntries,
          mMemoryCacheParams.maxCacheEntries - getInUseCount());
      int maxSize = Math.min(
          mMemoryCacheParams.maxEvictionQueueSize,
          mMemoryCacheParams.maxCacheSize - getInUseSizeInBytes());
      oldEntries = trimExclusivelyOwnedEntries(maxCount, maxSize);
      makeOrphans(oldEntries);
    }
    maybeClose(oldEntries);
}
```
先获取到当前可用的最大缓存个数和最大缓存空间大小，然后执行trimExclusivelyOwnedEntries方法去删除mExclusiveEntries中的数据，前提条件是这个map中的数据个数和数据空间大小分别大于maxCount和maxSize。

对于已经在上一步中从Map中remove的数据，都保存下来，通过makeOrphans方法，把每一个entry都设置orphan为true。

最后对于entry符合orphan = true && clientCount = 0的都关闭掉，彻底释放。

到这里基本将内存缓存的存储和释放过程说完了，回到CountingMemoryCache的cache方法中。

经过newClientReference将缓存之后的数据重新封装到CloseableReference之后，再一次执行maybeEvictEntries判断是否需要压缩一下内存缓存数据，就回到了上面分析的删除缓存数据的过程了。

这之后将重新封装的CloseableReference对象返回。

返回的数据回到EncodedMemoryCacheProducer类的内部类EncodedMemoryCacheConsumer中，在onNewResultImpl函数中执行完cache方法之后，继续将返回的CloseableReference封装到EncodeImage中：
```java
//EncodedMemoryCacheProducer.EncodedMemoryCacheConsumer
EncodedImage cachedEncodedImage = new EncodedImage(cachedResult);
cachedEncodedImage.copyMetaDataFrom(newResult);
getConsumer().onNewResult(newResult, status);
```
copyMetaDataFrom是读取新返回数据的元数据，这个元数据图片数据的各种参数：
```java
//EncodedImage
public void copyMetaDataFrom(EncodedImage encodedImage) {
    mImageFormat = encodedImage.getImageFormat();//格式
    mWidth = encodedImage.getWidth();//宽度
    mHeight = encodedImage.getHeight();//高度
    mRotationAngle = encodedImage.getRotationAngle();//旋转方向
    mExifOrientation = encodedImage.getExifOrientation();//图片方向，定义在android.media.ExifInterface中
    mSampleSize = encodedImage.getSampleSize();//JPEG的采样大小
    mStreamSize = encodedImage.getSize();//byte数据大小
    mBytesRange = encodedImage.getBytesRange();//数据范围
    mColorSpace = encodedImage.getColorSpace();//色彩空间，比如RGB
}
```
这些参数解析出来之后，封装到了EncodedImage的对象中，用于下一步处理图片。

而内存此时缓存的是从上一步传递过来的MemoryPooledByteBuffer类型的未解码（编码）的数据。

