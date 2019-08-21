[TOC]

#### 1 什么是 HashedWheelTimer 

- 它是一种高效的定时任务处理类
- 它通过将任务按延时时间放入一个轮子中不同的桶中，按照自定义的时钟，不断循环遍历每一个桶，执行每一个桶中到期的任务

- 它只能添加一个定时执行一次的任务，而不是像 quartz 那样指定一个任务定时定期循环执行
- 如果需要一个任务定时定期循环执行，只能在任务本身，重复调用 newTimeout() 方法，将任务重复加入时间轮中

#### 2 如何使用

- 新建一个 HashedWheelTimer 实例

```java
HashedWheelTimer timer = new (ThreadFactory threadFactory,long tickDuration, TimeUnit unit, int ticksPerWheel, boolean leakDetection, long maxPendingTimeouts);
// threadFactory：用来创建 Thread 的工厂，指定创建出来的线程的一些属性，如线程名，优先级等
// tickDuration：每次轮转的间隔，越小则定时器精度越高，默认为 100ms
// unit：tickDuration 的时间单位
// ticksPerWheel：轮子的大小，默认为 512，会被规整为 2 的整数次幂
// leakDetection：默认为 true，用于泄露检测，如果为 false，只有 worker 线程不是守护线程时，才会进行泄露检测
// maxPendingTimeouts：最大的任务数量
```

- 开启一个任务，用于指定时间运行

```java
//这样就将 task 放入了时间轮中，并在规定的时刻，将会被 worker 线程执行
timer.newTimeout(TimerTask task, long delay, TimeUnit unit);
// task：用于执行任务
// delay：任务延时多长时间运行
// unit：delay 的时间单位
```

#### 3 源码解析

##### newTimeout(...)

- 通过这个方法将 task 放入 timeouts 中，这时并没有放入时间轮中！

```java
public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {
	//如果 task == null,unit == null 执行异常流程
    //原则性递增任务数量
    long pendingTimeoutsCount = pendingTimeouts.incrementAndGet();
	//默认 maxPendingTimeouts == -1 的，如果指定了这个，当时间任务数量大于这个是报错
    if (maxPendingTimeouts > 0 && pendingTimeoutsCount > maxPendingTimeouts) {...}
	//启动
    start();

    long deadline = System.nanoTime() + unit.toNanos(delay) - startTime;
    // Guard against overflow.
    if (delay > 0 && deadline < 0) {
        deadline = Long.MAX_VALUE;
    }
    //将 timeout 添加到 HashedWheelTimeout 队列中，该队列将在下一个轮转时处理。
    //在处理过程中，所有排队的 HashedWheelTimeouts 都将添加到正确的hashedWheelBucket中。
    HashedWheelTimeout timeout = new HashedWheelTimeout(this, task, deadline);
    timeouts.add(timeout);
    return timeout;
}
```

##### start()

- 每次调用 newTimeout() 方法都会调用 start() 方法
- 只有第一次调用 newTimeout() 方法，才会开启 worker 线程
- **为什么这样设计呢？**

```java
public void start() {
    //确保 worker 线程只被启动一次
    switch (WORKER_STATE_UPDATER.get(this)) {
        case WORKER_STATE_INIT:
            if (WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_INIT, WORKER_STATE_STARTED)) {
                //这里开启 worker 线程执行 worker() 类中的 run() 方法
                workerThread.start();
            }
            break;
        case WORKER_STATE_STARTED:
            break;
        case WORKER_STATE_SHUTDOWN:
            throw new IllegalStateException("cannot be started once stopped");
        default:
            throw new Error("Invalid WorkerState");
    }

    // startTime 是在启动 worker 线程后初始化的，并且被 volatile 修饰
    // 说明 start() 方法要等到 worker 线程启动后，才退出
    while (startTime == 0) {
        try {
            startTimeInitialized.await();
        } catch (InterruptedException ignore) {
            // Ignore - it will be ready very soon.
        }
    }
}
```

##### worker 线程执行的逻辑

1. 休眠到下次轮转的时间，返回从 初始化 startTime 到此时的时间
2. 如果时间大于0，获取当前轮转对应的桶（轮子索引）
3. 移除所有已取消的任务
4. 将 timeout 中的任务，移到轮子中！
5. 执行目标桶中到期的任务

