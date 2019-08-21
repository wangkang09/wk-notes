[TOC]

### 1 Redis 分布式锁的安全性和可靠性保证

- 安全属性：互斥，任何时候，只有一个客户端能持有同一个锁
- 效率属性A：不会死锁，最终一定会得到锁，且一定会释放锁（即使持有锁的客户端宕机或发生网络分区）
- 效率属性B：容错，只要大多数 Redis 节点正常工作，客户端应该都能获取和释放锁

### 2 Redis 分布式锁最简单方案

- 在 Redis 实例中创建一个key，并且附带**超时时间**（故障自动释放，不会死锁）。当key存在时，返回0，说明已经有线程创建了这个key 锁，要等待。返回1时，说明当前线程获得这个 key 的锁，可以执行业务逻辑
- **问题：** 当 Redis 的实例宕机时，别的线程可能切换到别的实例，这样别的线程也会得到锁，也会去执行业务逻辑
- **解决：** 因为 SheinCache 使用分布式锁来解决缓存击穿问题，其实完全可以忍受有2个线程同时获取锁，不会造成多大影响

### 3 简单设置 key 的超时时间存在的问题

- 如果超时时间过长，客户端宕机时，别的客户端线程就会被阻塞很久
- 如果超时时间过短，先获得锁的线程可能还没执行完业务，锁已经失效了，这是别的线程也获得了锁。这样就会存在**第一个线程可能会主动释放第二个线程的分布式锁**的情况
- **解决：**设置的超时时间加上一个**随机时间**，只有总时间==之前的时间时，才主动释放锁（不相等时，说明已经被别的线程取到锁了）
- **解决方案存在的问题：** 判断和释放锁不是同步的

```java
public void releaseLock(key,random_time) {
    //不同步！
    if(getKey(key) == random_time) {
        deleteKey(key);
    }
}
```

### 4 使用 Lua 脚本解决 3 中的问题

- 由于 Lua 脚本的原子性，在 Redis 执行脚本的过程中，其他客户端命令都需要等待 Lua 脚本执行完才能执行，这样就解决了 3 中的不同步问题

```Lua
//Redis在2.6后内部内嵌Lua脚本解释器，所以我们可以通过简单的Lua脚本来保证上述操作的原子性。代码中的Lua脚本的的意思是：我们把LockKey赋值给KEYS[1]，把RequestId赋值给ARGV[1]，如果key中的值等于RequestId，返回true否则返回false。这样就保证了释放锁操作时原子的，并且当前客户端只会释放当前客户端的锁。
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else 
    return 0
end
```

### zk 锁性能问题

http://stor.51cto.com/art/201906/597813.htm



```java
boolean flag = pool.connection().sync().lock(key,200,200);
loaderResult result;
if(flag) {
    if (result = getFromL2(key) != null) {
        return result;
    }
    result = invokeLoader(key, finalLoader);
    finalLoader.callback(result);
    releaseLock();
} else {
   return getFromL2(key);
}
```

```java
boolean flag = pool.connection().sync().lockImmediately(key,1000);
loaderResult result;
if(flag) {
    result = invokeLoader(key, finalLoader);
    finalLoader.callback(result);
} else {
    result = getFromL2(key);
    if (result != null) {
        return result;
    } else {
        Thread.sleep(200);
        return getFromL2(key);
    }
}
```



```java

```

