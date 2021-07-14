#	线程
##	三种线程实现方式
	根据操作系统内核是否对线程可感知,可以把线程分为内核线程和用户线程。目前系统内核对内核线程做了轻量级封装后暴露给用户程序,称为轻量级进程
	1、使用内核线程实现。
		Windows与Linux JDK 实现的线程模型 用户线程 与内核线程 1:1。程序线程实现依赖内核线程实现。
	2、使用用户线程实现。
		用户线程指的是完全建立在用户空间的线程库上,系统内核不能感知线程存在的实现。用户程序自己封装了线程的实现
	3、使用用户线程和轻量级进程混合实现。
		由CPU封装轻量级进程（内核线程暴露接口）,然后用户进程再一次进行封装,对应多个用户线程。

##	线程分配方式
	1、抢占式 目前Linux、windows等操作系统对线程的分配方式为抢占式
	2、调度式。由系统内核统一统一线程调起、释放

#	程序IO与内核IO
	Linux定义了一切皆文件。然而在计算机程序中,我们可以毫不客气地说一切皆IO。无论是传输、交互、读写、还是处理。
	上层程序的read write 方法本质上是内存在缓冲区的复制。而真正内核缓冲区与物理磁盘之间的内存交互上层应用程序并不感知,也不关心。
		内存read:物理磁盘->内核缓冲区->程序缓冲区
		内存write:程序缓冲区->内核缓冲区->物理磁盘
	所以同步等待IO事件的完成是一件很消耗的操作。在此背景下,探索新的IO方式才有存在的意义

##	IO读写模型
	目前有四种IO。
###	1、BIO 同步阻塞IO
	传统同步阻塞IO,不过多介绍
###	2、NIO 同步非阻塞IO
	每次发起的IO系统调用,在内核等待数据过程中可以立即返回。用户线程不会阻塞,实时性较好。
	不断地轮询内核,这将占用大量的CPU时间,效率低下
	在高并发应用场景下,同步非阻塞IO也是不可用的。一般Web服务器不使用这种IO模型。这种IO模型一般很少直接使用。在Java的实际开发中,也不会涉及这种IO模型
###	3、IO Multiplexing IO多路复用 
	由统一的选择器轮询IO事件,将就绪状态的事件分发给不同的事件处理器。依赖系统内核提供的接口实现。
	JAVA NIO库、Guava、reactor反应器都是基于IO多路复用模型设计。是重点介绍和掌握的模型。
		注册IO事件
		注册IO处理事件
		注册定时轮询任务
		定时轮询IO事件,并且处理就绪的IO事件
###	4、AIO 异步IO 
	真正的异步IO,由系统调起应用程序（目前只有Windows IIS实现了AIO）也不多做介绍

#	Java NIO 
	Java NIO不是传统意义上的NIO。指的是 Java new IO,对应的是 Java OIO(old IO)。
	同样使用IO多路复用模型实现。设计思路大致相同,这里只做简单介绍。
##	核心对象 buffer selector channel
##	使用
	 
###	线程回调 Runable -> Callable<T>
###	桥接 FutureTask<T>
	实现了 -> Future接口 用于执行callable任务 并且得到返回值
	(1)判断并发任务是否执行完成。
	(2)获取并发的任务完成后的结果。
	(3)取消并发执行中的任务。
####	FutureTask 
	主要方法 run() get() isDone() ....... 泛型类 T为线程执行的返回结果
	用于监听和返回线程执行结果

###	buffer
	是一个抽象类 一共有八种buffer子类 继承自buffer 分别为基本数据类型(不包括Boolean)和指向对象内存的MappedByteBuffer
		ByteBuffer
		CharBuffer
		DoubleBuffer
		FloatBuffer
		IntBuffer
		LongBuffer
		ShortBuffer
		MappedByteBuffer

#	Guava Java扩展包 
	支持异步回调 监听线程执行结果。同样简单了解。
	和reactor反应器模式有着异曲同工的地方
	FutureCallback 主要方法 onSuccess onFailure 支持添加监听
	ListenableFuture -> Future 方法 addListener()

#	Reactor反应器模式 
##	应用:Nginx Redis netty
##	网络服务程序发展史
	1>OIO while(true)
		阻塞进程、轮询是否有请求发生
	2>OIO One Connection PerThread
		每一个线程处理一个请求。
	3>Reactor