```java
@Override
public void run() {
    // Initialize the startTime.
    startTime = System.nanoTime();
    if (startTime == 0) {
        // 确保 worker 线程执行后，startTime 不为 0
        startTime = 1;
    }

    // 这里通知 start() 方法继续执行
    startTimeInitialized.countDown();

    do {
        //正常情况下，deadline 为 从 worker 线程启动到目前的 时间间隔
        final long deadline = waitForNextTick();
        if (deadline > 0) {
            //取模获取当前轮转的索引
            int idx = (int) (tick & mask);
            //移除已经取消的任务，通过 HashedWheelTimeout 的 cancel 方法可以将当前 timeout 加入 取消列表中
            processCancelledTasks();
            //取到当前索引下对应的 任务列表
            HashedWheelBucket bucket =
                wheel[idx];
            //轮中的任务是在这里放入的，我们定义的任务先主动 push 到 timeout 中
            //在这里将实时放入 timeout 的任务，移动到 轮中！！
            //这里就应用到 timeout 的数据结构了：安全的多写1读数据结构！
            transferTimeoutsToBuckets();
            //执行当前 bucket 对应的到期的任务
            bucket.expireTimeouts(deadline);
            tick++;
        }
        //只有 stop() 方法可以将 WORKER_STATE_STARTED 变为 WORKER_STATE_SHUTDOWN
    } while (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_STARTED);

    // Fill the unprocessedTimeouts so we can return them from stop() method.
    //遍历轮子，将所有轮子中没有执行的任务放入 unprocessedTimeouts 中
    for (HashedWheelBucket bucket: wheel) {
        bucket.clearTimeouts(unprocessedTimeouts);
    }
    //将所有在 timeout 列表中的任务放入 unprocessedTimeouts 中
    for (;;) {
        HashedWheelTimeout timeout = timeouts.poll();
        if (timeout == null) {
            break;
        }
        if (!timeout.isCancelled()) {
            unprocessedTimeouts.add(timeout);
        }
    }
    //清除被取消的任务
    processCancelledTasks();
}
```

waitForNextTick()











###### waitForNextTick()

- 如果 currentTime > = deadline，说明目前已经到了下次轮转的时间了，直接返回
- 如果没到，则休眠到下次轮转的实际

```java
private long waitForNextTick() {
    //下一次轮转的时间
    long deadline = tickDuration * (tick + 1);

    for (;;) {
        //从 worker 线程启动到目前的 时间间隔
        final long currentTime = System.nanoTime() - startTime;
        //下一次轮转的时间 - 目前经历的时间 == 还需要休眠的实际时间
        long sleepTimeMs = (deadline - currentTime + 999999) / 1000000;
		//如果休眠时间为负数，则返回一个负数或当前时间，使得下次轮转立刻执行
        if (sleepTimeMs <= 0) {
            //这是什么情况？如果是这种情况，会返回并再次进入此方法
            //再次进入时，重新计算 currentTime，可能就恢复正常了
            if (currentTime == Long.MIN_VALUE) {
                return -Long.MAX_VALUE;
            } else {
                return currentTime;
            }
        }
        // windows 的时间的最大精度为 10ms，这样会使得整体进度更高
        if (PlatformDependent.isWindows()) {
            sleepTimeMs = sleepTimeMs / 10 * 10;
        }
        try {
            Thread.sleep(sleepTimeMs);
        } catch (InterruptedException ignored) {
            if (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_SHUTDOWN) {
                return Long.MIN_VALUE;
            }
        }
    }
}
```

###### processCancelledTasks()

- 移除已经取消的任务

```java
private void processCancelledTasks() {
    for (;;) {
        HashedWheelTimeout timeout = cancelledTimeouts.poll();
        if (timeout == null) {
            // all processed
            break;
        }
        try {
            timeout.remove();
        } catch (Throwable t) {
            if (logger.isWarnEnabled()) {
                logger.warn("An exception was thrown while process a cancellation task", t);
            }
        }
    }
}
```

###### transferTimeoutsToBuckets()

- 该方法是从timeouts（就是前面newTimeout是放进去的那个queue）的queue中取出任务，放到格子里（HashedWheelBucket是一个链表）
- 为了防止这个操作销毁太多时间，导致更多的任务时间不准，因此一次最多操作10w个

