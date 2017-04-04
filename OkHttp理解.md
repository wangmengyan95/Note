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

## Interceptor
Interceptor机制是我认为OkHttp非常有趣的机制。它将不同的功能分割成了不同的Interceptor，不同的Interceptor可以自由的组合，实现一系列的功能。

Interceptor的定义如下
```
public interface Interceptor {
  Response intercept(Chain chain) throws IOException;

  interface Chain {
    Request request();

    Response proceed(Request request) throws IOException;

    Connection connection();
  }
}
```
Chain是由RealInterceptorChain实现的
```
public final class RealInterceptorChain implements Interceptor.Chain {
  private final List<Interceptor> interceptors;
  private final int index;

  ...

  public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      Connection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    ....

    RealInterceptorChain next = new RealInterceptorChain(
        interceptors, streamAllocation, httpCodec, connection, index + 1, request);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    ....

    return response;
  }
}

```

Interceptor的实现非常多，但是不同的Interceptor的均包括类似的逻辑。
```
public final class XXXInterceptor implements Interceptor {

  @Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();

    // Change request if necessary
    ....

    Response response = chain.proceed(newRequest);

    // Change response if necessary
    ...

    return newResponse;
  }
}
```
由上面的代码就可以看出，chain中维护了一个Interceptor的列表，并且维护了当前Interceptor的index，我们会从Interceptor的列表中取出对应的Interceptor，并且新生成一个index指向下一个Interceptor的chain，并调用interceptor.interceptor(newChain)。在Interceptor的intercept方法中，我会调用chain.proceed(request)，使得下一个Interceptor有机会去intercept chain。至此，request会先从前到后经过Interceptor list中的每一个Interceptor，直至最底层的Interceptor真正去执行这个request并且得到response。而response会从后到前经过Interceptor list中的每一个Interceptor，最后被最开始的chian.proceed()方法返回。

## ConnectionPool
OkHttp是通过ConnectionPool实现连接的重用。每次Client与Sever进行Socket的连接，都需要经过握手等等一系列步骤，当Client频繁与Server进行通信时，每次重新建立Socket连接显然会带来很多不必要的开销。在这种情况下，我们可以在Socket连接建立后，保持Socket一段时间再释放，从而避免频繁握手带来的开销。

```
public Builder() {
  ...
  connectionPool = new ConnectionPool();
  ...
}
```
每个OkHttpClient对应着一个ConnectionPool，当OkHttpClient初始化时，如果没有特别指定，OkHttpClient即会使用默认的ConnectionPool。

OkHttp通过StreamAllocation对Connection进行操作，如果我们把Connection看做是对Socket的封装，StreamAllocation就是对Connection的封装。之所以需要StreamAllocation的原因就是Http2协议支持对于Connection的复用。即我们能够利用一个Connection进行并发的请求。因此，一个Connection可能对应多个StreamAllocation对象。对于Http1协议，因为不支持并发请求，StreamAllocation与Connection一一对应。

```
private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
    boolean connectionRetryEnabled) throws IOException {
  Route selectedRoute;
  synchronized (connectionPool) {
    ...
    // Attempt to get a connection from the pool.
    Internal.instance.get(connectionPool, address, this);
    if (connection != null) {
      return connection;
    }

    selectedRoute = route;
  }

  ...
  synchronized (connectionPool) {
    // Pool the connection.
    Internal.instance.put(connectionPool, result);
    ....
  }
  ...
  return result;
}
```
```
RealConnection get(Address address, StreamAllocation streamAllocation) {
  assert (Thread.holdsLock(this));
  for (RealConnection connection : connections) {
    if (connection.isEligible(address)) {
      streamAllocation.acquire(connection);
      return connection;
    }
  }
  return null;
}

void put(RealConnection connection) {
  assert (Thread.holdsLock(this));
  if (!cleanupRunning) {
    cleanupRunning = true;
    executor.execute(cleanupRunnable);
  }
  connections.add(connection);
}
```

StreamAllocation获取Connection的方式比较容易理解。它首先会根据address从ConnectionPool中尝试获取可用的Connection，如果没有的话就会创建Connection，并且放入ConnectionPool中。ConnectionPool内部维护了一个队列connections，用于存储可用的Connection。

```
long cleanup(long now) {
  int inUseConnectionCount = 0;
  int idleConnectionCount = 0;
  RealConnection longestIdleConnection = null;
  long longestIdleDurationNs = Long.MIN_VALUE;

  // Find either a connection to evict, or the time that the next eviction is due.
  synchronized (this) {
    for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
      RealConnection connection = i.next();

      // If the connection is in use, keep searching.
      if (pruneAndGetAllocationCount(connection, now) > 0) {
        inUseConnectionCount++;
        continue;
      }

      idleConnectionCount++;

      // If the connection is ready to be evicted, we're done.
      long idleDurationNs = now - connection.idleAtNanos;
      if (idleDurationNs > longestIdleDurationNs) {
        longestIdleDurationNs = idleDurationNs;
        longestIdleConnection = connection;
      }
    }

    if (longestIdleDurationNs >= this.keepAliveDurationNs
        || idleConnectionCount > this.maxIdleConnections) {
      // We've found a connection to evict. Remove it from the list, then close it below (outside
      // of the synchronized block).
      connections.remove(longestIdleConnection);
    } else if (idleConnectionCount > 0) {
      // A connection will be ready to evict soon.
      return keepAliveDurationNs - longestIdleDurationNs;
    } else if (inUseConnectionCount > 0) {
      // All connections are in use. It'll be at least the keep alive duration 'til we run again.
      return keepAliveDurationNs;
    } else {
      // No connections, idle or in use.
      cleanupRunning = false;
      return -1;
    }
  }

  closeQuietly(longestIdleConnection.socket());

  // Cleanup again immediately.
  return 0;
}
```

ConnectionPool中有专门的ThreadPoolExecutor用来执行清理Connection的Runnable。这个Runnable是Connection被加入ConnectionPool时被启动。在这个Runnalbe中，我们尝试找出空闲时间最长的Connection。
1. 如果这个Connection空闲的时间超过了最长允许空闲时间，则清理这个Connection，并马上继续再次执行Runnable
2. 如果这个Connection空闲的时间没有超过最长允许空闲时间，则在keepAliveDurationNs - longestIdleDurationNs后再次执行Runnable
3. 如果没有空闲的Connection，在keepAliveDurationNs后再次执行Runnable
4. 如果没有Connection，则不继续执行Runnable，直达下次有新的Connection加入ConnectionPool时为止
