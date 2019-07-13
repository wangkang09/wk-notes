[TOC]

# 1 网络编程介绍

## 1.1 java.net 阻塞 io

- in.readLine() 方法会阻塞，直到方法返回。注意这是当前线程只能等到 io 的返回。不能做任何其他事情。造成线程的浪费。

## 1.2 java.nio 非阻塞 io

- 通过选择器 selector 实现一个线程处理多个读写操作

## 1.3 Netty：异步和事件驱动

- **channel**：nio 的一个基本构造，可以看作数据的载体。它可以被打开或关闭，连接或断开。
- 回调：将一个方法传递给另一个方法，当另一个方法完成时，可以直接操作此方法。如我把我的作业本给小明，小明写完作业，直接将作业再抄一遍到我的作业本中。
- **ChannelHandler**：当 channel 中的某个事件触发时（ channel 打开），会运行相关事件。实现业务逻辑的地方。它是一个接口，它的实现负责接收并响应事件通知。
- 一个 Channel 中可以有多个 ChannelHandler
- **channelFuture**：和普通的 future 相比，可以向 channelFuture 中注册监听器（ChannelFutureListener），进行相应的监听，如 operationComplete() 监听 channelFuture 完成后的操作

## 1.4 阻塞、非阻塞、异步、同步

- **阻塞**：一个线程调用了一个 io 方法，如果方法没有返回，则线程一直阻塞在这里。不能去干别的，也不能被其它人调用
- # **非阻塞**：一个线程执行了一个 io 方法，这个线程立即返回去做别的事情（selector）
- 非阻塞又可分为异步和同步
- **同步非阻塞**：线程虽然可以去干别的事情，但是还是需要主动轮训已经调用的 io 方法有没有返回
- **异步非阻塞**：线程去干别的事情，并且不需要轮训有没有返回值，而是 io 有返回值时，主动通知该线程（回调）



# 2 编写 Netty 服务器与客户端

## 2.1 服务器编写

- 通过实现 ChannelInboundHandler 接口来实现业务逻辑
- 创建一个ServerBootstrap 实例来引导和绑定服务器
- 创建并分配一个 NioEventLoopGroup 实例进行事件处理，如接受新连接以及读/写数据
- 指定服务器绑定的本地端口：InetSocketAddress
- 使用一个 EchoServerHandler 的实例初始化每一个新的 Channel
- 调用 ServerBootstrap.bind() 方法以绑定服务器

## 2.2 客户端编写

- 通过 SimpleChannelInboundHandler 实现业务逻辑
- 创建一个 Bootstrap 实例来引导和绑定客户端
- 为事件分配一个 EventLoopGroup 实例
- 绑定服务端地址
- 当连接被建立时，一个 EchoClientHandler 实例会被安装到 channelPipeline 中
- 调用 Bootstrap.connect() 方法连接服务端



# 3 Netty 组件与设计概要

- Netty 是基于 Java NIO 的异步的和事件驱动的实现，保证了高负载下应用程序性能的最大化和可伸缩性
- Netty 也包含一组设计模式，它将应用程序逻辑从网络层解耦，简化了开发过程，同时也最大限度地提高了可测试性、模块化以及代码的可重用性

## 3.1 Channel

- Channel 相当于 Socket 的抽象，用于 I/O 操作
- Netty 的 Channel 接口所提供的 API，大大降低了直接使用 Socket 类的复杂性
- 并且，Channel 也有了一些常用的实现类
- Channel 是线程安全的，可以多个线程使用同一个 Channel

## 3.2 EventLoop 

- # 是 Netty 的核心抽象，用于处理连接的生命周期中所发生的事情
- Channel、EventLoop、Thread、EventLoopGroup 具有以下关系
  - 一个 EventLoopGroup 包含一个或多个 EventLoop
  - 一个 EventLoop 只和一个 Thread 绑定
  - 所有有 EventLoop 处理的 I/O 事件都将在它专有的 Thread 上被处理
  - 一个 Channel 只能注册到一个 EventLoop 中
  - 一个 EventLoop 中会有多个 Channel
  - 注：在一个 EventLoop 中，所有 Channel 的 I/O 操作，都有相同（与次 EventLoop 绑定）的 Thread 执行

## 3.3 ChannelFuture

- ChannelFuture 是 Future 的升级版，可以向 channelFuture 中注册监听器（ChannelFutureListener），进行相应的监听，可以在某个操作完成时得到通知
- 可以将 ChannelFuture 看作是将来要执行的操作的结果的占位符

## 3.4 ChannelHandler

- ChannelHandler 是开发人员所用到的主要组件，它完成了所有入站和出站数据的应用程序逻辑
- ChannelInboundHandler 用于接收入站事件和数据
- 当你要给连接的客户端发送相应时，可以从 ChannelInboundHandler 冲刷数据
- 你的应用程序的业务逻辑通常有一个或多个 ChannelInboundHandler
- 经常用到的适配器类：
  - ChannelHandlerAdapter
  - ChannelInboundHandlerAdapter
  - ChannelOutboundHandlerAdapter
  - ChannelDuplexHandler
- ChannelHandler 的典型用途：
  - 将数据从一种格式转换为另一种格式
  - 提供异常的通知
  - 提供 Channel 变为活动的或者非活动的通知
  - 提供当 Channel 注册到 EventLoop 或者从 EventLoop 注销时的通知

## 3.5 ChannelPipeline

- ChannelPipeline 使多个 ChannelHandler 形成一个 ChannelHandler 链
- 当 Channel 被创建时，会自动分配到它专属的 ChannelPipeline 中：过程如下
  - 一个 ChannelInitializer 的实现被注册到了 ServerBootStrap 或 BootStrap 中
  - 当 ChannelInitializer.initChannel() 方法被调用时，ChannelInitializer 将在 ChannelPipeline 中按安装一组自定义的 ChannelHandler
  - ChannelInitializer 将它自己从 ChannelPipeline 中移除
- ChannelPipeline 实现了一种常见的设计模式——拦截过滤器。UNIX 管道是另外一个熟悉的例子：多个命令被链接在一起，其中一个命令的输出端将连接到命令行中下一个命令的输入端

