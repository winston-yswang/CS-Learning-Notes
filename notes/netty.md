## 一、Netty——异步和事件驱动

### 1.1 Java网络编程

> Java API（java.net）阻塞 I/O 示例

```java
// 创建一个新的 ServerSocket，用以监听指定端口上的连接请求
ServerSocket serverSocket = new ServerSocket(portNumber);
// 对 accept()方法的调用将被阻塞，直到一个连接建立
Socket clientSocket = serverSocket.accept();
// 这些流对象都派生于该套接字的流对象
BufferedReader in = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
PrintWriter out = new PrintWriter(clientSocket.getOutputStream(), true);
String request, response;
// 处理循环开始
while ((request = in.readLine()) != null) {
	if ("Done".equals(request)) {
		break;
	}
	response = processRequest(request);	// 请求被传递给服务器的处理方法
	out.println(response);
}
```

- ServerSocket 上的 accept()方法将会一直阻塞到一个连接建立 ，随后返回一个新的 Socket 用于客户端和服务器之间的通信。该 ServerSocket 将继续监听传入的连接。
- BufferedReader 和 PrintWriter 都衍生自 Socket 的输入输出流 。前者从一个字符输入流中读取文本，后者打印对象的格式化的表示到文本输出流。
- readLine()方法将会阻塞，直到在处一个由换行符或者回车符结尾的字符串被读取。

上面这段代码片段将只能同时处理一个连接，要管理多个并发客户端，需要为每个新的客户端Socket 创建一个新的 Thread，如图1-1所示。

![image-20211203194923007](pics\image-20211203194923007.png)

这种方案的不足之处：

1. **在任何时候都可能有大量的线程处于休眠状态**，只是等待输入或者输出数据就绪，对资源是一种浪费。
2. 需要为每个线程的调用栈都分配内存，其默认值大小区间为 64 KB 到 1 MB，具体取决于操作系统。
3. 即使 Java 虚拟机（JVM）在物理上可以支持非常大数量的线程，但是远在到达该极限之前，上下文切换所带来的开销就会带来麻烦，例如，在达到 10 000 个连接的时候。



> Java NIO

Java 对于非阻塞 I/O 的支持是在 2002 年引入的，位于 JDK 1.4 的 java.nio 包中。

Java解决上节弊端的设计是使用$class java.nio.channels.Selector$ 的非阻塞 I/O，如图1-2。

<img src="pics\image-20211203195942156.png" alt="image-20211203195942156" style="zoom: 67%;" />

它使用了事件通知 API以确定在一组非阻塞套接字中有哪些已经就绪能够进行 I/O 相关的操作。因为可以在任何的时间检查任意的读操作或者写操作的完成状态，所以如图 1-2 所示，一个单一的线程便可以处理多个并发的连接。

这种模型相对于阻塞 I/O 模型，提供了更好的资源管理：

- 使用较少的线程便可以处理许多连接，因此也减少了内存管理和上下文切换所带来开销
- 当没有 I/O 操作需要处理的时候，线程也可以被用于其他任务。



### 1.2 Netty的核心组件

面向对象的基本概念：**用较简单的抽象隐藏底层实现的复杂性**。

非阻塞网络调用使得我们可以不必等待一个操作的完成。选择器使得我们能够通过较少的线程便可监视许多连接上的事件。这使得处理大量事件时，非阻塞I/O相较于阻塞I/O更加快速和经济。

> Channel

Channel 是 Java NIO 的一个基本构造。它代表一个到实体（如一个硬件设备、一个问题或一个I/O操作的程序组件）的开放连接，如读操作和写操作。

目前，可以把 **Channel 看作是传入（入站）或者传出（出站）数据的载体**。因此，它可以被打开或者被关闭，连接或者断开连接。

> 回调

一个回调其实就是一个方法，一个指向已经被提供给另外一个方法的方法的引用。当一个应用异步调用给另一个应用时，会把自身引用也传过去，使得后者可以在适当的时候调用前者。

