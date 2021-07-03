#	线程
##	三种线程实现方式
	根据操作系统内核是否对线程可感知，可以把线程分为内核线程和用户线程
	1、使用内核线程实现 Windows与Linux JDK 实现的线程模型 用户线程 与内核线程 1:1
	2、使用用户线程实现
	3、使用用户线程和轻量级进程混合实现

##	线程分配方式
	1、抢占式 目前Linux、windows等操作系统对线程的分配方式为抢占式
	2、调度式

#	IO
##	IO原理 read、write
	上层程序的read write 方法本质上是内存在缓冲区的复制。而真正内核缓冲区与物理磁盘之间的内存交互上层应用程序并不感知,也不关心。
	内存read:物理磁盘->内核缓冲区->程序缓冲区
	内存write:程序缓冲区->内核缓冲区->物理磁盘
	所以同步等待IO事件的完成是一件很消耗的操作。在此背景下,探索新的IO才有存在的意义

##	IO读写模型
###	1、BIO 同步阻塞IO
###	2、NIO 同步非阻塞IO
		每次发起的IO系统调用，在内核等待数据过程中可以立即返回。用户线程不会阻塞，实时性较好。
		不断地轮询内核，这将占用大量的CPU时间，效率低下
		在高并发应用场景下，同步非阻塞IO也是不可用的。一般Web服务器不使用这种IO模型。这种IO模型一般很少直接使用。在Java的实际开发中，也不会涉及这种IO模型
###	3、IO Multiplexing IO多路复用 
		注册IO事件
		注册IO处理事件
		注册定时轮询任务
		定时轮询IO事件,并且处理就绪的IO事件
###	4、AIO 异步IO （目前只有Windows IIS实现了AIO）

#	Java NIO 
##	使用IO多路复用模型实现
##	核心对象 buffer selector channel
###	线程回调 Runable -> Callable<T>
###	桥接 FutureTask<T>实现了 -> Future接口 用于执行callable任务 并且得到返回值
	(1)判断并发任务是否执行完成。
	(2)获取并发的任务完成后的结果。
	(3)取消并发执行中的任务。
####	FutureTask 主要方法 run() get() isDone() ....... 泛型类 T为线程执行的返回结果

###	buffer是一个抽象类 一共有八种buffer子类 继承自buffer 分别为基本数据类型(不包括Boolean)和指向对象内存的MappedByteBuffer
		ByteBuffer
		CharBuffer
		DoubleBuffer
		FloatBuffer
		IntBuffer
		LongBuffer
		ShortBuffer
		MappedByteBuffer

#	Guava Java扩展包 支持异步回调 监听线程执行结果
	和reactor反应器模式有着异曲同工的地方
	FutureCallback 主要方法 onSuccess onFailure 支持添加监听
	ListenableFuture -> Future 方法 addListener()

#	Reactor线程模型 与IO多路复用
##	应用:Nginx Redis netty
##	网络服务程序发展史
	1>OIO while(true)
	2>OIO One Connection PerThread
	3>Reactor
##	核心对象
##	反应器模式由Reactor反应器线程、Handlers处理器两大角色组成：
	(1)Reactor反应器线程的职责：负责响应IO事件，并且分发到Handlers处理器。
	(2)Handlers处理器的职责：非阻塞的执行业务处理逻辑。
##	Reactor单线程模型
##	Reactor多线程模型
##	主从 Reactor多线程模型
##	发布订阅者模式
	订阅者对TOPIC有依赖关系
	同一个主题下的消息可以被多个订阅者订阅
	所以不同订阅者重复订阅 被订阅者订阅多次
##	生产消费者模式
	多个生产者与多个消费者通过队列连接起来
	只消费一次
##	reactor反应器模式
	handler与selectKey IO事件为一对一

#	netty框架相关

##	EventLoopGroup
	boss EventLoopGroup负责新连接的监听和接受，
	work EventLoopGroup负责IO事件处理

##	Bootstrap启动器类
	-	server端 ServerBootstrap
	-	client端 Bootstrap
	可类比 SpringApplication
	Bootstrap 意思是引导,一个 Netty 应用通常由一个 Bootstrap 开始,主要作用是配置整个 Netty 程序,串联各个组件,Netty 中 Bootstrap 类是客户端程序的启动引导类,ServerBootstrap 是服务端启动引导类。

###	父子通道 parent用于处理NioServerSocketChannel child用于处理 NioSocketChannel
###	基于 ServerBootstrap(服务端启动引导类)	启动流程
	!!这个地方需要重点说明一下。其实bind、connect、read、write、close等事件是由传输层socket定义的函数,并不是由netty定义的
####	1、创建父子反应器线程组,赋值给启动器类 
			boosGroup 用于 Accetpt 连接建立事件并分发请求,workerGroup 用于处理 I/O 读写事件和业务逻辑
			EventLoopGroup boosGroup = new NioEventLoopGroup(config.getBossLoopGroupThreads());
        	EventLoopGroup workerGroup = new NioEventLoopGroup(config.getWorkerLoopGroupThreads());
        	ServerBootstrap bootstrap = new ServerBootstrap();
        	bootstrap.group(boosGroup, workerGroup);