```java
ServerBootstrap b = new ServerBootstrap();
b.*.childHandler(new ChannelInitializer<SocketChannel>() {
    @Override
    public void initChannel(SocketChannel ch) throws Exception {
        ch.pipeline().addLast(serverHandler);
    }
});
Bootstrap b = new Bootstrap();
b.*.childHandler(new ChannelInitializer<SocketChannel>() {
    @Override
    public void initChannel(SocketChannel ch) throws Exception {
        ch.pipeline().addLast(serverHandler);
    }
});
```

- ChannelHandler 的执行顺序是由它们被添加的顺序决定的
- **出站**：事件的运动方向是从客户端到服务器，相对概念
- **入站**：事件的运动方向是从服务器到客户端，相对概念
- 入站和出站 ChannelHandler 可以被安装到同一个 ChannelPipeline 中；
  - 入站事件火车 ChannelPipeline 的头部开始流动，一直到尾部
  - 出站事件回城 ChannelOutboundHandler 链的尾部开始流动，一直到头部

## 3.6 ChannelHandler 子类实例说明：编解码器、SimpleChannelInboundHandler

### 3.6.1 编解码器

- 通过 ChannelHandler 来实现编解码器
- 入站数据被解码，出站数据被编码



## 3.7 引导

- 服务器需要两个 EventLoopGroup：
  - 第一组包含一个 ServerChannel，代表服务器本身的已绑定到某个本地端口的正在监听的 Socket
  - 第二组包含所有已创建的用来处理传入客户端连接的 Channel

- 与ServerChannel 相关联的 EventLoopGroup 将分配一个 EventLoop，用来为传入连接请求创建 Channel
- 第二个 EventLoopGroup 会给第一个创建的 Channel 分配一个 EventLoop



# 4 传输

- 流经网络的数据需要具有相同的类型：字节
- **网络传输**：一个帮我们抽象底层数据传输机制的概念

## 4.1 Channel 方法说明

| 方法名        | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| eventLoop     | 返回分配给 Channel 的 EventLoop                              |
| pipeline      | 返回分配给 Channel 的 ChannelPipeline                        |
| isActive      | 如果 Channel 是活动的，返回 true。活动的定义可能依赖于底层的传输。例如，一个 Socket 传输一旦连接了远程结点便是活动的，而一个 Datagram 传输一旦被打开便是活动的 |
| localAddress  | 返回本地的 SocketAddress                                     |
| remoteAddress | 返回远程的 SocketAddress                                     |
| write         | 将数据写到远程节点。这个数据将被传递给 ChannelPipeline，并且排队直到它被冲刷 |
| flush         | 将之前已写的数据冲刷到底层传输，如一个 Socket                |
| writeAndFlush | 一个简便的方法，等同于调用 write() 并接着调用 flush()        |

## 4.2 Netty 所提供的传输及其最佳应用

| 名称     | 包                          | 描述                                                         | 最佳应用                     |
| -------- | --------------------------- | ------------------------------------------------------------ | ---------------------------- |
| NIO      | io.netty.channel.socket.nio | 使用 java.nio.channels 包作为基础：基于选择器的方式          | 非阻塞代码库或一个常规的起点 |
| Epoll    | io.netty.channel.epoll      | 由 JNI 驱动的 epoll() 和非阻塞 IO。在 Linux 中比 NIO传输更快 | 同上但要在 Linux 环境        |
| OIO      | io.netty.channel.socket.oio | 使用 java.net 包作为基础：使用阻塞流                         | 阻塞代码库                   |
| Local    | io.netty.channel.local      | 可以在 VM 内部通过管道进行通信的本地传输                     | 在同一个 JVM 内部通信        |
| Emebedde | io.netty.channel.embedded   | Embedded 传输，运行使用 ChannelHandler 而不需要基于网络传输  | 测试 ChannelHandler的实现    |



# 5 ByteBuf

- 因为所有的网络通信都涉及字节序列的移动，所以高效易用的数据结构很关键
- Java NIO 提供的 ByteBuffer 作为字节容器，使用起来过于复杂其繁琐
- Netty 的 ByteBuf 是 ByteBuffer 的替代品，解决了 JDK API 的局限性

## 5.1 ByteBuf API 的优点

- 它可以被用户自定义的缓冲区类型扩展
- 通过内置的复合缓冲区类型实现了透明的零拷贝
- 容量可以按需增长（类似于 JDK 的 StringBuilder）
- 在读写模式之间切换不需要调用 ByteBuffer 的 flip() 方法
- 读写使用了不同的索引
- 支持方法的链式调用
- 支持引用计数
- 支持池化





# 6 ChannelHandler 和 ChannelPipeline

- channelHandlerContext.write()：将把缓冲区数据发送到下一个 ChannelHandler 中
- channel.write() 和 pipeline.write()：将 write 数据沿着整个 pipeline 传播
- 一个 Channel 对应一个 ChannelPipeline——每一个新创建的 Channel 都将会被分配一个新的 ChannelPipeline
- 在多个 ChannelPipeline 中安装同一个 ChannelHandler 的一个常见的原因是用于收集跨越多个 Channel 的统计信息
- 一个 Channel 中的所有 ChannelHandler 通过 ChannelPipeline 连接



# 7 EventLoop 与 线程模型

- 线程模型指定了操作系统、编程语言、框架或应用程序上下文中的线程管理的关键方面
- 如何以及何时创建线程将对应用程序代码的执行产生显著的影响

## 7.1 Java 线程模型演变

- 早期直接采用多线程的方式——在高负载下效果差，创建线程消耗较大
- Java 5 引入了线程池机制，通过缓存和重用线程极大提高了性能——随着线程数越来越多，上下文切换越加频繁，开销也变得显著
- Netty 中引入了 EventLoop 模型
  - 一个 EventLoop 将由一个永远都不会改变的 Thread 驱动，任务可以直接提交给 EventLoop 实现
  - 可以创建多个 EventLoop 实例来优化资源的使用
  - 单个 EventLoop 可能会被指派用于服务多个 Channel

## 7.2 JDK 和 Netty 任务调度方式

### 7.2.1 JDK 任务调度方式

- Java 5 之前，任务调度建立在 java.util.Timer 类之上，其使用了一个后台 Thread，之后，JDK 提供了 java.util.concurrent 包，它定义了 interface ScheduledExecutorService，来进行任务调度