Netty 在内部使用了回调来处理事件；当一个回调被触发时，相关的事件可以被一个 $interface ChannelHandler$ 的实现处理。代码清单 1-2 展示了一个例子：当一个新的连接已经被建立时，

ChannelHandler 的 channelActive()回调方法将会被调用，并将打印出一条信息。

```java
public class ConnectHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        // 当一个新的连接已经被建立时，channelActive(ChannelHandlerContext)将会被调用
		System.out.println("Client " + ctx.channel().remoteAddress() + " connected");
	} 
}
```

> Future

**Future 提供了另一种在操作完成时通知应用程序的方式**。这个对象可以看作是一个异步操作的结果的占位符；它将在未来的某个时刻完成，并提供对其结果的访问。

JDK 预置了 interface java.util.concurrent.Future，但是其所提供的实现，只允许手动检查对应的操作是否已经完成，或者一直阻塞直到它完成。这是非常繁琐的，所以 Netty 提供了它自己的实现——ChannelFuture，用于在执行异步操作的时候使用。

ChannelFuture提供了几种额外的方法，这些方法使得我们能够注册一个或者多个ChannelFutureListener实例。监听器的回调方法operationComplete()，将会在对应的操作完成时被调用。如果在 ChannelFutureListener 添加到 ChannelFuture 的时候，ChannelFuture 已经完成，那么该 ChannelFutureListener 将会被直接地通知。然后监听器可以判断该操作是成功地完成了还是出错了。如果是后者，我们可以检索产生的Throwable。

简而言之 ，**由ChannelFutureListener提供的通知机制消除了手动检查对应的操作是否完成的必要**。

每个 Netty 的出站 I/O 操作都将返回一个 ChannelFuture；也就是说，它们都不会阻塞。正如我们前面所提到过的一样，**Netty 完全是异步和事件驱动的**。

```java
/*
* 异步地建立连接
*/

Channel channel = ...;
// 异步地连接到远程节点
ChannelFuture future = channel.connect(new InetSocketAddress("192.168.0.1", 25));

// 要注册一个新的 ChannelFutureListener 到对 connect()方法的调用所返回的 ChannelFuture 上。
// 以便在操作完成时获得通知
future.addListener(new ChannelFutureListener() {
    @Override
    public void operationComplete(ChannelFuture future) {
        if (future.isSuccess()){	// 如果操作是成功的，则创建一个 ByteBuf 以持有数据
            ByteBuf buffer = Unpooled.copiedBuffer("Hello",Charset.defaultCharset());
            // 将数据异步地发送到远程节点。返回一个 ChannelFuture
            ChannelFuture wf = future.channel().writeAndFlush(buffer);
            ....
        } else {	// 如果发生错误，则访问描述原因的 Throwable
        	Throwable cause = future.cause();
        	cause.printStackTrace();
    } }
});
```

如果你把 **ChannelFutureListener 看作是回调的一个更加精细的版本**，那么你是对的。事实上，回调和 Future 是相互补充的机制；它们相互结合，构成了 Netty 本身的关键构件块之一。

> 事件和ChannelHandler

**Netty 使用不同的事件来通知我们状态的改变或者是操作的状态**。这使得我们能够基于已经发生的事件来触发适当的动作。这些动作可能是：

- 记录日志
- 数据转换
- 流控制
- 应用程序逻辑

Netty 是一个网络编程框架，所以事件是按照它们与入站或出站数据流的相关性进行分类的。可能由入站数据或者相关的状态更改而触发的事件包括：

- 连接已被激活或者连接失活；
- 数据读取；
- 用户事件
- 错误事件。

出站事件是未来将会触发的某个动作的操作结果，这些动作包括：

- 打开或者关闭到远程节点的连接；
- 将数据写到或者冲刷到套接字。

每个事件都可以被分发给 ChannelHandler 类中的某个用户实现的方法。这是一个很好的将事件驱动范式直接转换为应用程序构件块的例子。图 1-3 展示了一个事件是如何被一个这样的ChannelHandler 链处理的。

![image-20211203215515227](pics\image-20211203215515227.png)

