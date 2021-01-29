# AQS

**独占模式**

```
/**
 * Acquires in exclusive mode, ignoring interrupts.  Implemented
 * by invoking at least once {@link #tryAcquire},
 * returning on success.  Otherwise the thread is queued, possibly
 * repeatedly blocking and unblocking, invoking {@link
 * #tryAcquire} until success.  This method can be used
 * to implement method {@link Lock#lock}.
 *
 * @param arg the acquire argument.  This value is conveyed to
 *        {@link #tryAcquire} but is otherwise uninterpreted and
 *        can represent anything you like.
 */
public final void acquire(int arg) {
	// 获取锁不成功的话，就加入等待队列
  if (!tryAcquire(arg) &&
      acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
      // 如果过程中发生过中断，需要传播出去
      selfInterrupt();
}
/**
 * Releases in exclusive mode.  Implemented by unblocking one or
 * more threads if {@link #tryRelease} returns true.
 * This method can be used to implement method {@link Lock#unlock}.
 *
 * @param arg the release argument.  This value is conveyed to
 *        {@link #tryRelease} but is otherwise uninterpreted and
 *        can represent anything you like.
 * @return the value returned from {@link #tryRelease}
 */
public final boolean release(int arg) {
	if (tryRelease(arg)) {
		// 释放成功，唤醒下一节点，下一节点竞争成功后再晋升成为head
  	Node h = head;
  	if (h != null && h.waitStatus != 0)
  		unparkSuccessor(h);
  	return true;
  }
  return false;
}
```

**共享式模式**

等待队列，只能head节点去唤醒其后继节点，共享式涉及到唤醒的传播，也是通过流程保证其在成为head节点后，才会去唤醒下一节点，保证任何并发的情况下，节点是顺序线程安全的被唤醒的。**唤醒的传播的前提是：假设一次release操作，必定能让第一个等待的节点抢到锁，如果再来1~n次释放，无论是什么时候来，必须让第二等待节点也被唤醒，而且根据释放的时机，不确定的唤醒其他的等待节点**