```java
/**
 * 代码清单 7-2 使用 ScheduledExecutorService 调度任务
 * */
public static void schedule() {
    //创建一个其线程池具有 10 个线程的ScheduledExecutorService
    ScheduledExecutorService executor = Executors.newScheduledThreadPool(10);
    ScheduledFuture<?> future = executor.schedule(
        //创建一个 Runnable，以供调度稍后执行
        new Runnable() {
        @Override
        public void run() {
            //该任务要打印的消息
            System.out.println("Now it is 60 seconds later");
        }
    //调度任务在从现在开始的 60 秒之后执行
    }, 60, TimeUnit.SECONDS);
    //...
    //一旦调度任务执行完成，就关闭 ScheduledExecutorService 以释放资源
    executor.shutdown();
}
scheduleWithFixedDelay();//从一个执行终止到下一个执行开始的时间
scheduleAtFixedRate();//从第一次调度到下一次调度的时间
```

- 其关键使用了 new DelayedWorkQueue() 这个队列
- 如果有大量的任务被紧凑地调度，就会成为一个瓶颈。即使自定义线程池的其它参数，始终逃不过，创建线程和线程可能过多的情况

### 7.2.2 Netty 任务调度方式

- Netty 的 EventLoop 扩展了 ScheduledExecutorService
- Netty 通过 Channel 的 EventLoop 实现任务调度解决 JDK 瓶颈问题

```java
private static final Channel CHANNEL_FROM_SOMEWHERE = new NioSocketChannel();
/**
 * 代码清单 7-4 使用 EventLoop 调度周期性的任务
 * */
public static void scheduleFixedViaEventLoop() {
    Channel ch = CHANNEL_FROM_SOMEWHERE; // get reference from somewhere
    ScheduledFuture<?> future = ch.eventLoop().scheduleAtFixedRate(
       //创建一个 Runnable，以供调度稍后执行
       new Runnable() {
       @Override
       public void run() {
           //这将一直运行，直到 ScheduledFuture 被取消
           System.out.println("Run every 60 seconds");
       }
    //调度在 60 秒之后，并且以后每间隔 60 秒运行
    }, 60, 60, TimeUnit.SECONDS);
}
```



## 7.3 Netty 任务调度实现细节及优势

### 7.3.1 线程管理

`channel.eventLoop().execute(Task)`

- Netty 线程模型的卓越性能取决于对于当前执行的 Thread 的身份的确定
  - 把任务传递给 execute 方法之后，执行检查确定当前调用线程是否就是分配给 EventLoop 的那个线程
  - 如果是，就可以直接执行任务；如果不是，则将任务放入队列以便 EventLoop 下一次处理它的事件时执行

### 7.3.2 EventLoop/线程的分配

- 根据不同的传输实现，EventLoop 的创建和分配方式也不同

#### a 异步传输

- 异步传输实现只使用了少量的 EventLoop，且一个 EventLoop 可能被多个 Channel 共享，这使得可以通过少量的 Thread 来支撑大量的 Channel，而不是每一个 Channel 分配一个 Thread
- EventLoopGroup 负责为每个新创建的 Channel 分配一个 EventLoop。默认为顺序循环分配
- 注意：因为一个 EventLoop 中所关联的 Channel 的 ThreadLocal 都是一样的，这样使用 ThreadLocal 可能出现问题

#### b 阻塞传输

- 阻塞传输每一个 EventLoop 只关联一个 Channel，即，一个线程负责一个 Channel。Netty 用这种方式保证了设计上的一致性



# 8 引导

- 引导用来对应用程序进行配置，使得它能够将各个模块拼接起来，最后运行起来
- 引导使你的应用逻辑和网络层相隔离，无论它是客户端还是服务器
- 引导类的层次结构包括一个抽象的父类和两个具体的引导子类，分别代表客户端/服务器引导

## 8.1 引导类

- ServerBootstrap 代表的服务器，它使用一个父 Channel 来接受来自客户端的连接，并创建子 Channel 用于它们之间的通信
- Bootstrap 代表的客户端，它仅仅使用一个单独的、没有父 Channel 的 Channel 用于所有的网络交互
- 两种应用程序类型之间通用的引导步骤由 AbstractBootstrap 处理，而特定的引导步骤分别交个它的两个子类

## 8.2 从 Channel 引导客户端——即在同一个 EventLoop 下创建新的 Channel

- 假设你的服务器正在处理一个客户端的请求，这个请求需要它充当第三方系统的客户端
- 这种情况下，需要从已经被接收的子 Channel 中引导一个客户端 Channel（即这个 Channel 不是内部处理逻辑，而是网络处理逻辑）

- 一个 Channel 只能注册到一个 EventLoop，一个 EventLoop 可以关联多个 Channel，一个 Channel 对应多个 ChannelHandler

```java
public void channelActive(ChannelHandlerContext ctx)
    throws Exception {
    //创建一个 Bootstrap 类的实例以连接到远程主机
    Bootstrap bootstrap = new Bootstrap();
    //指定 Channel 的实现
    bootstrap.channel(NioSocketChannel.class).handler(
        //为入站 I/O 设置 ChannelInboundHandler
        new SimpleChannelInboundHandler<ByteBuf>() {
            @Override
            protected void channelRead0(
                ChannelHandlerContext ctx, ByteBuf in)
                throws Exception {
                System.out.println("Received data");
            }
        });
    //使用与分配给已被接受的子Channel相同的EventLoop
    bootstrap.group(ctx.channel().eventLoop());
    connectFuture = bootstrap.connect(
        //连接到远程节点
        new InetSocketAddress("www.manning.com", 80));
}
```

## 8.3 引导类中添加多个 ChannelHandler

- 通过 ChannelInitializer 类在引导类中添加多个 ChannelHandler

```java
/**
 * 代码清单 8-6 引导和使用 ChannelInitializer
 * */
public void bootstrap() throws InterruptedException {
    //创建 ServerBootstrap 以创建和绑定新的 Channel
    ServerBootstrap bootstrap = new ServerBootstrap();
    //设置 EventLoopGroup，其将提供用以处理 Channel 事件的 EventLoop
    bootstrap.group(new NioEventLoopGroup(), new NioEventLoopGroup())
        //指定 Channel 的实现
        .channel(NioServerSocketChannel.class)
        //注册一个 ChannelInitializerImpl 的实例来设置 ChannelPipeline
        .childHandler(new ChannelInitializerImpl());
    //绑定到地址
    ChannelFuture future = bootstrap.bind(new InetSocketAddress(8080));
    future.sync();
}

//用以设置 ChannelPipeline 的自定义 ChannelInitializerImpl 实现
final class ChannelInitializerImpl extends ChannelInitializer<Channel> {
    @Override
    //将所需的 ChannelHandler 添加到 ChannelPipeline
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new HttpClientCodec());
        pipeline.addLast(new HttpObjectAggregator(Integer.MAX_VALUE));

    }
}
```