```java
private void transferTimeoutsToBuckets() {
    // transfer only max. 100000 timeouts per tick to prevent a thread to stale the workerThread when it just
    // adds new timeouts in a loop.
    for (int i = 0; i < 100000; i++) {
        HashedWheelTimeout timeout = timeouts.poll();
        if (timeout == null) {
            // all processed
            break;
        }
        if (timeout.state() == HashedWheelTimeout.ST_CANCELLED) {
            // Was cancelled in the meantime.
            continue;
        }
		//timeout.deadline：worker 线程启动到放入 timeout 的时间 + delay
        //calculated 表示需要多长轮转才到此任务
        long calculated = timeout.deadline / tickDuration;
        //通过 calculated 计算出此任务还需多少轮才可以被执行
        timeout.remainingRounds = (calculated - tick) / wheel.length;
		//如果 目前轮次已经大于 calculated ，说明该任务应该在过去执行了
        //那就通过当前的 ticks 来计算索引，这样使得该任务会立马执行！
        //因为这样 计算出来的 index 和 run() 方法中计算出来的一模一样
        final long ticks = Math.max(calculated, tick); // Ensure we don't schedule for past.   //通过取模得到任务对应的 轮的索引
        int stopIndex = (int) (ticks & mask);
		//得到该索引对应的轮，放入轮对应的索引中
        HashedWheelBucket bucket = wheel[stopIndex];
        bucket.addTimeout(timeout);
    }
}
```

###### bucket.expireTimeouts(deadline)

- 执行轮子中已经到期的任务
- 如果remainingRounds（剩下的圈数）小于等于0，那么就把他移除并执行expire方法（即TimerTask的run方法）
- 如果任务被取消了，则直接移除
- 否则remainingRounds减1，等待下一圈

```java
public void expireTimeouts(long deadline) {
    HashedWheelTimeout timeout = head;
    //遍历当前桶(轮子对应的索引)，执行桶中已经到期的任务
    while (timeout != null) {
        HashedWheelTimeout next = timeout.next;
        if (timeout.remainingRounds <= 0) {
            next = remove(timeout);
            //timeout.deadline：worker 线程启动到放入 timeout 的时间 + delay
            //deadline：worker 线程启动到执行完 waitForNextTick() 方法的时间 
            //正常情况下，只有 deadline > timeout.deadline, waitForNextTick() 才会返回！
            if (timeout.deadline <= deadline) {
                //执行 task，如果执行时间很长的话，会导致后面的任务可能被一起放入下一个轮转中执行
                //所以，hashedWheelTimer 是不支持每个固定时间就执行的(不管任务有没有完成)
                //我们可以通过在 task 中异步执行，这样就可以支持了
                timeout.expire();
            } else {
                // The timeout was placed into a wrong slot. This should never happen.
                throw new IllegalStateException(String.format(
                        "timeout.deadline (%d) > deadline (%d)", timeout.deadline, deadline));
            }
        } else if (timeout.isCancelled()) {
            next = remove(timeout);
        } else {
            timeout.remainingRounds --;
        }
        timeout = next;
    }
}
```

###### stop() 

- 设置 worker 线程状态为 WORKER_STATE_SHUTDOWN
- 并在 worke 线程结束后，返回没有被执行的任务

```java
@Override
public Set<Timeout> stop() {
    if (Thread.currentThread() == workerThread) {
        throw new IllegalStateException(
                HashedWheelTimer.class.getSimpleName() +
                        ".stop() cannot be called from " +
                        TimerTask.class.getSimpleName());
    }

    if (!WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_STARTED, WORKER_STATE_SHUTDOWN)) {
        // workerState can be 0 or 2 at this moment - let it always be 2.
        if (WORKER_STATE_UPDATER.getAndSet(this, WORKER_STATE_SHUTDOWN) != WORKER_STATE_SHUTDOWN) {
            INSTANCE_COUNTER.decrementAndGet();
            if (leak != null) {
                boolean closed = leak.close(this);
                assert closed;
            }
        }

        return Collections.emptySet();
    }

    try {
        boolean interrupted = false;
        while (workerThread.isAlive()) {
            workerThread.interrupt();
            try {
                workerThread.join(100);
            } catch (InterruptedException ignored) {
                interrupted = true;
            }
        }

        if (interrupted) {
            Thread.currentThread().interrupt();
        }
    } finally {
        INSTANCE_COUNTER.decrementAndGet();
        if (leak != null) {
            boolean closed = leak.close(this);
            assert closed;
        }
    }
    return worker.unprocessedTimeouts();
}
```