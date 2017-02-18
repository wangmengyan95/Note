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

##添加Message

```
boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }

    synchronized (this) {
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }

        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            // 如果新加的消息是新的链表头，则更新链表头
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;

            // 找到msg应该插入的位置，最后退出循环时的状态是
            // prev.when <= msg.when < p.when
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

##删除Message

```
void removeMessages(Handler h, int what, Object object) {
    if (h == null) {
        return;
    }

    synchronized (this) {
        Message p = mMessages;

        // Remove all messages at front.
        // 从头开始遍历链表，如果Message满足条件就删除并且更新mMessages链表头，
        // 直到mMessages链表头不满足条件为止
        while (p != null && p.target == h && p.what == what
               && (object == null || p.obj == object)) {
            Message n = p.next;
            mMessages = n;
            p.recycleUnchecked();
            p = n;
        }

        // Remove all messages after front.
        // 继续遍历链表，删除满足条件结点，直至链表末尾
        while (p != null) {
            Message n = p.next;
            if (n != null) {
                if (n.target == h && n.what == what
                    && (object == null || n.obj == object)) {
                    Message nn = n.next;
                    n.recycleUnchecked();
                    p.next = nn;
                    continue;
                }
            }
            p = n;
        }
    }
}
```
删除Message还有几个其它的类似函数，删除的具体流程都是一样的，唯一区别就是判断Message是否满足的条件不同。

##关闭MessageQueue

```
void quit(boolean safe) {
    if (!mQuitAllowed) {
        throw new IllegalStateException("Main thread not allowed to quit.");
    }

    synchronized (this) {
        // 如果选择安全quit，则有可能MessageQueue中还有Message，用mQuitting标记，
        // 避免无意义多次执行检查
        if (mQuitting) {
            return;
        }
        mQuitting = true;

        if (safe) {
            removeAllFutureMessagesLocked();
        } else {
            removeAllMessagesLocked();
        }

        // We can assume mPtr != 0 because mQuitting was previously false.
        nativeWake(mPtr);
    }
}

private void removeAllFutureMessagesLocked() {
    final long now = SystemClock.uptimeMillis();
    Message p = mMessages;
    if (p != null) {
        if (p.when > now) {
            // p.when > now 说明链表内的所有Message均未到执行的时间，
            // 这种情况删除所有Message
            removeAllMessagesLocked();
        } else {
            Message n;
            // 遍历列表，找到第一个不可执行的Message，
            // 退出循环的状态是 p.when <= now < n.when
            for (;;) {
                n = p.next;
                if (n == null) {
                    return;
                }
                if (n.when > now) {
                    break;
                }
                p = n;
            }

            // 删除从Message n开始后的所有元素 
            p.next = null;
            do {
                p = n;
                n = p.next;
                p.recycleUnchecked();
            } while (n != null);
        }
    }
}

private void removeAllMessagesLocked() {
    // 遍历链表，移除所有Message
    Message p = mMessages;
    while (p != null) {
        Message n = p.next;
        p.recycleUnchecked();
        p = n;
    }
    mMessages = null;
}
```
关闭MessageQueue分两种情况，如果选择安全关闭，则MessageQueue中会保留所有已经到达执行时间的Message，等待它们被从MessageQueue中取出。如果选择不安全关闭，则会清楚MessageQueue中的所有Message。

## 取出Message

```
Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        // 这个native call会导致线程阻塞nextPollTimeoutMillis ms
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;

            // 如果链表头是一个Barrier，则遍历链表，找到第一个异步Message
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // 如果第一个Message仍然不到执行的时间，则更新nextPollTimeoutMillis
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 找到可执行的Message，返回此Message
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // 没有可执行的Message，则设置nextPollTimeoutMillis为-1，此时nativePollOnce会一直阻塞，直到被唤醒
                // No more messages.
                nextPollTimeoutMillis = -1;
            }

            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                return null;
            }

            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // Run the idle handlers.
        // We only ever reach this code block during the first iteration.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // Reset the idle handler count to 0 so we do not run them again.
        pendingIdleHandlerCount = 0;

        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
}
```

MessageQueue的基本返回逻辑就是
- 如果有Barrier，则返回第一个可执行的异步Message
- 如果没有Barrier，则返回第一个可执行的Message
- 如果没有可执行的Message，并且是第一次Idle，则运行IdleHandler，并且根据自定义的queueIdle()方法决定是否保留IdleHandler
- 如果没有可执行的Message，并且不是第一次Idle，则阻塞等待，直到timeout或者被唤醒
- 如果是在退出状态，则返回null，这也会导致Looper.loop()方法直接结束