## 8.4 ChannelOption 属性——统一配置所有 Channel 的通用属性

```java
//设置 ChannelOption，其将在 connect()或者bind()方法被调用时被设置到已经创建的 Channel 上
bootstrap.option(ChannelOption.SO_KEEPALIVE, true)
    .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000);
//存储该 id 属性
bootstrap.attr(id, 123456);
//使用配置好的 Bootstrap 实例连接到远程主机
ChannelFuture future = bootstrap.connect(...)
```

## 8.5 DatagramChannel 用于无连接协议

```java
//调用 bind() 方法，因为该协议是无连接的
ChannelFuture future = bootstrap.bind(new InetSocketAddress(0));
```

## 8.6 关闭

- 最重要的是关闭 EventLoopGroup

```java
public void bootstrap() {
	// ...
    //shutdownGracefully()方法将释放所有的资源，并且关闭所有的当前正在使用中的 Channel
    Future<?> future = group.shutdownGracefully();
    // block until the group has shutdown
    future.syncUninterruptibly();
}
```



# 9 EmbeddedChannel 测试 ChannelHandler

- ChannelHandler 是 Netty 应用程序的关键，必须彻底的测试它们
- 一个 Channel 中的所有 ChannelHandler 通过 ChannelPipeline 连接
- 通过 ChannelPipeline 形式，可以是一个大型任务分解为多个小的组件（ChannelHandler），易扩展，易修改

## 9.1 入站信息测试

```java
//将数据写入EmbeddedChannel，写入 inboundMessages 队列
assertTrue(channel.writeInbound(input.retain()));
// 从 inboundMessages 这个 队列中 poll 一个
ByteBuf read = (ByteBuf) channel.readInbound();
//返回从当前readerindex开始的缓冲区子区域的新切片，并将readerindex增加 3
ByteBuf b1 = buf.readSlice(3);
assertEquals(b1, read);//比较是否相同

//返回 false，因为没有一个完整的可供读取的帧 out为空，造成了 inboundMessages 为空，就返回null
//但是，只要后面 inboundMessages 不为空了，即使 out 为空，也返回 true
assertFalse(channel.writeInbound(input.readBytes(2)));

```

## 9.2 出站信息测试

```java
//(1) 创建一个 ByteBuf，并且写入 9 个负整数
ByteBuf buf = Unpooled.buffer();
for (int i = 1; i < 10; i++) {
    buf.writeInt(i * -1);
}
//(2) 创建一个EmbeddedChannel，并安装一个要测试的 AbsIntegerEncoder
EmbeddedChannel channel = new EmbeddedChannel(
    new AbsIntegerEncoder());
//(3) 写入 ByteBuf，并断言调用 readOutbound()方法将会产生数据
assertTrue(channel.writeOutbound(buf));
//(4) 将该 Channel 标记为已完成状态
assertTrue(channel.finish());
// read bytes
//(5) 读取所产生的消息，并断言它们包含了对应的绝对值
for (int i = 1; i < 10; i++) {
    assertEquals(i, channel.readOutbound());
}
assertNull(channel.readOutbound());
```

## 9.3 异常测试

```java
//channel 中有一个 decoder：判断如果输入的字节大于3，就报错
assertTrue(channel.writeInbound(input.readBytes(2)));
try {
    //写入一个 4 字节大小的帧，并捕获预期的TooLongFrameException
    channel.writeInbound(input.readBytes(4));
    //我们是异常测试，所以如果没有抛异常说明是不对的
    Assert.fail();
} catch (TooLongFrameException e) {
   //有异常反而直接通过
}
//写入剩余的2字节，并断言将会产生一个有效帧
assertTrue(channel.writeInbound(input.readBytes(3)));
//将该 Channel 标记为已完成状态
assertTrue(channel.finish());
```



# 10 编解码器

- 数据和网络字节流之间做相互转换是最常见的编程任务之一
- 我们需要处理标准的格式或者协议（如：FTP 或 Telnet）
- **编码器**：将应用程序的数据转换为网络格式
- **解码器**：将网络格式转换为应用程序的数据
- Netty 为许多通用协议提供了编解码器和处理器，几乎可以开箱即用

## 10.1 解码器

- 将字节解码为消息：ByteToMessageDecoder 和 ReplayingDecoder
- 将一种消息类型解码为另一种：MessageToMessageDecoder

### 10.1.1 ByteToMessageDecoder

- Netty 专门提供了一个抽象基类：ByteToMessageDecoder
- 这个类会对入站数据进行缓冲，直到准备好处理：因为不知道远程节点是否会一次性地发送一个完整的消息

| 方法                   | 描述                                                         |
| ---------------------- | ------------------------------------------------------------ |
| decode(ctx,in,out)     | 这是你必须实现的唯一抽象方法。decode()方法被调用时将会转入一个包含了传入数据的ByteBuf，以及一个用来添加解码消息的List。对这个方法的调用将会重复进行，直到确定没有新的元素被添加到该List，或该ByteBuf中没有更多可读取的字节时为止。然后，如果该List不为空，那么它的内容将会被传递给ChannelPipeline中的下一个ChannelInboundHandler |
| decodeLast(ctx,in,out) | Netty提供的默认实现只是简单地调用了decode()方法。当Channel的状态变为非活动时，这个方法将会被调用一次。可以重写该方法以提供特殊的处理 |

### 10.1.2 ReplayingDecoder

- ReplayingDecoder 扩展了 ByteToMessageDecoder 类，就不必调用 readableBytes() 方法了
- ReplayingDecoder 稍慢于 ByteToMessageDecoder
- 并不是所有的 ByteBuf 操作都被支持，如果调用一个不被支持的方法，将会抛出一个 UnsupportedOperationException

| 更多的解码器                                 | 描述                                         |
| -------------------------------------------- | -------------------------------------------- |
| io.netty.handler.codec.LineBasedFrameDecoder | 它使用了行尾控制字符(\n或\r\n)来解析消息数据 |
| io.netty.handler.codec.httpObjectDecoder     | 一个HTTP数据的解码器                         |
| io.netty.handler.codec 子包                  | 有更多用于特定用例的编解码实现               |

### 10.1.3 MessageToMessageDecoder