##	核心对象
	反应器模式由Reactor反应器线程、Handlers处理器两大角色组成：
		(1)Reactor反应器线程的职责：负责响应IO事件,并且分发到Handlers处理器。
		(2)Handlers处理器的职责：非阻塞的执行业务处理逻辑。
##	Reactor单线程模型

##	Reactor多线程模型
##	主从 Reactor多线程模型
##	比较reactor 与 生产消费者模式、发布订阅者模式
	发布订阅者模式
		订阅者对TOPIC有依赖关系
		同一个主题下的消息可以被多个订阅者订阅
		所以不同订阅者重复订阅 被订阅者订阅多次
	生产消费者模式
		多个生产者与多个消费者通过队列连接起来
		只消费一次
	reactor反应器模式
		handler与selectKey IO事件为一对一
		读写完全解耦

#	netty框架相关
	netty基于reactor反应器模式设计。
	netty是一个IO框架,并不是一个网络框架。可进行TCP/UDP传输层IO,可进行进程间IO。
	以下是重要对象
##	EventLoopGroup
	服务端反应器需要两个反应器线程组。
		boss EventLoopGroup 负责新连接的监听和接受
		work EventLoopGroup 负责IO事件处理

##	Bootstrap启动器类
	* server端 ServerBootstrap
	* client端 Bootstrap
	启动引导类 可类比 SpringApplication
	一个 Netty 应用通常由一个 Bootstrap 开始,主要作用是配置整个 Netty 程序,串联各个组件,Netty 中 Bootstrap 类是客户端程序的启动引导类,ServerBootstrap 是服务端启动引导类。

###	父子通道 parent用于处理NioServerSocketChannel child用于处理 NioSocketChannel
###	基于 ServerBootstrap(服务端启动引导类)	启动流程
	!!这个地方需要重点说明一下。其实bind、connect、read、write、close等连接事件是由传输层socket定义的协议,并不是由netty定义的。
	实现一个传输层框架必须要实现这些接口。可类比httpClient等
	
####	1、创建父子反应器线程组,赋值给启动器类 
			boosGroup 用于 Accetpt 连接建立事件并分发请求,workerGroup 用于处理 I/O 读写事件和业务逻辑
```java
			EventLoopGroup boosGroup = new NioEventLoopGroup(config.getBossLoopGroupThreads());
        	EventLoopGroup workerGroup = new NioEventLoopGroup(config.getWorkerLoopGroupThreads());
        	ServerBootstrap bootstrap = new ServerBootstrap();
        	bootstrap.group(boosGroup, workerGroup);
```
####	2、设置IO通道channel类型
			bootstrap.channel(NioServerSocketChannel.class) //这里设置的父通道
			实际上通道是成对存在的,会有父子通道分别用来处理客户端和服务端的请求
####	3、设置监听端口
```java
			b.localAddress(new InetSocketAddress(port));
```
####	4、设置channel配置选项。可设置父子通道配置选项。分别为option、childOption方法
```java
			bootstrap.option(ChannelOption.SO_BACKLOG, config.getSoBacklog())
			bootstrap.childOption(ChannelOption.SO_KEEPALIVE, config.isSoKeepalive())
```
####	5、创建pipeline,配置handler处理器
			pipeline为有序双向链表
			new ChannelInitializer<Channel>
####	6、开启绑定服务器新连接的监听端口
			服务端 b.bind().sync();
			
```java
		//3、6步骤可以合并成一步 
		bootstrap.bind(new InetSocketAddress(InetAddress.getByName(config.getHost()), config.getPort())).sync;
```
			sync 表示阻塞当前线程,直到处理完成。
			客户端建连连接 bootstrap.connect(host,port).sync()方法 host port 为要连接的父通道地址
####	7、阻塞,直到服务结束	
			可以采取任意阻塞手段,
			* spring托管
			* 甚至while(true)
			* Thread.sleep(Integer.MAX_SIZE)
			* channel.closeFuture().sync()
