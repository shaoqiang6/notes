# 两个重点方法的详细注释

## 1、await

两个队列：同步队列（双向链表）、等待队列（单链表）

```java
/**
 * Implements interruptible condition wait.
 * <ol>
 * <li> If current thread is interrupted, throw InterruptedException.
 * <li> Save lock state returned by {@link #getState}.
 * <li> Invoke {@link #release} with saved state as argument,
 *      throwing IllegalMonitorStateException if it fails.
 * <li> Block until signalled or interrupted.
 * <li> Reacquire by invoking specialized version of
 *      {@link #acquire} with saved state as argument.
 * <li> If interrupted while blocked in step 4, throw InterruptedException.
 * </ol>
 */
public final void await() throws InterruptedException {
    // 如果线程已经中断，抛出异常
    if (Thread.interrupted())
        throw new InterruptedException();
    // 当前线程加入到等待队列
    Node node = addConditionWaiter();
    // 释放锁（await方法执行时机是在获取到锁之后，需要等待的场景，）
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // while中断的条件：
    // 1.当前节点在同步队列
    // 2.当前线程被中断
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // CAS 获取lock
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

## 2、singnal、singnalAll

```java
/**
 * Moves the longest-waiting thread, if one exists, from the
 * wait queue for this condition to the wait queue for the
 * owning lock.
 *
 * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
 *         returns {@code false}
 */
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```

```java
/**
 * Removes and transfers nodes until hit non-cancelled one or
 * null. Split out from signal in part to encourage compilers
 * to inline the case of no waiters.
 * @param first (non-null) the first node on condition queue
 */
private void doSignal(Node first) {
    do {
        // 等待队列中的第二个节点设置为等待队列的头结点
        if ((firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    // transferForSignal 将first节点放到同步队列
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```

```java
 /**
     * Transfers a node from a condition queue onto sync queue.
     * Returns true if successful.
     * @param node the node
     * @return true if successfully transferred (else the node was
     * cancelled before signal)
     */
    final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
         // 放到同步队列中
        Node p = enq(node);
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```

`LockSupport`  todo

```java
/**
 * Makes available the permit for the given thread, if it
 * was not already available.  If the thread was blocked on
 * {@code park} then it will unblock.  Otherwise, its next call
 * to {@code park} is guaranteed not to block. This operation
 * is not guaranteed to have any effect at all if the given
 * thread has not been started.
 *
 * @param thread the thread to unpark, or {@code null}, in which case
 *        this operation has no effect
 */
public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
}
```

unpark 将线程由阻塞变为非阻塞

不响应中断支持

# 总结

整体来说就是使用 Lock.newCondition+ await() + singnal()/singnalAll() 代替了

synchronised + wairt() + notify()/notifyAll()