- 从一种 POJO 类型转换为另一种

```java
//扩展了MessageToMessageDecoder<Integer>
public class IntegerToStringDecoder extends
    MessageToMessageDecoder<Integer> {
    @Override
    public void decode(ChannelHandlerContext ctx, Integer msg,
        List<Object> out) throws Exception {
        //将 Integer 消息转换为它的 String 表示，并将其添加到输出的 List 中
        out.add(String.valueOf(msg));
    }
}
```

### 10.1.4 TooLongFrameException

- 由于 Netty 在解码之前要在内存中缓冲它们。因此不能让解码器缓冲大量的数据，以至于耗尽可用内存
- Netty 通过 TooLongFrameException 来限制缓冲的大小，但超过指定大小时，抛出这个异常

```java
//扩展 ByteToMessageDecoder 以将字节解码为消息
public class SafeByteToMessageDecoder extends ByteToMessageDecoder {
    private static final int MAX_FRAME_SIZE = 1024;
    @Override
    public void decode(ChannelHandlerContext ctx, ByteBuf in,
        List<Object> out) throws Exception {
            int readable = in.readableBytes();
            //检查缓冲区中是否有超过 MAX_FRAME_SIZE 个字节
            if (readable > MAX_FRAME_SIZE) {
                //跳过所有的可读字节，抛出 TooLongFrameException 并通知 ChannelHandler
                in.skipBytes(readable);
                throw new TooLongFrameException("Frame too big!");
        }
        // do something
        // ...
    }
}
```



## 10.2 编码器

- 将消息编码为字节
- 将消息编码为消息

### 10.2.1 抽象类 MessageToByteEncoder

- 和解码器相比，这个类只有一个方法，而解码器有两个：因为解码器通常需要在 Channel 关闭之后产生最后一个消息。而编码器不需要
- Netty 提供了 MessageToByteEncoder 抽象类来简化操作

```java
//扩展了MessageToByteEncoder
public class ShortToByteEncoder extends MessageToByteEncoder<Short> {
    @Override
    public void encode(ChannelHandlerContext ctx, Short msg, ByteBuf out)
        throws Exception {
        //将 Short 写入 ByteBuf 中
        out.writeShort(msg);
    }
}
```

### 10.2.2 抽象类 MessageToMessageEncoder

- Netty 提供了 MessageToMessageEncoder 抽象类来简化操作
- io.netty.handler.codec.protobuf.ProtobufEncoder：是这个抽象类的专业用法，它处理了有 Google 的 Protocol Buffers 规范所定义的数据格式

```java
//扩展了 MessageToMessageEncoder
public class IntegerToStringEncoder
    extends MessageToMessageEncoder<Integer> {
    @Override
    public void encode(ChannelHandlerContext ctx, Integer msg,
        List<Object> out) throws Exception {
        //将 Integer 转换为 String，并将其添加到 List 中
        out.add(String.valueOf(msg));
    }
}
```



## 10.3 抽象的编解码器类

- 在同一个类中管理入站和出站数据和消息的转换有时候很有用
- Netty 提供了一些抽象类来简化处理

### 10.3.1 抽象类 ByteToMessageCodec

- 任何请求/相应协议都可以作为使用 ByteToMessageCodec 的理想选择。
- 例如，在某个 SMTP 的实现中，编解码器将读取传入字节，并将它们解码为一个自定义的消息类型：如SmtpRequest。而在接收端，当一个相应被创建时，将会产生一个 SmtpResponse，其将被编码回字节以便进行传输

### 10.3.2 抽象类 MessageToMessageCodec

```java
//WebSocket 协议能实现 Web 浏览器到服务器之间的全双向通信
//代码略
```

### 10.3.3 CombinedChannelDuplexHandler 类

- 结合一个解码器和编码器可能会对可重用性造成影响
- 通过 CombinedChannelDuplexHandler 类来结合两个类，来避免这个问题

```java
//通过该解码器和编码器实现参数化 CombinedByteCharCodec
public class CombinedByteCharCodec extends
    CombinedChannelDuplexHandler<ByteToCharDecoder, CharToByteEncoder> {
    public CombinedByteCharCodec() {
        //将委托实例传递给父类
        super(new ByteToCharDecoder(), new CharToByteEncoder());
    }
}
```


# 11 预置的 ChannelHandler 和 编解码器

## 11.1 通过 SSL/TLS 保护 Netty 应用程序

- **SSL：**Secure Socket Layer，安全套接字层；是位于可靠的面向连接的网络层协议和应用层协议之间的协议层。SSL 通过互认证、使用数字签名确保完整性、使用加密确保私密性，以实现客户端和服务器之间的安全通行。该协议有两层组成：SSL 记录协议和 SSL 握手协议。
- **TLS：**Transport LayerSecurity，传输层安全协议；用于两个应用程序之间提供保密性和数据完整性。该协议有两层组成：TLS 记录协议和 TLS 握手协议。
- 这两个协议层叠在其他协议之上，用以实现数据安全。
- Netty 有基于 javax.net.ssl 包的两种协议实现，通过 SslHandler 类来实现
- Netty 还提供了基于 OpenSSL 工具包的 SSLEngine 实现。比JDK提供的 SSLEngine 性能更好

```java
public class SslChannelInitializer extends ChannelInitializer<Channel> {
    private final SslContext context;
    private final boolean startTls;
    public SslChannelInitializer(SslContext context,     //传入要使用的 SslContext
        boolean startTls) {                              //如果设置为 true，第一个写入的消息将不会被加密（客户端应该设置为 true）
        this.context = context;
        this.startTls = startTls;
    }
    @Override
    protected void initChannel(Channel ch) throws Exception {
        //对于每个 SslHandler 实例，都使用 Channel 的 ByteBufAllocator 从 SslContext 获取一个新的 SSLEngine
        SSLEngine engine = context.newEngine(ch.alloc());
        //将 SslHandler 作为第一个 ChannelHandler 添加到 ChannelPipeline 中
        ch.pipeline().addFirst("ssl",
            new SslHandler(engine, startTls));
    }
}
```

- 在握手阶段，两个节点将互相验证并且商定一种加密方式
- 握手阶段完成后，所有的数据都将会被加密
- SSL/TLS 握手将会被自动执行

## 11.2 基于 Netty 的 HTTP/HTTPS 应用程序

### 11.2.1 HTTP 解码器、编码器、编解码器

