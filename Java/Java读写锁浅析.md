[TOC]

# 1 获取读锁

- 首先尝试获取读锁
- 获取读锁失败后，尝试插入AQS队列（加入队列前，重新尝试获取读锁）

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)//1.1
        doAcquireShared(arg);//1.2 当返回小于0是，尝试加入AQS队列
}
```

## 1.1 尝试获取读锁

- 当有写锁时，返回-1，获取失败
- 判断是否应该阻塞：默认为非公平判断
- 有几个counter类，是计重入次数的，不阻塞，相应的次数+1
- 之前判断不了，则重新深入获取读锁

```java
protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    int c = getState();
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;//当有写锁时，走这一步，因为写锁setState了
    int r = sharedCount(c);
    //readerShouldBlock()方法有公平和非公平的区别
    //1.1.1
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;//不阻塞，才可能到这
    }
    //1.1.2 之前判断不了，则重新深入获取读锁
    return fullTryAcquireShared(current);
}
```

### 1.1.1 判断是否应该阻塞（重点）

- 公平锁：只要队列有值，且头结点不是本身，就加入队列，显然是顺序公平的
- 非公平锁：只有头结点不是shared节点才加入队列，等待。注意：即使写操作没获取到锁，只要写节点加入了AQS，则之后的读操作，都要加入AQS了，而不是只有写操作获取到锁时，读操作才阻塞
- 关键：当读操作释放锁时，新获取读锁的线程和AQS队列里的读线程，互补影响，同步运行

```java
//公平,返回true表示应该阻塞。只要队列中有值，且第一个不是当前节点时，阻塞
//只要前面有节点，就阻塞
public final boolean hasQueuedPredecessors() {
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
//非公平,返回true表示应该阻塞。头不为null，第一个节点不为null且第一个节点不是shared节点才阻塞
//只有第一个节点时Exclusive时才阻塞
final boolean apparentlyFirstQueuedIsExclusive() {
    Node h, s;
    return (h = head) != null &&
        (s = h.next)  != null &&
        !s.isShared()         &&
        s.thread != null;
}
```

### 1.1.2 深入获取读锁

- 如果有写锁，则直接失败
- 判断是否应该阻塞
  - 如果是，当重入计数为0时，返回-1，获取锁失败。这一步暂时没弄明白
  - 如果不是，CAS状态，设置状态成功，获取到锁，否则重新下一次的for

```java
//这个不太明白
final int fullTryAcquireShared(Thread current) {
    HoldCounter rh = null;
    for (;;) {
        int c = getState();
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1;//如果有写锁，则直接失败
        } else if (readerShouldBlock()) {
            // Make sure we're not acquiring read lock reentrantly
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
            } else {
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {
                        rh = readHolds.get();
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                //重入为0的时候，1.2尝试重新获取锁
                if (rh.count == 0)
                    return -1;//只有cachedHoldCounter==0时，才可能阻塞
            }
        }
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;//设置状态成功表示获取到锁
        }
    }
}
```

## 1.2 获取锁失败后，尝试加入AQS队列

- 当`tryAcquireShared(arg)`返回负数时，尝试加入AQS队列
- 首先将当前线程当作SHARED节点，加入AQS队列，用尾插法
- 获取队列最后一个节点的前一个节点
  - 如果前一个节点是头结点，则重新尝试获取锁，获取成功直接返回
  - 如果获取失败，

```java
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);//尾插法加入队列，并返回队列
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);//如果是头，则重新获取读锁
                if (r >= 0) {
                    setHeadAndPropagate(node, r);//获取成功，重新设置头结点，并唤醒头结点
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();//设置线程Interrupt属性
                    failed = false;
                    return;
                }
            }
            //1.2.1 判断当前节点的前一个节点是否是SIGNAL（-1）节点，返回true,如果是直接进入1.2.2挂起;如果不是，while循环将前面所有的是CANCELLED（1）的节点移除，如果节点不是1和-1，则将节点属性CAS成-1,返回false，进行下一个循环；
            //1.2.2 挂起
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        //return和异常之前，要经过这个判断，如果failed为true，则取消当前节点的获取
        //显然只有异常时，才会进入1.2.3
        if (failed)
            //1.2.3
            cancelAcquire(node);
    }
}
```

### 1.2.1 判断当前节点的前一个节点是否是SIGNAL节点

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        /* 只有cancelled节点值>0！
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;//移除前面节点，直到前面节点不是cancelled节点为止
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    //如果前面节点不是signal节点，就返回false
    return false;
}
```