####	8、进程结束时关闭反应器线程组 
```java
			boss.shutdownGracefully().syncUninterruptibly();
            worker.shutdownGracefully().syncUninterruptibly();
```
##	selector
	Netty 基于 Selector 对象实现 I/O 多路复用,通过 selector 一个线程可以监听多个连接的 Channel 事件。
	当向一个 Selector 中注册 Channel 后,Selector 内部的机制就可以自动不断地查询(Select) 这些注册的 Channel 是否有已就绪的 I/O 事件(例如可读,可写,网络连接完成等),这样程序就可以很简单地使用一个线程高效地管理多个 Channel 。

##	channel 负责处理网络连接
	每一个channel都代表一个网络连接,由它负责同对端进行网络通信,可以写入数据到对端,也可以从对端读取数据。
	Netty中不直接使用Java NIO的Channel通道组件,对Channel通道组件进行了自己的封装。
		*1 	NioSocketChannel：异步非阻塞TCP Socket传输通道。客户端使用
		*2 	NioServerSocketChannel：异步非阻塞TCP Socket服务器端监听通道。
		*3 	NioDatagramChannel：异步非阻塞的UDP传输通道。
		*4 	NioSctpChannel：异步非阻塞Sctp传输通道。
		*5 	NioSctpServerChannel：异步非阻塞Sctp服务器端监听通道。
		*6 	OioSocketChannel：同步阻塞式TCP Socket传输通道。
		*7 	OioServerSocketChannel：同步阻塞式TCP Socket服务器端监听通道。
		*8 	OioDatagramChannel：同步阻塞式UDP传输通道。
		*9 	OioSctpChannel：同步阻塞式Sctp传输通道。
		*10	OioSctpServerChannel：同步阻塞式Sctp服务器端监听通道。
###	Channel接口为网络连接主接口。
		*1	当前网络连接的通道的状态(例如是否打开？是否已连接？)
		*2	网络连接的配置参数 (例如接收缓冲区大小)
		*3	提供异步的网络 I/O 操作(如建立连接,读写,绑定端口),异步调用意味着任何 I/O 调用都将立即返回,并且不保证在调用结束时所请求的 I/O 操作已完成。
		*4	调用立即返回一个 ChannelFuture 实例,通过注册监听器到 ChannelFuture 上,可以 I/O 操作成功、失败或取消时回调通知调用方。
		*5	支持关联 I/O 操作与对应的处理程序。
###	id
	唯一标识
###	parent 
	父通道
	ServerSocketChannel 此属性为空。本身就是一个parent 
	SocketChannel 为父通道引用,也就是对应的 ServerSocketChannel
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
	调用ChannelFuture异步任务的sync( )方法来阻塞当前线程,一直等到通道关闭的异步任务执行完毕

###	write()
	启程将数据写入到通道中。返回值为出站的异步任务Future。
	write()为异步任务,并不能立即写出到对端,仅仅是将数据放入通道缓冲区
###	flush()
	将缓冲区数据立即写出到对端
###	writeAndFlush()
	将数据写入到通道并立即写出

###	channel close disconnect deregister 方法对比
	close 		销毁实例。推荐使用
	disconnect  只能销毁连接成功的实例
	deregister	注销

###	EmbeddedChannel 测试类
	可以在测试类中启动一条流水线
```java 
	public void testPipelineInBound() {
        ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
            protected void initChannel(EmbeddedChannel ch) {
                ch.pipeline().addLast(new SimpleInHandlerA());
                ch.pipeline().addLast(new SimpleInHandlerB());
                ch.pipeline().addLast(new SimpleInHandlerC());

            }
        };
        EmbeddedChannel channel = new EmbeddedChannel(i);
        ByteBuf buf = Unpooled.buffer();
        buf.writeInt(1);
        //向通道写一个入站报文
        channel.writeInbound(buf);
        try {
            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```
##	Handler 控制器
	在Reactor反应器经典模型中,反应器查询到IO事件后,分发到Handler业务处理器,由Handler完成IO操作和业务处理。
	handler对IO的处理与channel处理对应,以达到解耦channel与IO结果的目的。
	整个的IO处理操作环节包括:
###	入站ChannelInboundHandler:自底向上 从通道读数据包、数据包解码、业务处理
	需要注意的是,ChannelInboundHandler有很多个子类,在实际开发和生产后遇到的可能并不是这几个方法
		channelRegistered 		通道注册事件 通道注册完成后调用
		channelActive			通道激活事件 通道激活完成后调用
		channelRead				通道缓冲区可读 触发通道可读事件
		channelReadComplete		当通道缓冲区读完,Netty会触发通道读完事件
		channelInactive			当连接被断开或者不可用,Netty会触发连接不可用事件
		exceptionCaught			当通道处理过程中遇到异常,触发异常捕获事件
####	生命周期
	1、	handlerAdded() 			属于通道生命周期回调,当业务处理器被加到pipeline中被回调
	2、	channelRegistered() 	属于通道生命周期回调,当通道成功绑定NioEventLoop后
	3、	channelActive() 		属于通道生命周期回调,激活完成后回调:指的是所有的业务处理器添加、注册的异步任务完成,并且NioEventLoop线程绑定的异步任务完成。
	4、	channelRead() 			属于通道IO回调
	5、	channelReadComplete() 	属于通道IO回调
	6、	channelRead() 			属于通道IO回调
	7、	channelReadComplete() 	属于通道IO回调
	8、	channelInactive() 		属于通道生命周期回调,当通道的底层连接已经不是ESTABLISH状态,或者底层连接已经关闭时
	9、	channelUnregistered() 	属于通道生命周期回调,通道和NioEventLoop线程解除绑定,移除掉对这条通道的事件处理之后,回调所有业务处理器该方法
	10、	handlerRemoved() 		属于通道生命周期回调,Netty会移除掉通道上所有的业务处理器,并且回调所有的业务处理器
	
###出站ChannelOutboundHandler:自顶向下 目标数据编码、把数据包写到通道,然后由通道发送到对端
	当业务处理完成时需要调用的出站操作。比如写入通道、建立、断开连接等等
	*	bind					监听地址(IP+PORT)用于服务端
	*	connect 				完成底层IO连接操作。用户客户端
	*	write 					完成向通道的数据写入操作。仅仅是将数据写入通道缓冲区。并不是完成实际的数据写入操作
	*	flush 					刷新缓冲区数据,将缓冲区数据写入对端
	*	read 					从底层读取数据,从IO通道中读取数据
	*	disConnect 				断开服务器连接,此方法主要用于客户端,例如断开TCP连接
	*	close 					关闭底层通道

###	channelRead重写问题
	ChannelInboundHandler中 不调用基类channelRead方法会导致流水线截断。

##	ChannelHandlerContext
	保存 Channel 相关的所有上下文信息,同时关联一个 ChannelHandler 对象。表示handler和pipeline之间的关联关系
###	功能
	1、获取上下文所关联的Netty组件实例,如所关联的通道、所关联的流水线、上下文内部Handler业务处理器实例
	2、进行入站、出站IO操作。
###	方法 和channel handler 定义方法基本一致 具体使用不是很理解

	channel() 					获取channel
	fireChannelRegistered()
	fireChannelUnregistered()
	fireChannelActive()
	fireChannelInactive()
	fireExceptionCaught(Throwable cause)
	fireUserEventTriggered(Object evt)
	fireChannelRead(Object msg)
	fireChannelReadComplete()
	flush()
	pipeline()					获取pipeline
	attr(AttributeKey<T> key) 	获取属性

##	ChannelHandlerContext、channel、handler
	Channel、Handler、ChannelHandlerContext三者的关系为：Channel通道拥有一条ChannelPipeline通道流水线,
	每一个流水线节点为一个ChannelHandlerContext通道处理器上下文对象,每一个上下文中包裹了一个ChannelHandler通道处理器。
	在ChannelHandler通道处理器的入站/出站处理方法中,Netty都会传递一个Context上下文实例作为实际参数。
	通过Context实例的实参,在业务处理中,可以获取ChannelPipeline通道流水线的实例或者Channel通道的实例。
###	三者读写IO上的区别
	IO操作(write、read)可以由channel 发起、可以由pipeline发起,可以由ChannelHandlerContext发起,也可以由handler发起。
	通过Channel或ChannelPipeline的实例来调用这些方法,它们就会在整条流水线中传播。
	然而,如果是通过ChannelHandlerContext通道处理器上下文进行调用,就只会从当前的节点开始执行Handler业务处理器,并传播到同类型处理器的下一站（节点）
	*上面这段话不是很理解,看起来是影响范围不同。之后具体测试一下吧