```java
/**
 * Sets head of queue, and checks if successor may be waiting
 * in shared mode, if so propagating if either propagate > 0 or
 * PROPAGATE status was set.
 *
 * @param node the node
 * @param propagate the return value from a tryAcquireShared
 */
private void setHeadAndPropagate(Node node, int propagate) {
  Node h = head; // Record old head for check below
  setHead(node);
  /*
   * Try to signal next queued node if:
   *   Propagation was indicated by caller,
   *     or was recorded (as h.waitStatus either before
   *     or after setHead) by a previous operation
   *     (note: this uses sign-check of waitStatus because
   *      PROPAGATE status may transition to SIGNAL.)
   * and
   *   The next node is waiting in shared mode,
   *     or we don't know, because it appears null
   *
   * The conservatism in both of these checks may cause
   * unnecessary wake-ups, but only when there are multiple
   * racing acquires/releases, so most need signals now or soon
   * anyway.
   */
  
  /*
   * 这里是传播式的唤醒健壮性判断
   * 在共享模式下，主要考虑并发释放的情况：
   *  - 1. 第一个等待节点抢占state之前并发释放结束（前头节点的waiteStatus会被设置成-3，即PROPAGATE），
   *       那么无论其抢占结果（propagate）是什么，都会在其成为头节点后进行唤醒传播（h.waitStatus = -3）
   *       如果第二个等待节点抢占失败（没有剩余量，或者非公平模式其他线程抢占了），会将新head的waitStatus改回-1
   *  - 2. 第一个等待节点抢占state之后，并发释放结束:
   *				- 2.1 其成为head之前存在并发释放，其抢占结果(propagate)已经失效，
   *          正常情况下h.waitStatus会被设置为PROPAGATE，保证唤醒正确传播，同第一种情况的结果
   *
   *          如果刚好在进行释放导致的doReleaseShared中，判断h == head之前head替换成功，那么h.waitStatus = -3，
   *          但是doReleaseShared会自旋一次，直接unpark第二节点（此时唤醒是线程安全的），head.waitStatus = 0
   *           - 正常顺序下，第一节点执行doReleaseShared时，h.waitStatus = -3，会再传播一次，将head.waitStatus = -3
   *             那么在第二节点执行setHeadAndPropagate时，会唤醒第三节点
   *						 (第三节点只能是第二节点成为head节点后，才能被唤醒)
   *           - 如果在第一节点执行doReleaseShared前，第二节点抢先替换了head（特别极端的情况），
   *             无论是第一个节点执行还是第二节点执行doReleaseShared就会直接唤醒第三个节点(此时第二节点已经成为head)，
   *						 剩下的节点执行doReleaseShared时会将waitStatus = -3（2.1），（或者直接唤醒第四节点 - 2.2）
   *						 在第三节点抢占成功后，传播唤醒第四节点，这种情况就是新的一轮并发释放了
   * 					意思就是保证一定可以传播到下一节点，并且有可能会传播到之后的多个节点
   *        - 2.2 其成为head之后并发释放结束，其抢占结果(propagate)已经失效，释放操作直接唤醒其后继节点，waitStatus = 0
   *          - 此时如果前head节点，h.waiteStatus == 0（替换head节点前，没有后续的释放），
   *            此时当前head.waiteStatus有两种值：0 或者 -3
   *            - 如果head.waiteStaus == 0，即只有一次释放，那么不会再传播了（第二节点waitStatus = -1）
   *          	- 如果head.waiteStaus == -3，多次释放，会传播（第二节点waitStatus = -3）
   *
   * h == null || h.waitStatus < 0 为了解决并发释放时，可能导致唤醒无法被传播下去的问题
   *  - 第一次释放信号量，唤醒T1，head.waitStatus = 0，T1信号量竞争成功(doAcquireShared成功，剩余信号量为0)，
   *  - 在T1成功成为头节点之前(setHeadAndPropagate() -> setHead(node) -> head = node)
   *  - 第二次释放信号量，会设置head.waitStatus = -3(PROPAGATE)，表示需要传播唤醒
   *  - 如果是在head = node之后第二次释放，那就是直接通过doReleaseShared唤醒下一节点，并设置head.waitStatus = 0
   *  - 即只要在第一次唤醒节点还未成为head之前，进行了第二次唤醒，都是通过传播的方式进行唤醒第二个等待节点
   *  - 唤醒传播是在前一个节点成为head之后，才能传播下去的（等待队列中的节点只能等待其前驱节点唤醒自己）
   *  - 从而保证一定的有序性（非公平锁运行刚来的线程和队列中第一个节点进行CAS竞争），从而保证并发下唤醒操作的正确
   * 
   * (h = head) == null || h.waitStatus < 0 主要是为了保证在类似readWriteLock的情况下，read等待节点都可以被唤醒
   * 另外就是当其成为head后，多次并发释放，会让其waitStatus发生-1 -> 0 -> -3 -> -1的变化
   *
   * 在JDK6u11、JDK6u17中，只判断了propagate > 0 && node.waitStatus != 0
   * 会导致并发释放同步量的时候，线程无法被传播唤醒的问题，仅靠propagate > 0判断是不可行的
   * 详见 BUG - JDK-6801020
   * 
   */
  if (propagate > 0 || h == null || h.waitStatus < 0 ||
      (h = head) == null || h.waitStatus < 0) {
    Node s = node.next;
    // 等待节点不是shared的情况：readWriteLock中的write等待节点
    if (s == null || s.isShared())
      doReleaseShared();
  }
}

/**
 * Release action for shared mode -- signals successor and ensures
 * propagation. (Note: For exclusive mode, release just amounts
 * to calling unparkSuccessor of head if it needs signal.)
 */
private void doReleaseShared() {
  /*
   * Ensure that a release propagates, even if there are other
   * in-progress acquires/releases.  This proceeds in the usual
   * way of trying to unparkSuccessor of head if it needs
   * signal. But if it does not, status is set to PROPAGATE to
   * ensure that upon release, propagation continues.
   * Additionally, we must loop in case a new node is added
   * while we are doing this. Also, unlike other uses of
   * unparkSuccessor, we need to know if CAS to reset status
   * fails, if so rechecking.
   */
  for (;;) {
    Node h = head;
    if (h != null && h != tail) {
      int ws = h.waitStatus;
      if (ws == Node.SIGNAL) {
        if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
          continue;            // loop to recheck cases
        unparkSuccessor(h);
      }
      // 确保release操作存在并发的情况下，唤醒能向后传播
      // 这里并不会唤醒任何节点，节点的下一次唤醒应该是在已唤醒节点成为head之后，由这个成为head的节点进行唤醒
      else if (ws == 0 &&
               !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
        continue;                // loop on failed CAS
    }
    // 场景模拟1: 第一个被唤醒的节点，在这个if判断前成为了head，那么就需要进行自旋，再次releaseShared操作，保证唤醒向后传播
    //           此时之前的头节点已经被设置成了0或者-3，如果被设置成了-3，就会在setHeadAndPropagate中向后传播唤醒
    //           会再次进入该方法，而head.waitStatus已经是0，那么会设置head.waitStatus = -3，而不会进行再次唤醒
    //           会在该head进行是否传播判断的时候，再将唤醒传播下去
    if (h == head)                   // loop if head changed
      break;
  }
}
```

