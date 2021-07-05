# 消息队列与RocketMQ

## 消息队列的基础知识

### 什么是消息队列

一种用于分布式系统的中间件。

分布式系统的一部分产生消息，将其存储到消息队列中，而分布式系统的另一部分从消费队列中获取消息，然后对消息进行一定的处理。

这里的消息非常广义，可以是任何东西。比如，一个序列化了的java对象，一个字符串，一串比特数组等等。消息队列本身并不关注消息的实际内容是什么，它只负责接收消息、把消息提供给消费者。

### 为什么要用消息队列

至于为什么要用消息队列，实际上主要是在允许异步的场景下提高性能、降低系统之间的耦合，以及将流量峰值打平（削峰），考虑这样两个场景：

+ 一个订单在完成后，需要发短信、邮件给用户、给用户增加积分等等一系列操作，这些操作和订单本身的完成有关系但并不一定需要和订单一同完成。如果都放在订单的流程里，当这样的附加操作增加时，会明显拖慢主流程，并且，每次增加一个新的操作，就要改动订单系统的代码，这样不是很合理。

  此时引入消息队列，即当订单创建后，将订单创建的消息放到一个队列中，让消费者去订阅，这样一个是这些附加操作都是异步执行，不会影响主流程，一个是当增加附加操作时，只要增加一个监听消息队列的进程即可，降低了系统之间的耦合。

  没错，这就是设计模式里面经典的观察者模式。

+ 在某种秒杀场景下，可能忽然涌入极大量订单，如果直接全部交付处理，瞬时流量可能直接把服务器打垮，此时现将订单全部扔到消息队列，之后服务器按自己能力取订单处理，这样就将流量峰值削平了，避免了服务器崩溃。

+ 当生产者速度很高时，可以通过水平扩展消费者来增加消费能力，确保消费不成为性能瓶颈。

### 消息队列的问题

+ 系统复杂度升高。比如消息有没有丢失、会不会被重复消费、消息的顺序如何保证，等。
+ 系统可用性降低。MQ本身可能挂掉导致系统不可用。
+ 一致性问题。比如消息没有被正确消费，但有可能系统的其他部分不能得到通知。

### 几种常用的消息队列

`JAVA`制订了一套消息队列的实现规范，称为`JMS`，即`JAVA Message Service`，`ActiveMQ`就是基于`JMS`规范实现的。

但我们不准备讨论JMS，只是把常用的几个消息队列梳理一下，详细情况见：