Netty 的 ChannelHandler 为处理器提供了基本的抽象，目前你可以认为每个 ChannelHandler 的实例都类似于一种为了响应特定事件而被执行的回调。

> Future、回调和 ChannelHandler
>

Netty的异步编程模型是建立在Future和回调的概念之上的，而将事件派发到ChannelHandler的方法则发生在更深的层次上。结合在一起，这些元素就提供了一个处理环境，使你的应用程序逻辑可以独立于任何网络操作相关的顾虑而独立地演变。这也是 Netty 的设计方式的一个关键目标。

拦截操作以及高速地转换入站数据和出站数据，都只需要你提供回调或者利用操作所返回的Future。这使得链接操作变得既简单又高效，并且促进了可重用的通用代码的编写。

> 选择器、事件和 EventLoop
>

**Netty 通过触发事件将 Selector 从应用程序中抽象出来，消除了所有本来将需要手动编写的派发代码。在内部，将会为每个 Channel 分配一个 EventLoop，用以处理所有事件**，包括：

- 注册感兴趣的事件
- 将事件派发给ChannelHandler
- 安排进一步的动作

EventLoop 本身只由一个线程驱动，其处理了一个 Channel 的所有 I/O 事件，并且在该EventLoop 的整个生命周期内都不会改变。



## 二、你的第一款Netty应用程序

<img src="C:\Users\Lenovo\Desktop\笔记\CS-Learning-Notes\notes\pics\image-20211204151834837.png" alt="image-20211204151834837" style="zoom:50%;" />

### 2.1 Netty 服务器端

所有的 Netty 服务器都需要以下两部分：

- 至少一个ChannelHandler—该组件实现了服务器对从客户端接收的数据的处理，即它的业务逻辑。
- 引导—这是配置服务器的启动代码。至少，它会将服务器绑定到它要监听连接请求的端口上。

> ChannelHandler 和业务逻辑
>

ChannelHandler，它是一个接口族的父接口，它的实现负责接收并响应事件通知。在 Netty 应用程序中，所有的数据处理逻辑都包含在这些核心抽象的实现中。

因为要实现的 Echo 服务器会响应传入的消息，所以它需要实现 ChannelInboundHandler 接口，用 来定义响应入站事件的方法。这个简单的应用程序只需要用到少量的这些方法，所以继承 ChannelInboundHandlerAdapter 类也就足够了，它提供了 ChannelInboundHandler 的默认实现。

我们通常会用到的方法例如：

- channelRead()—对于每个传入的消息都要调用；
- channelReadComplete()—通知ChannelInboundHandler最后一次对channelRead()的调用是当前批量读取中的最后一条消息；
- exceptionCaught()—在读取操作期间，有异常抛出时会调用。

```java
/*
* 该 Echo 服务器的 ChannelHandler 实现是 EchoServerHandler
*/

@Sharable	// 标示一个ChannelHandler 可以被多个 Channel 安全地共享
public class EchoServerHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ByteBuf in = (ByteBuf) msg;
        System.out.println("Server received: " + in.toString(CharsetUtil.UTF_8));
        ctx.write(in);	// 将接收到的消息写给发送者，而不冲刷出站消息
	}
    
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        // 将未决消息冲刷到远程节点，并且关闭该 Channel
       ctx.writeAndFlush(Unpooled.EMPTY_BUFFER).addListener(ChannelFutureListener.CLOSE);
    }
    
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();	// 打印异常栈跟踪
        ctx.close();	// 关闭该Channel
	} 
}
```

> 如果不捕获异常，会发生什么呢?

**每个 Channel 都拥有一个与之相关联的 ChannelPipeline，其持有一个 ChannelHandler 的实例链**。在默认的情况下，ChannelHandler 会把对它的方法的调用转发给链中的下一个 ChannelHandler。因此，如果 exceptionCaught()方法没有被该链中的某处实现，那么所接收的异常将会被传递到 ChannelPipeline 的尾端并被记录。为此，你的应用程序应该提供至少有一个实现了exceptionCaught()方法的 ChannelHandler。

