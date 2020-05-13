---
layout: post
title:  "Android OKHttp系列1-流程总结"
date:   2019-5-13 11:09:22 +0800
categories: Android
---

> 文章将会被同步至微信公众号：Android部落格
> 源码地址：https://github.com/square/okhttp

### 1、 调用示例
- 同步方式：

```java
new Thread(new Runnable() {
    @Override
    public void run() {
        try {
            OkHttpClient client = new OkHttpClient();
            Request request = new Request.Builder().url("http://www.baidu.com").build();
            Response response = client.newCall(request).execute();
            Log.d(TAG, "response sync:" + response.toString());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}).start();
```

通过追溯源码，流程图如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200331173538465.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTExOTg3NjQ=,size_16,color_FFFFFF,t_70#pic_center)

- 异步方式：

```java
try {
    OkHttpClient client = new OkHttpClient();
    Request request = new Request.Builder().url("http://www.baidu.com").build();
    client.newCall(request).enqueue(new Callback() {
        @Override
        public void onFailure(Request request, IOException e) {

        }

        @Override
        public void onResponse(Response response) throws IOException {
            Log.d(TAG, "response async:" + response.toString());
        }
    });
} catch (Exception e) {
    e.printStackTrace();
}
```

通过追溯源码，流程图如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200331173623311.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTExOTg3NjQ=,size_16,color_FFFFFF,t_70#pic_center)

#### 分析
同步和异步请求的核心方法都是`getResponseWithInterceptorChain()`，需要注意的是，同步方法没有在工作线程干活，而异步方法是在线程池里面执行，Android不允许在主线程里面做网络请求操作，如果同步请求的话，还必须在非主线程中。

异步方式请求是执行enqueue方法，有两个列表维护执行状态，`runningAsyncCalls`和`readyAsyncCalls`，分别是正在执行和等待执行列表，而同步方式则是直接提交请求。异步请求当一次AsyncCall执行完毕之后，在Dispatcher的`promoteCalls`方法会做两个状态列表的切换，等待列表切换到正在执行列表，同时删除等待列表中最前面的Call。如下：

```java
private void promoteCalls() {
  if (runningAsyncCalls.size() >= maxRequests) return; // Already running max capacity.
  if (readyAsyncCalls.isEmpty()) return; // No ready calls to promote.

  for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
    AsyncCall call = i.next();

    if (runningCallsForHost(call) < maxRequestsPerHost) {
      i.remove();
      runningAsyncCalls.add(call);
      executorService().execute(call);
    }

    if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
  }
}
```

遍历`readyAsyncCalls`列表里面的call,如果正在运行并小于最大的请求数，就可以加到`runningAsyncCalls`中了，然后接下来`execute`执行这个请求。
而`promoteCalls`方法是被`Dispatcher`的`finished`方法执行。`RealCall`中同步或异步`execute`方法执行完毕后会在`finally`中执行`client.dispatcher().finished(this)`方法，如下：

```java
void finished(AsyncCall call) {
  finished(runningAsyncCalls, call, true);
}

/** Used by {@code Call#execute} to signal completion. */
void finished(RealCall call) {
  finished(runningSyncCalls, call, false);
}
private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
  int runningCallsCount;
  Runnable idleCallback;
  synchronized (this) {
    if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
    if (promoteCalls) promoteCalls();
    runningCallsCount = runningCallsCount();
    idleCallback = this.idleCallback;
  }

  if (runningCallsCount == 0 && idleCallback != null) {
    idleCallback.run();
  }
}
```

第一个是finished异步请求完成之后调用，第二finished是同步请求完成之后调用，最终都会调用到带泛型参数的finished，并将执行完的call从runningAsyncCalls(异步)或runningSyncCalls(同步)中删除。promoteCalls参数用来区分是否是异步请求，如果是的话，执行promoteCalls方法。

微信公众号：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200331173712101.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTExOTg3NjQ=,size_16,color_FFFFFF,t_70#pic_center)