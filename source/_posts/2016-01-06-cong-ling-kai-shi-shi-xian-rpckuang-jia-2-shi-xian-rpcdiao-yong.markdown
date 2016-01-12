---
layout: post
title: "从零开始实现RPC框架--(2)实现RPC调用"
date: 2016-01-06 22:05:15 +0800
comments: true
categories: 中间件,rpc
---
#前言

在上一篇文章"[从零开始实现RPC框架--(1)基本原理][previous]"中，大致讲述了RPC框架的原理、可能遇到的问题及一些解决的方案。  
之后我实现了RPC框架的基础功能--客户端和服务端之间的调用，代码已经上传至github（[项目链接][project]）。    
这个版本tag名为"getting-through"，目前客户端选择服务端地址时暂时以常量形式设置，服务注册发现和负载均衡的特性会在下一个版本完成，本次主要功能放在客户端和服务端之间的调用过程的实现上。

#PRC调用的实现

RPC框架主要可以分为启动和服务调用两个阶段，下面分阶段分别来看客户端和服务端的实现：

##1.启动阶段   
###1.1 客户端启动--初始化代理类

####像本地方法一样调用
客户端在启动时，需要为客户端所依赖的各个service生成各自的proxy bean，并且借助spring在启动时注入到对应的业务组件中，proxy bean应该拥有和service完全相同的接口，来接管所有对service的方法调用

以一个sayHello例子为例，程序很简单，客户端告知我是“zxc”，服务端返回”Hello zxc!”：

HelloService接口：
```java
public interface HelloService {
     String sayHello(String name);
}
```

其实现类HelloServiceImpl：
```java
public class HelloServiceImpl implements HelloService {
     public String sayHello(String name) {
          return “Hello " + name;
     }
}
```

在进行远程调用的时候，把HelloService接口作为api的一部分，单独打jar包，再由客户端应用引用。

我们的目标要达到的效果是能够在客户端代码中自然的进行调用，效果如代码所示：
```java
public class SomeClientBiz {
    @Autowired
    private HelloService helloService;

    public void doSomething() {
        ...
        String reply = helloService.sayHello("zxc");
        ...
    }
}
```
这里注入的service bean并不是HelloServiceImpl，因为HelloServiceImpl并不在客户端上，这个bean只是实现了HelloService接口的动态代理类，在调用时其真正的行为是发起一次rpc调用。

####代理类的实现方法
代理类可以利用JDK中的动态代理方式来实现：
```java
Object proxy = Proxy.newProxyInstance(classLoader, interfaceClassList, invocationHandler);
```
第一个参数是classLoader   
第二个参数是要代理的接口的class的list，事例中是只有一个元素即HelloService.class的list  
第三个参数是InvocationHandler，代理类的主逻辑就在这里，需要实现其invoke接口：  
```java
public ServiceProxy implements InvocationHandler {
  @Override
    public Object invoke(Object obj, Method method, Object[] arguments) throws Throwable {
        //proxy logic here
    }
}
```
这里invoke接口参数中，obj参数就是之前newProxyInstance所返回的proxy，在客户端调用的那个对象；method就是调用的方法，在例子中就是sayHello方法；arguments是此次调用所传入的参数队列，在例子中是String类型的"zxc"。

这样就得到一个有着HelloService接口的代理类，这个类上任何方法的调用都会被传递给内部实现了InvocationHanlder接口的代理来处理

PS：   
google的guava框架对动态代理相关类Proxy.newProxyInstance和InvocationHandler也有对应的封装，分别是Reflection.newProxy和AbstractInvocationHandler，
```java
//隐藏了classLoader的逻辑，并且支持泛型，也就是直接返回HelloService类型的对象
HelloService service = Reflection.newProxy(serviceClass, invocationHandler);
```
```java
public abstract class AbstractInvocationHandler implements InvocationHandler {
  @Override
    public final Object invoke(Object proxy, Method method, Object[] args) {
      //先排除 hashCode,equals,toString 的调用
        //只将有意义的调用传递给handlerInvocation方法
        handleInvocation(proxy, method, args);
    }
} 
```
让我们的代码更加简单优雅。