> 关键点

- 针对不同类型的事件来调用 ChannelHandler；
- 应用程序通过实现或者扩展 ChannelHandler 来挂钩到事件的生命周期，并且提供自定义的应用程序逻辑；
- 在架构上，ChannelHandler 有助于保持业务逻辑与网络处理代码的分离。这简化了开发过程，因为代码必须不断地演化以响应不断变化的需求。



> 引导服务器

引导服务器涉及以下内容：

- 绑定到服务器将在其上监听并接受传入连接请求的端口；
- 配置 Channel，以将有关的入站消息通知给 EchoServerHandler 实例。

```java
public class EchoServer {
    private final int port;
    public EchoServer(int port) {
    	this.port = port;
    }
    public static void main(String[] args) throws Exception {
        if (args.length != 1) {
            System.err.println("Usage: " + EchoServer.class.getSimpleName() + " <port>");
        }
        int port = Integer.parseInt(args[0]);	// 设置端口值
        new EchoServer(port).start();	// 调用服务器的 start()方法
	}
    public void start() throws Exception {
        final EchoServerHandler serverHandler = new EchoServerHandler();
        EventLoopGroup group = new NioEventLoopGroup();	// 1、创建EventLoopGroup
        try {
            ServerBootstrap b = new ServerBootstrap();	// 2、创建ServerBootstrap
            b.group(group)
             .channel(NioServerSocketChannel.class) 	// 3、指定所使用的 NIO传输 Channel
             .localAddress(new InetSocketAddress(port)) 	// 4、使用指定的端口设置套接字地址
                // 5、添加一个EchoServerHandler 到子Channel的 ChannelPipeline
             .childHandler(new ChannelInitializer<SocketChannel>(){
                 @Override
                 public void initChannel(SocketChannel ch)
                     throws Exception {
                     ch.pipeline().addLast(serverHandler);① }
               });
            // 6、异步地绑定服务器；调用 sync()方法阻塞等待直到绑定完成
            ChannelFuture f = b.bind().sync();
            // 7、获取 Channel 的CloseFuture，并且阻塞当前线程直到它完成
			f.channel().closeFuture().sync();
		} finally {
            // 8、关闭 EventLoopGroup，释放所有的资源
			group.shutdownGracefully().sync();
		} 
    } 
}
```

上例中，因为正在使用的是 NIO 传输，所以指定了 NioEventLoopGroup 来接受和处理新的连接，并且将 Channel 的类型指定为 NioServerSocketChannel 。在此之后，将本地地址设置为一个具有选定端口的 InetSocketAddress 。服务器将绑定到这个地址以监听新的连接请求。

**当一个新的连接被接受时，一个新的子 Channel 将会被创建，而 ChannelInitializer 将会把一个EchoServerHandler 的实例添加到该 Channel 的 ChannelPipeline 中**。正如我们之前所解释的，这个 ChannelHandler 将会收到有关入站消息的通知。

接下来步骤⑥绑定了服务器 ，并等待绑定完成。（**对 sync()方法的调用将导致当前 Thread阻塞，一直到绑定操作完成为止**）。在⑦处，该应用程序将会阻塞等待直到服务器的 Channel关闭（因为你在 Channel 的 CloseFuture 上调用了 sync()方法）。然后，你将可以关闭EventLoopGroup，并释放所有的资源，包括所有被创建的线程

回顾一下刚完成的服务器实现中的重要步骤：

- EchoServerHandler 实现了业务逻辑； 

- main()方法引导了服务器；

引导过程中所需要的步骤如下：

- 创建一个 ServerBootstrap 的实例以引导和绑定服务器； 

- 创建并分配一个 NioEventLoopGroup 实例以进行事件的处理，如接受新连接以及读/写数据； 

- 指定服务器绑定的本地的 InetSocketAddress； 

- 使用一个 EchoServerHandler 的实例初始化每一个新的 Channel； 

- 调用 ServerBootstrap.bind()方法以绑定服务器。



### 2.2 Netty 客户端

Echo 客户端将会：

