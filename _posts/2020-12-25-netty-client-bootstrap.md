---
title:  "Netty-使用Bootstrap进行客户端初始化的分析"
category: "netty"
---

## 先以一个Netty官网的EchoClient为例

Bootstrap 是用来初始化 Netty 客户端的工厂类，客户端使用 NioSocketChannel 作为 Socket 进行读写。

```
基于Netty 4.1.x版本
```

```java
public class EchoClient {
    public static void main(String[] args) throws Exception {
        // 1、分析EventLoopGroup初始化
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap();
            b.group(group)
             //2、分析channel方法
             .channel(NioSocketChannel.class)
             .option(ChannelOption.TCP_NODELAY, true)
             //3、分析Handler的添加过程
             .handler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     if (sslCtx != null) {
                         p.addLast(sslCtx.newHandler(ch.alloc(), HOST, PORT));
                     }
                     //p.addLast(new LoggingHandler(LogLevel.INFO));
                     p.addLast(new EchoClientHandler());
                 }
             });
            
            // 发起同步连接操作
            //4、分析connect方法
            ChannelFuture f = b.connect(HOST, PORT).sync();

            // Wait until the connection is closed.
            f.channel().closeFuture().sync();
        } finally {
            // 关闭，释放线程池资源
            group.shutdownGracefully();
        }
    }
}
```

## 1、分析 EventLoopGroup 初始化
```java
public class NioEventLoopGroup extends MultithreadEventLoopGroup {
    public NioEventLoopGroup() {
        //默认线程数为0
        this(0);
    }

    public NioEventLoopGroup(int nThreads, Executor executor, final SelectorProvider selectorProvider,
                             final SelectStrategyFactory selectStrategyFactory) {
        super(nThreads, executor, selectorProvider, selectStrategyFactory, RejectedExecutionHandlers.reject());
    }
}

public abstract class MultithreadEventLoopGroup extends MultithreadEventExecutorGroup implements EventLoopGroup {
    protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
        //NioEventLoopGroup会调用超类MultithreadEventLoopGroup的构建方法
        //如果初始化NioEventLoopGroup没指定线程数，到这里就会取DEFAULT_EVENT_LOOP_THREADS的值
        super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
    }

    private static final int DEFAULT_EVENT_LOOP_THREADS;

    static {
        DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
                "io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));

        if (logger.isDebugEnabled()) {
            logger.debug("-Dio.netty.eventLoopThreads: {}", DEFAULT_EVENT_LOOP_THREADS);
        }
    }

}
```

## 2、分析 channel 方法

channel 方法在 AbstractBootstrap 类中

```java
public abstract class AbstractBootstrap<B extends AbstractBootstrap<B, C>, C extends Channel> implements Cloneable {
    //这里返回类一个ReflectiveChannelFactory实例
    public B channel(Class<? extends C> channelClass) {
        //这里的channelClass就是前面BootStrap初始化时候传的NioSocketChannel
        return channelFactory(new ReflectiveChannelFactory<C>(
                ObjectUtil.checkNotNull(channelClass, "channelClass")
        ));
    }
}

public class ReflectiveChannelFactory<T extends Channel> implements ChannelFactory<T> {
    private final Constructor<? extends T> constructor;
    
    public ReflectiveChannelFactory(Class<? extends T> clazz) {
        ObjectUtil.checkNotNull(clazz, "clazz");
        try {
            //这里的clazz就是NioSocketChannel.class
            this.constructor = clazz.getConstructor();
        } catch (NoSuchMethodException e) {
            throw new IllegalArgumentException("Class " + StringUtil.simpleClassName(clazz) +
                    " does not have a public non-arg constructor", e);
        }
    }

    @Override
    public T newChannel() {
        try {
            //ReflectiveChannelFactory这里只重写了newChannel方法，NioSocketChannel实例就是调用这个方法创建的
            return constructor.newInstance();
        } catch (Throwable t) {
            throw new ChannelException("Unable to create Channel from class " + constructor.getDeclaringClass(), t);
        }
    }

}

```

## 3、分析Handler的添加过程

Handler 是 netty 用户实现自定义插件处理的地方，比如自定义的协议、编解码等。

```java
public abstract class ChannelInitializer<C extends Channel> extends ChannelInboundHandlerAdapter {
    //Bootstrap初始化时，实现了initChannel方法
    protected abstract void initChannel(C ch) throws Exception;

    @Override
    @SuppressWarnings("unchecked")
    public final void channelRegistered(ChannelHandlerContext ctx) throws Exception {
        //在这里调用了用户实现的initChannel方法
        if (initChannel(ctx)) {
            ctx.pipeline().fireChannelRegistered();
            removeState(ctx);
        } else {
            ctx.fireChannelRegistered();
        }
    }

    @SuppressWarnings("unchecked")
    private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
        if (initMap.add(ctx)) {
            try {
                initChannel((C) ctx.channel());
            } catch (Throwable cause) {
                exceptionCaught(ctx, cause);
            } finally {
                ChannelPipeline pipeline = ctx.pipeline();
                if (pipeline.context(this) != null) {
                    //在initChannel方法中，用户自定义的Handler已经被添加到Pipline
                    //这里就将用户自定义的ChannelInitializer实现移除
                    pipeline.remove(this);
                }
            }
            return true;
        }
        return false;
    }
}

```

## 4、分析 connect 方法

```java
public class Bootstrap extends AbstractBootstrap<Bootstrap, Channel> {
    public ChannelFuture connect(SocketAddress remoteAddress) {
        ObjectUtil.checkNotNull(remoteAddress, "remoteAddress");
        validate();
        return doResolveAndConnect(remoteAddress, config.localAddress());
    }

    private ChannelFuture doResolveAndConnect(final SocketAddress remoteAddress, final SocketAddress localAddress) {
        //重点看initAndRegister方法
        final ChannelFuture regFuture = initAndRegister();
        final Channel channel = regFuture.channel();

        if (regFuture.isDone()) {
            if (!regFuture.isSuccess()) {
                return regFuture;
            }
            return doResolveAndConnect0(channel, remoteAddress, localAddress, channel.newPromise());
        }//省略部分代码
    }
}

public abstract class AbstractBootstrap<B extends AbstractBootstrap<B, C>, C extends Channel> implements Cloneable {
    final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            //到这里，我们终于看见了NioSocketChannel是如何被实例化的
            //channelFactory就是前面分析的ReflectiveChannelFactory
            channel = channelFactory.newChannel();
            init(channel);
        } catch (Throwable t) {
            //...
        }

        //这里将channel注册到Selector
        ChannelFuture regFuture = config().group().register(channel);
    }
}
```