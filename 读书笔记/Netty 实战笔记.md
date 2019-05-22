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

- 是 Netty 的核心抽象，用于处理连接的生命周期中所发生的事情
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
- 一个 Channel 对应一个 ChannelPipeline
- 在多个 ChannelPipeline 中安装同一个 ChannelHandler 的一个常见的原因是用于收集跨越多个 Channel 的统计信息



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