（1）连接到服务器； 

（2）发送一个或者多个消息； 

（3）对于每个消息，等待并接收从服务器发回的相同的消息； 

（4）关闭连接。

> 通过ChannelHandler 实现客户端逻辑

如同服务器，客户端将拥有一个用来处理数据的 ChannelInboundHandler。在这个场景下，可以扩展 SimpleChannelInboundHandler 类以处理所有必须的任务，这要求重写下面的方法：

- channelActive()——在到服务器的连接已经建立之后将被调用；
- channelRead0()——当从服务器接收到一条消息时被调用；
- exceptionCaught()——在处理过程中引发异常时被调用。

```java
/*
* 客户端的 ChannelHandler
*/

@Sharable
public class EchoClientHandler extends SimpleChannelInboundHandler<ByteBuf> {
    /* 将在一个连接建立时被调用 */
    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        ctx.writeAndFlush(Unpooled.copiedBuffer("Netty rocks!", CharsetUtil.UTF_8)); 
    }
    /* 每当接收数据时，都会调用;
    由服务器发送的消息可能会被分块接收。也就是说，如果服务器发送了 5 字节，那么不
	能保证这 5 字节会被一次性接收。即使是对于这么少量的数据，channelRead0()方法也可能
	会被调用两次，第一次使用一个持有 3 字节的 ByteBuf（Netty 的字节容器），第二次使用一个
	持有 2 字节的 ByteBuf。作为一个面向流的协议，TCP 保证了字节数组将会按照服务器发送它
	们的顺序被接收。
	*/
    @Override
    public void channelRead0(ChannelHandlerContext ctx, ByteBuf in) {
    	System.out.println("Client received: " + in.toString(CharsetUtil.UTF_8));
    }
    
	@Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
    cause.printStackTrace();
    ctx.close();
	} 
}
```



> SimpleChannelInboundHandler 与 ChannelInboundHandler

你可能会想：为什么我们在客户端使用的是 SimpleChannelInboundHandler，而不是在 EchoServerHandler 中所使用的 ChannelInboundHandlerAdapter 呢？这和两个因素的相互作用有关：业务逻辑如何处理消息以及 Netty 如何管理资源。

**在客户端，当 channelRead0()方法完成时，你已经有了传入消息，并且已经处理完它了。当该方法返回时，SimpleChannelInboundHandler 负责释放指向保存该消息的 ByteBuf 的内存引用**。

在 EchoServerHandler 中，你仍然需要将传入消息回送给发送者，而 write()操作是异步的，直到 channelRead()方法返回后可能仍然没有完成。为此，EchoServerHandler扩展了 ChannelInboundHandlerAdapter，其在这个时间点上不会释放消息。

消息在 EchoServerHandler 的 channelReadComplete()方法中，当 writeAndFlush()方法被调用时被释放。

> 引导客户端

引导客户端类似于引导服务器，不同的是，客户端是使用主机和端口参数来连接远程地址。

```java
/*
* 客户端的主类
*/
public class EchoClient {
	private final String host;
	private final int port;
	public EchoClient(String host, int port) {
		this.host = host;
		this.port = port;
	}
    
	public void start() throws Exception {
		EventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap();
            b.group(group)
             .channel(NioSocketChannel.class)	// 客户端使用 OIO 传输
             .remoteAddress(new InetSocketAddress(host, port))
             .handler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                	ch.pipeline().addLast(new EchoClientHandler());}
             });
            // 在一切都设置完成后，调用 Bootstrap.connect()方法连接到远程节点
            ChannelFuture f = b.connect().sync();
            f.channel().closeFuture().sync();
        } finally {
        	group.shutdownGracefully().sync();
        } 
    }
	public static void main(String[] args) throws Exception {
        if (args.length != 2) {
            System.err.println("Usage: " + EchoClient.class.getSimpleName() + " <host> <port>");
            return;
        }
        String host = args[0];
        int port = Integer.parseInt(args[1]);
        new EchoClient(host, port).start();
    } 
}
```



## 三、Netty的组件和设计

