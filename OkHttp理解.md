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

AsyncCall execute方法
```
@Override protected void execute() {
  boolean signalledCallback = false;
  try {
    Response response = getResponseWithInterceptorChain();
    ...
  } catch (IOException e) {
    ...
  } finally {
    client.dispatcher().finished(this);
  }
}
```
当AsyncCall被加入ThreadPoolExecutor后，最终会被执行，即调用AsyncCall execute方法。getResponseWithInterceptorChain()是真正执行http request的方法，我们以后进行详细分析。在执行完request获得response后，最后会执行Dispatcher finished方法。

Dispatcher finished方法 promoteCalls方法
```
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
Dispatcher finished方法最后会调用promoteCalls方法。该方法会从readyAsyncCalls列表中不断取出新的AsyncCall，加入ThreadPoolExecutor，直到达到该HttpClient可同时执行request的上限。

综上所述，Dispatcher内部维护一个runningAsyncCalls队列，一个readyAsyncCalls队列，通过ThreadPoolExecutor不断执行runningAsyncCalls中的AsyncCall，执行完后通过promoteCalls方法将readyAsyncCalls的AsyncCall移动到runningAsyncCalls，至此实现单一httpClient多线程执行网络请求。