- HTTP 是基于请求/响应模式的：客户端向服务器发送一个 HTTP 请求，然后服务器将返回一个 HTTP 响应

```java
@Override
protected void initChannel(Channel ch) throws Exception {
    ChannelPipeline pipeline = ch.pipeline();
    if (client) {
        //如果是客户端，则添加 HttpResponseDecoder 以处理来自服务器的响应
        pipeline.addLast("decoder", new HttpResponseDecoder());
        //如果是客户端，则添加 HttpRequestEncoder 以向服务器发送请求
        pipeline.addLast("encoder", new HttpRequestEncoder());
    } else {
        //如果是服务器，则添加 HttpRequestDecoder 以接收来自客户端的请求
        pipeline.addLast("decoder", new HttpRequestDecoder());
        //如果是服务器，则添加 HttpResponseEncoder 以向客户端发送响应
        pipeline.addLast("encoder", new HttpResponseEncoder());
    }
}
```

### 11.2.2 聚合 HTTP 消息

- 通过 HttpPipelineInitializer 我们可以出来不同类型的 HttpObject 消息了。但由于 HTTP 的请求和响应可能有许多部分组成，所以还需要聚合它们形成完整的信息
- Netty 就提供了一个聚合器
- 可以将消息分段缓冲，然后直接转发一个完整的消息给下一个 ChannelInboundHandler

```java
@Override
protected void initChannel(Channel ch) throws Exception {
    ChannelPipeline pipeline = ch.pipeline();
    if (isClient) {
        //如果是客户端，则添加 HttpClientCodec
        pipeline.addLast("codec", new HttpClientCodec());
    } else {
        //如果是服务器，则添加 HttpServerCodec
        pipeline.addLast("codec", new HttpServerCodec());
    }
    //将最大的消息大小为 512 KB 的 HttpObjectAggregator 添加到 ChannelPipeline
    pipeline.addLast("aggregator",
                     new HttpObjectAggregator(512 * 1024));
}
```

### 11.2.3 HTTP 压缩

- 当使用 HTTP 是，建议开启压缩功能以尽可能地减少传输数据的大小（特别是对文本数据）。虽然压缩会有 CPU 开销

```java
@Override
protected void initChannel(Channel ch) throws Exception {
    ChannelPipeline pipeline = ch.pipeline();
    if (isClient) {
        //如果是客户端，则添加 HttpClientCodec
        pipeline.addLast("codec", new HttpClientCodec());
        //如果是客户端，则添加 HttpContentDecompressor 以处理来自服务器的压缩内容
        pipeline.addLast("decompressor",
                         new HttpContentDecompressor());
    } else {
        //如果是服务器，则添加 HttpServerCodec
        pipeline.addLast("codec", new HttpServerCodec());
        //如果是服务器，则添加HttpContentCompressor 来压缩数据（如果客户端支持它）
        pipeline.addLast("compressor",
                         new HttpContentCompressor());
    }
}
```

### 11.2.4 使用 HTTPS

- 使用 HTTPS 只需要将 SslHandler 添加到 ChannelPipeline 中

```java
@Override
protected void initChannel(Channel ch) throws Exception {
    ChannelPipeline pipeline = ch.pipeline();
    SSLEngine engine = context.newEngine(ch.alloc());
    //将 SslHandler 添加到ChannelPipeline 中以使用 HTTPS
    pipeline.addFirst("ssl", new SslHandler(engine));
    if (isClient) {
        //如果是客户端，则添加 HttpClientCodec
        pipeline.addLast("codec", new HttpClientCodec());
    } else {
        //如果是服务器，则添加 HttpServerCodec
        pipeline.addLast("codec", new HttpServerCodec());
    }
}
```

### 11.2.5 WebSocket

- WebSocket 解决了一个长期存在的问题：（服务端）如何实时发布信息
- AJAX 提供了一定程度的概述，但数据流仍然是由客户端所发送的请求驱动
- WebSocket 提供了“在一个单个的 TCP 连接上提供双向的通信…结合 WebSocket API ... 它为网页和远程服务器之前的双向通信提供了一种替代 HTTP 轮询的方案”
- WebSocket 在客户端和服务器之间提供了真正的双向数据交换

```java
//https://github.com/netty/netty/tree/4.1/example/src/main/java/io/netty/example/http/websocketx/client
public class WebSocketServerInitializer extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ch.pipeline().addLast(
            //这里可以添加 SslHandler，来提供安全性
            new HttpServerCodec(),
            //为握手提供聚合的 HttpRequest
            new HttpObjectAggregator(65536),
            //如果被请求的端点是"/websocket"，则处理该升级握手
            new WebSocketServerProtocolHandler("/websocket"),
            //TextFrameHandler 处理 TextWebSocketFrame
            new TextFrameHandler(),
            //BinaryFrameHandler 处理 BinaryWebSocketFrame
            new BinaryFrameHandler(),
            //ContinuationFrameHandler 处理 ContinuationWebSocketFrame
            new ContinuationFrameHandler());
    }
    public static final class TextFrameHandler extends
        SimpleChannelInboundHandler<TextWebSocketFrame> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx,
            TextWebSocketFrame msg) throws Exception {
            // Handle text frame
        }
    }
    public static final class BinaryFrameHandler extends
        SimpleChannelInboundHandler<BinaryWebSocketFrame> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx,
            BinaryWebSocketFrame msg) throws Exception {
            // Handle binary frame
        }
    }
    public static final class ContinuationFrameHandler extends
        SimpleChannelInboundHandler<ContinuationWebSocketFrame> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx,
            ContinuationWebSocketFrame msg) throws Exception {
            // Handle continuation frame
        }
    }
}
```

## 11.3 空闲的连接和超时

- 检测空闲连接以及超时对于及时释放资源来说是至关重要的
- Netty 提供了几个 ChannelHandler 来实现这些任务

| 名称                | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| IdleStateHandler    | 当连接空闲时间太长时，将会触发一个 IdleStateEvent 事件。然后，你可以通过在你的 ChannelInboundHandler 中重写 userEventTriggered() 方法来处理该 IdleStateEvent 事件 |
| ReadTimeoutHandler  | 如果在指定的时间间隔内没有收到任何的入站数据，则抛出一个 ReadTimeoutException 并关闭对应的 Channel。可以通过重写你的 ChannelHandler 中的 exceptionCaught() 方法来检测该 ReadTimeoutException |
| WriteTimeoutHandler | 如果在指定的时间间隔内没有任何出站数据写入，则抛出一个 WriteTimeoutException 并关闭对应的 Channel。可以通过重写你的 ChannelHandler 中的 exceptionCaught() 方法来检测该 WriteTimeoutException |