### 3.1 Channel、EventLoop 和 ChannelFuture

这些类合在一起，可以被认为是 Netty 网络抽象的代表：

- Channel—Socket；
- EventLoop—控制流、多线程处理、并发；
- ChannelFuture—异步通知。

> Channel 接口

在基于 Java 的网络编程中，其基本的构造是 class Socket。Netty 的 Channel 接口所提供的 API，大大地降低了直接使用 Socket 类的复杂性。

> EventLoop 接口

EventLoop 定义了 Netty 的核心抽象，用于处理连接的生命周期中所发生的事件。图 3-1在高层次上说明了 Channel、EventLoop、Thread 以及 EventLoopGroup 之间的关系。

<img src="pics\image-20211204170046610.png" alt="image-20211204170046610" style="zoom: 67%;" />

- 一个 EventLoopGroup 包含一个或者多个 EventLoop； 
- 一个 EventLoop 在它的生命周期内只和一个 Thread 绑定； 
- 所有由 EventLoop 处理的 I/O 事件都将在它专有的 Thread 上被处理； 
- 一个 Channel 在它的生命周期内只注册于一个 EventLoop； 
- 一个 EventLoop 可能会被分配给一个或多个 Channel。

> ChannelFuture 接口

因为一个操作可能不会立即返回，所以我们需要一种用于在之后的某个时间点确定其结果的方法。为此，Netty 提供了ChannelFuture 接口，其 addListener()方法注册了一个 ChannelFutureListener，以便在某个操作完成时（无论是否成功）得到通知。

可以将 ChannelFuture 看作是将来要执行的操作的结果的占位符。它究竟什么时候被执行则可能取决于若干的因素，因此不可能准确地预测，但是可以肯定的是它将会被执行。此外，所有属于同一个 Channel 的操作都被保证其将以它们被调用的顺序被执行。



### 3.2 ChannelHandler 和 ChannelPipeline

现在，细致地看一看那些管理数据流以及执行应用程序处理逻辑的组件。

> ChannelHandler 接口

从应用程序开发人员的角度来看，Netty 的主要组件是 ChannelHandler，它充当了所有处理入站和出站数据的应用程序逻辑的容器。事实上，ChannelHandler 可专门用于几乎任何类型的动作，例如将数据从一种格式转换为另外一种格式，或者处理转换过程中所抛出的异常。

> ChannelPipeline 接口

ChannelPipeline 提供了 ChannelHandler 链的容器，并定义了用于在该链上传播入站和出站事件流的 API。当 Channel 被创建时，它会被自动地分配到它专属的 ChannelPipeline。

ChannelHandler 安装到 ChannelPipeline 中的过程如下所示：

- 一个ChannelInitializer的实现被注册到了ServerBootstrap中；
- 当 ChannelInitializer.initChannel()方法被调用时，ChannelInitializer将在 ChannelPipeline 中安装一组自定义的 ChannelHandler；
- ChannelInitializer 将它自己从 ChannelPipeline 中移除。

ChannelHandler 是专为支持广泛的用途而设计的，可以将它看作是处理往来 ChannelPipeline 事件（包括数据）的任何代码的通用容器。图 3-2 说明了这一点，其展示了从 ChannelHandler 派生的 ChannelInboundHandler 和 ChannelOutboundHandler 接口。

![image-20211205193300962](pics\image-20211205193300962.png)

使得事件流经 ChannelPipeline 是 ChannelHandler 的工作，它们是在应用程序的初始化或者引导阶段被安装的。这些对象接收事件、执行它们所实现的处理逻辑，并将数据传递给链中的下一个 ChannelHandler。它们的执行顺序是由它们被添加的顺序所决定的。实际上，被我们称为 **ChannelPipeline 的是这些 ChannelHandler 的编排顺序**。

图 3-3 说明了一个 Netty 应用程序中入站和出站数据流之间的区别。从一个客户端应用程序的角度来看，如果事件的运动方向是从客户端到服务器端，那么我们称这些事件为出站的，反之则称为入站的。