####	2、设置IO通道channel类型
			bootstrap.channel(NioServerSocketChannel.class) //这里设置的父通道
			实际上通道是成对存在的,会有父子通道分别用来处理客户端和服务端的请求
####	3、设置监听端口
			b.localAddress(new InetSocketAddress(port));
####	4、设置channel配置选项。可设置父子通道配置选项。分别为option、childOption方法
			bootstrap.option(ChannelOption.SO_BACKLOG, config.getSoBacklog())
			bootstrap.childOption(ChannelOption.SO_KEEPALIVE, config.isSoKeepalive())
####	5、创建pipeline,配置handler处理器
			pipeline为有序双向链表
			new ChannelInitializer<Channel>
####	6、开启绑定服务器新连接的监听端口
			服务端 b.bind().sync();
			3、6步骤可以合并成一步 bootstrap.bind(new InetSocketAddress(InetAddress.getByName(config.getHost()), config.getPort())).sync;
			sync 表示阻塞当前线程,直到处理完成。
			客户端建连连接 bootstrap.connect(host,port).sync()方法 host port 为要连接的父通道地址
####	7、阻塞,直到服务结束	
			可以采取任意阻塞手段,
			spring托管
			甚至while(true)
			channel.closeFuture().sync()
####	8、进程结束时关闭反应器线程组 
			workerLoopGroup.shutdownGracefully() bossLoopGroup.shutdownGracefully()


##	channel 负责处理网络连接
	每一个channel都代表一个网络连接,由它负责同对端进行网络通信，可以写入数据到对端，也可以从对端读取数据。
	Netty中不直接使用Java NIO的Channel通道组件，对Channel通道组件进行了自己的封装。
		1 、NioSocketChannel：异步非阻塞TCP Socket传输通道。客户端使用
		2 、NioServerSocketChannel：异步非阻塞TCP Socket服务器端监听通道。
		3 、NioDatagramChannel：异步非阻塞的UDP传输通道。
		4 、NioSctpChannel：异步非阻塞Sctp传输通道。
		5 、NioSctpServerChannel：异步非阻塞Sctp服务器端监听通道。
		6 、OioSocketChannel：同步阻塞式TCP Socket传输通道。
		7 、OioServerSocketChannel：同步阻塞式TCP Socket服务器端监听通道。
		8 、OioDatagramChannel：同步阻塞式UDP传输通道。
		9 、OioSctpChannel：同步阻塞式Sctp传输通道。
		10、OioSctpServerChannel：同步阻塞式Sctp服务器端监听通道。
###	Channel接口为网络连接主接口 网络层实现了http、webSocket等多种处理机制
		1、当前网络连接的通道的状态(例如是否打开？是否已连接？)
		2、网络连接的配置参数 (例如接收缓冲区大小)
		3、提供异步的网络 I/O 操作(如建立连接,读写,绑定端口),异步调用意味着任何 I/O 调用都将立即返回,并且不保证在调用结束时所请求的 I/O 操作已完成。
		4、调用立即返回一个 ChannelFuture 实例,通过注册监听器到 ChannelFuture 上,可以 I/O 操作成功、失败或取消时回调通知调用方。
		5、支持关联 I/O 操作与对应的处理程序。
###	id
	唯一标识
###	parent 
	父通道
	ServerSocketChannel此属性为空
	SocketChannel 为父通道引用
###	pipeline
	流水线。每一个channel都会拥有自己的流水线。
###	connect()
	SocketChannel 连接 server 使用
###	bind()
	ServerSocketChannel 绑定端口
	服务端使用
###	close()
	channel关闭事件。关闭通道连接
###	read()
	读取通道数据,并且启动入站处理handler,启动pipeline
	调用ChannelFuture异步任务的sync( )方法来阻塞当前线程，一直等到通道关闭的异步任务执行完毕

###	write()
	启程将数据写入到通道中。返回值为出站的异步任务Future。
	write()为异步任务,并不能立即写出到对端,仅仅是将数据放入通道缓冲区
###	flush()
	将缓冲区数据立即写出到对端
###	writeAndFlush()
	将数据写入到通道并立即写出

###	channel close disconnect deregister 方法对比
	close销毁实例
	disconnect只能销毁连接成功的实例
	deregister

###	EmbeddedChannel 测试类

##	Handler
	在Reactor反应器经典模型中，反应器查询到IO事件后，分发到Handler业务处理器，由Handler完成IO操作和业务处理。
	整个的IO处理操作环节包括:
###	入站ChannelInboundHandler:自底向上 从通道读数据包、数据包解码、业务处理
####	业务处理逻辑	SimpleChannelInboundHandler类
#####	channelRead0
#####	channelActive
#####	channelInactive
#####	channelReadComplete
###出站ChannelOutboundHandler:自顶向下 目标数据编码、把数据包写到通道，然后由通道发送到对端



###	channelRead0重写问题

