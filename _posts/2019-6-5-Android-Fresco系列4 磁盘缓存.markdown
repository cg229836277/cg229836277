---
layout: post
title:  "Android-Fresco系列4 磁盘缓存"
date:   2019-6-5 9:53:25 +0800
categories: Android
---

先看流程图：
![](https://i.loli.net/2019/06/05/5cf7ab7c4597630289.jpg)

## 一、DiskCacheWriteProducer
从NetworkFetchProducer传递过来的数据是EncodedImage类型，里面的未解码数据是CloseableReference<MemoryPooledByteBuffer>类型。

#### 1) produceResults

定义了一个DiskCacheWriteConsumer用于消费接收到的消息。
```java
//DiskCacheWriteProducer
Consumer<EncodedImage> consumer;
if (producerContext.getImageRequest().isDiskCacheEnabled()) {
    consumer = new DiskCacheWriteConsumer(
        consumerOfDiskCacheWriteProducer,
        producerContext,
        mDefaultBufferedDiskCache,
        mSmallImageBufferedDiskCache,
        mCacheKeyFactory
    );
}
```
#### 2) DiskCacheWriteConsumer
```java
//DiskCacheWriteConsumer
@Override
public void onNewResultImpl(EncodedImage newResult, @Status int status) {
    final ImageRequest imageRequest = mProducerContext.getImageRequest();
    final CacheKey cacheKey = mCacheKeyFactory.getEncodedCacheKey(imageRequest, mProducerContext.getCallerContext());
    mDefaultBufferedDiskCache.put(cacheKey, newResult);
    getConsumer().onNewResult(newResult, status);
}
```
mCacheKeyFactory对应的是在ImagePipelineConfig类中默认初始化的`DefaultCacheKeyFactory.getInstance()`，如果用户不设置的话。
```java
//DefaultCacheKeyFactory
@Override
public CacheKey getEncodedCacheKey(
  ImageRequest request,
  Uri sourceUri,
  @Nullable Object callerContext) {
    return new SimpleCacheKey(getCacheKeySourceUri(sourceUri).toString());
}
```
实际上就是把uri封装到SimpleCacheKey对象。

#### 3) 关键变量初始化
mDefaultBufferedDiskCache对应的是ImagePipelineFactory类中getProducerFactory方法时初始化的,调用getMainBufferedDiskCache方法后返回一个BufferedDiskCache对象给mDefaultBufferedDiskCache：
```java
//ImagePipelineFactory
public BufferedDiskCache getMainBufferedDiskCache() {
    if (mMainBufferedDiskCache == null) {
      mMainBufferedDiskCache =
          new BufferedDiskCache(
              getMainFileCache(),
              mConfig.getPoolFactory().getPooledByteBufferFactory(mConfig.getMemoryChunkType()),
              mConfig.getPoolFactory().getPooledByteStreams(),
              mConfig.getExecutorSupplier().forLocalStorageRead(),
              mConfig.getExecutorSupplier().forLocalStorageWrite(),
              mConfig.getImageCacheStatsTracker());
    }
    return mMainBufferedDiskCache;
}
```
- DiskCacheConfig

其中getMainFileCache初始化了一个DiskCacheConfig类，主要是磁盘缓存的一些配置信息。
```java
//ImagePipelineFactory
public FileCache getMainFileCache() {
    if (mMainFileCache == null) {
      DiskCacheConfig diskCacheConfig = mConfig.getMainDiskCacheConfig();
      mMainFileCache = mConfig.getFileCacheFactory().get(diskCacheConfig);
    }
    return mMainFileCache;
}
```
- DynamicDefaultDiskStorageFactory

getFileCacheFactory初始化一个mFileCacheFactory对象,默认是：
```java
//ImagePipelineConfig
mFileCacheFactory = builder.mFileCacheFactory == null
    ? new DiskStorageCacheFactory(new DynamicDefaultDiskStorageFactory())
    : builder.mFileCacheFactory;

//DynamicDefaultDiskStorageFactory
public class DynamicDefaultDiskStorageFactory implements DiskStorageFactory {

  @Override
  public DiskStorage get(DiskCacheConfig diskCacheConfig) {
    return new DynamicDefaultDiskStorage(
        diskCacheConfig.getVersion(),
        diskCacheConfig.getBaseDirectoryPathSupplier(),
        diskCacheConfig.getBaseDirectoryName(),
        diskCacheConfig.getCacheErrorLogger());
  }
}

//DiskStorageCacheFactory
@Override
public FileCache get(DiskCacheConfig diskCacheConfig) {
    return buildDiskStorageCache(diskCacheConfig, mDiskStorageFactory.get(diskCacheConfig));
}
```
DiskStorageCacheFactory封装了一个DiskStorageCacheFactory对象，DiskStorageCacheFactory是一个磁盘缓存空间大小的管理类。
- DiskStorageCache

最后通过DiskStorageCacheFactory.get方法生成一个FileCache对象返回（DiskStorageCache实现了FileCache接口）：
```java
//DiskStorageCache
public static DiskStorageCache buildDiskStorageCache(
  DiskCacheConfig diskCacheConfig,
  DiskStorage diskStorage,
  Executor executorForBackgroundInit) {
    DiskStorageCache.Params params = new DiskStorageCache.Params(
        diskCacheConfig.getMinimumSizeLimit(),
        diskCacheConfig.getLowDiskSpaceSizeLimit(),
        diskCacheConfig.getDefaultSizeLimit());
    
    return new DiskStorageCache(
        diskStorage,
        diskCacheConfig.getEntryEvictionComparatorSupplier(),
        params,
        diskCacheConfig.getCacheEventListener(),
        diskCacheConfig.getCacheErrorLogger(),
        diskCacheConfig.getDiskTrimmableRegistry(),
        diskCacheConfig.getContext(),
        executorForBackgroundInit,
        diskCacheConfig.getIndexPopulateAtStartupEnabled());
}
```
- DiskCacheConfig

看看磁盘缓存的参数设置吧：
最小缓存限制：

`private long mMaxCacheSizeOnVeryLowDiskSpace = 2 * ByteConstants.MB;`

低存储设置空间大小限制：

`private long mMaxCacheSizeOnLowDiskSpace = 10 * ByteConstants.MB;`

默认大小限制：

`private long mMaxCacheSize = 40 * ByteConstants.MB;`

最后将FileCache封装到BufferedDiskCache对象返回,构造函数如下：
```java
//BufferedDiskCache
public BufferedDiskCache(){
    mStagingArea = StagingArea.getInstance();
}

//StagingArea
private StagingArea() {
    mMap = new HashMap<>();
}
```
实际上StagingArea封装了一个`private Map<CacheKey, EncodedImage> mMap;`,只是用来存储key和编码的value，实际的操作在BufferedDiskCache中。

在上面的堆栈中，调用到`mDefaultBufferedDiskCache.put(cacheKey, newResult);`，实际是调用BufferedDiskCache的put方法：
```java
//BufferedDiskCache
public void put(final CacheKey key, EncodedImage encodedImage) {
    mStagingArea.put(key, encodedImage);
    mWriteExecutor.execute(new Runnable() {
        @Override
        public void run() {
            writeToDiskCache(key, finalEncodedImage);
        }
    });
}
//BufferedDiskCache
private void writeToDiskCache(final CacheKey key, final EncodedImage encodedImage) {
    mFileCache.insert(key, new WriterCallback() {
        @Override
        public void write(OutputStream os) throws IOException {
            mPooledByteStreams.copy(encodedImage.getInputStream(), os);
        }
    });
}
```

mFileCache实际对应的是DiskStorageCache类，看看这个类中的insert方法：
```java
//DiskStorageCache
public BinaryResource insert(CacheKey key, WriterCallback callback){
    String resourceId = CacheKeyUtil.getFirstResourceId(key);
    DiskStorage.Inserter inserter = startInsert(resourceId, key);
    inserter.writeData(callback, key);
    // Committing the file is synchronized
    BinaryResource resource = endInsert(inserter, key, resourceId);
    return resource;
}

//DiskStorageCache
private DiskStorage.Inserter startInsert(
  final String resourceId,
  final CacheKey key)
  throws IOException {
    maybeEvictFilesInCacheDir();
    return mStorage.insert(resourceId, key);
}
```
## 二、开始缓存

mStorage对应的是DynamicDefaultDiskStorage类，看看insert方法：
```java
//DynamicDefaultDiskStorage
@Override
public Inserter insert(String resourceId, Object debugInfo) throws IOException {
    return get().insert(resourceId, debugInfo);
}
//DynamicDefaultDiskStorage
synchronized DiskStorage get() throws IOException {
    if (shouldCreateNewStorage()) {
      // discard anything we created
      deleteOldStorageIfNecessary();
      createStorage();
    }
    return Preconditions.checkNotNull(mCurrentState.delegate);
}
```
#### 1) 创建文件夹

第一次调用磁盘缓存时，是没有创建对应的缓存目录的，所以需要调用createStorage方法：
```java
//DynamicDefaultDiskStorage
private void createStorage() throws IOException {
    File rootDirectory = new File(mBaseDirectoryPathSupplier.get(), mBaseDirectoryName);
    createRootDirectoryIfNecessary(rootDirectory);
    DiskStorage storage = new DefaultDiskStorage(rootDirectory, mVersion, mCacheErrorLogger);
    mCurrentState = new State(rootDirectory, storage);
}
```
mBaseDirectoryPathSupplier是在DiskCacheConfig的build方法中定义的：
```java
//DiskCacheConfig
public DiskCacheConfig build() {
    if (mBaseDirectoryPathSupplier == null && mContext != null) {
        mBaseDirectoryPathSupplier = new Supplier<File>() {
          @Override
          public File get() {
            return mContext.getApplicationContext().getCacheDir();
          }
        };
    }
}
```
可见根目录是所在应用的缓存路径/data/data/packageName/cache，然后根据根目录和版本号格式化生成一个子目录，最后统一封装成一个DefaultDiskStorage对象：
```java
//DefaultDiskStorage
public DefaultDiskStorage(){
    mVersionDirectory = new File(mRootDirectory, getVersionSubdirectoryName(version));
}

//DefaultDiskStorage
static String getVersionSubdirectoryName(int version) {
return String.format(
    (Locale) null,
    "%s.ols%d.%d",
    DEFAULT_DISK_STORAGE_VERSION_PREFIX,
    SHARDING_BUCKET_COUNT,
    version);
}
```
此时mVersionDirectory对应的路径是:/data/data/packageName/cache/image_cache/v2.ols100.1
storage对象对应的是DefaultDiskStorage类，然后封装到State类中，这个类中初始化了delegate对象，代理的就是DefaultDiskStorage的storage对象。

接下来就在DefaultDiskStorage中执行insert方法了：
```java
//DefaultDiskStorage
public Inserter insert(){
    ileInfo info = new FileInfo(FileType.TEMP, resourceId);
    File parent = getSubdirectory(info.resourceId);
    File file = info.createTempFile(parent);
    return new InserterImpl(resourceId, file);
}
```
resourceId的生成规则是：
```java
//CacheKeyUtil
private static String secureHashKey(final CacheKey key)
    return SecureHashUtil.makeSHA1HashBase64(key.getUriString().getBytes("UTF-8"));
}
```
接着依据resourceId的hash值与SHARDING_BUCKET_COUNT值取余再生成一层子目录：
```java
////DefaultDiskStorage
private String getSubdirectoryPath(String resourceId) {
    String subdirectory = String.valueOf(Math.abs(resourceId.hashCode() % SHARDING_BUCKET_COUNT));
    return mVersionDirectory + File.separator + subdirectory;
}
```
最后子目录可能是：/data/data/com.chuck.demo/cache/image_cache/v2.ols100.1/94/

#### 2) 创建临时文件
接下来在这个文件夹下创建临时文件createTempFile：
```java
public File createTempFile(File parent) throws IOException {
  File f = File.createTempFile(resourceId + ".", TEMP_FILE_EXTENSION, parent);
  return f;
}
```
此时后缀是.tmp，前缀是resourceId。

接下来将resourceId和临时文件封装到InserterImpl对象返回。

回到DiskStorageCache类的insert方法，调用startInsert返回的DiskStorage.Inserter对象，其对应的是InserterImpl类，接着调用writeData方法：
```java
//InserterImpl
@Override
public void writeData(WriterCallback callback, Object debugInfo){
    FileOutputStream fileStream = new FileOutputStream(mTemporaryFile);
    CountingOutputStream countingStream = new CountingOutputStream(fileStream);
    callback.write(countingStream);
    // just in case underlying stream's close method doesn't flush:
    // we flush it manually and inside the try/catch
    countingStream.flush();
}
```
此时通过临时文件创建一个输出流对象countingStream，然后通过callBack对象回调到BufferedDiskCache的writeToDiskCache方法中定义WriterCallback内部类的地方，实际就一行代码：
```java
//BufferedDiskCache
private void writeToDiskCache(
final CacheKey key,
final EncodedImage encodedImage) {
    mFileCache.insert(key, new WriterCallback() {
        @Override
        public void write(OutputStream os)  {
            mPooledByteStreams.copy(encodedImage.getInputStream(), os);
        }
    });
}

//EncodedImage
public @Nullable InputStream getInputStream() {
    CloseableReference<PooledByteBuffer> pooledByteBufferRef =
        CloseableReference.cloneOrNull(mPooledByteBufferRef);
    if (pooledByteBufferRef != null) {
      try {
        return new PooledByteBufferInputStream(pooledByteBufferRef.get());
      } finally {
        CloseableReference.closeSafely(pooledByteBufferRef);
      }
    }
    return null;
}
```
这行代码的意思是将已经编码图片的stream复制到os对象中，这个os指输出流，对应的是mTemporaryFile临时文件。

看看这个copy方法：
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
看到mByteArrayPool.get这里，又用到了Bucket那一套分配内存了，因为mByteArrayPool是GenericByteArrayPool类型，这里的mTempBufSize默认是16kb。

#### 3) 创建正式文件

此时就已经完成了已编码图片的数据流拷贝工作。回到
DiskStorageCache类的insert方法，调用endInsert方法：
```java
//DiskStorageCache
private BinaryResource endInsert(
  final DiskStorage.Inserter inserter,
  final CacheKey key,
  String resourceId) throws IOException {
    synchronized (mLock) {
      BinaryResource resource = inserter.commit(key);
      mResourceIndex.add(resourceId);
      mCacheStats.increment(resource.size(), 1);
      return resource;
    }
}
```
inserter.commit在DefaultDiskStorage的内部类InserterImpl中调用：
```java
//DefaultDiskStorage.InserterImpl
@Override
public BinaryResource commit(Object debugInfo){
    File targetFile = getContentFileFor(mResourceId);
    FileUtils.rename(mTemporaryFile, targetFile);
    if (targetFile.exists()) {
    targetFile.setLastModified(mClock.now());
  }
  return FileBinaryResource.createOrNull(targetFile);
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
此时就是正式文件了，type对应的是FileType.CONTENT，是cnt字符，然后依据resourceId的hash值与SHARDING_BUCKET_COUNT取余的绝对值再一次生成一个文件路径，这个文件路径与临时文件路径一样，但是后缀变成了cnt，接下来将另外文件修改名称为正式文件名，并修改文件最后修改时间。最后的操作是返回一个BinaryResource对象，实际是子类FileBinaryResource，这个对象里面还封装了一个File对象，包含读文件相关的操作。

#### 4) 缓存空间检查
在所有数据都写入完毕之后，在DiskStorageCache类的startInsert方法中的`mStorage.insert`方法调用之前，还调用了maybeEvictFilesInCacheDir方法，主要做一些缓存空间的控制工作：
```java
//DiskStorageCache
private void maybeEvictFilesInCacheDir() throws IOException {
    synchronized (mLock) {
      boolean calculatedRightNow = maybeUpdateFileCacheSize();
    
      // Update the size limit (mCacheSizeLimit)
      updateFileCacheSizeLimit();
    
      long cacheSize = mCacheStats.getSize();
      // If we are going to evict force a recalculation of the size
      // (except if it was already calculated!)
      if (cacheSize > mCacheSizeLimit && !calculatedRightNow) {
        mCacheStats.reset();
        maybeUpdateFileCacheSize();
      }
    
      // If size has exceeded the size limit, evict some files
      if (cacheSize > mCacheSizeLimit) {
      evictAboveSize(
          mCacheSizeLimit * 9 / 10,
          CacheEventListener.EvictionReason.CACHE_FULL); // 90%
      }
    }
}
```
这个方法中的现看看内外部存储是否符合默认存储大小的限制，符合的话，就更新相关限制；对于当前缓存大小已经超出限制的部分，默认的话就是40M的0.9（36M），则删除超出的部分。evictAboveSize最终调用purgeUnexpectedResources方法：
```java
//DefaultDiskStorage
@Override
public void purgeUnexpectedResources() {
    FileTree.walkFileTree(mRootDirectory, new PurgingVisitor());
}

//FileTree
public static void walkFileTree(File directory, FileTreeVisitor visitor) {
    visitor.preVisitDirectory(directory);
    File[] files = directory.listFiles();
    if (files != null) {
      for (File file: files) {
        if (file.isDirectory()) {
          walkFileTree(file, visitor);
        } else {
          visitor.visitFile(file);
        }
      }
    }
    visitor.postVisitDirectory(directory);
}
```
删除的策略在PurgingVisitor中：
```java
private class PurgingVisitor implements FileTreeVisitor {
    private boolean insideBaseDirectory;
    
    @Override
    public void preVisitDirectory(File directory) {
      if (!insideBaseDirectory && directory.equals(mVersionDirectory)) {
        // if we enter version-directory turn flag on
        insideBaseDirectory = true;
      }
    }
    
    @Override
    public void visitFile(File file) {
      if (!insideBaseDirectory || !isExpectedFile(file)) {
        file.delete();
      }
    }
    
    @Override
    public void postVisitDirectory(File directory) {
      if (!mRootDirectory.equals(directory)) { // if it's root directory we must not touch it
        if (!insideBaseDirectory) {
          // if not in version-directory then it's unexpected!
          directory.delete();
        }
      }
      if (insideBaseDirectory && directory.equals(mVersionDirectory)) {
        // if we just finished visiting version-directory turn flag off
        insideBaseDirectory = false;
      }
    }
    
    private boolean isExpectedFile(File file) {
      FileInfo info = getShardFileInfo(file);
      if (info == null) {
        return false;
      }
      if (info.type == FileType.TEMP) {
        return isRecentFile(file);
      }
      Preconditions.checkState(info.type == FileType.CONTENT);
      return true;
    }
    
    /**
     * @return true if and only if the file is not old enough to be considered an old temp file
     */
    private boolean isRecentFile(File file) {
      return file.lastModified() > (mClock.now() - TEMP_FILE_LIFETIME_MS);
    }
};
```
对于在v2.ols100.1目录下的文件，如果文件夹名称策略与既有策略不相符，则删除；对于tmp结尾的临时文件，创建时间超过30min，则删除；对于不在/data/data/packageName/cache/image_cache/目录，且不在v2.ols100.1目录下的文件夹，则删除。

到这里磁盘缓存告一段落。