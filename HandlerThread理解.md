HandlerThread顾名思义是一种特殊的Thread子类，在它创建的过程中会为我们建立好与这个Thread相对应的Looper，方便我们以此为基础创建Handler。

HandlerThread的源码在[这里](https://github.com/android/platform_frameworks_base/blob/master/core/java/android/os/HandlerThread.java)。

##源码

```
public void run() {
    mTid = Process.myTid();
    Looper.prepare();
    // 必须保证创建Looper被synchronized保护因为getLooper()可能在其他线程被调用
    synchronized (this) {
        mLooper = Looper.myLooper();
        notifyAll();
    }
    Process.setThreadPriority(mPriority);
    onLooperPrepared();
    Looper.loop();
    mTid = -1;
}

public Looper getLooper() {
    if (!isAlive()) {
        return null;
    }
    
    // If the thread has been started, wait until the looper has been created.
    synchronized (this) {
        while (isAlive() && mLooper == null) {
            try {
                wait();
            } catch (InterruptedException e) {
            }
        }
    }
    return mLooper;
}
```

HandlerThread的源码比较简单，从run()函数中可以看出当线程启动时我们创建了一个与这个线程对应的Looper，HandlerThread同时也提供了getLooper()函数方便我们取得这个Looper，创建相应的Handler。从源码中可以看出，所用基于同一个HandlerThread的Handler里的Message是被串行处理的，因为这些Message都对应着相同的Looper，即内部相同的MessageQueue。

## 用法
```
HandlerThread workThread = new HandlerThread("WorkHandler");
// 在创建Handler必须首先调用Thread.start()函数。start()函数会创建Thread,调用Thread.run()函数，进而创建出对应的Looper
workThread.start();
Handler workHandler = new WorkHandler(workThread.getLooper());
workHandler.sendEmptyMessage(0);
```
