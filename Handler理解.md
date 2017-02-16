Handler也是android的重要组件，它一般与Looper和MessageQueue配合，负责分发和处理Message。

Handler的源码在[这里](https://github.com/android/platform_frameworks_base/blob/master/core/java/android/os/Handler.java)。

```
public Handler(Callback callback, boolean async) {
    ...
    
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```
Handler的构造函数简单明了。从中我们能够很容易看出一个Handler只对应一个Looper和MessageQueue。


```
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```
Handler的绝大部分方法都是用来send Message的。各种各样不同的方法其实最后就是调用了sendMessageAtTime这个函数。从源代码看出，所有Message的target就是Handler自身。

```
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```
另外一个值得注意的方法就是getPostMessage。Handler send相关的函数一般也有提供Runnable参数的版本。其实本质上，Handler只不过将Runnable作为了Message的callback暂存起来，最终还是通过Message完成的消息传递。

```
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}

private static void handleCallback(Message message) {
    message.callback.run();
}

public interface Callback {
    public boolean handleMessage(Message msg);
}
```

dispatchMessage是另外一个比较重要的方法。它是在Looper.loop()中被调用的。它定义了Handler处理msg的顺序。

1. 如果msg包含Callback，则优先用此Callback处理。Callback如上所述，就是定义msg时传入的Runnable参数。
2. 如果Handler定义了Callback，则用此Callback处理。这个Callback是Handler类里定义的一个接口，只包含了handleMessage(Message msg)这一个函数。
3. 如果上述条件都不成立，则默认调用handleMessage方法。handleMessage是一个需要被子类重写的方法。