####让spring管理代理类的生命周期   

代理类需要spring来管理其生命周期，才能完成代理类到业务Biz的注入   

定义代理类的bean：
```
<bean id="helloService" class="org.zxc.zing.client.proxy.ServiceProxyBeanFactory" factory-method="getService">
  <constructor-arg value="org.zxc.zing.demo.api.HelloService" />
</bean>
```
ServiceProxyBeanFactory是代理的工厂类，getService是工厂方法，而生成代理bean的参数，目前为止，只需要一个类的全名，以得到对应的class来生成代理类
```java
public class ServiceProxyBeanFactory {
    public static Object getService(String serviceName) throws ClassNotFoundException {
        Class<?> serviceClass = Class.forName(serviceName);
        return Reflection.newProxy(serviceClass, new ServiceProxy(serviceName));
    }
}
```

这样，启动阶段客户端的初始化就完成了，这些被注入到客户端业务逻辑中的代理配在之后调用阶段就会派上用场了。


###1.2 服务端启动--加载service接口到实现的映射并启动netty   
####加载service接口到实现类映射
在服务端，我们有接口HelloService及其实现类HelloServiceImpl，要保证在rpc请求到来时能找到要请求服务对应的实现逻辑，就需要在服务启动之时在内存中维护好这个映射。

还是借助spring，我们来定义这些bean
```xml
<!-- service实现类的bean -->
<bean id="helloService" class="org.zxc.zing.demo.service.impl.HelloServiceImpl" />
<!-- service映射bean-->
<bean class="org.zxc.zing.server.remote.RemoteServiceBean" init-method="init">
  <property name="serviceName" value="org.zxc.zing.demo.api.HelloService"/>
    <property name="serviceImpl" ref="helloService"/>
</bean> 
```
RemoteServiceBean中有两个参数，serviceName即之前和客户端统一的服务接口类的全名，serviceImpl即服务接口对应的服务实现类  
我们把这个映射用一个静态类的静态成员Map<String, Object>的方式维护在内存中，并且在spring加载这个bean的时候执行init方法，把当前的Impl加入Map
```java
public class RemoteServiceBean {
    private String serviceName;
    private Object serviceImpl;

    public void init() {
        log.info("spring add serviceBean init");
        RemoteServiceServer.addService(serviceName, serviceImpl);
    }
}
```
```java
public class RemoteServiceServer {
    private static ConcurrentHashMap<String, Object> serviceImplMap = new ConcurrentHashMap<String, Object>();

  public static void addService(String serviceName, Object serviceImpl) {
        serviceImplMap.putIfAbsent(serviceName, serviceImpl);
    }
}
```
虽然目前只有spring加载bean初始化时串行的进行add，但之后可能涉及到一些场景在其它时机对map进行增删，比如管理服务时需要停止/恢复提供某个service，先用一个ConcurrentHashMap来保证线程安全。

随着spring bean一个个加载完成，服务接口到实现类的映射也就加载完毕了。

可以看到，在addService之前还有一个startup方法，即启动netty服务，下面就来看看netty服务的启动。

