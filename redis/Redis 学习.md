[TOC]

##  1 什么是 Redis

- Redis 是一个速度非常快的非关系型数据库
- 它以 key/value 键值对的形式存储数据
- key 为 String 类型，Value 可以有 5 中不同类型的值
- 可以通过两种方式做数据持久化
- 有**主从复制、分片**特性分别扩展读性能、写性能

## 2 Redis 适用的场景

- 用户要求性能且对一致性要求不高的场景（如 Web 网页）——相比于关系型
- 要求性能且要**数据持久化**的场景——相比于其它的 NoSQL
- 要求性能且对存储的 Value 值的类型有要求的场景——相比于其它的 NoSQL
- 排行榜

## 3 为什么使用 Redis

- 速度快（内存数据库）
- 存储类型多样化（支持5中类型的值）
- 可靠性高（数据持久化、主从复制）
- 扩展性强（很容易的数据分片）
- 支持分布式锁、消息队列

## 4 五种数据结构

- **容器类型(list/set/hash/zset)的通用规则**
  - create if not exists：不存在就创建一个再操作
  - drop if no elements：容器里没有元素了，直接删除容器，是否内存

### 4.1 string

- **用途：**一般将用户信息结构体使用 JSON 序列化成字符串，然后放入 Redis 中
- 用户取是需要反序列化
- 字符串是动态的，可以修改，类似于 ArrayList，当长度小于 1MB 时，按倍数扩容，当大于 1MB 时，每次扩容 1MB。有最大长度（目前 512MB）

-----

- **设置键值对**

```java
set name wangkang
get name
exists name
del name
```

- **批量读写**

```java
//可以对多个字符串批量读写，节省网络耗时开销
mset name1 wk name2 ww name3 kk
mget name1 name2 name3
```

- **过期设置**
  - 过期是以对象为单位的，如果是一个 hash 结果的过期，是整个 hash 对象的过期，而不是某个 key 的过期
  - 如果一个字符串设置了过期时间，后有调用 set 方法修改了它，过期时间会消失

```java
set name guoqi
get name
expire name 5			//5s 后过期
setex name 5 guoqi		//原子操作
set name guoqi ex 5		//等效
ttl name				//返回有效时间，无限则返回-1
```

- **putIfAbsent**

```java
setnx name absentPut   //返回1
setnx name absentPut   //返回0
```

- **string 代表整数来计数**

```java
set age 30
incr age
incrby age 5
decr age
decrby age 5
```

- **字符串操作**

```java

append name world        //追加
strlen key  			 //字符串长度,中文占3个字节
getrange key 2 4 		 //截取字符串
getrange key 0 -1
```

### 4.2 list

- list 相当于 Java 中的 **LinkedList**，双向链表
- 插入和删除为 O(1)、搜索为 O(n)
- **用途：**通常用来做异步队列，用户将需要处理的任务结构体序列化成字符串，放入 list 中，另一个线程从这个列表中轮询数据进行处理

- 因为是双向链表，既可以当队列使用（先进先出），也可以当栈使用（后进先出）

------

- **list 操作**

```java
rpush key v1 v2 v3
llen key
lpush key vv1 vv2  //vv2 vv1 v1 v2 v3
lpop key //vv2
rpop key //vv1
```

- **慢操作**
  - lindex 相当于 LinkedList 中的 get(index) 方法
  - ltrim i0 i1：保留 i0-i1之间的数据

```java
lindex key 0
lrange key 0 -1         //-1 表示最后
ltrim key 1 -1          //保留第2个到最后一个元素
ltrim key 2 1           //区间为负时，表示清空列表
```

- **快速列表**
  - 在列表元素较少的情况下，会使用连续的内存存储，结构为 ziplist（压缩列表）
  - 在数据较多时，变为 quicklist（由多个 ziplist 组成）

### 4.3 hash 

- 相当于 Java 中的 HashMap 
- Redis 的 hash 的值只能是字符串，且 rehash 是通过渐进（后台）的方法做的（rehash时保留旧的，主线程查的时候，遍历新旧两个）

```java
hset mapName key value
hset books java "think in java"     //字符串包含空格时，用引号括起来
hset books python "python cookbook"
hset books java "dd"  					//更新操作，覆盖旧值
hget books java  							//如果没有返回 nil
hgetall books							     //获取所有成员
hlen books									 //长度
hmset books java "java javav" golong "go go" //批量添加
hdel hash-name sub-key
```

### 4.4 set

- 相当于 Java 中的 hashSet
- **用途：**去重 list

```java
sadd books python
sadd books java golang		//批量添加
smembers books				//列出所有成员
sismember books rust		//判断是否是成员
scard books  				//获取长度
spop books 					//随机弹出
srem set-keys key
```

### 4.5 zset（最有特色）

- 相当于 Java 的 SortedSet 和 HashMap 的结合体
- 一方面它是一个 set，保证了 key 的唯一性
- 另一方面它赋予每个 value 一个 score
- **用途：**
  - 存储粉丝列表，key 为用户 ID，score 为关注时间
  - 存储学生成绩，key 为学生 ID，score 为成绩
  - 其他排名列表

```java
zadd books 9.0 thinkinjava
zadd books 9.9 redis
zadd books 1.1 japen			  //添加
zrange books 0 -1				  //升序
zrevrange books 0 -1              //降序
zcard books                       //count
zscore books thinkinjava          //获取指定 key 的 score
zrank books thinkinjava			  //获取指定 key 的排名
zrangebyscore books 0 9.1         //根据分值区间遍历
zrangebyscore books -inf inf      //负无穷到正无穷
zrem books thinkinjava            //删除指定 key
```