```java
public class IdleStateHandlerInitializer extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(
                //(1) IdleStateHandler 将在被触发时发送一个IdleStateEvent 事件
            	//如果连接超过60秒没有接收或者发送任何数据，那么 IdleStateHandler 将会使用一个 IdleStateEvent 事件来调用 fireUserEventTriggered() 方法。这将会触发后面的 HeartbeanHandler
                new IdleStateHandler(0, 0, 60, TimeUnit.SECONDS));
        //将一个 HeartbeatHandler 添加到ChannelPipeline中
        pipeline.addLast(new HeartbeatHandler());
    }
    //实现 userEventTriggered() 方法以发送心跳消息
    public static final class HeartbeatHandler
        extends ChannelInboundHandlerAdapter {
        //发送到远程节点的心跳消息
        private static final ByteBuf HEARTBEAT_SEQUENCE =
                Unpooled.unreleasableBuffer(Unpooled.copiedBuffer(
                "HEARTBEAT", CharsetUtil.ISO_8859_1));
        @Override
        public void userEventTriggered(ChannelHandlerContext ctx,
            Object evt) throws Exception {
            //(2) 发送心跳消息，并在发送失败时关闭该连接
            if (evt instanceof IdleStateEvent) {
                ctx.writeAndFlush(HEARTBEAT_SEQUENCE.duplicate())
                     .addListener(
                         ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                //不是 IdleStateEvent 事件，所以将它传递给下一个 ChannelInboundHandler
                super.userEventTriggered(ctx, evt);
            }
        }
    }
}
```

## 11.4 解码基于分隔符的协议和基于长度的协议

- 在使用 Netty 的过程中，你将会遇到需要解码器的基于分隔符和帧长度的协议

### 11.4.1 基于分隔符的协议

- 基于分隔符的消息协议使用定义的字符来标记消息或消息段的开头或结尾(一帧)。由 RFC 文档正式定义的许多协议（如SMTP、POP3、IMAP、Telnet）
- 消息也是按一帧一帧处理的
- 当定义自己的专有格式协议时，都可以由下表提取任意标记序列分割的帧的自定义解码器

| 名称                       | 描述                                                         |
| -------------------------- | ------------------------------------------------------------ |
| DelimiterBasedFrameDecoder | 使用任何由用户提供的分隔符来提取帧的通用解码器               |
| LineBasedFrameDecoder      | 提取由行尾符（\n或\r\n）分隔的帧的解码器。这个解码器比 DelimiterBasedFrameDecoder 更快 |

```java
public class LineBasedHandlerInitializer extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        //该 LineBasedFrameDecoder 将提取的帧转发给下一个 ChannelInboundHandler
        pipeline.addLast(new LineBasedFrameDecoder(64 * 1024));
        //添加 FrameHandler 以接收帧
        pipeline.addLast(new FrameHandler());
    }
    public static final class FrameHandler
        extends SimpleChannelInboundHandler<ByteBuf> {
        @Override
        //传入了单个帧的内容
        public void channelRead0(ChannelHandlerContext ctx,
            ByteBuf msg) throws Exception {
            // Do something with the data extracted from the frame
        }
    }
}
```

### 11.4.2 基于长度的协议

- 即一帧有固定的长度，而不是用分隔符来标记一帧
- Netty 提供了两种解码器

| 名称                         | 描述                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| FixedLengthFrameDecoder      | 提取在调用构造函数时指定的定长帧                             |
| LengthFieldBasedFrameDecoder | 根据编码进帧头部中的长度值提取帧；该字段的偏移量以及长度在构造函数中指定。一般在帧的起始的前8个字节规定帧长 |

```java
@Override
protected void initChannel(Channel ch) throws Exception {
    ChannelPipeline pipeline = ch.pipeline();
    pipeline.addLast(
            //使用 LengthFieldBasedFrameDecoder 解码将帧长度编码到帧起始的前 8 个字节中的消息
        	//第一个为 MaxFrameLength
            new LengthFieldBasedFrameDecoder(64 * 1024, 0, 8));
    //添加 FrameHandler 以处理每个帧
    pipeline.addLast(new FrameHandler());
}
```

## 11.5 写大型数据

- 因为网络饱和的可能性（网速），在异步框架高效地写大块数据是一个问题
- 因为写操作是非阻塞的，导致不停的写入，就会有之前写的数据没处理完，产生内存耗尽的风险
- 在写大型数据时，需要处理到远程节点的连接是慢速连接的情况，这会导致内存释放的延迟
- FileRegion：通过支持零拷贝的文件传输的 Channel 来发送的文件区域

```java
//这个示例只适用于文件内容的直接传输，不包括应用程序对数据的任何处理
public class FileRegionWriteHandler extends ChannelInboundHandlerAdapter {
    private static final Channel CHANNEL_FROM_SOMEWHERE = new NioSocketChannel();
    private static final File FILE_FROM_SOMEWHERE = new File("");
    @Override
    public void channelActive(final ChannelHandlerContext ctx) throws Exception {
        File file = FILE_FROM_SOMEWHERE; //get reference from somewhere
        Channel channel = CHANNEL_FROM_SOMEWHERE; //get reference from somewhere
        //...
        //创建一个 FileInputStream
        FileInputStream in = new FileInputStream(file);
        //以该文件的完整长度创建一个新的 DefaultFileRegion
        FileRegion region = new DefaultFileRegion(
                in.getChannel(), 0, file.length());
        //发送该 DefaultFileRegion，并注册一个 ChannelFutureListener
        channel.writeAndFlush(region).addListener(
            new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future)
               throws Exception {
               if (!future.isSuccess()) {
                   //处理失败
                   Throwable cause = future.cause();
                   // Do something
               }
            }
        });
    }
}
```

- 在需要将数据从文件系统复制到用户内存中时，可以使用 ChunkedWriteHandler ，它支持异步写大型数据流，而不会导致大量的内存消耗