####启动netty服务   
之前说rpc服务端启动时需要加载服务映射和启动netty服务两件事，先后顺序是怎样的呢？   
考虑到之后有了服务注册中心，服务端加载完service bean的时候要告知注册中心，这台服务器可以提供该service的服务，这样客户端就能拿到这台服务端的地址从而发送请求，期望请求到来之时服务端是已经启动好的，也就是说启动netty服务需要在加载service bean之前完成。
```java
public class RemoteServiceServer {
    private static volatile boolean started = false;

    private static ConcurrentHashMap<String, Object> serviceImplMap = new ConcurrentHashMap<String, Object>();
    
    static {
      bootstrap();
    }

  public static void addService(String serviceName, Object serviceImpl) {
        serviceImplMap.putIfAbsent(serviceName, serviceImpl);
    }
    
    public static void bootstrap() {
        if (!started) {
            synchronized (RemoteServiceServer.class) {
                if (!started) {
                    doStartup();
                }
            }
        }
    }
    
    private static void doStartup() {
      //netty bootstrap
    }
    
    public static Object getActualServiceImpl(String serviceName) {
        if (!started) {
            log.warn("server not started");
            bootstrap();
        }
        log.info("cur map when get:"+serviceImplMap);
        return serviceImplMap.get(serviceName);
    }
}
```
在RemoteServiceServer的静态块中执行启动，当spring加载第一个RemoteServiceBean并执行其init方法时，RemoteServiceServer.addService会触发JVM加载RemoteServiceServer类，进而执行启动bootstrap流程。  
除去服务端第一次启动时需要执行bootstrap之外，考虑到某些情况下，比如初始化启动失败，在请求到来时发现服务未能启动成功而尝试启动等，用volatile的started标志位加上double-check的方式对bootstrap做一个并发的控制以保证线程安全。
```java
    private static void doStart() {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ch.pipeline().addLast(new NettyDecoder(), new NettyEncoder(), new NettyServerHandler());
                    }
                }).option(ChannelOption.SO_BACKLOG, 128)
                .childOption(ChannelOption.SO_KEEPALIVE, true);

        try {
            ChannelFuture f = bootstrap.bind(8080).sync();
            f.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    if (future.isSuccess()) {
                        started = true;
                        log.info("server started!");
                    }
                }
            });
        } catch (InterruptedException e) {
            log.error("server started failed:"+e.getMessage(), e);
        }
    }
```
doStart中调用netty的api进行启动：   
EventLoop可以理解为类似守护进程，不断处理到来的请求，这里定义了一个bossGroup，一个workerGroup，bossGroup接收请求后分发给workerGroup，workerGroup在自己的线程池中处理这些请求。   
数据到达当前机器，服务端进行逻辑处理，处理完成的数据传输回去这些逻辑都是依赖ChannelHandler来实现，代码中我们定义的NettyDecoder，NettyEncoder，NettyServerHandler都是ChannelHandler，具体的机制在调用阶段详述。  

PS：
netty框架屏蔽了很多的底层网络细节，api封装的简单易用，如果希望对netty有个大致的了解，可以参考[官方UserGuide][netty]，netty 3.x/4.x/5.x版本api都有较大的差异，本项目依赖的是4.0.32版本。

这样，客户端完成了服务代理类的注入，服务端启动了netty并加载了service对应的实现，一切就绪等待RPC调用开始。

##2.调用阶段   
[上一篇文章][previous]中主要描述了一次RPC调用的过程，为了读起来方便，这里再引用一遍：   
![RPC调用流程][rpcflow]   
1.  客户端调用了某个服务的某个方法，期望得到处理的结果
2.  把本次调用的上下文，如服务名、方法签名、参数等信息序列化，构造request
3.  根据被调用的服务名，方法签名等信息找到可以提供服务的server列表
4.  根据负载均衡的规则，选出其中一个server作为目标来调用
5.  向选出的server发送该请求，客户端线程挂起
6.  server接收到请求，反序列并解析得到对应的服务名，方法签名，参数信息
7.  server根据调用信息找到真正的业务服务实例，调用业务服务该方法
8.  把方法的返回值序列化，构造返回response
9.  把response传回给client
10.  client接收并反序列化response，得到服务处理结果，返回给1中调用的地方，唤醒对应的客户端线程