- **跳跃链表**
  - zset 内部的排序功能是通过跳跃列表来实现的

## 5 分布式锁

```java
set lock:key value ex 5 nx //ex 表示失效，nx 表示存在才设置
```

## 6 消息队列

- 专用的消息中间件可能使用起来比较繁琐
- 如果对消息不要求极高的可靠性，且只有一组消费者队列，可以考虑使用 redis

### 6.1 异步消息队列

- 使用 list 结构做异步消息队列，最一个先进先出的队列

```java
rpush notify-queue apple banana   //生产者线程1
lpop notify-queue                 //消费者线程1
lpop notify-queue 				  //消费者线程2
rpush notify-queue pear bear      //生产者线程2
```

### 6.2 队列空了

- 队列空了后，客户端可能会陷入 pop 死循环，造成 客户端CPU消耗，REDIS 的负载也无故增加
- **解决方法**
  - 客户端线程休眠：会导致消息的延迟进一步增大
  - 使用 blpop/brpop 命令：没有数据时休眠，一旦有数据，立即醒来

### 6.3 空闲连接自动断开

- 当使用 blpop/brpop 命令被阻塞时，这个连接就成了闲置连接，一般一段时间后服务器会自动断开闲置连接，来减小闲置资源。这时就会抛出异常
- **解决方法：**重试..

### 6.4 延时/定时消息

- 通过 zset 来实现，将处理时间设置为 score
- 用多个线程（保证高可用）轮询 zset 获取到期的任务进行处理

```java
flag = setTrue();//只有一个线程能设置成功
if (flag == true) {
    task = zrangebyScore delay-queue 0 1;//获取最小时间的任务
    success = run(task);
    if (success) {
        zrem delay-queue task;
    }
} else {
    return;
}
```

## 7 位图

- 位图主要是操作每个字节的8个位

## 8 HyperLogLog 统计 UV

- 统计网页每天的 PV：直接使用 Redis 计数器就可以了
- 统计网页每天的 UV：必须更加 ID 去重，用 HyperLogLog 来实现

```java
pfadd pfName user1
pfadd pfName user2
pfcount pfName
```

### 8.1 HyperLogLog 的核心思想

#### 8.1.1 伯努利大数定律

- 随便去一个数，转换为二进制后：

- 它的末尾是 0 的概率为 1/2，按照伯努利大数定律，取 2 个数，出现 0 的概率接近 1
- 它的末尾是 00 的概率为 1/4，按照伯努利大数定律，取 4 个数，出现 00 的概率接近 1
- 它的末尾是 000 的概率为 1/8，按照伯努利大数定律，取 8 个数，出现 000 的概率接近 1
- 综上，取 n 个数，末尾出现 k 个 0 的概率接近 1。 k,n 满足 2^k = n
- 这样我们就可以近似的认为，如果一堆数中，末尾为 0 的个数的**最大值是 k** 的话，那么这堆数近似于 2^k 个

#### 8.1.2 大数定律估算优化

- 我们直接用大数定律来估算集合中数的个数会出现以下问题
  - 一个集合中只有2个数：1048576，1048578。集合数的末尾最大 0 的个数为 20
  - 按照大数定律，则估算的这个集合的数为 1048576 个，很不准确，虽然这种情况的概率很低(大约 1/ 1048576)，但是仍然会出现
- **解决方案：**
  - 1个集合分成 n 个集合，分别取每个集合最大末尾 0  的个数
  - **LogLog 估算**：取上述的平均数，做最终的 0 的个数
  - **HyperLogLog 估算**：取上述的调和平均数，做最终的 0  的个数
  - 做调和平均数的好处是，可以**有效的平滑离群值的影响**
- 点击[查看](<http://www.rainybowe.com/blog/2017/07/13/%E7%A5%9E%E5%A5%87%E7%9A%84HyperLogLog%E7%AE%97%E6%B3%95/index.html>)更详细分析

#### 8.1.3 Redis 中 HyperLogLog 流程

- redis 中 HyperLogLog 使用 前 14位来定位统计数组的位置，后50为用来记录第一个1出现的位置

- redis 中 HyperLogLog 算法的核心思想就是以上两节，细节可以查看[源码实现](<https://github.com/antirez/redis/blob/unstable/src/hyperloglog.c>)

## 9 布隆过滤器

### 9.1 什么是布隆过滤器

- 可以将布隆过滤器看作是一个不怎么精确的 set 结构，用它来判断某个对象是否存在
- 当它判断一个对象不存在时，则这个对象肯定不存在
- 当他判断一个对象存在时，这个对象**仍有小概率存在**

### 9.2 布隆过滤器的使用场景

- 需要**判断一个对象是否已经存在**
- 并且内存有限(如果使用 set，会消耗太多内存)
- 并且可以忍受一定的误判

### 9.3 redis 中布隆过滤器的基本用法

- redis 中的布隆过滤器是到 redis 4.0 提供了插件功能后，才作为一个插件加载到 redis server 中的

```java
bf.add bfName user1
bf.madd bfName user2 user3 user4
bf.exists bfName user1
bf.exists bfName user0
bf.mexists bfName user2 user user4
```