[消息队列对比](https://www.jianshu.com/p/e6cd4b4139ee)

简单地讲，常用的消息队列有ActiveMQ，RabbitMQ，RocketMQ和kafka，其对比如下：

![消息队列对比](https://images2018.cnblogs.com/blog/1073393/201807/1073393-20180717163401137-604348200.png)

总之，看了一圈，RabbitMQ的延时很低，ActiveMQ不太活跃，RocketMQ从阿里巴巴交到Apache维护了，性能比较综合，而kafka更多的是流计算框架，大概综合了一下，不如先了解了解RocketMQ好了。于是就选择了RocketMQ来了解。

## RocketMQ

官方git在此：[github:apache/rocketmq](https://github.com/apache/rocketmq)

RocketMQ最早是阿里的一个开源项目，自从阿里开始重视开源建设以后，这个项目就搞到apache去了，无论如何，不管这些细枝末节，我们开始了解这个消息队列。以下大部分内容摘自github上的中文文档。

[github:apache/rocketMQ中文文档](https://github.com/apache/rocketmq/tree/master/docs/cn)

### RocketMQ的基础概念

+ Producer（生产者）

消息生产者。一般由业务系统负责生产消息。一个消息生产者会把业务应用系统里产生的消息发送到broker服务器。RocketMQ提供多种发送方式，同步发送、异步发送、顺序发送、单向发送。同步和异步方式均需要Broker返回确认信息，单向发送不需要。

+ Consumer（消费者）

消息消费者。一般是后台系统负责异步消费。一个消息消费者会从Broker服务器拉取消息、并将其提供给应用程序。从用户应用的角度而言提供了两种消费形式：push和pull。

+ Broker Server（队列服务器）

消息存储的物理服务器，消息中转角色，负责存储消息、转发消息。代理服务器在RocketMQ系统中负责接收从生产者发送来的消息并存储、同时为消费者的拉取请求作准备。代理服务器也存储消息相关的元数据，包括消费者组、消费进度偏移和主题和队列消息等。

显然是整个RocketMQ的核心实现部分了，而且的确也是最复杂的一个部分。

+ Name Server（注册中心）

分布式系统里为了提高灵活性和可用性，一般通信双方都不会写死对方的地址，而是提供了一个注册中心，通信的多方将自身的通信地址写到注册中心，从而降低单点出错对系统的影响，也降低了系统的管理难度。Name Server可以集群部署，但相互独立，彼此之间并没有通信。

这个服务在RocketMQ中的角色就像Zookeeper在Dubbo中的角色。

+ Topic（主题）

表示一类消息的集合，每个主题包含若干条消息，每条消息只能属于一个主题，是RocketMQ进行消息订阅的基本单位。

因为消息队列是个中间件，所以可能被多个不同种类的Producer和Consumer访问，而Producer和Consumer正是通过Topic来标识相关数据。

+ Tag（标签）

只有Topic其实有时也足够标识一些消息，但是有时候，我们需要更详细地标识消息，既要凸显这些消息属于一种大的类别，又要区分这些消息的小类别从而让不同的消费者进行消费。根据文档中最佳实践的那部分内容，应该是

> 一个应用尽可能用一个Topic，而消息子类型则可以用tags来标识。tags可以由应用自由设置，只有生产者在发送消息设置了tags，消费方在订阅消息时才可以利用tags通过broker做消息过滤：message.setTags("TagA")。

+ 生产者组（Producer Group）

同一类Producer的集合，这类Producer发送同一类消息且发送逻辑一致。如果发送的是事务消息且原始生产者在发送之后崩溃，则Broker服务器会联系同一生产者组的其他生产者实例以提交或回溯消费。

其实没看懂官方文档这一段是啥意思，因为主要我不太明白事务消息的相关内容。

+ 消费者组（Consumer Group）

同一类Consumer的集合，这类Consumer通常消费同一类消息且消费逻辑一致。消费者组使得在消息消费方面，实现负载均衡和容错的目标变得非常容易。要注意的是，消费者组的消费者实例必须订阅完全相同的Topic。RocketMQ 支持两种消息模式：集群消费（Clustering）和广播消费（Broadcasting）。

消费者组的意义非常容易理解，比如说，一个订单完成的消息，我这里要进行：1.发邮件；2.发短信，3.加积分，4.送到大数据系统，分析这个用户喜欢啥以便以后更好地嫖他。

那么这么多的下游应用在等着，都订阅了同一个Topic，每一个应用都可能有水平副本，那么显然我们的需求是，对于同一类的应用，我们要求他们进行负载均衡，但是对于不同类的应用，我们要求所有历史消息都投递到位。为了应对这类场景，ConsumerGroup的概念就非常有用了。在实际的测试中发现以下默认行为：

每一个新的ConsumerGroup创建时，根据ConsumerClient的设置（ConsumeFromWhere），其表现会不一致。该参数默认是CONSUME_FROM_LAST_OFFSET，也就是说，新的ConsumerGroup接入后，不会消费历史信息。但该参数有个例外，就是：

> if the consumer group is created so recently that the earliest message being subscribed has yet expired, which means the consumer group represents a lately launched business, consuming will start from the very beginning
>
> 如果这个ConsumerGroup是新建的，broker里的最早的消息都还没过期（在broker的参数设置中，默认是72小时），那消费行为将从broker中最早的消息开始消费。（这也是很多人疑惑为什么该参数设置成CONSUME_FROM_LAST_OFFSET没有起作用的原因）

而当设置为CONSUME_FROM_FIRST_OFFSET时，从broker中尚未被删除的最早的消息开始消费，当设置为CONSUME_FROM_TIMESTAMP时，从指定的timestamp开始消费。

要注意的是，该参数只对新的ConsumerGroup第一次接入broker有用，而同Group的Consumer之后的接入都是从上一次消费的进度开始消费，进度首先保存在消费者本地，与broker进行同步。

参考：[官方文档：Best Practice For Consumer](https://rocketmq.apache.org/docs/best-practice-consumer/)

+ Message（消息）

消息系统所传输信息的物理载体，生产和消费数据的最小单位，每条消息必须属于一个主题。RocketMQ中每个消息拥有唯一的Message ID，且可以携带具有业务标识的Key。系统提供了通过Message ID和Key查询消息的功能。

因为其实，重复消费总是不可能完全避免的，因此一些重要的消息，需要在消费端有逻辑判断是否已经处理过。

+ 普通顺序模式

  在该模式下，消费者通过同一个消费队列收到的消息是有顺序的，不同消息队列收到的消息则可能是无顺序的。

+ 严格顺序消息

  严格顺序消息模式下，消费者收到的所有消息均是有顺序的。但有副作用，那就是当一个broker down掉以后，整个集群都不可用。在有slave的情况下，消费仍然可用但是

### 系统架构

![](https://www.history-of-my-life.com/imgs-for-md/rocketMQ-arch.png)

#### 参考资料

[RocketMQ特性及面试（下）](https://juejin.im/post/5d231b9d5188251b201d619a)

#### MessageQueue 与 消息的有序性

消息有序性是消息队列需要保证的一个比较重要的特性。在最简单的情况下，每个消息与其他消息都完全独立，其顺序完全没有关系，但实际上并非如此，比如说在削峰的应用场景下，订单的创建、支付、锁库存、减库存可能是一系列消息，但需要按顺序消费，因为你不可能为一个没有创建的订单创建支付数据，也不可能为一个没有支付的订单减库存，此时消息的顺序就很重要。

那么RocketMQ是怎么实现有序性的呢？它通过一个成为MessageQueue的内部组件来实现这个功能。如下图：

![rocketmq_broker_detail](https://www.history-of-my-life.com/imgs-for-md/rocketmq_broker_detail.png)

可以看到当生产者的消息到达以后，会首先被放入commitLog，进行刷盘，只要刷盘成功，消息就不会丢失。但因为所有的消息不分topic全都刷入同一个commitLog，且消息显然不等长，对commitLog遍历查找特定Topic数据的时间不可接受，所以建立了一些queue，实际上是一个字典表，从而允许通过topic对数据进行快速定位。

这个queue就是message queue，这是对应生产端的概念，而其实一个message Queue对应一个consumer queue，而一个consumer queue对应了一个consumer。

理论上来讲，只要一个message queue就能满足需要，但考虑到消费者的水平扩展问题，假如说一个consumer queue接入了多个消费者，那么本应该被顺序取出的消息就可能第一个被消费者A取走，第二个被消费者B取走，就破坏了顺序性。通过其他方式来保证顺序性，比如说分组、要求一个分组只能被一个消费者取走，但因为同组消息之间可能插入其他消息，而一组有多少消息也不能明确，组之间还要考虑uniqueId的问题，设计复杂度会升高。

可能是基于这种考虑，在RocketMQ中，默认一个Topic有4个message queue，也就是说，在默认情况下，一个topic最多有四个消费者，当消费者数量多于message queue时，多出来的consumer无法接入message queue，也就不能起到增大消费能力的作用。

#### 落盘为安

理论上，只有数据写到硬盘中了，才能认为消息才不会丢失。为了保证可用性，Broker提供了一些机制。在消息刷盘模式上，broker提供了以下两种方式

> 同步刷盘：如上图所示，只有在消息真正持久化至磁盘后RocketMQ的Broker端才会真正返回给Producer端一个成功的ACK响应。同步刷盘对MQ消息可靠性来说是一种不错的保障，但是性能上会有较大影响，一般适用于金融业务应用该模式较多。
>
> 异步刷盘：能够充分利用OS的PageCache的优势，只要消息写入PageCache即可将成功的ACK返回给Producer端。消息刷盘采用后台异步线程提交的方式进行，降低了读写延迟，提高了MQ的性能和吞吐量。
>
> 扩展阅读：
>
> 页缓存（PageCache)是OS对文件的缓存，用于加速对文件的读写。一般来说，程序对文件进行顺序读写的速度几乎接近于内存的读写速度，主要原因就是由于OS使用PageCache机制对读写访问操作进行了性能优化，将一部分的内存用作PageCache。对于数据的写入，OS会先写入至Cache内，随后通过异步的方式由pdflush内核线程将Cache内的数据刷盘至物理磁盘上。

而为了进一步提高可用性，Broker还提供了主从模式，一个Master可以有多个Slave。Slave只保存Master的CommitLog文件。

当Broker为同步模式时，每次master在将producer发送来的消息写入内存（磁盘）的时候会同步等待master将消息传输到slave。当Broker为异步模式时，消息会异步复制到Slave。

同一个Broker的主从服务器的brokerName一样，而brokerId不一致，0表示master，其他正整数表示slave，brokerRole可选SYNC_MASTER/ASYNC_MASTER/SLAVE。

### 启动NameServer和Broker

我们现在来看怎么启动NameServer和Broker。可以按照官方文档整，但是官方文档着实有些坑，我这里来讲一下我的编译过程。

```shell
cd ~/IdeaProjects/
git clone git@github.com:apache/rocketmq.git
cd ~/IdeaProjects/rocketmq
# 接下来要修改一堆编译语言版本，不然一定会出错，主要包括以下文件
# rocketmq-broker/pom.xml
# rocketmq-client/pom.xml
# rocketmq-common/pom.xml
# rocketmq-loging/pom.xml
# rocketmq-remoting/pom.xml
# 在里面加上下面一段

<properties>
    <java.version>1.8</java.version>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <maven.compiler.target>1.8</maven.compiler.target>
    <maven.compiler.source>1.8</maven.compiler.source>
</properties>

# 然后运行
mvn -Prelease-all -DskipTests clean install -U


# 然后我这里写了个脚本，供快速启动停止
if [ $# != 1 ]; then
   echo "-start, -produce, -consume, -kill"
   exit
fi
export JAVA_HOME='/Library/Java/JavaVirtualMachines/jdk1.8.0_131.jdk/Contents/Home'
if [ $1 = '-start' ]; then
    nohup /bin/bash /Users/mateng/IdeaProjects/rocketmq/distribution/target/rocketmq-4.7.0/rocketmq-4.7.0/bin/mqnamesrv > /dev/null 2>&1 &
    nohup /bin/bash /Users/mateng/IdeaProjects/rocketmq/distribution/target/rocketmq-4.7.0/rocketmq-4.7.0/bin/mqbroker -n localhost:9876 > /dev/null 2>&1 &
    sleep 5
    umount `mount | grep '/Volumes/RAMDisk 1' | awk '{print $1}'`
elif [ $1 = '-produce' ]; then
    export NAMESRV_ADDR=localhost:9876
    /bin/bash ~/IdeaProjects/rocketmq/distribution/target/rocketmq-4.7.0/rocketmq-4.7.0/bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
elif [ $1 = '-consume' ]; then
    export NAMESRV_ADDR=localhost:9876
    /bin/bash ~/IdeaProjects/rocketmq/distribution/target/rocketmq-4.7.0/rocketmq-4.7.0/bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
elif [ $1 = '-kill' ]; then
    kill `jps | grep NamesrvStartup | awk '{print $1}'`
    kill `jps | grep BrokerStartup | awk '{print $1}'`
    while [[ `jps | grep NamesrvStartup | awk '{print $1}'` != '' || `jps | grep BrokerStartup | awk '{print $1}'` != '' ]]
    do
        sleep 1
    done
    umount `mount | grep /Volumes/RAMDisk | awk '{print $1}'`
fi

## 然后直接运行下面四个命令就可以启动、测试和停止NameServer和Broker了
bash rocketmq.sh -start
```

我们用`jps -v`查看一下这俩服务运行的参数

```
>~ jps -v
33803 NamesrvStartup -Xms4g -Xmx4g -Xmn2g -XX:MetaspaceSize=128m 
			-XX:MaxMetaspaceSize=320m -XX:+UseConcMarkSweepGC 
			-XX:+UseCMSCompactAtFullCollection -XX:CMSInitiatingOccupancyFraction=70 
			-XX:+CMSParallelRemarkEnabled -XX:SoftRefLRUPolicyMSPerMB=0 
			-XX:+CMSClassUnloadingEnabled -XX:SurvivorRatio=8 -XX:-UseParNewGC 
			-verbose:gc -Xloggc:/Volumes/RAMDisk/rmq_srv_gc_%p_%t.log 
			-XX:+PrintGCDetails -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 
			-XX:GCLogFileSize=30m -XX:-OmitStackTraceInFastThrow -XX:-UseLargePages 
			-Djava.ext.dirs=/Library/Java/JavaVirtualMachines/jdk1.8.0_131.jdk/Contents/Home/jre/lib/ext:/Users/xxx/IdeaProjects/rocketmq/distribution/target/rocketmq-4.7.0/rocketmq-4.7.0/bin/../lib
33805 BrokerStartup -Xms8g -Xmx8g -Xmn4g -XX:+UseG1GC -XX:G1HeapRegionSize=16m
			-XX:G1ReservePercent=25 -XX:InitiatingHeapOccupancyPercent=30 
			-XX:SoftRefLRUPolicyMSPerMB=0 -verbose:gc 
			-Xloggc:/Volumes/RAMDisk/rmq_broker_gc_%p_%t.log -XX:+PrintGCDetails 
			-XX:+PrintGCDateStamps -XX:+PrintGCApplicationStoppedTime 
			-XX:+PrintAdaptiveSizePolicy -XX:+UseGCLogFileRotation 
			-XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=30m 
			-XX:-OmitStackTraceInFastThrow -XX:+AlwaysPreTouch 
			-XX:MaxDirectMemorySize=15g -XX:-UseLargePages -XX:-UseBiasedLocking 
			-Djava.ext.dirs=/Library/Java/JavaVirtualMachines/jdk1.8.0_131.jdk/Contents/Home/jre/lib/ext:/Users/xxx/IdeaProjects/rocketmq/distribution/target/rocketmq-4.7.0/rocketmq-4.7.0/bin/../lib
```

可以看到这俩一开始就吃了不少内存，一个4G堆不允许动态调整堆大小，一个8G堆不仅不允许调整堆大小还直接`AlwaysPreTouch`，在启动时一口气直接从系统申请8G内存不释放，爽歪歪。可见这玩意真的是应对大流量的，流量不够大不如想想在其他方面怎么努力提高效率...

#### Broker的参数们

| 参数名                  | 默认值                    | 说明                                                         |
| ----------------------- | ------------------------- | ------------------------------------------------------------ |
| listenPort              | 10911                     | 接受客户端连接的监听端口                                     |
| namesrvAddr             | null                      | nameServer 地址                                              |
| brokerIP1               | 网卡的 InetAddress        | 当前 broker 监听的 IP                                        |
| brokerIP2               | 跟 brokerIP1 一样         | 存在主从 broker 时，如果在 broker 主节点上配置了 brokerIP2 属性，broker 从节点会连接主节点配置的 brokerIP2 进行同步 |
| brokerName              | null                      | broker 的名称                                                |
| brokerClusterName       | DefaultCluster            | 本 broker 所属的 Cluser 名称                                 |
| brokerId                | 0                         | broker id, 0 表示 master, 其他的正整数表示 slave             |
| storePathCommitLog      | $HOME/store/commitlog/    | 存储 commit log 的路径                                       |
| storePathConsumerQueue  | $HOME/store/consumequeue/ | 存储 consume queue 的路径                                    |
| mappedFileSizeCommitLog | 1024 * 1024 * 1024(1G)    | commit log 的映射文件大小                                    |
| deleteWhen              | 04                        | 在每天的什么时间删除已经超过文件保留时间的 commit log        |
| fileReservedTime        | 72                        | 以小时计算的文件保留时间                                     |
| brokerRole              | ASYNC_MASTER              | SYNC_MASTER/ASYNC_MASTER/SLAVE                               |
| flushDiskType           | ASYNC_FLUSH               | SYNC_FLUSH/ASYNC_FLUSH SYNC_FLUSH 模式下的 broker 保证在收到确认生产者之前将消息刷盘。ASYNC_FLUSH 模式下的 broker 则利用刷盘一组消息的模式，可以取得更好的性能。 |

好吧，好多参数都挺重要的，根据我的理解如下：

brokerIP2看起来是为了让slave使用内网IP进行同步

deleteWhen和fileReservedTime共同确定了一个commit Log文件被保存的最长的时间，看起来是默认最小保存72小时。

刷盘类型也非常重要，同步双写最稳妥，主从同时确认后才能返回，、是最能保证数据不丢失的策略，但是这玩意同时也效率最低。异步的话就是批量刷盘，同时主从同步也异步。

### 没有Spring Boot时使用的基本类

#### POM依赖

```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-client</artifactId>
    <version>4.6.0</version>
</dependency>
```

#### DefaultMQProducer

其实官方包里提供的Producer有两种，一个是`DefaultMQProducer`，一个是`TransactionMQProducer`，鉴于我现在还没有能力理解分布式事务的那些问题，我这里只看看`DefaultMQProducer`，正如其文档所言：

> It's fine to tune fields which exposes getter/setter methods, but keep in mind, all of them should work well out of box for most scenarios.
>
> 虽然你可以用各种Getter和Setter方法瞎调参数，但是请记得，我们这默认参数对大部分情况都很好用。
>
> this class aggregates various <code>send</code> methods to deliver messages to brokers. Each of them has pros and cons;  you'd better understand strengths and weakness of them before actually coding.
>
> 这个类提供了大量的send方法，各有优劣。搞明白再用。
>
> 这个类显然是个很重的对象，创建和销毁都需要时间，幸好，这个类的实例是线程安全的。

首先创建一个实例：

```java
public class TestProducer {

    public static void main(String[] args) throws MQClientException {
        DefaultMQProducer defaultMQProducer = new DefaultMQProducer("testProducerGroup");
        defaultMQProducer.setNamesrvAddr("localhost:9876");
        defaultMQProducer.start();
				// our code goes here
        defaultMQProducer.shutdown();
    }
}
```

总之有正儿八经的`start()`和`shutdown()`方法的类，一看就很重，不适合创建来创建去的那种。

##### 参数设置系列方法

直接解释参数的含义好了，简单的就不讲了。只列出来提醒一下还有这功能。

```java
/**
 * 该参数是指所有broker上，一个topic的messageQueue加起来有多少个。
 * 会受brokerConfig里defaultTopicQueueNums参数的影响，会取两个里面最小的那个
 * 但这又是个迷惑行为~因为假如多broker的设置不同怎么办呢？
 * 不过该参数只有允许自动创建topic时才有用，而生产环境可能用的少一点？
 */
volatile int defaultTopicQueueNums = 4;
int sendMsgTimeout = 3000;
boolean retryAnotherBrokerWhenNotStoreOK = false;
int maxMessageSize = 1024 * 1024 * 4; // 4M

/**
 * 同步/异步模式下默认重发次数，可能造成潜在的重复消息，这些重复消息由rocketMQ的使用者解决
 */
int retryTimesWhenSendFailed = 2;
int retryTimesWhenSendAsyncFailed = 2;

/**
 * Compress message body threshold, namely, message body larger than 4k will be compressed on default.
 */
private int compressMsgBodyOverHowmuch = 1024 * 4;

/**
 * 设置异步发送的线程池。如果不设置，有个默认值。
 */
void setAsyncSenderExecutor(final ExecutorService asyncSenderExecutor);
// 默认值的代码如下
this.defaultAsyncSenderExecutor = new ThreadPoolExecutor(
    Runtime.getRuntime().availableProcessors(),
    Runtime.getRuntime().availableProcessors(),
    1000 * 60,
    TimeUnit.MILLISECONDS,
    this.asyncSenderThreadPoolQueue,
    // ThreadFactory 的作用果然就是给命名，哈哈
    new ThreadFactory() {
        private AtomicInteger threadIndex = new AtomicInteger(0);

        @Override
        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncSenderExecutor_" + this.threadIndex.incrementAndGet());
        }
    });

/**
 * 设置执行异步发送callback的代码，如果不设置，有个默认值
 */
void setCallbackExecutor(final ExecutorService callbackExecutor);
// 默认的 publicExecutor 代码如下，这是RemoteClient中的成员
this.publicExecutor = Executors.newFixedThreadPool(publicThreadNums, new ThreadFactory() {
    private AtomicInteger threadIndex = new AtomicInteger(0);

    @Override
    public Thread newThread(Runnable r) {
        return new Thread(r, "NettyClientPublicExecutor_" + this.threadIndex.incrementAndGet());
    }
});

/**
 * 在broker出现问题时的躲避策略。这两个参数是紧密相连的。长度必须一致。
 * 其实际含义是：当延迟大于 latencyMax 中的第 i 个元素的值时
 * 设置对应broker不可用时间为notAvailableDuration的第 i 个元素
 * 在这段时间不会向该 broker 发送消息
 * 仅当 sendLatencyFaultEnable 时会启动这个机制，默认为false
 */
void setSendLatencyFaultEnable(final boolean sendLatencyFaultEnable);
void setLatencyMax(final long[] latencyMax);
void setNotAvailableDuration(final long[] notAvailableDuration);

/**
 * 第二个方法的实现就是调用第一个方法，所以效果完全一致。
 * 至于什么是VipChannel，下面详述
 */
void setVipChannelEnabled(final boolean vipChannelEnabled);
void setSendMessageWithVIPChannel(final boolean sendMessageWithVIPChannel);
```

关于VIP通道，github上有个[issue](https://github.com/apache/rocketmq/issues/1510)，我把问题和回复粘贴下来：

> ### Some question about "vip channel"?
>
> - What’s the function of “vip channel”?
> - Why did we use it in RocketMQ?
> - Where use it?

> VIP channel used for command message transfer to prevent the message sending/consume socket is very busy lead to the command cannot be executed in time.

> `brokerVIPChannel` will return 2 possible broker's addresses. One is the origin address which has been passed in, another is an address with a different port. For example:
> `brokerVIPChannel(true, "127.0.0.1:9876") -> "127.0.0.1:9874" brokerVIPChannel(false, "127.0.0.1:9876") -> "127.0.0.1:9876"`
> The origin one is the normal channel and the different one is the VIP channel.
> BrokerController will initialize 2 NettyRemotingServer instance when a broker starts with the same configuration, whick means these 2 server instances listening 2 different port has the same function. In other words, the VIP channel has the same function with the normal channel.
> So why broker needs 2 server instance?
> The answer is to isolate the read and write operation. Among the API of the message , the most important one is sending message which requires high RTT . It will block netty's IO thread when it occurs message accumulation and impact to write link . The request for consuming message will full of the IO thread pool which will result in the write operation are blocked . In this case , we could send message to VIP channel to guarantee the RTT of sending messages .

##### Message的构成

见官方文档：[Message的数据结构]([https://github.com/apache/rocketmq/blob/master/docs/cn/best_practice.md#5--message%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84](https://github.com/apache/rocketmq/blob/master/docs/cn/best_practice.md#5--message数据结构))，粘贴过来如下：

| 字段名         | 默认值 | 说明                                                         |
| -------------- | :----: | ------------------------------------------------------------ |
| Topic          |  null  | 必填，消息所属topic的名称                                    |
| Body           |  null  | 必填，消息体                                                 |
| Tags           |  null  | 选填，消息标签，方便服务器过滤使用。目前只支持每个消息设置一个tag |
| Keys           |  null  | 选填，代表这条消息的业务关键词，服务器会根据keys创建哈希索引，设置后，可以在Console系统根据Topic、Keys来查询消息，由于是哈希索引，请尽可能保证key唯一，例如订单号，商品Id等。 |
| Flag           |   0    | 选填，完全由应用来设置，RocketMQ不做干预                     |
| DelayTimeLevel |   0    | 选填，消息延时级别，0表示不延时，大于0会延时特定的时间才会被消费 |
| WaitStoreMsgOK |  TRUE  | 选填，表示消息是否在服务器落盘后才返回应答。                 |

其中DelayTimeLevel是指如果定时消息，即到达broker后，首先会被放入定时任务队列，直到定时到达后才会放入正式的Topic队列。

##### send()系列方法

接着我们来研究他的一系列`send()`方法，首先，从函数命名上来看分为两大类：

`send()`，`sendOneway()`

这两个大类的差别很简单，`send()`要求`broker`提供ack消息，因此适用于一些对消息丢失比较敏感的场合，而`sendOneway()`不care这些东西，适用于发日志这种可以容忍丢失的消息。

而根据`send()`的参数中有没有`SendCallback`这个类型的参数，`send()`又分为两大类，没有该参数的`send()`是同步发送，返回一个`SendResult`实例，而有该参数的是异步发送，没有返回值，而`SendResult`在提供的`SendCallBack`中进行消费。

OK，大概知道了这么多，接下来我们来仔细看一看这些方法，借此阐明RocketMQ中的一些概念。

```java
SendResult send(Message);
SendResult send(Message, long);
SendResult send(Message, MessageQueue);
SendResult send(Message, MessageQueue, long);
SendResult send(Message, MessageQueueSelector, Object);
SendResult send(Message, MessageQueueSelector, Object, long);
```

这里面全部都是同步方法，如我们所见，Message参数是必须的，我们接下来再说Message的事情，其他几个参数，其中long表示超时时间，在不设置时采取的是默认时间，也就是`3000ms`，而`MessageQueue`这个参数的意义之前解释过了，在需要保证消息有序的情况下，需要指定messageQueue。而MessageQueueSelector则提供了更加灵活的机制，即在投递时通过MessageSelector来选择到底使用哪个MessageQueue，而有MessageQueueSelector参数的方法中，都有一个Object参数，它是提供给MessageSelector的参数，这里提供一个官网的demo来加强理解。

```java
public class Producer {

   public static void main(String[] args) throws Exception {
			 // ...省略部分内容
       
     	 // 订单列表
       String[] tags = new String[]{"TagA", "TagC", "TagD"};
       List<OrderStep> orderList = buildOrders(); // 该方法省略

       String dateStr = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date());
       for (int i = 0; i < 10; i++) {
           // 加个时间前缀
           String body = " Hello RocketMQ " + orderList.get(i);
           Message msg = new Message("TopicTest", tags[i % tags.length], "KEY" + i, body.getBytes());

           SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
               @Override
               public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                   Long id = (Long) arg;  //根据订单id选择发送queue
                   long index = id % mqs.size();
                   return mqs.get((int) index);
               }
           }, orderList.get(i).getOrderId());	//订单id
         
       }

       producer.shutdown();
   }
}
```

接下来看异步方法：

```java
void send(Message, SendCallback);
void send(Message, SendCallback, long);
void send(Message, MessageQueue, SendCallback);
void send(Message, MessageQueue, SendCallback, long);
void send(Message, MessageQueueSelector, Object, SendCallback);
void send(Message, MessageQueueSelector, Object, SendCallback, long);
```

异步方法除了多了一个SendCallBack参数，其他参数的意义和之前一样。而CallBack如下：

```java
public class AsyncProducer {
    public static void main(String[] args) throws Exception {
        // 省略实例化消息生产者Producer
      	// 将重试次数设置为0
        producer.setRetryTimesWhenSendAsyncFailed(0);

        int messageCount = 100;
        // 根据消息数量实例化倒计时计算器
        final CountDownLatch2 countDownLatch = new CountDownLatch2(messageCount);
        for (int i = 0; i < messageCount; i++) {
            final int index = i;
            // 创建消息，并指定Topic，Tag和消息体
            Message msg = new Message("TopicTest",
                "TagA",
                "OrderID188",
                "Hello world".getBytes(RemotingHelper.DEFAULT_CHARSET));
            // SendCallback接收异步返回结果的回调
            producer.send(msg, new SendCallback() {
                @Override
                public void onSuccess(SendResult sendResult) {
                    System.out.printf("%-10d OK %s %n", index,
                        sendResult.getMsgId());
                    countDownLatch.countDown();	// 官方样例竟然没有这一句，就很离谱
                }

                @Override
                public void onException(Throwable e) {
                    System.out.printf("%-10d Exception %s %n", index, e);
                    e.printStackTrace();
                    countDownLatch.countDown();	// 官方样例竟然没有这一句，就很离谱
                }
            });
        }
        // 等待5s
        countDownLatch.await(5, TimeUnit.SECONDS);
        // 如果不再发送消息，关闭Producer实例。
        producer.shutdown();
    }
}
```

接下来就是batch模式：

```java
//for batch
SendResult send(Collection<Message>);
SendResult send(Collection<Message>, long);
SendResult send(Collection<Message>, MessageQueue);
SendResult send(Collection<Message>, MessageQueue, long);
```

这个看起来很明确。之所以要batch，是因为batch能显著提高性能，而他们最终将会被发送到同一个messageQueue中去。

之后是sendOneway，这些只管发不管到，跟UDP一样不负责任，用于消息可以容忍丢失的场合。

```java
void sendOneway(Message);
void sendOneway(Message, MessageQueue);
void sendOneway(Message, MessageQueueSelector, Object);
```

##### 其他重要方法

```java
// 获得该topic对应的所有消息队列
List<MessageQueue> fetchPublishMessageQueues(String topic);

// request 系列方法。request有意思的是会一直阻塞知道消费者消费完消息并且回复以后才会返回
// 更像是RPC的用法了。可能会用于必须确认消费结果的场合吧。
// for rpc // 这是代码里的原注释，for rpc，看起来确实是RPC的用法
Message request(Message, long);
void request(Message, RequestCallback, long);
Message request(Message, MessageQueueSelector, Object, long);
void request(Message, MessageQueueSelector, Object, RequestCallback, long);
Message request(Message, MessageQueue, long);
void request(Message, MessageQueue, RequestCallback, long);
```

##### SendResult

根据BestPratice的建议

> 消息发送成功或者失败要打印消息日志，务必要打印SendResult和key字段。send消息方法只要不抛异常，就代表发送成功。发送成功会有多个状态，在sendResult里定义。以下对每个状态进行说明：
>
> - **SEND_OK**
>
> 消息发送成功。要注意的是消息发送成功也不意味着它是可靠的。要确保不会丢失任何消息，还应启用同步Master服务器或同步刷盘，即SYNC_MASTER或SYNC_FLUSH。
>
> - **FLUSH_DISK_TIMEOUT**
>
> 消息发送成功但是服务器刷盘超时。此时消息已经进入服务器队列（内存），只有服务器宕机，消息才会丢失。消息存储配置参数中可以设置刷盘方式和同步刷盘时间长度，如果Broker服务器设置了刷盘方式为同步刷盘，即FlushDiskType=SYNC_FLUSH（默认为异步刷盘方式），当Broker服务器未在同步刷盘时间内（默认为5s）完成刷盘，则将返回该状态——刷盘超时。
>
> - **FLUSH_SLAVE_TIMEOUT**
>
> 消息发送成功，但是服务器同步到Slave时超时。此时消息已经进入服务器队列，只有服务器宕机，消息才会丢失。如果Broker服务器的角色是同步Master，即SYNC_MASTER（默认是异步Master即ASYNC_MASTER），并且从Broker服务器未在同步刷盘时间（默认为5秒）内完成与主服务器的同步，则将返回该状态——数据同步到Slave服务器超时。
>
> - **SLAVE_NOT_AVAILABLE**
>
> 消息发送成功，但是此时Slave不可用。如果Broker服务器的角色是同步Master，即SYNC_MASTER（默认是异步Master服务器即ASYNC_MASTER），但没有配置slave Broker服务器，则将返回该状态——无Slave服务器可用。

#### DefaultPushMQConsumer

Consumer有两大类，一类是PullConsumer，一类是PushConsumer。显然一类是主动去broker拉取数据，一类是由broker主动推送数据给Consumer。根据[官方设计文档](https://github.com/apache/rocketmq/blob/master/docs/cn/design.md)：

> 而Push模式只是对pull模式的一种封装，其本质实现为消息拉取线程在从服务器拉取到一批消息后，然后提交到消息消费线程池后，又“马不停蹄”的继续向服务器再次尝试拉取消息。如果未拉取到消息，则延迟一下又继续拉取。
>
> 在两种基于拉模式的消费方式（Push/Pull）中，均需要Consumer端在知道从Broker端的哪一个消息队列—队列中去获取消息。因此，有必要在Consumer端来做负载均衡，即Broker端中多个MessageQueue分配给同一个ConsumerGroup中的哪些Consumer消费。

那么为什么这种马不停蹄地拉数据的行为也能达到push级别的响应速度呢？这是因为在pushConsumer在pull的时候使用长连接，在没有数据时，会hang住一段时间等待新的数据来临，在这段时间里假如来了新数据，就立即返回，这样就达到了push的实时性。

总之不用太纠结这俩的不一样，在我测试使用的这个版本里，官当的Pull实现类是`DefaultLitePullConsumer`，而push实现类是`DefaultMQPushConsumer`，原来的pull实现类`DefaultMQPullConsumer`已经标记为弃用。

其实从接口来看，`PullConsumer`和`PushConsumer`差别并不大，我们先来看PushConsumer。

```java
public class TestPushConsumer {

    public static void main(String[] args) throws MQClientException {

        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("testProducerGroup");
        consumer.setNamesrvAddr("localhost:9876");
      
      	// 集群模式还是广播模式
        consumer.setMessageModel(MessageModel.CLUSTERING);

        // 订阅一个或者多个Topic，以及Tag来过滤需要消费的消息
        consumer.subscribe("testTopic", "*");
        
        // 注册监听器，有两种，一个是顺序，一个是并行
//        consumer.registerMessageListener(new MessageListenerOrderly() {
//            @Override
//            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context) {
//                for (MessageExt msg : msgs) {
//                    System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), new String(msg.getBody()));
//                }
//                // 标记该消息已经被成功消费
//                return ConsumeOrderlyStatus.SUCCESS;
//            }
//        });
        consumer.registerMessageListener(
            new MessageListenerConcurrently() {
                @Override
                public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                    for (MessageExt msg : msgs) {
                        System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), new String(msg.getBody()));
                    }
                    // 标记该消息已经被成功消费
                    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
                }
            }
        );
      	consumer.start();
        System.out.printf("Consumer Started.%n");
    }

}
```

##### subscribe()

```java
void subscribe(String, MessageSelector);
void subscribe(String, String);
```

这两个方法都是为了过滤消息，即从同一个Topic里选择不同类型的子消息。上面介绍的tag就是区分子主题的一个方便功能，但显然RocketMQ最后认为这个子主题还是不全面，因此又定义了一种功能更强大的信息筛选方法，见[过滤信息样例](https://github.com/apache/rocketmq/blob/master/docs/cn/RocketMQ_Example.md#5-过滤消息样例)：

> 只有使用push模式的消费者才能用使用SQL92标准的sql语句。

第一个是全功能版，第二个是根据tag选择的易用版本。

和我最初想的不同，经过测试，其实一个Consumer可以订阅多个topic，与此配合，有个`unsubscribe(String)`方法，用来取消对某个Topic的订阅。

另外，要注意的是，subscribe()方法实现的筛选是在broker服务器上的筛选，这是利用broker的闲置CPU资源来换取紧张的网络资源。在较早的版本（也是很多教程上的版本），说是实现MessageFilter接口，但在4.6.0版本中，已经无法找到传输该类的接口，应该认为是彻底废弃了。

现在subscribe使用的两种方法，实际都会被转换为SubscriptionData。

##### registerMessageListener

```java
void registerMessageListener(MessageListenerConcurrently);
void registerMessageListener(MessageListenerOrderly);
```

Consumer内部也有线程池，线程池的大小由几个参数指定，那么问题就来了，假如我们从broker里顺序拿到了一系列需要顺序消费的消息，但却将其一口气交给内部的线程池并行处理，而并行处理显然是乱序的，那我们废那么大功夫保证消费者顺序拿到消息又有什么用呢？因此，在消费者拿到消息消费时，还需要进一步保证消息是按照拿到的顺序进行消费的。

这就是消费端的同步模式和顺序模式的差别。

**一个Consumer只能注册一个Listener，重复注册时，旧的会被顶掉，并且需要在start()方法之前调用，不然木已成舟，就改不了了**

本来我还比较疑惑，在顺序模式下还需要消费端线程池吗，单线程池就够了吧？后来看了看源码，发现确实还需要，而且不同的Listener会导致内部使用线程池的策略不同。

在使用MessageListenerConcurrently时，所有的消息到来之后立即交给各个线程运行，这当然无需赘言，但当使用MessageListenerOrderly时情况略为复杂，即虽然我们在默认情况下建立了20个线程，但是，这些线程在run时，会对Consumer对应的MessageQueue实例加锁，也就是说，一个Consumer有可能对应多个Broker中的MessageQueue，而Consumer只保证对于每一个MessageQueue是有序的，因此，20个线程都会拿到处理对应消息的任务，但对于同一个MessageQueue阻塞，只有同一MessageQueue的一个任务处理完之后，同MessageQueue的其他任务才有机会得到处理。

其实这里还有个问题，假如Consumer在拿到10个消息后，一口气把这些消息全部打包成Runnable交给线程池的话，那么其实Consumer的执行顺序还是无法保证，因为在最初获得MessageQueue锁的线程释放锁后，其他线程获得锁的顺序仍然是不固定的。因此实际上肯定他还用了其他更巧妙的方式保证了消费的顺序。

但我看了半天源码，把自己绕进去了。可以确认的是，RocketMQ使用单线程来pullMessage，而它本身拉取的数据是保证有序的，所以只需要用某种方式，保证在一次性拉取到的N个消息消费完之前，不再创建新的拉取任务，或者说虽然没有停止拉取，但停止将拉取的数据打包提交给线程池就可以保证顺序了，但到底用了哪一种方式，我把自己绕晕了所以没搞明白。

##### MessageListener的返回值

我们看一下MessageListener的接口定义：

```java
public interface MessageListenerConcurrently extends MessageListener {
    /**
     * It is not recommend to throw exception,rather than returning ConsumeConcurrentlyStatus.RECONSUME_LATER if
     * consumption failure
     *
     * @param msgs msgs.size() >= 1<br> DefaultMQPushConsumer.consumeMessageBatchMaxSize=1,you can modify here
     * @return The consume status
     */
    ConsumeConcurrentlyStatus consumeMessage(final List<MessageExt> msgs,
        final ConsumeConcurrentlyContext context);
}
public interface MessageListenerOrderly extends MessageListener {
    /**
     * It is not recommend to throw exception,rather than returning ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT
     * if consumption failure
     *
     * @param msgs msgs.size() >= 1<br> DefaultMQPushConsumer.consumeMessageBatchMaxSize=1,you can modify here
     * @return The consume status
     */
    ConsumeOrderlyStatus consumeMessage(final List<MessageExt> msgs,
        final ConsumeOrderlyContext context);
}
```

也就是说，这两个接口都不建议抛出异常（实际上如果抛出异常，会被当做消费失败，和稍后重试含义接近），而是明确返回值。那么这个返回值都有哪些取值呢？

```java
// 这里是并行消费的返回值
public enum ConsumeConcurrentlyStatus {
    CONSUME_SUCCESS,
    // 下面是该批消息稍候重试，重试次数太多就进死信队列了。
    RECONSUME_LATER;
}
```

```java
// 这是顺序消费的返回值
public enum ConsumeOrderlyStatus {
    /**
     * Success consumption
     */
    SUCCESS,
    /**
     * Rollback consumption(only for binlog consumption)
     */
    @Deprecated
    ROLLBACK,
    /**
     * Commit offset(only for binlog consumption)
     */
    @Deprecated
    COMMIT,
    /**
     * 因为顺序消费隐含着后面的消息必须后消费，所以会无限阻塞队列，而且没有重试次数限制
     * 消息不会进入死信队列，而是一直、一直、一直重试。似乎有点爽歪歪。
     */
    SUSPEND_CURRENT_QUEUE_A_MOMENT;
}
```

##### 线程池调整方法

```java
void setAdjustThreadPoolNumsThreshold(long);
void setConsumeThreadMin(int);
void setConsumeThreadMax(int);
```

第一个方法很有意思，设计初衷是这样的：假如说broker里消息堆积的数量太多了，超过了这个方法参数设置的值，我们就扩大本地consumer线程池的大小。不得不说，这个设计初衷是好的，但是这种来回的交互方式的确有些奇怪，就是上下级模块之间，在程序员控制之外静默调整线程池，感觉不是好的设计。所以虽然这个方法并没有标记为废弃，但实际上最终调整线程池大小的方法`org.apache.rocketmq.client.impl.consumer.ConsumeMessageConcurrentlyService#incCorePoolSize()`和`decCorePoolSize()`里面的代码都被完全注释掉了，也就是说实际废弃了这个线程池调整方法的意义。

第二三个参数的意义很简单，一个是线程池核心线程数量，一个是线程池最大线程数量。在默认情况下，这个值都是20。实际上，观察了具体实现代码以后，我发现设置比核心线程数量更大的最大线程数量并无意义，因为创建消费线程池的那一段代码是这样的：

```java
// client 4.6.0版本
this.consumeRequestQueue = new LinkedBlockingQueue<Runnable>();

this.consumeExecutor = new ThreadPoolExecutor(
    this.defaultMQPushConsumer.getConsumeThreadMin(),
    this.defaultMQPushConsumer.getConsumeThreadMax(),
    1000 * 60,
    TimeUnit.MILLISECONDS,
    this.consumeRequestQueue,
    new ThreadFactoryImpl("ConsumeMessageThread_"));
```

让我们来复习一下线程池的知识：

> 一个`ThreadPoolExecutor`线程池将根据核心线程数量和最大线程数量自动调整线程池大小。当一个新任务提交时，如果此时正在运行的线程比corePoolSize数量小，那么线程池将会创建一个新线程来处理这个任务，即使现在还有线程处于空闲状态。如果正在运行的线程数量大于corePoolSize但是小于maximumPoolSize，任务将会先暂存到任务队列里，**除非任务队列也满了，这时候才会新建一个线程。**

但是，因为这里写死提供了一个无界队列，也就是说，假如消费速度不够快，其实直到OOM错误发生，线程池也并不会新建线程。是一段迷之源码。所以其实设置`setConsumeThreadMin()`就可以了，另一个方法并不会起作用= =

##### 流控方法

呃，虽然上面的线程池使用方式理论上可能造成OOM错误，但实际上因为消息队列作为中间件本身可以存储消息，因此实际上只要控制拉取速度，就不会造成消费端的消息积压。这也是使用消息队列最初的目的之一：削峰。

所以，控制拉取速度的行为称为流控。有以下概念与流控相关。

+ max offset

  字面上可以理解为这是标识message queue中的max offset表示消息的最大offset。但是从源码上看，这个offset实际上是最新消息的offset+1，即：下一条消息的offset。

+ min offset

  标识现存在的最小offset。而由于消息存储一段时间后，消费会被物理地从磁盘删除，message queue的min offset也就对应增长。这意味着比min offset要小的那些消息已经不在broker上了，无法被消费。

+ consumer offset

  字面上，可以理解为标记Consumer Group在一条逻辑Message Queue上，消息消费到哪里即消费进度。但从源码上看，这个数值是消费过的最新消费的消息offset+1，即实际上表示的是**下次拉取的offset位置**。

  消费者拉取消息的时候需要指定offset，broker不主动推送消息， offset的消息返回给客户端。

  consumer刚启动的时候会获取持久化的consumer offset，用以决定从哪里开始消费，consumer以此发起第一次请求。

  每次消息消费成功后，这个offset在会先更新到内存，而后定时持久化。在集群消费模式下，会同步持久化到broker，而在广播模式下，则会持久化到本地文件。

+ span

  中文翻译是跨度，那这个跨度到底是什么跨度呢？在consumer中有个方法名字中带span，那span到底是什么呢？我们来看看这个方法的相关代码部分：

  ```java
  // 设置并行消费时的最大跨度
  void setConsumeConcurrentlyMaxSpan(int consumeConcurrentlyMaxSpan) {
      this.consumeConcurrentlyMaxSpan = consumeConcurrentlyMaxSpan;
  }
  
  // 对顺序消费无效。
  /**
  * Concurrently max span offset.it has no effect on sequential consumption
  */
  private int consumeConcurrentlyMaxSpan = 2000;
  
  // 在下面这个类中，该参数找到用处
  // org.apache.rocketmq.client.impl.consumer.DefaultMQPushConsumerImpl
  if (!this.consumeOrderly) {
    if (processQueue.getMaxSpan() > this.defaultMQPushConsumer.getConsumeConcurrentlyMaxSpan()) {
      this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
      if ((queueMaxSpanFlowControlTimes++ % 1000) == 0) {
        log.warn(
          "the queue's messages, span too long, so do flow control, minOffset={}, maxOffset={}, maxSpan={}, pullRequest={}, flowControlTimes={}",
          processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), processQueue.getMaxSpan(),
          pullRequest, queueMaxSpanFlowControlTimes);
      }
      return;
    }
  } else {
    // 顺序消费模式的代码
  }
  
  
  // 可以看到，只要最大跨度超过了这个参数，就触发流控，接下来消费端会延迟从消息队列拉消息
  // 那么这个maxSpan是怎么算的呢？我们追踪一下
  public long getMaxSpan() {
      try {
          this.lockTreeMap.readLock().lockInterruptibly();
          try {
              if (!this.msgTreeMap.isEmpty()) {
                	// 核心代码就这一句
                  return this.msgTreeMap.lastKey() - this.msgTreeMap.firstKey();
              }
          } finally {
              this.lockTreeMap.readLock().unlock();
          }
      } catch (InterruptedException e) {
          log.error("getMaxSpan exception", e);
      }
      return 0;
  }
  // 这个tree的声明如下：
  TreeMap<Long, MessageExt> msgTreeMap = new TreeMap<>();
  // 那么他的Key到底是什么呢？
  public boolean putMessage(final List<MessageExt> msgs){
    	// 省略无关代码
      MessageExt old = msgTreeMap.put(msg.getQueueOffset(), msg);
  }
  
  // 可见是用msg的queue offset做key
  ```
  
  那么含义就很明显了，就是当当前消费者拉到本地的消息中，最早的一个消息的offset和最新的消息的offset差别超过了consumeConcurrentlyMaxSpan，就触发流控。说明第2000个之前的消息都没处理完，这很可能说明某个并发线程出现了问题，卡了，这时候一方面触发流控，可能一方面就要排查问题了。
  
  除此之外，还有些其他的流控参数和相关set方法，意思是达到相关阈值后就触发流控。
  
    ```java
  // 每个消费者本地的queue（对应一个messageQueue）最多缓存多少个数据
  pullThresholdForQueue = 1000;
  // 每个消费者本地的queue（对应一个messageQueue）最多缓存多少大小（以Mb为单位）的数据
  int pullThresholdSizeForQueue = 100;
  // 消费者对于每个Topic的最大缓存，-1默认值表示无限制。不过还受上两个参数的限制
  // 最终还是有限制
  int pullThresholdForTopic = -1;
  // 显而易见
  int pullThresholdSizeForTopic = -1;
  // 似乎是以毫秒计时的拉取时间间隔，但感觉在绝大多数场景下没有限制的必要
  long pullInterval = 0;
  // 当触发流控时，延迟从broker中拉取消息多久，默认看起来是1s
  long suspendCurrentQueueTimeMillis = 1000;
    ```

##### batch相关参数设置

```java
// 一次向broker的请求最多请求多少数量的消息
int pullBatchSize = 32;

// 一次消费几个消息
int consumeMessageBatchMaxSize = 1;
// 对于这个参数要稍微做一些解释，我们可以在上面注册Listener的代码里看一下
public interface MessageListenerConcurrently extends MessageListener {
    ConsumeConcurrentlyStatus consumeMessage(final List<MessageExt> msgs,
        final ConsumeConcurrentlyContext context);
}
public interface MessageListenerOrderly extends MessageListener {
    ConsumeOrderlyStatus consumeMessage(final List<MessageExt> msgs,
        final ConsumeOrderlyContext context);
}
// 这里面consumeMessage的参数都是List，那这个List从哪里来呢
// 在ConsumeMessageConcurrentlyService里是这样的
public void submitConsumeRequest(
    final List<MessageExt> msgs, final ProcessQueue processQueue,
    final MessageQueue messageQueue, final boolean dispatchToConsume) {
    
    final int consumeBatchSize = this.defaultMQPushConsumer.getConsumeMessageBatchMaxSize();
    // 这里得到的msgs的最大数量，由pullBatchSize限制，源代码不贴了
    if (msgs.size() <= consumeBatchSize) {
        ConsumeRequest consumeRequest = new ConsumeRequest(msgs, processQueue, messageQueue);
        try {
            this.consumeExecutor.submit(consumeRequest);
        } catch (RejectedExecutionException e) {
            this.submitConsumeRequestLater(consumeRequest);
        }
    } else {
        for (int total = 0; total < msgs.size(); ) {
            List<MessageExt> msgThis = new ArrayList<MessageExt>(consumeBatchSize);
            for (int i = 0; i < consumeBatchSize; i++, total++) {
                if (total < msgs.size()) {
                    msgThis.add(msgs.get(total));
                } else {
                    break;
                }
            }

            ConsumeRequest consumeRequest = new ConsumeRequest(msgThis, processQueue, messageQueue);
            try {
                this.consumeExecutor.submit(consumeRequest);
            } catch (RejectedExecutionException e) {
                for (; total < msgs.size(); total++) {
                    msgThis.add(msgs.get(total));
                }

                this.submitConsumeRequestLater(consumeRequest);
            }
        }
    }
}
// 在ConsumeMessageOrderlyService里是这样的，前后省略无关代码
final int consumeBatchSize = ConsumeMessageOrderlyService.this.defaultMQPushConsumer.getConsumeMessageBatchMaxSize();
List<MessageExt> msgs = this.processQueue.takeMessags(consumeBatchSize);
// 所以在我们不修改consumeMessageBatchMaxSize参数时，Listener拿到的List都只有1个元素
```

##### setConsumeFromWhere(ConsumeFromWhere)

对消费位置的精确定位，参数是一个enum。

根据网上的提示：

> 这个参数只对一个新的consumeGroup第一次启动时有效。
>  就是说，如果是一个consumerGroup重启，他只会从自己上次消费到的offset，继续消费。这个参数是没用的。 而判断是不是一个新的ConsumerGroup是在broker端判断。
>  要知道，消费到哪个offset最先是存在Consumer本地的，定时和broker同步自己的消费offset。broker在判断是不是一个新的consumergroup，就是查broker端有没有这个consumergroup的offset记录。
> 链接：https://www.jianshu.com/p/dafc7b3b8988

```java
ConsumeFromWhere consumeFromWhere = ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET;
public enum ConsumeFromWhere {
    /**
     * 一个新的订阅组第一次启动从队列的最后位置开始消费
     * 后续再启动接着上次消费的进度开始消费
     */
    CONSUME_FROM_LAST_OFFSET,

    // 下面三个已废弃的在目前的实现中和上一个参数意义一样了。
    @Deprecated
    CONSUME_FROM_LAST_OFFSET_AND_FROM_MIN_WHEN_BOOT_FIRST,
    @Deprecated
    CONSUME_FROM_MIN_OFFSET,
    @Deprecated
    CONSUME_FROM_MAX_OFFSET,
    /**
     * 一个新的订阅组第一次启动从队列的最前位置开始消费
     * 后续再启动接着上次消费的进度开始消费
     */
    CONSUME_FROM_FIRST_OFFSET,
    
    /**
     * 一个新的订阅组第一次启动从指定时间点开始消费
     * 后续再启动接着上次消费的进度开始消费
     * 时间点设置参见DefaultMQPushConsumer.consumeTimestamp参数
     * 也就是要配合 setConsumeTimestamp()方法使用
     */
    CONSUME_FROM_TIMESTAMP,
}

/** 附上setConsumeTimestamp所设置的参数的含义
 * Backtracking consumption time with second precision. Time format is
 * 20131223171201<br>
 * Implying Seventeen twelve and 01 seconds on December 23, 2013 year<br>
 * Default backtracking consumption time Half an hour ago.
 * 默认值半小时前
 */
private String consumeTimestamp = UtilAll.timeMillisToHumanString3(System.currentTimeMillis() - (1000 * 60 * 30));
```

##### NameSpace

看起来是在4.4.0以后新增的概念，在[这个issue中提出](https://github.com/apache/rocketmq/issues/1120)，在[这个PR中新增](https://github.com/apache/rocketmq/pull/1122)

至于是干嘛的，我把issue的内容贴过来就懂了

> **FEATURE REQUEST**
>
> 1. Please describe the feature you are requesting.
>    Support namespace for RocketMQ，the same topic with different namespaces represent different topics in RocketMQ Namesrv/Broker. Changes will be hold in the rocketmq-client module.
> 2. Provide any additional detail on your proposed use case for this feature.
>    If you have three environments called daily/pre-online/online, you may only want to change the endpoint or the namespace in your configs to send messages to the different topics in different environments.

理论上，这看起来像是Client网络通信端该完成的事情。

在Consumer中有以下方法来使用namespace的概念：

```java
void setNamespace(String namespace);
// 看起来下面这些更像是内部使用的方法，我们只需要设置上面的方法就够了
MessageQueue queueWithNamespace(MessageQueue queue);
Collection<MessageQueue> queuesWithNamespace(Collection<MessageQueue> queues);
String withNamespace(String resource);
String withoutNamespace(String resource);
Set<String> withNamespace(Set<String> resourceSet);
Set<String> withoutNamespace(Set<String> resourceSet);
```

##### 其他定制方法

```java
/**
 * 设置IP，一般来说都是从NameServer那里弄到的，就是说NameServer看到的是啥就是啥
 * 也不需要单独设置，但有时候内外网问题可能需要单独设置
 * 我觉得应该会更新NameServer中的IP设置
 */
void setClientIP(String clientIP);
/**
 * 这个要配合TlsSystemConfig设置好多东西，毕竟TLS不是想开随意就开了的
 * 感觉内网环境的话，似乎必要性一般
 */
void setUseTLS(boolean useTLS);
/**
 * Max re-consume times. -1 means 16 times. 默认值就是-1
 * 在重试次数达到一定值时，消息会进入死信队列。
 * 在并行消费的情况下才会起作用，因为顺序消费隐含着必须前面消费完才能消费后面的消息的限制
 * 所以顺序消费下这个重试次数实际上是无限大的。
 */
void setMaxReconsumeTimes(final int maxReconsumeTimes);

/**
 * 客户端通信层接收到网络请求的时候，处理器的核数
 * Runtime.getRuntime().availableProcessors()
 */
clientCallbackExecutorThreads;

/**
 * 轮询从NameServer获取路由信息的时间间隔。这个间隔决定了新服务上线/下线，
 * 客户端最长多久能探测得到。默认是30秒，就是说如果做broker扩容，
 * 最长需要30秒客户端才能感知得到新broker的存在。
 */
pollNameServerInterval;

/**
 * RocketMQ采取的是定期批量ack的机制以持久化消费进度。也就是说每次消费消息结束后，
 * 并不会立刻ack，而是定期的集中的更新进度。 由于持久化不是立刻持久化的，
 * 所以如果消费实例突然退出（如断电）或者触发了负载均衡分consue queue重排，
 * 有可能会有已经消费过的消费进度没有及时更新而导致重新投递。故本配置值越小，
 * 重复的概率越低，但同时也会增加网络通信的负担。
 */
persistConsumerOffsetInterval;

/**
 * 一个消息最多被消费多久，默认15分钟没回应就算消费失败了。
 */
consumeTimeout;

/**
 * 从上文可以看到，vipChannel是给Producer用的，Consumer并不能从该参数中获益。
 */
vipChannelEnabled;
```

#### DefaultLitePullConsumer

啊，懒得再分析了。里面多了一些跟主动pull相关的参数，比如pull哪些queue，暂停pull哪些queue，设置pull线程池等等。然后是通过`poll()`或者`poll(long timeout)`来获取`List<MessageExt>`，之后再手动消费。

总之，看起来显然没有pushConsumer方便。实际上用的也确实不怎么多，所以~Emm，我就懒得搞了。

### 与Spring Boot的整合

anyway，java开发几乎已经绕不开Spring Boot了，好用的框架都需要和Spring Boot整合，rocketMQ也不例外。

#### POM依赖

```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-spring-boot-starter</artifactId>
    <version>2.1.0</version>
</dependency>
```

#### RocketMQTemplate

这是DefaultMQProducer与Spring Boot整合后的一个类，是Producer。他的一系列方法都不需要单独介绍，因为要是搞明白RocketMQ的相关概念，这些方法的命名就足以告诉我们这些方法的作用了。唯一有一点需要介绍的是这几个方法：

```java
/**
 * 设置QueueSelector，我们上面已经看过Selector的代码了，它其实需要一个额外参数即：arg
 * 来选择队列，那么这个arg参数到底是什么呢？
 */
void setMessageQueueSelector(MessageQueueSelector messageQueueSelector);

// 这个arg参数，是所有send方法中的hashKey参数。
// 如果没有该参数，实际上也用不到MessageQueueSelector，而是会使用DefaultMQProducer
// 的默认队列分配操作，也就是：轮询。

/**
 * 这是默认的实现
 */ 
public class SelectMessageQueueByHash implements MessageQueueSelector {

    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        int value = arg.hashCode();
        if (value < 0) {
            value = Math.abs(value);
        }

        value = value % mqs.size();
        return mqs.get(value);
    }
}

/**
 * 还记得有个类似于RPC的调用方式吧，DefaultMQProducer的request系列方法。
 * Template也进行了支持，即 sendAndReceive 系列方法。
 * 这一系列方法，根据是否提供 CallBack 对象，又分为有返回值、需要手动消费，和
 * 直接调用callBack消费两种。举一个CallBack的例子
 */
void sendAndReceive(String destination, Message<?> message, 
                    RocketMQLocalRequestCallback rocketMQLocalRequestCallback);
```

基本上，在Spring Boot中使用RocketMQ，Producer端使用这个就可以了。

#### 消费端相关接口

为了在Spring中注册一个处理消息的Bean，我们需要分三步走：

1. 扩展以下两个接口中的一个

   ```java
   /**
    * 这明显对应着那些不需要向producer回复消息的consumer行为
    * 也应该是用的最多的一种consumer行为了
    */
   public interface RocketMQListener<T> {
       void onMessage(T message);
   }
   
   /**
    * 这是应对Producer端request时的consumer行为
    */
   public interface RocketMQReplyListener<T, R> {
       /**
        * @param message data received by the listener
        * @return data replying to producer
        */
       R onMessage(T message);
   }
   ```

2. 将其标记为Spring中的一个Bean。

3. 使用rocketMQ的特有注解，提供 Broker 的基本信息。

   ```java
   @Service
   @RocketMQMessageListener(topic = "testTopic", consumerGroup = "testProducerGroup")
   public class StudentConsumer implements RocketMQListener<Student> {
   
       @Override
       public void onMessage(Student message) {
           System.out.println(message);
       }
   }
   ```

我们详细地看一下RocketMQMessageListener注解的相关参数。

```java
public @interface RocketMQMessageListener {

    String NAME_SERVER_PLACEHOLDER = "${rocketmq.name-server:}";
    String ACCESS_KEY_PLACEHOLDER = "${rocketmq.consumer.access-key:}";
    String SECRET_KEY_PLACEHOLDER = "${rocketmq.consumer.secret-key:}";
    String TRACE_TOPIC_PLACEHOLDER = "${rocketmq.consumer.customized-trace-topic:}";
    String ACCESS_CHANNEL_PLACEHOLDER = "${rocketmq.access-channel:}";

    /**
     * Consumers of the same role is required to have exactly same subscriptions and consumerGroup to correctly achieve
     * load balance. It's required and needs to be globally unique.
     *
     *
     * See <a href="http://rocketmq.apache.org/docs/core-concept/">here</a> for further discussion.
     */
    String consumerGroup();

    /**
     * Topic name.
     */
    String topic();

    /**
     * Control how to selector message.
     *
     * @see SelectorType
     */
    SelectorType selectorType() default SelectorType.TAG;

    /**
     * Control which message can be select. Grammar please see {@link SelectorType#TAG} and {@link SelectorType#SQL92}
     */
    String selectorExpression() default "*";

    /**
     * Control consume mode, you can choice receive message concurrently or orderly.
     */
    ConsumeMode consumeMode() default ConsumeMode.CONCURRENTLY;

    /**
     * Control message mode, if you want all subscribers receive message all message, broadcasting is a good choice.
     */
    MessageModel messageModel() default MessageModel.CLUSTERING;

    /**
     * Max consumer thread number.
     */
    int consumeThreadMax() default 64;

    /**
     * Max consumer timeout, default 30s.
     */
    long consumeTimeout() default 30000L;

    /**
     * The property of "access-key".
     */
    String accessKey() default ACCESS_KEY_PLACEHOLDER;

    /**
     * The property of "secret-key".
     */
    String secretKey() default SECRET_KEY_PLACEHOLDER;

    /**
     * Switch flag instance for message trace.
     */
    boolean enableMsgTrace() default true;

    /**
     * The name value of message trace topic.If you don't config,you can use the default trace topic name.
     */
    String customizedTraceTopic() default TRACE_TOPIC_PLACEHOLDER;

    /**
     * The property of "name-server".
     */
    String nameServer() default NAME_SERVER_PLACEHOLDER;

    /**
     * The property of "access-channel".
     */
    String accessChannel() default ACCESS_CHANNEL_PLACEHOLDER;
}
```

遗憾的是，我们可以看到Spring Boot的整合目前还没有增加对 NameSpace 的支持。

上面的两个接口都没有提供向broker的返回值，也就是我们在DefaultPushMQConsumer中使用registerMessageListener方法提供的Listener的consumeMessage()方法的返回值。通过查看源代码，我们可以得知以下事实：

1. 内部实现中使用DefaultPushMQConsumer来消费消息。
2. 如果我们提供的`RocketMQListener`或者`RocketMQReplyListener`没有抛出异常，则向broker返回消息消费成功的状态，即根据并发处理模式还是顺序处理模式，分别有可能返回`ConsumeConcurrentlyStatus.CONSUME_SUCCESS`或者`ConsumeOrderlyStatus.SUCCESS`。假如抛出了异常，那么将向broker返回`ConsumeConcurrentlyStatus.RECONSUME_LATER`或者`ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT`。

为了增加对Consumer的控制，我们可以让我们的类在实现`RocketMQListener`或`RocketMQReplyListener`的同时，实现`RocketMQPushConsumerLifecycleListener`，对Spring Boot整合的`DefaultRocketMQPushConsumer`提供更多的自定义能力，如下[官方示例](https://github.com/apache/rocketmq-spring/blob/master/rocketmq-spring-boot-samples/rocketmq-consume-demo/src/main/java/org/apache/rocketmq/samples/springboot/consumer/MessageExtConsumer.java)

```java
@Service
@RocketMQMessageListener(topic = "${demo.rocketmq.msgExtTopic}", selectorExpression = "tag0||tag1", consumerGroup = "${spring.application.name}-message-ext-consumer")
public class MessageExtConsumer implements RocketMQListener<MessageExt>, RocketMQPushConsumerLifecycleListener {
    @Override
    public void onMessage(MessageExt message) {
        System.out.printf("------- MessageExtConsumer received message, msgId: %s, body:%s \n", message.getMsgId(), new String(message.getBody()));
    }

    @Override
    public void prepareStart(DefaultMQPushConsumer consumer) {
        // set consumer consume message from now
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_TIMESTAMP);
        consumer.setConsumeTimestamp(UtilAll.timeMillisToHumanString3(System.currentTimeMillis()));
    }
}
```

#### application.properties中的配置

```properties
# 必要的参数非常简单
# 这是Producer和Consumer都需要的参数
rocketmq.name-server=localhost:9876
# accessChannel的含义下述
rocketmq.access-channel=LOCAL

# producer 专属设置
rocketmq.producer.group=my-group1
rocketmq.producer.sendMessageTimeout=300000
rocketmq.producer.retry-times-when-send-failed=2
# 这是 ACL，即权限控制相关代码
rocketmq.producer.access-key=mockAccessKey
rocketmq.producer.secret-key=mockSecretKey
rocketmq.producer.compress-message-body-threshold=1024*4
# 消息轨迹是rocketMQ自4.4.0版本开始支持的一个特性，下详述
rocketmq.producer.customized-trace-topic=my-trace-topic-name
rocketmq.producer.enable-msg-trace=true
rocketmq.producer.max-message-size=1024*1024*4
rocketmq.producer.retry-next-server=false
rocketmq.producer.retry-times-when-send-async-failed=2
rocketmq.producer.send-message-timeout=30000

# consumer 专属设置，用来启用/禁用一些listener，非常合理的设计
rocketmq.consumer.listeners.consumegroup1.topic1=true
rocketmq.consumer.listeners.consumegroup1.topic2=false
# 这两个参数用的并不是Spring-Properties-Processor的机制，而是一个SPEL的trick
# 即在RocketMQMessageListener中将accessKey属性的默认值设置为一个SPEL
# String SECRET_KEY_PLACEHOLDER = "${rocketmq.consumer.secret-key:}"
# 我觉得这种方式不太好，因为令人疑惑
rocketma.consumer.access-key=AK
rocketma.consumer.secret-key=SK
```

1. access-channel，似乎用于追踪messageTrace

```java
/**
 * Used for set access channel, if need migrate the rocketmq service to cloud, it is We recommend set the value with
 * "CLOUD". otherwise set with "LOCAL", especially used the message trace feature.
 */
public enum AccessChannel {
    /**
     * Means connect to private IDC cluster.
     */
    LOCAL,

    /**
     * Means connect to Cloud service.
     */
    CLOUD,
}
```

2. 消息轨迹。请看[消息轨迹](http://wuwenliang.net/2019/07/04/跟我学RocketMQ之消息轨迹实战与源码分析/)

3. ACL (access control list) 相关功能，请参见[官方文档](https://github.com/apache/rocketmq/blob/master/docs/cn/acl/user_guide.md)。简单地讲，broker要开启这个功能，就要修改`distribution/conf/plain_acl.yml`文件里的相关参数，并且在broker的config中设置下列属性：

   ```properties
   aclEnable=true
   ```

4. 其他参数含义明确，不再赘述。

## 分布式事务

这一块东西太多，我对基础知识都不了解，先搁置。