步骤1中service代理类已经在客户端启动过程中完成注入  
步骤2中，当service代理类被调用时，调用相关的信息就会被传递给启动阶段所说的InvocationHandler，即ServicePorxy的invoke中
```java
public ServiceProxy implements InvocationHandler {
  @Override
    public Object invoke(Object obj, Method method, Object[] arguments) throws Throwable {
        //步骤3，应该是从注册中心拉到的可提供服务的服务器列表中根据负载均衡规则选出一个，此版本上的实现暂以常量设置
        ProviderInfo provider = ServiceProviderManager.getProvider(serviceName);
        //步骤4，构造远程调用的request
        RemoteRequest request = new RemoteRequest();
        request.setRequestId(UUID.randomUUID().toString());
        request.setServiceName(serviceName);
        request.setMethodName(method.getName());
        request.setParameterTypes(method.getParameterTypes());
        request.setArguments(args);
    //步骤5
        RemoteClient client = new RemoteClient(provider);
        RemoteResponse response = client.send(request);

        return response.getResponseValue();
    }
}
```
步骤4中网络之间请求和响应的设计如下：
```java
public class ServiceRequest{
    private String requestId; //可代表一次请求的唯一编号
    private String serviceName; //调用的服务名
    private String methodName; //调用的函数名
    private Class<?>[] parameterTypes; //调用的函数的参数的类型
    private Object[] parameters; //调用函数传递的参数
}

public class ServiceResponse{
    private String requestId; //唯一请求的编号
    private int responseCode; //代表返回值是否正常的结果码，比如可定义200为正常，500为异常
    private Object responseValue; //返回的结果
}
```

步骤5中客户端找到了对应的服务端机器，需要发起netty连接服务端、序列化request、发送请求、线程挂起
```java
    public RemoteResponse send(RemoteRequest request) throws TimeoutException, ExecutionException, InterruptedException {
        final SettableFuture<RemoteResponse> future = SettableFuture.create();
        //➀启动netty连接
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(bossGroup)
                    .channel(NioSocketChannel.class)
                    .option(ChannelOption.SO_KEEPALIVE, true)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                        //➃处理我们业务逻辑的ChannelHandler
                            ch.pipeline().addLast(new NettyDecoder(), new NettyEncoder(), new NettyClientHandler(future));
                        }
                    });
      //➁连接到netty服务端
            ChannelFuture f = bootstrap.connect(providerInfo.getAddress(), providerInfo.getPort()).sync();
            //➂发送request
            ChannelFuture writeFuture = f.channel().writeAndFlush(request);
            return future.get(1000, TimeUnit.MILLISECONDS);
        } finally {
            bossGroup.shutdownGracefully();
        }
    }
```
➀在客户端启动netty，与服务端不同，只需要一个group，之后进行一系列的配置  
➁调用bootstrap.connect连接目标netty服务器➂  
➂连接成功后，writeAndFlush发送请求  
➃之前提到netty中的业务逻辑都是在ChannelHandler中实现，ChannelHandler分为ChannelInboundHandler和ChannelOutboundHandler，InboundHandler会在数据传入当前服务器时进行处理，OutboundHandler在当前服务器发送数据时进行处理，在这里使用到NettyDecoder，NettyEncoder和NettyClientHandler三个类，其中NettyDecoder和NettyClientHandler属于InboundHandler，NettyEncoder为OutBoundHandler，所以在➂处writeAndFlush之后，request会经过OutboundHandler的处理，这里就是NettyEncoder，在NettyEncoder中完成序列化的工作
```java
public class NettyEncoder extends MessageToByteEncoder {
    @Override
    protected void encode(ChannelHandlerContext ctx, Object msg, ByteBuf out) throws Exception {
      //序列化
        byte[] bytes = Serializer.serialize(msg);
        //先写入数据的长度
        out.writeInt(bytes.length); 
        //写入要传输数据的字节流
        out.writeBytes(bytes);
    }
}
```
```java
public class Serializer {
    public static byte[] serialize(Object obj) throws IOException {
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        Hessian2Output out = new Hessian2Output(bos);
        out.writeObject(obj);
        out.close();
        return bos.toByteArray();
    }

    public static Object deserialize(byte[] data) throws IOException {
        ByteArrayInputStream bis = new ByteArrayInputStream(data);
        Hessian2Input in = new Hessian2Input(bis);
        return in.readObject();
    }
}
```
Serializer.serialize中使用hession完成Object到字节流的转换，而Encoder中在写入这部分字节流之前，先将数据的长度写入，以供之后数据的接收端校验是否接收到了完成的数据。

