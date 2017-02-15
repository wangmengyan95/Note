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
Handler的绝大部分方法都是用来send Message的。各种各样不同的方法其实最后就是调用了sendMessageAtTime这个函数。从源代码看出，所有Message的target就是handler自身。