![image-20211205193537865](C:\Users\Lenovo\Desktop\笔记\CS-Learning-Notes\notes\pics\image-20211205193537865.png)

图 3-3 也显示了入站和出站 ChannelHandler 可以被安装到同一个 ChannelPipeline中。如果一个消息或者任何其他的入站事件被读取，那么它会从 ChannelPipeline 的头部开始流动，并被传递给第一个 ChannelInboundHandler。这个 ChannelHandler 不一定会实际地修改数据，具体取决于它的具体功能，在这之后，数据将会被传递给链中的下一个ChannelInboundHandler。最终，数据将会到达 ChannelPipeline 的尾端，届时，所有处理就都结束了。

数据的出站运动（即正在被写的数据）在概念上也是一样的。在这种情况下，数据将从ChannelOutboundHandler 链的尾端开始流动，直到它到达链的头部为止。在这之后，出站数据将会到达网络传输层，这里显示为 Socket。通常情况下，这将触发一个写操作。

**当ChannelHandler 被添加到ChannelPipeline 时，它将会被分配一个ChannelHandlerContext，其代表了 ChannelHandler 和 ChannelPipeline 之间的绑定。**虽然这个对象可以被用于获取底层的 Channel，但是它主要还是被用于写出站数据。



### 3.3 引导

Netty 的引导类为应用程序的网络层配置提供了容器，这涉及将一个进程绑定到某个指定的端口，或者将一个进程连接到另一个运行在某个指定主机的指定端口上的进程。

有两种类型的引导：一种用于客户端（简单地称为 Bootstrap），而另一种（ServerBootstrap）用于服务器。无论你的应用程序使用哪种协议或者处理哪种类型的数据，唯一决定它使用哪种引导类的是它是作为一个客户端还是作为一个服务器。

Bootstrap 和 ServerBootstrap两者的区别：

1. **ServerBootstrap 将绑定到一个端口，因为服务器必须要监听连接，而 Bootstrap 则是由想要连接到远程节点的客户端应用程序所使用的**。
2. 引导一个客户端只需要一个 EventLoopGroup，但是一个ServerBootstrap 则需要两个。

服务器需要两组不同的 Channel：

1. 第一组将只包含一个 ServerChannel，代表服务器自身的已绑定到某个本地端口的正在监听的套接字。
2. 第二组将包含所有已创建的用来处理传入客户端连接（对于每个服务器已经接受的连接都有一个）的 Channel

![image-20211205195507491](pics\image-20211205195507491.png)

与 ServerChannel 相关联的 EventLoopGroup 将分配一个负责为传入连接请求创建Channel 的 EventLoop。一旦连接被接受，第二个 EventLoopGroup 就会给它的 Channel分配一个 EventLoop。



## 四、传输

**流经网络的数据总是具有相同的类型：字节**。



## 五、ByteBuf

### 5.1 ByteBuf 类——Netty 的数据容器

**ByteBuf 维护了两个不同的索引：一个用于读取，一个用于写入。**当你从 ByteBuf 读取时，它的 readerIndex 将会被递增已经被读取的字节数。同样地，当你写入 ByteBuf 时，它的writerIndex 也会被递增。图 5-1 展示了一个空 ByteBuf 的布局结构和状态。

![image-20211210210528527](pics\image-20211210210528527.png)

要了解这些索引两两之间的关系，请考虑一下，如果打算读取字节直到 readerIndex 达到和 writerIndex 同样的值时会发生什么。在那时，你将会到达“可以读取的”数据的末尾。就如同试图读取超出数组末尾的数据一样，试图读取超出该点的数据将会触发一个 IndexOutOfBoundsException。 

名称以 read 或者 write 开头的 ByteBuf 方法，将会推进其对应的索引，而名称以 set 或 者 get 开头的操作则不会。后面的这些方法将在作为一个参数传入的一个相对索引上执行操作。

可以指定 ByteBuf 的最大容量。试图移动写索引（即 writerIndex）超过这个值将会触发一个异常。（默认的限制是 Integer.MAX_VALUE。）