步骤6 服务端收到字节流，需要在接收完整后进行反序列化   
对于服务端来说，接收数据就要需要InboundHandler来处理了，在1.2服务端启动阶段可以看到NettyDecoder和NettyServerHandler来处理

关于NettyDecoder，[Netty UserGuide-基于流的传输][transport]中的提到，在TCP/IP使用这种基于流传输的协议时，收到的数据会被存入一个buffer中，而这个buffer并不是数据包的buffer，而是字节的buffer。也就是说你的系统可能接收到这样的三个数据包：   
![系统收到的数据包][transport-packet]   
而你的应用收到字节片段很可能是这样的：   
![应用收到的字节片段][transport-byte]   
对于这个问题，需要在新的字节数据放入buffer时检测一下是否符合我们期望的长度
```java
public class NettyDecoder extends ByteToMessageDecoder{

  //每当有新的字节接收到时，decode就会被调用
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
      //收到的字节还不足一个int，即Encode阶段写入的数据总长度，先不处理
        if (in.readableBytes() < 4) {
            log.info("no enough readable bytes");
            return;
        }

    //此时收到的字节达到4个字节，提取一个int，即期望接收的数据总长度
        int dataLength = in.readInt();
        if (dataLength < 0) {
            ctx.close();
        }

    //接收的字节流除去int剩余的字节长度还未达到期望的长度，表示数据未接收完整
        if (in.readableBytes() < dataLength) {
            in.resetReaderIndex();
        }

    //长度达到了，已经足够，读取出完整的数据
        byte[] data = new byte[dataLength];
        in.readBytes(data);

    //把完整的数据反序列化为对象
        Object deserialized = Serializer.deserialize(data);
        //当decode中把一个对象加入到out中，代表已经解析成功了，之后decode不再被调用
        out.add(deserialized);
    }
}
```

步骤7、8、9： request完成了反序列化，就会被传递给下一个ChannelInboundHandler，即服务端的NettyServerHandler
```java
public class NettyServerHandler extends SimpleChannelInboundHandler<RemoteRequest> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, RemoteRequest request) throws Exception {
    //收到了request，从启动时加载好的map中找到request请求中的serviceImpl实例
        Object actualServiceImpl = RemoteServiceServer.getActualServiceImpl(request.getServiceName());

        if (actualServiceImpl != null) {
          //根据request中的方法名，参数类型，参数等信息，反射调用serviceImpl
          Class<?> serviceInterface = actualServiceImpl.getClass();
            Method method = serviceInterface.getMethod(request.getMethodName(), request.getParameterTypes());
            //反射计算得到结果
            Object result = method.invoke(actualServiceImpl, request.getArguments());
            //构造response
            RemoteResponse response = new RemoteResponse();
            response.setRequestId(request.getRequestId());
            response.setResponseValue(result);
            //向客户端返回构造完成的response
        ctx.writeAndFlush(response)
            .addListener(ChannelFutureListener.CLOSE);
        }
    }
}
```
服务端发送给客户端时，response经过NettyEncoder进行序列化。
PS:  
这里的method的反射调用，即服务端的业务逻辑，是跑在workerEventLoopGroup线程池的线程中，但是如果业务逻辑中涉及到一些连接数据库、远程连接等耗时操作，建议新开线程池来执行，执行完成后再调用netty返回response，而让netty线程池专注处理网络通讯。  


