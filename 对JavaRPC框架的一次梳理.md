# 对JavaRPC框架的一次梳理

## 一些基本概念

### 参考链接

[知乎:如何给老婆解释什么是RPC](https://zhuanlan.zhihu.com/p/36427583)

[简书:什么是RPC](https://www.jianshu.com/p/7d6853140e13)

[2020-3-18：如何编写高性能RPC框架](https://blog.csdn.net/wufaliang003/article/details/79344357)

[dubbo官网:为什么要用dubbo](http://dubbo.apache.org/zh-cn/docs/user/preface/background.html)

### 概念解读

1. RPC

   Remote Procedure Call，即远程过程调用。简单来说说就是调用者和被调用者不在同一个地址空间，真正函数功能的运算不是在本地，而是在远端实现，调用者和被调用者通过网络进行通信的一种方式。

2. 为什么要使用RPC

   在一个小型系统里，最典型的比如一个本地实现的离线桌面软件，所有的运算都在当前进程内部调用和执行，调用者和被调用者处于同一个进程。这是小型系统最常见的一种代码组织方式。

   但随着代码规模的膨胀，小型系统逐渐膨胀为大型系统，运算量需求不断加大，单机的性能达到瓶颈，分布式的需求开始凸显。此时系统中一部分运算转移到其他主机，远程过程调用的需求出现了。此外，在现代大型系统里，一部分代码可能由js书写，比如网页前端，一部分由java书写，比如网页后端，而另外一部分由python写，比如算法模块。不同的代码由于ABI不兼容，不方便整合成一个进程，而必然存在调用其他地址空间代码的需求。

   可见，RPC本身只是一个概念，与具体的实现方式并没有绑定。

   对我上面讲述的发展脉络，[dubbo官网:为什么要用dubbo](http://dubbo.apache.org/zh-cn/docs/user/preface/background.html)这篇文章梳理地更清晰，我摘抄如下：

   > 随着互联网的发展，网站应用的规模不断扩大，常规的垂直应用架构已无法应对，分布式服务架构以及流动计算架构势在必行，亟需一个治理系统确保架构有条不紊的演进。
   >
   > **单一应用架构**
   >
   > 当网站流量很小时，只需一个应用，将所有功能都部署在一起，以减少部署节点和成本。此时，用于简化增删改查工作量的数据访问框架(ORM) 是关键。
   >
   > **垂直应用架构**
   >
   > 当访问量逐渐增大，单一应用增加机器带来的加速度越来越小，将应用拆成互不相干的几个应用，以提升效率。此时，用于加速前端页面开发的Web框架(MVC) 是关键。
   >
   > **分布式服务架构**
   >
   > 当垂直应用越来越多，应用之间交互不可避免，将核心业务抽取出来，作为独立的服务，逐渐形成稳定的服务中心，使前端应用能更快速的响应多变的市场需求。
   >
   > 此时，用于提高业务复用及整合的分布式服务框架(RPC) 是关键。
   >
   > **流动计算架构**
   >
   > 当服务越来越多，容量的评估，小服务资源的浪费等问题逐渐显现，此时需增加一个调度中心基于访问压力实时管理集群容量，提高集群利用率。
   >
   > 此时，用于提高机器利用率的资源调度和治理中心(SOA) 是关键。

3. RPC调用和本地调用有什么不同。

   本地调用因为在一个进程内，遵循函数调用的基本范式，如下代码：

   ```cpp
   int Multiply(int l, int r) {
      int y = l * r;
      return y;
   }
   
   int lvalue = 10;
   int rvalue = 20;
   int l_times_r = Multiply(lvalue, rvalue);
   ```

   最后一行的代码执行时实际发生了这些工作：

   > 1. 将 lvalue 和 rvalue 的值压栈
   > 2. 进入Multiply函数，取出栈中的值10 和 20，将其赋予 l 和 r
   > 3. 执行第2行代码，计算 l * r ，并将结果存在 y
   > 4. 将 y 的值压栈，然后从Multiply返回
   > 5. 第8行，从栈中取出返回值 200 ，并赋值给 l_times_r

   当这个方法存在于远程时，调用方向被调用方发送请求，首先需要告诉被调用端调用的是哪个函数，之后需要将参数序列化后传给被调用端，被调用端经过计算返回后，将结果再序列化后返回调用端，调用端对此进行反序列化，得到所需要的结果。

   大概可以写出下面的伪代码：

   ```cpp
   int l_times_r = Call(ServerAddr, Multiply ,lvalue, rvalue)
   ```

4. RPC大致需要提供哪些功能

   因为RPC的需求起于分布式，所以RPC的框架需要解决的几个基本问题应该包括：

   + 有一个或多个调用方（consumer）
   + 有一个或多个被调用方（provider）
   + 有一个注册中心（register）
   + （可选的）有一个数据统计中心，可以用于检测系统运行状况。（monitor）

   实际上，这差不多就是Dubbo框架的基本构成。

   另外，对于一个RPC框架，最好是让被调用方使用起来和本地调用一样，感知不到是一个远端调用，那就更加excited了。

   此外，当然还需要考虑性能问题等等。

5. RestAPI与RPC

   从本质上讲，RestAPI是RPC的一种实现方式。一般来讲：

   > RestAPI通常以业务为导向，将业务对象上执行的操作映射到HTTP动词，格式非常简单，可以使用浏览器进行扩展和传输，通过JSON数据完成客户端和服务端之间的消息通信，直接支持请求/响应方式的通信。不需要中间的代理，简化了系统的架构，不同系统之间只需要对JSON进行解析和序列化即可完成数据的传递。

   RestAPI有不少优势，比如架构比较简单，HTTP是现有的协议，Json的序列化和反序列化在各语言中都有完善的支持。因此也得到了广泛的应用，比如spring cloud就是使用Http协议作为通信协议。

   但Rest也存在一些缺点，比如：

   > 比如只支持请求/响应这种单一的通信方式，对象和字符串之间的序列化操作也会影响消息传递速度，客户端需要通过服务发现的方式，知道服务实例的位置，在单个请求获取多个资源时存在着挑战，而且有时候很难将所有的动作都映射到HTTP动词。

   假如没有开启Http的长连接支持，每次请求都会重新进行TCP握手，开销也比较大。序列化和反序列化采用的是字符串的表达方式，有时候也会受限，比如说要传递一些二进制数据，等等。因此，一些使用自有序列化反序列化机制、自有通信机制的RPC框架逐渐流行起来。

### 一次不完整的RPC流程

1. 注册中心启动，开放注册功能
2. 服务端 启动，到注册中心进行注册(register)，声明自己可用、上报自己地址等
3. 客户端 启动，到注册中心查找(lookup)可用服务端，获取其地址。在某些实现中，客户端在注册中心也进行注册，这样服务端的情况发生变化时，注册中心可以将结果推送到客户端。
4. 客户端 获取到 UserService 接口的 Refer: userServiceRefer
5. 客户端 调用 userServiceRefer.verifyUser(email, pwd)
6. 客户端 获取到 请求方法 和 请求数据
7. 客户端 把 请求方法 和 请求数据 序列化为 传输数据
8. 进行网络传输
9. 服务端 获取到 传输数据
10. 服务端 反序列化获取到 请求方法 和 请求数据
11. 服务端 获取到 UserService 的 Invoker: userServiceInvoker
12. 服务端 userServiceInvoker 调用 userServiceImpl.verifyUser(email, pwd) 获取到 响应结果
13. 服务端 把 响应结果 序列化为 传输数据
14. 进行网络传输
15. 客户端 接收到 传输数据
16. 客户端 反序列化获取到 响应结果
17. 客户端 userServiceRefer.verifyUser(email, pwd) 返回 响应结果

## 手撸一个简易的RPC

首先是Server端，漏洞很多，比如根本没有考虑到方法重载等，但意思是那么个意思。

基础类Student

```java
package com.lifeStory.myRPC;

import lombok.Data;
import java.io.Serializable;

@Data
public class Student implements Serializable {
    private static final long serialVersionUID = 1L;
  
    private String id;
    private String name;
    private int age;
}
```

Server端：

```java
package com.lifeStory.myRPC;

import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.HashMap;
import java.util.Map;

public class Server {

    private final Map<String, Method> methodMap = new HashMap<>();

    public Server() {

        for (Method method : Server.class.getDeclaredMethods()) {
            methodMap.putIfAbsent(method.getName(), method);
        }
    }

    public void start() throws IOException {
        ServerSocket serverSocket = new ServerSocket(9527);
        // 接收到客户端连接请求之后为每个客户端创建一个新的线程进行链路处理
        while (true) {
            try {
                // 每一个新的连接都创建一个线程，负责读取数据
                Socket socket = serverSocket.accept();
                new Thread(() -> {
                    try (ObjectOutputStream oos = new ObjectOutputStream(socket.getOutputStream());
                         ObjectInputStream ois = new ObjectInputStream(socket.getInputStream())) {
                        String methodName = ois.readUTF();
                        Method method = methodMap.get(methodName);
                        Object[] arguments = (Object[]) ois.readObject();
                        Object result = method.invoke(this, arguments);
                        oos.writeObject(result);
                    } catch (IOException | ClassNotFoundException | IllegalAccessException | InvocationTargetException e) {
                        e.printStackTrace();
                    }
                }).start();
            } catch (IOException e) {
                //
            }

        }
    }

    public static void main(String[] args) throws IOException {

        Server server = new Server();
        server.start();

    }

    private Student addAge(Student s, int deltaAge) {
        s.setAge(s.getAge() + deltaAge);
        return s;
    }

    private Student setName(Student s, String name) {
        s.setName(name);
        return s;
    }

}
```

然后是Client 1.0版：

```java
package com.lifeStory.myRPC;

import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.net.InetSocketAddress;
import java.net.Socket;

public class RPCClient {

    public Student addAge(Student student, int deltaAge) {
        try (Socket socket = new Socket("127.0.0.1", 9527)) {
            ObjectOutputStream oos = new ObjectOutputStream(socket.getOutputStream());
            ObjectInputStream ois = new ObjectInputStream(socket.getInputStream());
            oos.writeUTF("addAge");
            oos.writeObject(new Object[]{student, deltaAge});
            oos.flush();
            return (Student) ois.readObject();
        } catch (IOException | ClassNotFoundException e) {
            throw new RuntimeException("连接错误", e);
        }
    }

    private Student setName(Student student, String name) {
        try (Socket socket = new Socket("127.0.0.1", 9527);
             ObjectOutputStream oos = new ObjectOutputStream(socket.getOutputStream());
             ObjectInputStream ois = new ObjectInputStream(socket.getInputStream())) {
            // 读取数据
            oos.writeUTF("setName");
            oos.writeObject(new Object[]{student, name});
            oos.flush();
            return (Student) ois.readObject();
        } catch (IOException | ClassNotFoundException e) {
            throw new RuntimeException("连接错误", e);
        }
    }

    public static void main(String[] args) {

        Student student = new Student();
        student.setAge(18);
        student.setName("小马");
        student.setId("2008012826");

        RPCClient rpcClient = new RPCClient();
        System.out.println(rpcClient.addAge(student, 2));
        System.out.println(rpcClient.setName(student, "小刘"));

    }

}
```

这么写看起来一点也不通用，重复代码太多，需要手动实现那么多代码，完全不能认为是无感，因此我们将这玩意加个动态代理，看起来更通用一些。

Client2.0版：

```java
package com.lifeStory.myRPC;

import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Proxy;
import java.net.InetSocketAddress;
import java.net.Socket;

public class RPCClient {

    @SuppressWarnings("unchecked")
    public static <T> T getRemoteProxyObj(final Class<T> serviceInterface, final InetSocketAddress addr) {
        // 1.将本地的接口调用转换成JDK的动态代理，在动态代理中实现接口的远程调用
        return (T) Proxy.newProxyInstance(serviceInterface.getClassLoader(), new Class<?>[]{serviceInterface},
            (proxy, method, args) -> {
                Socket socket = null;
                ObjectOutputStream output = null;
                ObjectInputStream input = null;
                try {
                    socket = new Socket();
                    socket.connect(addr);

                    // 3.将远程服务调用所需的接口类、方法名、参数列表等编码后发送给服务提供者
                    output = new ObjectOutputStream(socket.getOutputStream());
                    output.writeUTF(method.getName());
                    output.writeObject(args);

                    // 4.同步阻塞等待服务器返回应答，获取应答后返回
                    input = new ObjectInputStream(socket.getInputStream());
                    return input.readObject();
                } finally {
                    if (socket != null) {
                        socket.close();
                    }
                    if (output != null) {
                        output.close();
                    }
                    if (input != null) {
                        input.close();
                    }
                }
            });
    }

    public static void main(String[] args) {
        // 创建各个空对象
        Student student = new Student();
        student.setAge(18);
        student.setName("小马");
        student.setId("2008012826");

        StudentOperator operator = RPCClient.getRemoteProxyObj(StudentOperator.class, new InetSocketAddress("127.0.0.1", 9527));
        System.out.println(operator.addAge(student, 2));
        System.out.println(operator.setName(student, "小刘"));

    }

    public interface StudentOperator {
        Student addAge(Student student, int deltaAge);
        Student setName(Student student, String name);
    }
}
```

不再深究了，大概就是这么个意思。如果要稍微呈现完整的RPC功能，需要实现一个类似注册中心的东西，做provider和consumer的register，但实现起来太麻烦了，不做了那就。

## JAVA RMI是什么

### 参考链接

[简书:理解Java RMI 一篇就够](https://www.jianshu.com/p/5c6f2b6d458a)

[CSND:JAVA RMI 原理和使用浅析](https://blog.csdn.net/qq_28081453/article/details/83279066)

### 正文

所谓Java RMI，就是Java RMI（Java Remote Method Invocation），即Java远程方法调用。是Java编程语言里，一种用于实现远程过程调用的应用程序**编程接口**。RMI 使用 JRMP（Java Remote Message Protocol，Java远程消息交换协议）实现，使得客户端运行的程序可以调用远程服务器上的对象。是实现RPC的一种方式。用于不同虚拟机之间的通信，这些虚拟机可以在不同的主机上、也可以在同一个主机上。

对此有一些重要的概念，比如客户端的Stub、服务器端的Skeleton、远程对象等，听起来比较玄乎，但理解起来很简单。

所谓Stub，就是Client端对于业务Interface的一种代理，类似我们手撸代码里的RPCClient里的那个静态方法所生成的动态代理，负责序列化、连接远端服务、反序列化之类的功能，但它是由Java自动生成的，我们不用实现。

所谓Skeleton，就是服务端对本地业务class的一种代理，负责接受请求、反序列化、调用本地业务class实例的相应方法，并将结果序列化，传输回客户端的stub。

我们依然以上面的Student为例，使用RMI写出来的代码是这样的：

Server和Client都需要的，显然是业务接口了：

```java
import java.rmi.Remote;
import java.rmi.RemoteException;

// 必须扩展Remote接口，否则不生成桩代码
public interface StudentService extends Remote {

  	// 必须抛出 RemoteException，不然会报错
    Student addAge(Student student, int deltaAge) throws RemoteException;
    Student setName(Student student, String name) throws RemoteException;

}
```

Server端，业务接口的实现只需要server端有就可以：

```java
import java.rmi.RemoteException;
import java.rmi.server.UnicastRemoteObject;

// 必须扩展 UnicastRemoteObject
public class StudentServiceImpl extends UnicastRemoteObject implements StudentService {

    public StudentServiceImpl() throws RemoteException {
        super();
    }

    @Override
    public Student addAge(Student student, int deltaAge) {
        student.setAge(student.getAge() + deltaAge);
        return student;
    }

    @Override
    public Student setName(Student student, String name) {
        student.setName(name);
        return student;
    }

}
```

Server端启动RMI的代码，有两种风格：

```java
import java.net.MalformedURLException;
import java.rmi.AlreadyBoundException;
import java.rmi.Naming;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;

// 一种是这样
public class RMIServer {

    public static void main(String[] args) throws RemoteException, AlreadyBoundException, MalformedURLException {
        StudentServiceImpl studentService = new StudentServiceImpl();
        LocateRegistry.createRegistry(1099);
        Naming.bind("student", studentService);
    }
}

// 第二种是这样
public class RMIServer {

    public static void main(String[] args) throws RemoteException, AlreadyBoundException, MalformedURLException {
        StudentServiceImpl studentService = new StudentServiceImpl();
        LocateRegistry.createRegistry(1099);
        Registry registry = LocateRegistry.getRegistry();
        registry.bind("student", studentService);
    }
}
```

要注意的是，其实`Naming.bind()`看起来更像是一个shortcut，实际上还是引用了`Registry`。其实Naming.bind()看源码以后，感觉正确用法应该是`Naming.bind("rmi://127.0.0.1/student", studentService);`

而`Registry`也有大量的默认设置，getRegistry()方法就默认连接本地1099接口的`Registry`。为了验证这一点，我们将代码拆开为三个部分，如下：

```java
// 第一部分，注册中心
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;

public class RMIRegistry {
    public static void main(String[] args) throws RemoteException, InterruptedException {
        LocateRegistry.createRegistry(1099);
      	// 否则会直接退出
        System.in.read();
    }
}

// 第二部分，Server端
import java.net.MalformedURLException;
import java.rmi.AlreadyBoundException;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class RMIServer {

    public static void main(String[] args) throws RemoteException, AlreadyBoundException, MalformedURLException {
        StudentServiceImpl studentService = new StudentServiceImpl();
        Registry registry = LocateRegistry.getRegistry("127.0.0.1", 9527);
        registry.bind("student", studentService);
    }
}

// 第三部分，Client端
import java.net.MalformedURLException;
import java.rmi.NotBoundException;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class RMIClient {

    public static void main(String[] args) throws RemoteException, NotBoundException, MalformedURLException {
        Student student = new Student();
        student.setAge(18);
        student.setName("小马");
        student.setId("2008012826");

        Registry registry = LocateRegistry.getRegistry("127.0.0.1", 9527);
        StudentService studentService = (StudentService) registry.lookup("student");
        System.out.println(studentService.addAge(student, 2));
        System.out.println(studentService.setName(student, "小胡"));
    }
}
```

这样也可以运行起来。但是假如我们想做个负载均衡，即在两台机器上都运行`StudentServiceImpl`的实例，都注册到`student`路径上，这是不可行的，会抛出`AlreadyBoundException`，这显然也是Java RMI的重要局限性之一。

这里还要注意的是，Service和Client的业务接口的包路径都必须完全一致。

### JAVA RMI的优势和劣势

优势已经很明显了，就是极大地解放了我们的代码，不用像我手撸的那个版本一样处理这么多与通信、序列化反序列化相关的细节，也不用维护所谓MethodID和Method的对应之类的。

但是显然JAVA RMI有很大的局限性，局限性大到它根本没有成为主流。

首先，我们在获取Registry时就要指定IP和端口，从我个人角度来看，至少有以下几个问题

1. 注册中心本身是没有保活机制的，如果死了，那就完了，当然可以做多个注册中心，但是有几个注册中心就注册几次嘛，Client从哪个注册中心取呢？
2. 注册中心本身并没有对同路径的负载均衡机制，也就是说，比如我们不能把两个Server注册到同一个path上，这显然无法应对大规模流量，算不上支持分布式。
3. RMI是Java语言的远程调用，通信双方要求都必须用Java，显然不适应现在大型系统的实际。
4. RMI要求接口必须扩展`Remote`，而且所有方法都必须抛出`RemoteException`，这限制了对旧系统的改造。

因此，Java RMI作为一个了解项目即可，实际上也从来没有走向主流。

## RPC框架速览

### 参考资料

[2017-9-21:知乎:分布式RPC框架性能大比拼](https://zhuanlan.zhihu.com/p/80068285)

[2017-5-16:RPC框架调查](https://cwiki.apache.org/confluence/display/GEODE/RPC+framework+evaluation)

[2018-4-14:The Rough Guide to Java RPC Frameworks](https://speakerdeck.com/galderz/the-rough-guide-to-java-rpc-frameworks)

[2018-2-10:CSDN:流行的rpc框架性能测试对比](https://blog.csdn.net/quuqu/article/details/79304614)

[2020-3-18：如何编写高性能RPC框架](https://blog.csdn.net/wufaliang003/article/details/79344357)  看起来是一篇神文

这篇文章中列出的几个框架，如dubbo、motan，基于go语言的rpcx，跨语言的gRPC和thrift，同时，还有其他文章提到了一个Hessian，以及其后续版本Hessian2。经过简单的梳理，发现这些框架提供的功能其实并不完全重合，有些功能范围要少，有些要很大。比如Hessian，更多地被认为是序列化和反序列化协议，等等。对此，我现在还没有全面的认识，接下来逐一进行梳理。

有意思的是，可能我搜索的是RPC框架，搜索结果里鲜有提到提到SpringCloud，但其实dubbo经常拿来和SpringCloud对比，就两者解决的问题来看，实际上确实有大量的重叠，因此后面我会很肤浅地讨论SpringCloud的问题，但讨论到SpringCloud和微服务管理，就肯定得提到docker和k8s，k8s野心勃勃，servicemesh自2018年开始大火，我们也做肤浅地讨论，更深入的讨论可能得等我哪天有精力之后再做。

## Hessian and Hessian2

### 参考资料

[Hessian官网](http://hessian.caucho.com/)

[2015-2-2:cnBlogs:Hessian 原理分析](https://www.cnblogs.com/happyday56/p/4268249.html)

基于上述资料，我们可以这么说，Hessian 是 caucho 开发的一种二进制 WebService 协议，用来实现Web服务，使用简单的方法提供了序列化、反序列化和RMI功能。他是一个跨语言的二进制协议，通过Http封包传输。

它的优点包括：跨语言、跨平台、二进制封装因此体积较小，通过Http封包因此不容易受到防火墙的限制。

我们也因此可见，Hessian是一个`low-level`的框架/协议，并非一个all-in-one的RPC框架，我们实现一个简单的demo，可见其使用很类似于`Java RMI`。实际上，Hessian经常在其他RPC框架中被当做序列化和反序列化工具，比如说dubbo。而Hessian2直接将序列化和反序列化模块与web模块进行了拆分，分别出具了协议标准化文档，网页分别如下[Hessian 2.0 Serialization Protocol](http://hessian.caucho.com/doc/hessian-serialization.html)，[Hessian 2.0 Web Services Protocol](http://hessian.caucho.com/doc/hessian-ws.html)

### demo

为演示其跨语言能力，我们首先用java实现完整的服务端和客户端，然后用python再实现一遍，并跨语言连接。

#### java

server端，参考[SpringBoot整合Hessian](https://www.jianshu.com/p/9136aa36cffb)

因为这玩意实际上是写Servlet，为了避免还要启动tomcat、servlet、mapping.xml，我们直接将Hessian整合到SpringBoot中，毕竟考古这种事情干个一两次知道怎么回事就行了

1. pom文件

   ```xml
   <dependencies>
   
       <dependency>
           <groupId>com.caucho</groupId>
           <artifactId>hessian</artifactId>
           <version>4.0.63</version>
       </dependency>
   
       <dependency>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-starter-web</artifactId>
       </dependency>
   
   </dependencies>
   ```

2. 接口与实现

   ```java
   public interface StudentService {
   
       Student addAge(Student student, int deltaAge);
   
       Student setName(Student student, String name);
   
   }
   
   public class StudentServiceImpl implements StudentService {
   
       @Override
       public Student addAge(Student student, int deltaAge) {
           student.setAge(student.getAge() + deltaAge);
           return student;
       }
   
       @Override
       public Student setName(Student student, String name) {
           student.setName(name);
           return student;
       }
   
   }
   ```

3. SpringConfig

   ```java
   @Configuration
   public class HessianConfigure {
   
     	// HessianServiceExporter 是 Spring 提供的包
     	// 我们的Bean名看起来就像个路径，实际上它真的是个路径
       @Bean("/studentHessian")
       public HessianServiceExporter exportHelloHessian() {
           HessianServiceExporter exporter = new HessianServiceExporter();
           exporter.setService(new StudentServiceImpl());
           exporter.setServiceInterface(StudentService.class);
           return exporter;
       }
   }
   ```

4. SpringMain

   ```java
   @SpringBootApplication
   public class HessianServer {
       public static void main(String[] args) {
           SpringApplication.run(HessianServer.class);
       }
   }
   ```

客户端

```java
public class HessianClient {

    @SuppressWarnings("unchecked")
    public static <T> T getHessianClientBean(Class<T> clazz, String url) throws Exception {
        // 客户端连接工厂,这里只是做了最简单的实例化，还可以设置超时时间，密码等安全参数
        HessianProxyFactory factory = new HessianProxyFactory();
        return (T) factory.create(clazz, url);
    }


    public static void main(String[] args) {

        // 服务器暴露出的地址
        String url = "http://localhost:8080/studentHessian";

        // 客户端接口，需与服务端对象一样
        try {
            StudentService studentService = HessianClient.getHessianClientBean(StudentService.class, url);
            Student student = new Student();
            student.setAge(18);
            student.setName("小马");
            student.setId("2008012826");

            System.out.println(studentService.addAge(student, 2));
            System.out.println(studentService.setName(student, "小胡"));

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### python

client端，连接刚才Java写的Server，python3准备工作：`pip install python-hessian`

其官方github包在这里：[bgilmore/mustaine](https://github.com/bgilmore/mustaine)

```python
import json
from pyhessian.client import HessianProxy


def run():
  url = "http://localhost:8080/studentHessian"
  param = {
      "student": {
          "id": "2008012826",
          "name": "小马",
          "age": 18
      },
      "deltaAge": 2
  }

  service = HessianProxy(url)
  # 这里的操作就比较骚了
  response = service.addAge(param['student'], param['deltaAge'])
  response2 = service.setName(param['student'], '小胡')
  return (response, response2)

if __name__ == '__main__':
  for s in run():
    print(s.id, s.name, s.age)
```

而Hessian看起来并没有提供py库来实现server端，我们就不探究了。总之跨语言看起来很美，实则，至少在java和python之间也就那么回事了。

## let's try dubbo

接下来我们看看dubbo，之前面试时就被问起来，一脸懵逼，这次好歹搞个表面文章。

### 参考资料

[2019-7-8:知乎:阿里巴巴为什么主推HSF?比Dubbo有哪些优势?](https://www.zhihu.com/question/39560697/answer/741819355)

### dubbo简介

如果在网上搜dubbo的相关信息，会发现不少帖子将dubbo与HSF（High-speed Service Framework）这个分布式RPC服务框架进行比较，并有很多帖子说Dubbo属于被阿里放弃的RPC框架等等。我们在这里简单梳理一下Dubbo的发展脉络，以及与HSF的关系。

> 开源期(2011-2013，横空出世)：dubbo是阿里巴巴B2B开发的，2011年开源。hsf是淘宝开发的，要早2-3年。大家知道，淘系（淘宝、天猫、一淘/阿里妈妈等，甚至包括阿里云）发展的比阿里巴巴B2B要好的多。高并发服务化的场景其实也主要在淘系，而且淘宝的中间件团队比较强大。所以大概在B2B从香港退市的时候，B2B团队的研发面临跟淘系包括支付宝进行合并的过程，这个时候dubbo与hsf两个团队进行过一次”PK“，PK的结果是dubbo的部分功能合并到hsf，dubbo团队成员基本上都分流到了其他业务团队，比如梁飞（花名虚极）转岗到天猫客户端团队。这个时期的一个明显特征是，dubbo墙里开花墙外香，框架特性丰富、功能成熟、使用方便，在分布式服务化领域，迅速成为业内的事实标准。
>
> 沉寂期(2013-2017，潜龙在渊)：dubbo团队分流以后，明显可以看到从2013年到2017年，dubbo的维护程度很低，社区活跃度也很低。这几年中，比较有亮点的是当当网做了rest协议的实现。当时这一时期也培养了一大批的国内使用dubbo的程序员，dubbo在开源圈的影响力一直在上升。当然，也错过了进一步发展的好机会。Spring Cloud项目逐步流行开来。
>
> 复兴期(2017-2019，朝花夕拾)：2017年8月份重启维护以来，作为阿里的一个开源的拳头产品，一方面引入公司内部外资源，迅速调整定位，修复这几年积累的问题，添加各种新的特性，以更加开放的心态引入大量外部贡献者，与阿里巴巴以及其他公司的开源技术整合（Sentinel，Seata，Nacos，Spring Cloud等），规划Dubbo3.0，引入Reactive MicroServices，拥抱ServiceMesh，不断完善以dubbo为核心的服务化生态，通过频繁的技术布道和技术运营建设开源社区，同时一直在探索dubbo与云基础设施之间的应用效应。加入Apache基金会，并与2019年5月顺利毕业，成为国人开源的一个新名片。

所以，现在搜Dubbo，可以看到是apache基金会的一个项目，其官方地址为：[apache-dubbo](http://dubbo.apache.org/zh-cn/)

至此，我们也就基本梳理清楚dubbo和hsf之间的关系了。鉴于dubbo仍然是国内使用非常广泛的RPC框架，并且重新进入了积极维护期，学一下不亏。

### demo first

在这里，我准备由浅入深地使用一下dubbo，因此会有多个实现，分别体验dubbo多层次的功能。当然，以下所有示例在可能的情况下都与Spring整合在一起。

and，我不得不吐槽dubbo的文档和源代码。是不是国内的开源都不喜欢写文档和Java Doc啊，相比Spring只看Java Doc就能搞清楚很多问题，dubbo连废弃一个方法都不带注释，不说替代方法的，简直无情。

#### 没有注册中心的直连

显然这种方式一般不会用在生产环境，因为如果架构这么简单，也用不到dubbo。但直连可能的一个使用场景如下:

> 在开发及测试环境下，经常需要绕过注册中心，只测试指定服务提供者。

API方式直接写

```java
// Server端
public class Dubbo1Server {

    public static void main(String[] args) throws IOException {

        ConfigManager configManager = ApplicationModel.getConfigManager();
        configManager.setApplication(new ApplicationConfig("test"));

      	// 不得不吐槽，dubbo将config的setApplication()标记为废弃，但
      	// 1没提供替代方法，2不能不调用该方法……不调用该方法的办法就是加上上面这两行- -
        ServiceConfig<StudentService> config = new ServiceConfig<>();
        config.setInterface(StudentService.class);
      	// 不允许不提供注册中心配置，但提供"N/A"表示不使用注册中心
        config.setRegistry(new RegistryConfig("N/A"));
        config.setRef(new StudentServiceImpl());
        config.setPath("/sth");
        config.export();

        System.in.read();
    }

}
```

```java
// Client端
public class Dubbo1Client {

    public static void main(String[] args) {

        ConfigManager configManager = ApplicationModel.getConfigManager();
        configManager.setApplication(new ApplicationConfig("test"));

        Student student = new Student();
        student.setAge(18);
        student.setName("小马");
        student.setId("2008012826");

        ReferenceConfig<StudentService> config = new ReferenceConfig<>();
        config.setUrl("dubbo://localhost:20880/sth");
        config.setInterface(StudentService.class);
        StudentService studentService = config.get();
        System.out.println(studentService.addAge(student, 2));
        System.out.println(studentService.setName(student, "小胡"));

    }
}
```

#### 来个注册中心

这次用XML配置的方法，服务端XML

```xml
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
    <dubbo:application name="demo-provider"/>
  	<dubbo:registry address="zookeeper://127.0.0.1:2181"/>
    <dubbo:protocol name="dubbo" port="20880"/>
    <bean id="studentService" class="com.lifeStory.dubbo1.StudentServiceImpl"/>
    <dubbo:service interface="com.lifeStory.dubbo1.StudentService" ref="studentService"/>
</beans>
```

client端XML

```XML
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
    <dubbo:application name="demo-consumer"/>
    <dubbo:registry address="zookeeper://127.0.0.1:2181"/>
    <dubbo:reference id="demoService" check="false" interface="com.lifeStory.dubbo1.StudentService"/>
</beans>
```

服务端代码：

```java
public class XMLDubboServer {

    public static void main(String[] args) throws IOException {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("provider.xml");
        context.start();
        System.out.println("dubbo服务提供端已启动....");
        System.in.read(); // 按任意键退出
    }
  
}
```

客户端代码：

```java
public class XMLDubboClient {

    public static void main(String[] args) {

        Student student = new Student();
        student.setAge(18);
        student.setName("小马");
        student.setId("2008012826");

        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext( "consumer.xml" );
        context.start();
        StudentService studentService = (StudentService)context.getBean( "studentService" );// 获取远程服务代理
        System.out.println(studentService.addAge(student, 2));
        System.out.println(studentService.setName(student, "小胡"));
    }
  
}
```

#### 与SpringBoot的整合

这里比较复杂，我们将代码分为三个模块。其在maven中的组织结构如下图:

![springboot-dubbo-code-tree](https://www.history-of-my-life.com/imgs-for-md/spring-dubbo-code-tree.png)

其中，DubboInterface定义了通用接口和模型，DubboServer模块定义了Provider的行为，DubboClient模块定义了Consumer的行为，可以说Dubbo和Spring的整合相当好，我们将核心代码粘贴如下：

##### Server端

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>SpringDubboTest</artifactId>
        <groupId>com.lifeStory</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>DubboClient</artifactId>

    <dependencies>
        <dependency>
            <groupId>com.lifeStory</groupId>
            <artifactId>DubboInterface</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>

</project>
```

```yaml
# application.yml
dubbo:
  application:
    name: boot-duboo-provider
  registry:
    protocol: zookeeper
    address: 127.0.0.1:2181
  protocol:
    name: dubbo
  consumer:
    check: false
```

```java
package com.lifeStory.workers.impl;

import com.lifeStory.models.Student;
import com.lifeStory.workers.StudentService;

// 我这么写是刻意强调这两个 @Service 不是同一个注解
@org.springframework.stereotype.Service
@org.apache.dubbo.config.annotation.Service
public class StudentServiceImpl implements StudentService {

    @Override
    public Student addAge(Student student, int deltaAge) {
        student.setAge(student.getAge() + deltaAge);
        return student;
    }

    @Override
    public Student setName(Student student, String name) {
        student.setName(name);
        return student;
    }

}
```

```java
package com.lifeStory;

import org.apache.dubbo.config.spring.context.annotation.EnableDubbo;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@EnableDubbo
@SpringBootApplication
public class DubboServer {

    public static void main(String[] args) {
        SpringApplication.run(DubboServer.class, args);
    }

}
```

##### Client端

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>SpringDubboTest</artifactId>
        <groupId>com.lifeStory</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>

    <modelVersion>4.0.0</modelVersion>

    <artifactId>DubboServer</artifactId>

    <dependencies>
        <dependency>
            <groupId>com.lifeStory</groupId>
            <artifactId>DubboInterface</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>

</project>
```

```yml
# application.yml 可见和Server端没有区别，实际上，Dubbo并没有明确的Server和Client之分
# 从这个角度上讲似乎可以相互依赖，但Spring的依赖注入发生在启动时，所以相互依赖时，先启动的
# 怎么获得后启动的服务中的Service，我猜可以用AOP在发生变化时重新注入
dubbo:
  application:
    name: boot-dubbo-procider
  registry:
    protocol: zookeeper
    address: 127.0.0.1:2181
  protocol:
    name: dubbo
```

```java
@Service
public class StudentServiceConsumer {

    // 必须有的注解，否则整个SpringApplicationContext中不会有StudentService的Bean
  	// 根据我的猜测，可能是用AOP进行的运行时注入，这样在发生变化时StudentService的情况
  	// 也可以发生变化，这也是为什么@Reference不能用于构造函数的原因。（不负责的猜测）
    @Reference
    private StudentService studentService;

    public Student addAge(Student s, Integer deltaAge) {
        return studentService.addAge(s, deltaAge);
    }

    public Student setName(Student s, String name) {
        return studentService.setName(s, name);
    }

}
```

```java
@SpringBootApplication
@EnableDubbo
public class DubboClient {

    public static void main(String[] args) {

        Student student = new Student();
        student.setAge(18);
        student.setName("小马");
        student.setId("2008012826");

        ApplicationContext context = SpringApplication.run(DubboClient.class, args);
        StudentServiceConsumer studentService = context.getBean(StudentServiceConsumer.class);
        System.out.println(studentService.addAge(student, 2));
        System.out.println(studentService.setName(student, "小胡"));
    }

}
```

##### 最外层的parent Pom

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.lifeStory</groupId>
    <artifactId>SpringDubboTest</artifactId>
    <packaging>pom</packaging>
    <version>1.0-SNAPSHOT</version>
    <modules>
        <module>DubboServer</module>
        <module>DubboClient</module>
        <module>DubboInterface</module>
    </modules>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.2.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <java.version>1.8</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <maven.compiler.target>1.8</maven.compiler.target>
        <maven.compiler.source>1.8</maven.compiler.source>
        <dubbo.version>2.7.6</dubbo.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.10</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-logging</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </dependency>

        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-spring-boot-starter</artifactId>
            <version>${dubbo.version}</version>
        </dependency>

        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo</artifactId>
            <version>${dubbo.version}</version>
        </dependency>

        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-dependencies-zookeeper</artifactId>
            <version>${dubbo.version}</version>
            <type>pom</type>
            <exclusions>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>*</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>log4j</groupId>
                    <artifactId>*</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

    </dependencies>

</project>
```

#### 加个 monitor 吧

dubbo官方有提供一个监控网页，即很多教程里写的`dubbo-admin`，现在这个github地址如下：[dubbo-admin](https://github.com/apache/dubbo-admin)。具体使用非常简单。以下几个简单的操作就可以实现

```shell
git clone git@github.com:apache/dubbo-admin.git
cd dubbo-admin
## 修改 dubbo-admin-server/src/main/resources/application.properties
server.port = 8080
admin.registry.address=zookeeper://127.0.0.1:2181
admin.config-center=zookeeper://127.0.0.1:2181
admin.metadata-report.address=zookeeper://127.0.0.1:2181

admin.root.user.name=root
admin.root.user.password=root
#group  #注释掉这一段
#admin.registry.group=dubbo
#admin.config-center.group=dubbo
#admin.metadata-report.group=dubbo

admin.apollo.token=e16e5cd903fd0c97a116c873b448544b9d086de9
admin.apollo.appId=test
admin.apollo.env=dev
admin.apollo.cluster=default
admin.apollo.namespace=dubbo

## package
mvn clean package -Dmaven.test.skip=true

## run
mvn --projects dubbo-admin-server spring-boot:run
```

之后访问[localhost:8080](localhost:8080)，使用root/root登陆，即可看到注册的服务，具体不截图了，可以自己体验一下。

不过这个项目之所以叫admin，就是因为除了统计数据以外，还可以做一些其他配置，这个需要对dubbo有比较深入的了解，我这里具体还没来得及探索。

### 认识dubbo的架构

[dubbo: 官方文档: 框架设计](http://dubbo.apache.org/zh-cn/docs/dev/design.html)

可能是个人习惯，我觉得这篇文档的顺序不是很合理，按我的想法调整一下，应该先讲这一段：

#### 依赖关系

![/dev-guide/images/dubbo-relation.jpg](http://dubbo.apache.org/docs/zh-cn/dev/sources/images/dubbo-relation.jpg)

从这里很清晰地看到了一个经典的RPC框架设计思路，服务提供者、消费者、注册中心和监控中心。

#### 调用链

而本文开始时描述的那个经典的RPC调用链路，则通过下图可以有个清晰的认识：

![dubbo调用链](http://dubbo.apache.org/docs/zh-cn/dev/sources/images/dubbo-extension.jpg)

这里面每一层旁边的文字概念，可以重温一下[2020-3-18：如何编写高性能RPC框架，对每一层的概念可能会有更深入的了解。用文字描述这一过程，就如下：

服务消费端调用的是一个代理对象，这个代理对象在相应方法被调用时，发生了查询可用提供端、负载均衡、条件过滤等，最终把调用方法信息和参数进行序列化，通过netty等IO框架进行IO，而接收端在接到请求时，先对相应数据进行反序列化，在工作线程池查询可用工作线程，在实际调用接口的实现时，该实现对象依然是一个被包装了的代理对象，在实际实现对象工作之前，代理对象中已经发生了不少事情。

#### 整体设计

![dubbo整体设计](http://dubbo.apache.org/docs/zh-cn/dev/sources/images/dubbo-framework.jpg)

对这个图的详细解释请自行查询文档，说的相当清晰。对于理解其他RPC框架应该说也是大有裨益。

### Best practice

显然对于我这种刚接触Dubbo的，没资格总结什么Best Practice，但是官网的确给出了一些开发的建议，在这里放出来，为了做一提醒。

[dubbo官网：服务化最佳实践](http://dubbo.apache.org/zh-cn/docs/user/best-practice.html)

[dubbo官网：推荐用法](http://dubbo.apache.org/zh-cn/docs/user/recommend.html)

[dubbo官网：容量规划](http://dubbo.apache.org/zh-cn/docs/user/capacity-plan.html)

另外，注册中心官方推荐使用Zookeeper，在上文的示例中，我的zookeeper是用docker启动的。

## Spring Cloud

其实和Dubbo直接对标的一些框架还是有的，比如说新浪的motan，跨语言的gRPC，但是这些要么影响力还不够，要么是其他语言的，我们做java的还是搞Spring系列吧。所以先搞SpringCloud。

那么SpringCloud和Dubbo到底有哪些不同？在知乎上的讨论倒是看起来比较完善，链接如下：

[spring cloud 和 dubbo 各自的优缺点是什么?](https://www.zhihu.com/question/45413135)

[2018-08-03:Java微服务框架选型,dubbo or spring cloud](https://cloud.tencent.com/developer/article/1177574)

[2019-2-16:ServiceMesh和负载均衡](https://www.jianshu.com/p/27a742e349f7)

对其中一些比较有代表性的观点摘录如下：

> Dubbo确实类似于Spring Cloud的一个子集，Dubbo功能和文档完善，在国内有很多的成熟用户
>
> Dubbo具有调度、发现、监控、治理等功能，**支持相当丰富的服务治理能力**。Dubbo架构下，注册中心对等集群，并会缓存服务列表已被数据库失效时继续提供发现功能，本身的服务发现结构有很强的**可用性与健壮性**，足够支持高访问量的网站。
>
> 虽然Dubbo 支持短连接大数据量的服务提供模式，但绝大多数情况下都是使用长连接小数据量的模式提供服务使用的。所以，**对于类似于电商等同步调用场景多并且能支撑搭建Dubbo 这套比较复杂环境的成本的产品而言，Dubbo 确实是一个可以考虑的选择**。但如果产品业务中由于后台业务逻辑复杂、时间长而导致异步逻辑比较多的话，可能Dubbo 并不合适。同时，对于人手不足的初创产品而言，这么重的架构维护起来也不是很方便。

> 提起Spring Cloud，一些开发的第一印象是Http+JSON的rest通信，性能上难堪重用，其实这也是一种误读。(但其他回答中的性能对比，确实会显示使用Http+Json+短连接的SpringCloud速度明显低于Dubbo，大概响应速度是dubbo的2到3倍)
>
> Spring Cloud也并不是和Http+JSON强制绑定的，如有必要Thrift、protobuf等高效的RPC、序列化协议同样可以作为替代方案

> SpringCloud提供了完备的微服务架构解决方案，包括网关、断路器、分布式配置、分布式追踪系统、信息总线、数据流、批量任务等等，而Dubbo主要是服务治理，功能完备性上相比SpringCloud要差一点。

> 如今国内微服务格局，SpringCloud和dubbo二分天下，servicemesh方兴未艾，云原生时代来临，微服务、容器技术和devops将变得越来越紧密，而未来，一定是一个百花齐放的时代。

> 如今2019年来看的话，我觉得其实选择更多了，且先不谈java以外的微服务平台，从框架层面来看目前国内和2年前仍然是sc和dubbo二分天下，这个格局仍然没有改变，这几年要说变化不得不提k8s的大火，作为平台级的容器编排事实标志，k8s一开始就站得更高，早已经具备了微服务架构中很多核心组件的生产级解决方案，后续顺势而出的servicemesh理念，istio等产品则目标瞄准了微服务框架的痛点，野心勃勃的把自己定位在下一代微服务的标签上。

> 我们现在所使用的 Spring Cloud 技术体系，实际上是 Spring Cloud Netflix 为主，例如说：
>
> - Netflix Eureka 注册中心
> - Netflix Hystrix 熔断组件
> - Netflix Ribbon 负载均衡
> - Netflix Zuul 网关服务
>
> 但是，开源的世界，总是这么有趣。目前 Alibaba 基于 Spring Cloud 的**接口**，对的是接口，实现了一套 [Spring Cloud Alibaba](https://link.zhihu.com/?target=https%3A//github.com/spring-cloud-incubator/spring-cloud-alibaba) 技术体系，并且已经获得 Spring Cloud 的认可，处于孵化状态。组件如下：
>
> - Nacos 注册中心 + 配置中心，对标 Eureka 。并且，还对标了 Spring Cloud Config 。
> - Sentinel 服务保障，对标 Hystrix 。
> - Dubbo 服务调用( 包括负载均衡 )，对标 Ribbon + Feign 。
> - **缺失** 网关服务。

> [spring cloud和dubbo的区别](https://blog.csdn.net/anningzhu/article/details/76599875)

> Dubbo使用RPC实现服务间调用，确实提高性能，但也存在一些痛点，比如说：
>
> 服务提供方与调用方接口依赖方式太强：我们为每个微服务定义了各自的service抽象接口，并通过持续集成发布到私有仓库中，调用方应用对微服务提供的抽象接口存在强依赖关系，因此不论开发、测试、集成环境都需要严格的管理版本依赖，才不会出现服务方与调用方的不一致导致应用无法编译成功等一系列问题，以及这也会直接影响本地开发的环境要求，往往一个依赖很多服务的上层应用，每天都要更新很多代码并install之后才能进行后续的开发。若没有严格的版本管理制度或开发一些自动化工具，这样的依赖关系会成为开发团队的一大噩梦。而REST接口相比RPC更为轻量化，服务提供方和调用方的依赖只是依靠一纸契约，不存在代码级别的强依赖，当然REST接口也有痛点，因为接口定义过轻，很容易导致定义文档与实际实现不一致导致服务集成时的问题，但是该问题很好解决，只需要通过每个服务整合swagger，让每个服务的代码与文档一体化，就能解决。所以在分布式环境下，REST方式的服务依赖要比RPC方式的依赖更为灵活。
>
> 服务对平台敏感，难以简单复用：通常我们在提供对外服务时，都会以REST的方式提供出去，这样可以实现跨平台的特点，任何一个语言的调用方都可以根据接口定义来实现。那么在Dubbo中我们要提供REST接口时，不得不实现一层代理，用来将RPC接口转换成REST接口进行对外发布。若我们每个服务本身就以REST接口方式存在，当要对外提供服务时，主要在API网关中配置映射关系和权限控制就可实现服务的复用了。

另外发现神文一篇，还未见得有时间看：[微服务的定义](https://martinfowler.com/articles/microservices.html)

不过到了SpringCloud这一步，实际上我们的讨论范围已经由RPC转化为微服务了，似乎有点不公平，不过既然都到了这里了，不妨稍微搞一搞。

// 但忽然发现偏离了本来研究数据库的主线，这个暂时放下吧