```java
public class ChunkedWriteHandlerInitializer
    extends ChannelInitializer<Channel> {
    private final File file;
    private final SslContext sslCtx;
    public ChunkedWriteHandlerInitializer(File file, SslContext sslCtx) {
        this.file = file;
        this.sslCtx = sslCtx;
    }
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        //将 SslHandler 添加到 ChannelPipeline 中
        pipeline.addLast(new SslHandler(sslCtx.newEngine(ch.alloc())));
        //添加 ChunkedWriteHandler 以处理作为 ChunkedInput 传入的数据
        pipeline.addLast(new ChunkedWriteHandler());
        //一旦连接建立，WriteStreamHandler 就开始写文件数据
        pipeline.addLast(new WriteStreamHandler());
    }
    public final class WriteStreamHandler
        extends ChannelInboundHandlerAdapter {
        @Override
        //当连接建立时，channelActive() 方法将使用 ChunkedInput 写文件数据
        public void channelActive(ChannelHandlerContext ctx)
            throws Exception {
            super.channelActive(ctx);
            ctx.writeAndFlush(
            new ChunkedStream(new FileInputStream(file)));
        }
    }
}
```

## 11.6 序列化

- JDK 提供了 ObjectOutputStream 和 ObjectInputStream，用于通过网络对 POJO 进行序列化和反序列化。此 API 并不复制，但性能也不高。

### 11.6.1 JDK 序列化

- 如果应用程序必须要和使用了 ObjectOutputStream 和 ObjectInputStream 的远程节点交互，且关心兼容性，那么 JDK 序列化是正确的选择
- Netty 提供了几个用于和 JDK 进行互操作的序列化类

| 名称                    | 描述                                                         |
| ----------------------- | ------------------------------------------------------------ |
| CompatibleObjectEncoder | 和使用 JDK 序列化的非基于 Netty 的远程节点进行互操作的编码器 |
| ObjectDecoder           | 构建于 JDK 序列化之上的使用自定义的序列化来解码的解码器；当没有其他的外部依赖时，它提供了速度上的改进。否则其他的序列化实现更加可取 |
| ObjectEncoder           | 构建于 JDK 序列化之上的使用自定义的序列化来编码的编码器；当没有其他的外部依赖时，它提供了速度上的改进。否则其他的序列化实现更加可取 |

### 11.6.2 JBoss Marshalling 序列化

- 如果你可以自由地使用外部依赖，那么 JBoss Marshalling 将是个理想的选择
- 它比 JDK 序列化最多快3倍，而且更加紧凑
- 保留了与 Serializable 相关类的兼容性，并可通过工厂配置实现可插拔特性
- Netty 提供了2组类对 JBoss Marshalling 提供了支持

| 名称                                                       | 描述                                                    |
| ---------------------------------------------------------- | ------------------------------------------------------- |
| CompatibleMarshallingDecoder、CompatibleMarshallingEncoder | 与只使用 JDK 序列化的远程节点兼容                       |
| MarshallingDecoder、MarshallingEncoder                     | 适用于使用 JBoss Marshalling 的节点。这些类必须一起使用 |

```java
public class MarshallingInitializer extends ChannelInitializer<Channel> {
    private final MarshallerProvider marshallerProvider;
    private final UnmarshallerProvider unmarshallerProvider;
    public MarshallingInitializer(
            UnmarshallerProvider unmarshallerProvider,
            MarshallerProvider marshallerProvider) {
        this.marshallerProvider = marshallerProvider;
        this.unmarshallerProvider = unmarshallerProvider;
    }
    @Override
    protected void initChannel(Channel channel) throws Exception {
        ChannelPipeline pipeline = channel.pipeline();
        //添加 MarshallingDecoder 以将 ByteBuf 转换为 POJO
        pipeline.addLast(new MarshallingDecoder(unmarshallerProvider));
        //添加 MarshallingEncoder 以将POJO 转换为 ByteBuf
        pipeline.addLast(new MarshallingEncoder(marshallerProvider));
        //添加 ObjectHandler，以处理普通的实现了Serializable 接口的 POJO
        pipeline.addLast(new ObjectHandler());
    }
    public static final class ObjectHandler
        extends SimpleChannelInboundHandler<Serializable> {
        @Override
        public void channelRead0(
            ChannelHandlerContext channelHandlerContext,
            Serializable serializable) throws Exception {
            // Do something
        }
    }
}
```

### 11.6.3 Protocol Buffers 序列化

- Netty 序列化的最后一个解决方案是利用 Protocol Buffers 的编解码器

| 名称                                 | 描述                                                         |
| ------------------------------------ | ------------------------------------------------------------ |
| ProtobufDecoder                      | 使用 protobuf 对消息进行解码                                 |
| ProtobufEncoder                      | 使用 protobuf 对消息进行编码                                 |
| ProtobufVarint32FrameDecoder         | 根据消息中的 Google Protocol Buffers 的 “Base 128 Varints” 整型长度字段值动态地分割所接收到的 ByteBuf |
| ProtobufVarint32LengthFiledPrepender | 向 ByteBuf 前追加一个 Google Protocol Buffers 的 “Base 128 Varints” 整型的长度字段值 |

```java
private final MessageLite lite;
@Override
protected void initChannel(Channel ch) throws Exception {
    ChannelPipeline pipeline = ch.pipeline();
    //添加 ProtobufVarint32FrameDecoder 以分隔帧
    pipeline.addLast(new ProtobufVarint32FrameDecoder());
    //添加 ProtobufEncoder 以处理消息的编码
    pipeline.addLast(new ProtobufEncoder());
    //添加 ProtobufDecoder 以解码消息
    pipeline.addLast(new ProtobufDecoder(lite));
    //添加 ObjectHandler 以处理解码消息
    pipeline.addLast(new ObjectHandler());
}
```

# 12 WebSocket

- WebSocket 协议是完全重新设计的协议，旨在为 Web 上的双向数据传输问题提供一个切实可行的解决方案，使得客户端和服务器之间可以在任意时刻传输消息
- 因此，要求它们异步地处理消息回执

# 13 UDP

- 通常在要求性能且能够容忍一定的数据包丢失的情况下，可以使用 UDP 协议
- 相比于 TCP，UDP 取消了所有的握手、纠错及其它消息管理机制的开销
- TCP 就行打电话一样，信息必须顺序达到接收方，而 UDP 就像发邮件，邮件到达的顺序可能和发送的顺序不一样

## 13.1 UDP 广播

- **单播传输**：发送消息给一个由唯一的地址所标识的单一的网络目的地。面向连接和无连接协议都支持这种模式
- **多播**：传输到一个预定义的主机组
- **广播**：传输到网络上的所有主机