### 1.2.2 如果当前节点的前一节点是Signal节点，则挂起，否则继续下一循环

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();//设置当前线程为中断属性
}
```

### 1.2.3 如果线程异常退出，取消获取锁的节点

```java
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    if (node == null)
        return;

    node.thread = null;

    // Skip cancelled predecessors
	//跳过为cancelled属性的前节点
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    // predNext is the apparent node to unsplice. CASes below will
    // fail if not, in which case, we lost race vs another cancel
    // or signal, so no further action is necessary.
    Node predNext = pred.next;

    // Can use unconditional write instead of CAS here.
    // After this atomic step, other Nodes can skip past us.
    // Before, we are free of interference from other threads.
    node.waitStatus = Node.CANCELLED;

    // If we are the tail, remove ourselves.
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        // If successor needs signal, try to set pred's next-link
        // so it will get one. Otherwise wake it up to propagate.
        //不是头结点，将前节点设置为Signal属性
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            unparkSuccessor(node);//如果为头节点，且为Signal则唤醒该节点
        }

        node.next = node; // help GC
    }
}
```

# 2 获取写锁

```java
public final void acquire(int arg) {
    //2.1 尝试获取锁
    //2.2 加入AQS队列
    //2.3 先重新尝试获取锁，不成功在挂起
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();//设置中断属性
}
```

## 2.1 尝试获取写锁

```java
protected final boolean tryAcquire(int acquires) {

    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;//如果有写锁了，直接获取失败返回
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        setState(c + acquires);//获取重入锁，成功返回
        return true;
    }
    //2.1.1 判断写锁是否应该阻塞
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;//如果应该或设置状态出错都获取失败返回
    setExclusiveOwnerThread(current);
    return true;//获取成功
}
```

### 2.1.1 判断写锁是否应该阻塞

```java
//非公平锁直接返回失败
final boolean writerShouldBlock() {
    return false; // writers can always barge
}
//公平锁，如果有前一个节点，则应该阻塞，直接返回
//如果没有，尝试设置state，成功则获取到锁，失败则获取失败
final boolean writerShouldBlock() {
	return hasQueuedPredecessors();
}
```

## 2.2 将当前节点加入等待队列

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    //如果tail不为null，直接CAS将当前节点设置到尾部，设置成功直接返回
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //2.2.1不成功，再循环CAS设置
    enq(node);
    return node;
}
```

### 2.2.1简单的CAS不成功，再循环CAS将当前节点设置到队列尾部

```java
//当获取写锁失败后，会将写线程加入队列
private Node enq(final Node node) {//这里的node类型是exclude，读锁是SHARED
    //尾插法
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;//到达这不是，队列的头结点就是写结点了，这时所有的读请求就会失败了！
                return t;
            }
        }
    }
}
```

## 2.3 再一次尝试获取锁，不成功挂机当前节点

```java
//和读操作不一样，因为这里已经将当前节点加入队列了
//基本上和其它锁一样
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            //首先判断当前节点是否为头结点，如果是再一次尝试获取锁，成功则直接返回
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //不成功判断前一节点是否是SIGNAL节点
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

# 3 释放读锁

```java
public final boolean releaseShared(int arg) {
    //3.1尝试释放
    if (tryReleaseShared(arg)) {
        //3.2释放
        doReleaseShared();
        return true;
    }
    return false;
}
```

## 3.1 尝试释放读锁

```java
protected final boolean tryReleaseShared(int unused) {
    //重入锁-1
    Thread current = Thread.currentThread();
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else
            firstReaderHoldCount--;
    } else {
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != getThreadId(current))
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        --rh.count;
    }
    //循环CAS设置状态
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            // Releasing the read lock has no effect on readers,
            // but it may allow waiting writers to proceed if
            // both read and write locks are now free.
            return nextc == 0;//如果设置nextc不为0，则锁释放失败，因为重入嘛！
    }
}
```

## 3.2 如果重入为0了，则正式释放读锁，唤醒头结点！

```java
//重入为0是，唤醒头结点
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);//唤醒头结点
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        //如果头结点没变，返回
        //可能因为别的线程导致头结点变了，则继续唤醒头结点
        if (h == head)                   // loop if head changed
            break;
    }
}
```

# 4 释放写锁，

```java
public final boolean release(int arg) {
    //4.1 尝试释放写锁，重入次数减1，如果为0，则返回true
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);//释放成功唤醒头结点
        return true;
    }
    return false;
}
```

## 4.1 尝试释放写锁，重入次数减1，如果为0，则返回true

```java
protected final boolean tryRelease(int releases) {
    //如果持有写锁的不是当前线程，锁状态肯定有问题呀
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    boolean free = exclusiveCount(nextc) == 0;//如果重入次数为0，则可以释放锁
    if (free)
        setExclusiveOwnerThread(null);
    setState(nextc);//设置状态
    return free;
}
```

## 4.2 如果4.1返回true，则正式唤醒头结点！

