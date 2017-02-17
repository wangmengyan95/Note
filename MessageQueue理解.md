MessageQueue顾名思义就是一个消息队列，我们可以通过Handler不断将Message装入MessageQueue中等待处理，Looper则会不断从MessageQueue中取出Message，交给Message的target Handler去处理。

MessageQueue的源码在[这里](https://github.com/android/platform_frameworks_base/blob/master/core/java/android/os/MessageQueue.java)。本人才疏学浅，这里只侧重于分析MessageQueue java层的代码，native层的留待以后再说。

##添加Barrier

```
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}

private int postSyncBarrier(long when) {
    // Enqueue a new sync barrier token.
    // We don't need to wake the queue because the purpose of a barrier is to stall it.
    synchronized (this) {
        final int token = mNextBarrierToken++;
        
        // 创建一个新的Message作为Barrier，对于Barrier的Message，它的target为null，
        // 这经常作为日后判断一个Message是否是Barrier的条件
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        // 通过以下的while循环，我们找到了第一个Message，使得Barrier.when < p.when
        // 退出循环时的状态是 prev.when <= barrier.when < p.when
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }

        // 如果prev存在则把Barrier插到prev和p之间，否则Barrier就是新的链表头部
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```

Barrier的作用相当于一个阻塞器，它会阻止Barrier之后的所有同步Message的执行，直到Barrier被移除。我们会在next()函数中看到Barrier是如何发挥作用的。

##移除Barrier

```
public void removeSyncBarrier(int token) {
    // Remove a sync barrier token from the queue.
    // If the queue is no longer stalled by a barrier then wake it.
    synchronized (this) {
        Message prev = null;
        Message p = mMessages;

        // 这个while循环即从表头开始遍历链表寻找指定的Barrier，
        // 找Barrier的条件就是 p.target == null && p.arg1 == token
        while (p != null && (p.target != null || p.arg1 != token)) {
            prev = p;
            p = p.next;
        }
        if (p == null) {
            throw new IllegalStateException("The specified message queue synchronization "
                    + " barrier token has not been posted or has already been removed.");
        }

        // 判断是否需要唤醒等待的线程，nativePollOnce()会导致线程在native层等待，
        // 而nativeWake()可以唤醒它，nativePollOnce()会在next()函数中看到应用。
        final boolean needWake;
        if (prev != null) {
            // 如果Barrier不是链表头，无需唤醒，因为不可能有线程因为这个Barrier进入等待状态
            prev.next = p.next;
            needWake = false;
        } else {
            // 如果Barrier的后面节点还是Barrier，则无需唤醒
            mMessages = p.next;
            needWake = mMessages == null || mMessages.target != null;
        }
        p.recycleUnchecked();

        // If the loop is quitting then it is already awake.
        // We can assume mPtr != 0 when mQuitting is false.
        if (needWake && !mQuitting) {
            nativeWake(mPtr);
        }
    }
}
```
