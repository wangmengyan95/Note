OkHttp是目前最流行的android httpClient，源代码在[这里](https://github.com/square/okhttp)。我对我比较感兴趣的几点做一点笔记。

## Dispatcher
OkHttp通过Dispatcher机制来实现单一httpClient多线程执行网络请求。
```
OkHttpClient okHttpClient = new OkHttpClient();
// Sync
Response response = okHttpClient.newCall(request).execute();
// Async
okHttpClient.newCall(request).enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {

    }

    @Override
    public void onResponse(Call call, Response response) throws IOException {

    }
});
```

okHttpClient.newCall(request)实际上生成了一个RealCall的对象。

RealCall enqueue方法
```
@Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    // 加入Dispatcher
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```
这个方法非常直接，将responseCallback包装成AsyncCall对象，加入client的Dispatcher。Client的Dispatcher是在client初始化的时候指定的。

Dispatcher enqueue方法

```
synchronized void enqueue(AsyncCall call) {
  if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
    runningAsyncCalls.add(call);
    executorService().execute(call);
  } else {
    readyAsyncCalls.add(call);
  }
}
```

Dispatcher executorService方法

```
public synchronized ExecutorService executorService() {
  if (executorService == null) {
    executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
        new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
  }
  return executorService;
}
```
该方法返回一个ThreadPoolExecutor。比较重要的参数是SynchronousQueue，即这个Executor对于执行的Runnable是先进先出的。