## 流水线 pipeline 
	pipeline 将handler 整合在一起 用于处理channel事件 
	在pipeline中 handler 对IO的处理有顺序定义 实际设计为一个双向链表
	工作流程 
		入站 解码(ByteBuf) -decoder将入站的ByteBuf转成netty对象-> Polo(FullHttpRequest) --> ProtoBuf/Json
		出站
##	Future
	继承并且扩展了Java Future对象
	netty中Future添加了addListener、removeListener方式 支持添加对结果监听的处理
	在Netty的网络编程中，网络连接通道的输入和输出处理都是异步进行的，都会返回一个ChannelFuture接口的实例。可以通过ChannelFuture添加异步回调的监听器
	Future.syncUninterruptibly 不会被中断的sync
	Future.sync 当前线程等待直接Future执行完毕。但是如果Future执行失败则会抛出失败原因

###	ChannelFuture、Future 异步通知和监听机制
	在 Netty 中所有的 IO 操作都是异步的,不能立刻得知消息是否被正确处理。但是可以过一会等它执行完成或者直接注册一个监听,具体的实现就是通过 Future 和 ChannelFutures,他们可以注册一个监听,当操作执行成功或失败时监听会自动触发注册的监听事件。
	常见有如下操作:
	1、通过 isDone 方法来判断当前操作是否完成;
	2、通过 isSuccess 方法来判断已完成的当前操作是否成功;
	3、通过 getCause 方法来获取已完成的当前操作失败的原因;
	4、通过 isCancelled 方法来判断已完成的当前操作是否被取消;
	5、通过 addListener 方法来注册监听器,当操作已完成(isDone 方法返回完成),将会通知指定的监听器;如果 Future 对象已完成(无论是否成功),则理解通知指定的监听器。可设置超时处理逻辑
	6、通过sync等操作阻塞和调度NIO(高风险慎用,精通线程调度大神自动忽略)

###	ChannelPromise是一种可写的特殊 ChannelFuture 继承自Future
    ChannelPromise setSuccess(Void var1);
    ChannelPromise setSuccess();
    boolean trySuccess();
    ChannelPromise setFailure(Throwable var1);

##	selector
	Netty 基于 Selector 对象实现 I/O 多路复用,通过 Selector 一个线程可以监听多个连接的 Channel 事件。
	当向一个 Selector 中注册 Channel 后,Selector 内部的机制就可以自动不断地查询(Select) 这些注册的 Channel 是否有已就绪的 I/O 事件(例如可读,可写,网络连接完成等),这样程序就可以很简单地使用一个线程高效地管理多个 Channel 。

##	ChannelHandlerContext
	保存 Channel 相关的所有上下文信息,同时关联一个 ChannelHandler 对象。

##	ByteBuf二进制流

##	Decoder与Encoder解码器
	将TCP二进制传输 转化为Java Polo。这一层定义的是通道channel read/write 的数据编码格式。
	例如在网络编程中我们经常用到的HttpServerCodec

##	json与protobuf编解码
	涉及对象的序列化/反序列化的问题，例如一个对象从客户端通过TCP方式发送到服务器端；因为TCP协议（UDP等这种低层协议）只能发送字节流
	所以需要应用层将Java POJO对象序列化成字节流，数据接收端再反序列化成Java POJO对象。
	注意与HttpServerCodec编解码定义的区别。 这里定义的是httpRequest数据包的编码格式。
###	httpRequest content->json/protobuf
	tcp传输层传输数据为二进制包,需要将二进制包解码成Java Polo。

	在这里简单说个理解。netty不是个网络框架,是一个IO框架。它不仅可以处理TCP连接,更可以通过自定义编解码协议,处理和传输层相关的IO事件。
###	半包、粘包问题

#netty WebSocketFrame
##BinaryWebSocketFrame 发送二进制消息
##CloseWebSocketFrame 关闭
##ContinuationWebSocketFrame
##PingWebSocketFrame ping
##PongWebSocketFrame pong
##TextWebSocketFrame 发送文本消息

##	设置cookie
	res.headers().set(HttpHeaderNames.SET_COOKIE, new DefaultCookie("IP", "123456"));
##	获取cookie
	HttpRequest request = (HttpRequest) e.getMessage();
	String value = request.getHeader("Cookie");
	System.out.println(value);
##	netty没有实现session 需要自己实现
##  跨域问题解决
    //允许携带的header中属性 支持 * 通配符 ajax cookie跨域访问中 * 无法支持 需要显示指定可以携带哪些header信息
    res.headers().set(HttpHeaderNames.ACCESS_CONTROL_ALLOW_HEADERS, "Content-Type,X-Requested-With");
    //是否允许携带cookie
    res.headers().set(HttpHeaderNames.ACCESS_CONTROL_ALLOW_CREDENTIALS, "true");
    //支持的跨域名访问 ajax cookie跨域访问中 * 无法支持 ORIGIN 属性只能设置 一个 需要服务端手动配置 白名单匹配
    res.headers().set(HttpHeaderNames.ACCESS_CONTROL_ALLOW_ORIGIN, "*");