**condition**

```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer.ConditionObject
public final void await() throws InterruptedException {
  if (Thread.interrupted())
    throw new InterruptedException();
  Node node = addConditionWaiter();
  int savedState = fullyRelease(node);
  int interruptMode = 0;
  while (!isOnSyncQueue(node)) {
    LockSupport.park(this);
    // 这里唤醒可能有两种情况，1. 其他线程调用了signal；2. 当前线程interrupt了，可能会和signal产生竞争
    // 线程interrupt之后，需要加入到sync队列中去，根据不同情况确定不同的interrupt传播
    if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
      break;
  }
  if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
    interruptMode = REINTERRUPT;
  if (node.nextWaiter != null) // clean up if cancelled
    unlinkCancelledWaiters();
  if (interruptMode != 0)
    reportInterruptAfterWait(interruptMode);
}

private int checkInterruptWhileWaiting(Node node) {
  return Thread.interrupted() ?
    (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
  0;
}

private void doSignal(Node first) {
  do {
    if ( (firstWaiter = first.nextWaiter) == null)
      lastWaiter = null;
    first.nextWaiter = null;
    // 转移节点到sync队列中去，如果失败了，说明节点因为其他原因cancle了，继续转移下一节点
  } while (!transferForSignal(first) && 
           (first = firstWaiter) != null);
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer

final boolean transferForSignal(Node node) {
  /*
   * If cannot change waitStatus, the node has been cancelled.
   */
  // 节点可能已经cancel了，即已经转入sync队列了，不用进行处理了
  if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
    return false;

  /*
   * Splice onto queue and try to set waitStatus of predecessor to
   * indicate that thread is (probably) waiting. If cancelled or
   * attempt to set waitStatus fails, wake up to resync (in which
   * case the waitStatus can be transiently and harmlessly wrong).
   */
  Node p = enq(node);
  int ws = p.waitStatus;
  if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
    LockSupport.unpark(node.thread);
  return true;
}
final boolean transferAfterCancelledWait(Node node) {
  if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
    // cancel成功，说明当前线程唤醒是因为中断，中断应该抛出
    enq(node);
    return true;
  }
  // 否则，是在signal()之后才产生的中断，需要等待节点成功加入sync队列，中断不抛出
  /*
   * If we lost out to a signal(), then we can't proceed
   * until it finishes its enq().  Cancelling during an
   * incomplete transfer is both rare and transient, so just
   * spin.
   */
  while (!isOnSyncQueue(node))
    Thread.yield();
  return false;
}
```