## 流水线 pipeline ChannelInitializer
	pipeline 将handler 整合在一起 用于处理channel事件 
	在pipeline中 handler 对IO的处理有顺序定义 实际设计为一个双向链表
	ChannelInitializer用于创建一条流水线
```java
	new ChannelInitializer<NioSocketChannel>() {
       @Override
       protected void initChannel(NioSocketChannel ch) {
           ChannelPipeline pipeline = ch.pipeline();
           if (sslCtx != null) {
               pipeline.addFirst(sslCtx.newHandler(ch.alloc()));
           }
           //编解码控制器,之后会详细说明netty编解码。解决拆包等问题
           pipeline.addLast(new HttpServerCodec());
           pipeline.addLast(new HttpObjectAggregator(65536));
           //跨域问题解决.netty作为一个IO框架,帮助我们解决了大部分应用层和传输层的协议控制
           pipeline.addLast(new CorsHandler(corsConfig));
           //空闲回收。解决连接的假死问题
           pipeline.addLast(new IdleStateHandler(config.getHttpReaderIdleTimeSeconds(), config.getHttpWriterIdleTimeSeconds(), config.getHttpAllIdleTimeSeconds()));
           pipeline.addLast(new HttpServerHandler(pojoEndpointServer, config, finalEventExecutorGroup));

       }
   });
```
	工作流程 
		入站 正序执行 
			1、	TCP传输层 :解码(ByteBuf) -decoder将入站的ByteBuf转成netty对象-> Polo(FullHttpRequest) 
			2、	HTTP网络层:基于request对象content解析(ProtoBuf/Json)
			3、	业务处理器A->B->C
###	截断流水线
	channelRead()没有调用super.channelRead() 流水线将不会再像后传递
###	热插拔
	动态地增加、删除流水线上的业务处理器Handler。包括但不仅限于以下方法
```java
	addFirst(EventExecutorGroup group, ChannelHandler... handlers)
	addLast(ChannelHandler... handlers)
	remove(ChannelHandler handler)
	remove(String name)
```
##	Future
	继承并且扩展了Java Future对象
	netty中Future添加了addListener、removeListener方式 支持添加对结果监听的处理
	在Netty的网络编程中,网络连接通道的输入和输出处理都是异步进行的,都会返回一个ChannelFuture接口的实例。可以通过ChannelFuture添加异步回调的监听器
	* Future.syncUninterruptibly 	不会被中断的sync
	* Future.sync 					当前线程等待直接Future执行完毕。但是如果Future执行失败则会抛出失败原因
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
##	ByteBuf缓冲区
	ByteBuf编程中,时刻注意内存的释放,注意内存耗尽的问题。byteBuf.release()
###	与Java NIO的ByteBuffer相比,ByteBuf的优势
	* Pooling（池化,这点减少了内存复制和GC,提升了效率）之后会详细说明
	* 复合缓冲区类型,支持零复制
	* 不需要调用flip()方法去切换读/写模式
	* 扩展性好,例如StringBuffer
	* 可以自定义缓冲区类型
	* 读取和写入索引分开
	* 方法的链式调用
	* 可以进行引用计数,方便重复使用
###结构
	ByteBuf是一个字节容器,内部是一个字节数组。从逻辑上来分,字节容器内部可以分为四个部分。四部分为连续的空间,由指针区分
	读写分离,指针互不干扰。
	一、已废弃:已用字节,表示已经使用完成的无效字节
	二、可读:保存的有效数据,所有读取的数据来自这一部分
	三、可写:读写BUF索引是分开的 IO写入的byte都会写到这一部分
	四、可扩容字节:表示该ByteBuf还能扩容多大的空间