步骤10 客户端收到返回的response，经历NettyDecoder反序列化后，交给NettyClientHandler处理，这里需要提一下：netty的网络操作都是异步的，但是客户端代码调用时却需要同步的得到结果，这里设计到一个异步转同步的问题，客户端发出请求后，要拿到结果，但是结果返回之前需要先挂起线程，拿到结果后再唤醒线程，是类似一个Future，原理如下：
```java
public class ResponseFuture{
  //默认future没有完成，result为空
  private boolean completed = false;
    private Object result;

  public Object get() {
      //如果未完成时来get，就会一直wait下去
      synchronized(this){
          //直到set被调用，标记Future为完成，线程被唤醒，返回result
          while(!completed){
              this.wait();
            }
            return result;
        }
    }
    
    //当接收到返回的response时，设置任务完成，通知挂起的线程
  public void set(Object result) {
      this.completed = true;
        this.result = result;
        this.notifyAll();
    }
}
```
以上代码只是为了说明原理，真实情况不会这么简单，需要一些异常处理和超时机制防止线程一直沉睡等，我直接使用了guava框架的SettableFuture，其get方法可以传入timeout时间，set方法来标记Future得到结果，原理是一致的。   

先再回顾一下客户端在发送请求的片段
```java
    public RemoteResponse send(RemoteRequest request){
        final SettableFuture<RemoteResponse> future = SettableFuture.create();
        //启动netty连接
        ...
        //添加ChannelHandler
        ch.pipeline().addLast(new NettyDecoder(), new NettyEncoder(), new NettyClientHandler(future));
    //连接到netty服务端，发送请求
        ...
        //调用future拿结果，得到结果之前挂起
        return future.get(1000, TimeUnit.MILLISECONDS);
        ...
    }
```
在发送之前创建一个Response的SettableFuture，创建连接时作为NettyClientHandler的成员变量，发送请求之后，调用future.get，在得到response之前，挂起线程。

在接收到response，反序列化后，reponse交给NettyClientHandler：
```java
public class NettyClientHandler extends SimpleChannelInboundHandler<RemoteResponse> {

    private SettableFuture<RemoteResponse> future;

    public NettyClientHandler(SettableFuture<RemoteResponse> future) {
        this.future = future;
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, RemoteResponse msg) throws Exception {
        future.set(msg);
    }
```
调用future.set设置response，唤醒线程，把response的结果返回给代码调用处，rpc框架的基本调用就完成了。


#测试程序
此版本提供rpc程序的本地测试：
在zing-demo模块的test目录下，有一个DemoService的测试程序，先运行DemoServerTest的main函数，会在本地以8080端口启动netty服务，再运行DemoClientTest会以rpc的方式发起请求。

#总结
这个版本主要目标是实现RPC框架的联通，演示其中一些关键点的实现，其中很多地方有优化的空间，比如客户端连接可以重用，服务端业务逻辑在新的线程池中执行，还有下一次会实现的服务注册中心功能，
如果对文章中有任何意见和建议，欢迎指正和共同讨论。

[previous]:http://zxcpro.github.io/blog/2015/12/10/cong-ling-kai-shi-shi-xian-rpc-kuang-jia-1-ji-ben-yuan-li/
[project]:https://github.com/zxcpro/zing
[netty]:http://netty.io/wiki/user-guide.html
[transport]:http://netty.io/wiki/user-guide-for-4.x.html#wiki-h3-11
[transport-packet]:https://camo.githubusercontent.com/24ed1176ecca468dfb2b8b017bb927a8715a16f2/687474703a2f2f756d6c2e6d766e7365617263682e6f72672f676973742f3832653366626530653264346466323833323262
[transport-byte]:https://camo.githubusercontent.com/5b595baf5071bf669f81d08b7554064f4142cc69/687474703a2f2f756d6c2e6d766e7365617263682e6f72672f676973742f6233316330626437626266633639666438326436
