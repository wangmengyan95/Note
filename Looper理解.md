
Looper是Android线程通信的基础组件之一。顾名思义，Looper循环不断的从MessageQueue中取出message，交给Handler来处理。从后面的代码分析中我们可以发现，Looper与MessageQueue是一一对应的关系。

Looper的源代码在[这里](https://github.com/android/platform_frameworks_base/blob/master/core/java/android/os/Looper.java)。

```
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
    
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```
Looper并没有public的构造函数。使用的时候需要用`Looper.prepare()`调用私有的构造函数完成初始化。从构造函数可以看出，Looper与Thread和MessageQueue都是一一对应的关系。从prepare函数中我们也可以看出，对于每个线程来说，prepare函数只能被执行一次。ThreadLocal变量只能被同一个线程读写，即两个线程执行相同代码，但是两个线程里的ThreadLocal并不在线程间共享。

```
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;
    ...
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        
        ...
        try {
            msg.target.dispatchMessage(msg);
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }
        
        ....
        msg.recycleUnchecked();
    }
}
```
loop()函数是Looper工作的核心。不过其实也比较容易理解。Looper不断的阻塞的从MessageQueue中取出message，并交给message的target来处理。msg.target实际上是一个Handler，在接下来的文章中会继续理解Handler是怎么工作的。