###	重要属性
	* readerIndex
		指示读取的起始位置。每读取一个字节,readerIndex自动增加1。一旦readerIndex与writerIndex相等,则表示ByteBuf不可读了。
	* writerIndex
		指示写入的起始位置。每写一个字节,writerIndex自动增加1。一旦增加到writerIndex与capacity()容量相等,则表示ByteBuf已经不可写了。
		capacity()是一个成员方法,不是一个成员属性,它表示ByteBuf中可以写入的容量。注意,它不是最大容量maxCapacity
	* maxCapacity
		表示ByteBuf可以扩容的最大容量。当向ByteBuf写数据的时候,如果容量不足,可以进行扩容。扩容的最大限度由maxCapacity的值来设定,超过maxCapacity就会报错。
###	重要方法
####	容量相关
			capacity()
			maxCapacity()
####	写入相关
			isWritable()			是否可读
			writableBytes()	
			maxWritableBytes()
			writeBytes(byte[] src)	
			write**(TYPE value）		**表示基本数据类型 写入基础数据类型数据
				writeByte()、writeBoolean()、writeChar()、writeShort()、writeInt()、writeLong()、writeFloat()、writeDouble()
			set**(TYPE value）		**表示基本数据类型 不改变writerIndex指针值，包含了8大基础数据类型的设置。

####	读取相关
			isReadable()
			readableBytes()				返回表示ByteBuf当前可读取的字节数，它的值等于writerIndex减去readerIndex
			readBytes(byte[] dst)		readBoolean()、readChar()、readShort()、readInt()、readLong()、readFloat()、readDouble()
			markReaderIndex() resetReaderIndex()
				前一个方法表示把当前的读指针ReaderIndex保存在markedReaderIndex属性中。
				后一个方法表示把保存在markedReaderIndex属性的值恢复到读指针ReaderIndex中。markedReaderIndex属性定义在AbstractByteBuf抽象基类中。

###	内存回收与引用计数
	ByteBuf的内存回收工作是通过引用计数的方式管理的
	retain()	引用计数+1
	release() 	引用计数-1	
		在Netty的业务处理器开发过程中，应该坚持一个原则：retain和release方法应该结对使用。，在一个方法中，调用了retain，就应该调用一次release。
	当引用计数已经为0，Netty会进行ByteBuf的回收。分为两种情况：
	（1）Pooled池化的ByteBuf内存，回收方法是：放入可以重新分配的ByteBuf池子，等待下一次分配。
	（2）Unpooled未池化的ByteBuf缓冲区，回收分为两种情况：如果是堆（Heap）结构缓冲，会被JVM的垃圾回收机制回收；如果是Direct类型，调用本地方法释放外部内存（unsafe.freeMemory）

###	Allocator分配器
	前面说到ByteBuf有pool池化的特性。池化的ByteBuf可以避免重复创建和销毁ByteBuf对象。提升内存空间利用效率。
	* Netty通过ByteBufAllocator分配器来创建缓冲区和分配内存空间。Netty提供了ByteBufAllocator的两种实现：PoolByteBufAllocator和UnpooledByteBufAllocator。
	* 可以通过Java系统参数（System Property）的选项io.netty.allocator.type进行配置，配置时使用字符串值："unpooled"，"pooled"。
	* 在Netty程序中设置启动器Bootstrap的时候，将PooledByteBufAllocator设置为默认的分配器。代码如下所示
```java
	bootstrap.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
```
	* 目前netty4.1之后默认分配器为PooledByteBufAllocator。代码如下所示
```java
		ByteBufAllocator alloc;
        if ("unpooled".equals(allocType)) {
            alloc = UnpooledByteBufAllocator.DEFAULT;
            logger.debug("-Dio.netty.allocator.type: {}", allocType);
        } else if ("pooled".equals(allocType)) {
            alloc = PooledByteBufAllocator.DEFAULT;
            logger.debug("-Dio.netty.allocator.type: {}", allocType);
        } else {
            alloc = PooledByteBufAllocator.DEFAULT;
            logger.debug("-Dio.netty.allocator.type: pooled (unknown: {})", allocType);
        }
        DEFAULT_ALLOCATOR = alloc;
```

####	Jemalloc内存管理库
	//todo待补充
	在底层，Netty为我们干了所有“脏活、累活”！这主要是因为Netty用到了Java的Jemalloc内存管理库。有兴趣的小伙伴可以了解一下

###	ByteBuf缓冲区的类型
	根据内存的管理方不同，分为堆缓存区和直接缓存区，也就是HeapByteBuf和DirectByteBuf。另外，为了方便缓冲区进行组合，提供了一种组合缓存区。
	三种缓冲区的类型，无论哪一种，都可以通过池化（Pooled）、非池化（Unpooled）两种分配器来创建和分配内存空间。如下所示
####	HeapByteBuf 内存分配在Java heap上
```java
		ByteBuf heapBuf =  ByteBufAllocator.DEFAULT.buffer();//池化
		ByteBuf heapBuf = Unpooled.buffer();//非池化
```
####	DirectByteBuf 内存分配在直接内存中
	  		* Direct Memory不属于Java堆内存，所分配的内存其实是调用操作系统malloc()函数来获得的；由Netty的本地内存堆Native堆进行管理。
			* Direct Memory容量可通过-XX:MaxDirectMemorySize来指定，如果不指定，则默认与Java堆的最大值（-Xmx指定）一样。注意：并不是强制要求，有的JVM默认Direct Memory与-Xmx无直接关系。
			* Direct Memory的使用避免了Java堆和Native堆之间来回复制数据。在某些应用场景中提高了性能。
			* 在需要频繁创建缓冲区的场合，由于创建和销毁Direct Buffer（直接缓冲区）的代价比较高昂，因此不宜使用Direct Buffer。
			* DirectBuffer尽量在池化分配器中分配和回收。如果能将Direct Buffer进行复用，在读写频繁的情况下，就可以大幅度改善性能。
			* 对Direct Buffer的读写比Heap Buffer快，但是它的创建和销毁比普通Heap Buffer慢。
			* 在Java的垃圾回收机制回收Java堆时，Netty框架也会释放不再使用的Direct Buffer缓冲区，因为它的内存为堆外内存，所以清理的工作不会为Java虚拟机（JVM）带来压力。
			* 注意一下垃圾回收的应用场景：（1）垃圾回收仅在Java堆被填满，以至于无法为新的堆分配请求提供服务时发生；（2）在Java应用程序中调用System.gc()函数来释放内存”

```java
		ByteBuf directBuf =  ByteBufAllocator.DEFAULT.directBuffer();
		ByteBuf heapBuf = Unpooled.directBuffer();//非池化
```
####	CompositeBuf 组合类型BUF
```java
		CompositeByteBuf cbuf = ByteBufAllocator.DEFAULT.compositeBuffer();
		CompositeByteBuf cbuf = Unpooled.compositeBuffer(3);//非池化
		//消息头
        ByteBuf headerBuf = Unpooled.copiedBuffer("疯狂创客圈:", utf8);
        //消息体1
        ByteBuf bodyBuf = Unpooled.copiedBuffer("高性能 Netty", utf8);
        cbuf.addComponents(headerBuf, bodyBuf);
```

###	netty如何创建和释放ByteBuf实例
	在Netty开发中，必须密切关注Bytebuf缓冲区的释放，如果释放不及时，会造成Netty的内存泄露（Memory Leak），最终导致内存耗尽。
####	创建ByteBuf实例
####	入站释放ByteBuf实例
#####	TailHandler自动释放
	Netty默认会在ChannelPipline通道流水线的最后添加一个TailHandler末尾处理器，它实现了默认的处理方法，在这些方法中会帮助完成ByteBuf内存释放的工作。
	super.channelRead(ctx,msg);
	在需要截断流水线的情况下,需要考虑手动释放或者SimpleChannelInboundHandler

#####	SimpleChannelInboundHandler
	自动释放ByteBuf,业务处理需要调用channelRead0(ctx,msg)。实现请查看源码
```java
@Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        boolean release = true;
        try {
            if (acceptInboundMessage(msg)) {
                @SuppressWarnings("unchecked")
                I imsg = (I) msg;
                channelRead0(ctx, imsg);
            } else {
                release = false;
                ctx.fireChannelRead(msg);
            }
        } finally {
            if (autoRelease && release) {
                ReferenceCountUtil.release(msg);
            }
        }
    }

        public static boolean release(Object msg) {
        	//需要说明的是,FullHttpRequest、WebSocketFrame全部实现了ReferenceCounted接口
        if (msg instanceof ReferenceCounted) {
            return ((ReferenceCounted) msg).release();
        }
        return false;
    }
```
####	出站释放ByteBuf实例
#####	HeadHandler自动释放
	在出站处理流程中，申请分配到的ByteBuf主要是通过HeadHandler完成自动释放的。
	在write出站写入通道时，通过调用ctx.writeAndFlush(Bytebufmsg)，Bytebuf缓冲区进入出站处理的流水线。
	在出站流水线最后(head)会来到HeadHandler,在数据输出完成后ByteBuf被释放。如果计数器为零,彻底释放。(这里有个问题啊,如果不为零就不释放了???那岂不是也有问题)

##	Decoder与Encoder解码器
	Netty从底层Java通道读到ByteBuf二进制数据，传入Netty通道的流水线，随后开始入站处理。
	因为netty作为一个IO框架,所有收到的数据均为二进制数据。需要将IO二进制传输 转化为Java Polo。这一层定义的是通道channel的数据编码格式。
	编解码过程中需要解决半包、粘包、分包发送等问题。
	(在TCP中)ByteBuf可以直接转换成JSON或者其他JAVA POJO。并非必须和http扯上关系

###	HttpServerCodec	
	继承自CombinedChannelDuplexHandler。强制将编码器和解码器放在同一个控制器中。编解码器可以捆绑共同使用
	同时实现了Decoder与Encoder。实现了HttpMessage与ByteBuf 的转换
```java
        @Override
        protected void decode(ChannelHandlerContext ctx, ByteBuf buffer, List<Object> out) throws Exception {
            int oldSize = out.size();
            super.decode(ctx, buffer, out);
            int size = out.size();
            for (int i = oldSize; i < size; i++) {
                Object obj = out.get(i);
                if (obj instanceof HttpRequest) {
                    queue.add(((HttpRequest) obj).method());
                }
            }
        }
    }
```
###	WebSocketFrameAggregator

###	netty WebSocketFrame
	* BinaryWebSocketFrame 			发送二进制消息
	* TextWebSocketFrame 			发送文本消息
	* ContinuationWebSocketFrame	混合text\二进制 类型消息
	* PingWebSocketFrame 			ping
	* PongWebSocketFrame 			pong
	* CloseWebSocketFrame 			关闭

##	json与protobuf编解码
	涉及对象的序列化/反序列化的问题,例如一个对象从客户端通过TCP方式发送到服务器端；因为TCP协议（UDP等这种低层协议）只能发送字节流
	netty将字节流转化为对象的解析和反解析过程

###	半包、粘包问题
	数据传输过程中socket缓冲区大小是固定的,如果数据包过大或者过小,都会将一个数据包分成几个(或者几个包合并成一个)进行传输。
	解决的思路大致为
	netty的大多数编解码器为开发者解决了这个问题

#	实际问题解决
##	response设置cookie
```java	
	res.headers().set(HttpHeaderNames.SET_COOKIE, new DefaultCookie("IP", "123456"));
```

##	request获取cookie
```java
	HttpRequest request = (HttpRequest) e.getMessage();
	String value = request.getHeader("Cookie");
	System.out.println(value);
```
##	netty没有实现session 使用过程中如果需要绑定会话信息需要自己实现。可参考spring
##  跨域问题解决 这里简单记录下netty配置 详细原理和解决方案在Internet中记录
```java
    //允许携带的header中属性 支持 * 通配符 ajax cookie跨域访问中 * 无法支持 需要显示指定可以携带哪些header信息
    res.headers().set(HttpHeaderNames.ACCESS_CONTROL_ALLOW_HEADERS, "Content-Type,X-Requested-With");
    //是否允许携带cookie
    res.headers().set(HttpHeaderNames.ACCESS_CONTROL_ALLOW_CREDENTIALS, "true");
    //支持的跨域名访问 ajax cookie跨域访问中 * 无法支持 ORIGIN 属性只能设置 一个 需要服务端手动配置 白名单匹配
    res.headers().set(HttpHeaderNames.ACCESS_CONTROL_ALLOW_ORIGIN, "*");
```
##	即时通讯与webSocket
	即时通讯IM不一定要使用WS协议。TCP连接大多数情况下的性能更加优秀。但是传输层和应用层的问题需要自己解决。
	目前很多即时通讯客户端并没有使用WS协议