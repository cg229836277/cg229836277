---
layout: post
title:  "Android-Fresco系列7 解码数据内存缓存"
date:   2019-6-10 9:53:25 +0800
categories: Android
---

## 一、BitmapMemoryCacheProducer
从第三篇文章中可以看到Producer的初始化顺序是BitmapMemoryCacheProducer->DecodeProducer，由此看到解码成功的图片还要经过内存缓存，等于是说光内存缓存就有两份，一份编码的，一份解码的。

这边文章讲解码之后的数据缓存。

经过DecodeProducer解码之后的数据，被包裹在CloseableStaticBitmap对象中，然后向上传递。

####  1. produceResults
```java
//BitmapMemoryCacheProducer
@Override
public void produceResults(
final Consumer<CloseableReference<CloseableImage>> consumer, final ProducerContext producerContext) {
    CloseableReference<CloseableImage> cachedReference = mMemoryCache.get(cacheKey);
    
    if (cachedReference != null) {
        consumer.onNewResult(cachedReference, BaseConsumer.simpleStatusForIsLast(isFinal));
        cachedReference.close();
    } else {
        Consumer<CloseableReference<CloseableImage>> wrappedConsumer = wrapConsumer(
                  consumer, cacheKey, producerContext.getImageRequest().isMemoryCacheEnabled());
        mInputProducer.produceResults(wrappedConsumer, producerContext);
    }
}
```
先看看缓存里面有没有目标的数据，有的话通知其他consumer可以结束了，我这里已经取到数据了。当然目标数据要符合质量要求。

####  2. Consumer
当前缓存中找不到目标数据的话，就要设置消费者来拦截并处理数据了。消费者的定义在wrapConsumer方法中：
```java
//BitmapMemoryCacheProducer
protected Consumer<CloseableReference<CloseableImage>> wrapConsumer(){
    return new DelegatingConsumer<
    CloseableReference<CloseableImage>, CloseableReference<CloseableImage>>(consumer) {
        @Override
        public void onNewResultImpl(
        CloseableReference<CloseableImage> newResult, @Status int status) {
            final boolean isLast = isLast(status);
            if(newResult == null && isLast){
                getConsumer().onNewResult(null, status);
                return;
            }
            CloseableReference<CloseableImage> currentCachedResult = mMemoryCache.get(cacheKey);
            if (currentCachedResult != null) {
               getConsumer().onNewResult(currentCachedResult, status);
                return;
            }
            CloseableReference<CloseableImage> newCachedResult = null;
            if (isMemoryCacheEnabled) {
                newCachedResult = mMemoryCache.cache(cacheKey, newResult);
                getConsumer().onNewResult((newCachedResult != null) ? newCachedResult : newResult, status);
            }
        }
    }
}
```
三个步骤：

1. 状态检查，是结束标志且传过来的数据为null的话，不往下走了，直接往上传递null值；
2. 检查当前缓存里面有没有当前key对应的value，有且符合质量的话，就返回当前内存里面的缓存数据；
3. 前两个条件都不满足，就要将数据缓存了，这里缓存的逻辑与编码数据的缓存是一样的。

```java
//CountingMemoryCache
public @Nullable CloseableReference<V> cache(
      final K key, final CloseableReference<V> valueRef, final EntryStateObserver<K> observer) {
    maybeUpdateCacheParams();
    
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
    
    maybeEvictEntries();
}
```
五个步骤：
- 检查当前的内存空间是否符合内存缓存的大小限制
- 删除待删除和已缓存map中存在key值对应的旧数据
- 将Entry的orphan属性置为1，并且检查是否可以关闭释放
- 此时再检查数据大小是否符合单个数据缓存大小的限制

再贴一下内存缓存的空间大小限制：
```java
public class DefaultBitmapMemoryCacheParamsSupplier implements Supplier<MemoryCacheParams> {
  private static final int MAX_CACHE_ENTRIES = 256;
  private static final int MAX_EVICTION_QUEUE_SIZE = Integer.MAX_VALUE;
  private static final int MAX_EVICTION_QUEUE_ENTRIES = Integer.MAX_VALUE;
  private static final int MAX_CACHE_ENTRY_SIZE = Integer.MAX_VALUE;
  private static final long PARAMS_CHECK_INTERVAL_MS = TimeUnit.MINUTES.toMillis(5);

  private final ActivityManager mActivityManager;

  public DefaultBitmapMemoryCacheParamsSupplier(ActivityManager activityManager) {
    mActivityManager = activityManager;
  }

  @Override
  public MemoryCacheParams get() {
    return new MemoryCacheParams(
        getMaxCacheSize(),
        MAX_CACHE_ENTRIES,
        MAX_EVICTION_QUEUE_SIZE,
        MAX_EVICTION_QUEUE_ENTRIES,
        MAX_CACHE_ENTRY_SIZE,
        PARAMS_CHECK_INTERVAL_MS);
  }

  private int getMaxCacheSize() {
    final int maxMemory =
        Math.min(mActivityManager.getMemoryClass() * ByteConstants.MB, Integer.MAX_VALUE);
    if (maxMemory < 32 * ByteConstants.MB) {
      return 4 * ByteConstants.MB;
    } else if (maxMemory < 64 * ByteConstants.MB) {
      return 6 * ByteConstants.MB;
    } else {
      // We don't want to use more ashmem on Gingerbread for now, since it doesn't respond well to
      // native memory pressure (doesn't throw exceptions, crashes app, crashes phone)
      if (Build.VERSION.SDK_INT < Build.VERSION_CODES.HONEYCOMB) {
        return 8 * ByteConstants.MB;
      } else {
        return maxMemory / 4;
      }
    }
  }
}
```
> **getMemoryClass** 
Return the approximate per-application memory class of the current device. This gives you an idea of how hard a memory limit you should impose on your application to let the overall system work best. The returned value is in megabytes; the baseline Android memory class is 16 (which happens to be the Java heap limit of those devices); some devices with more memory may return 24 or even higher numbers.

- 符合条件，将CloseableReference<CloseableStaticBitmap>
类型的数据put到mCachedEntries中，同时将clientCount属相加1，orphan置为false。
- 再次检查内存空间，看看是否有可以删除的数据，腾出缓存空间。

缓存成功，返回缓存之后的数据，并将数据上传个下一个消费者。
