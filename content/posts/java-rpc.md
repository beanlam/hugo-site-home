---
title: "Java 简易RPC框架"
date: 2018-11-06T23:08:41+08:00
categories : ["java"]
---

# 需求分析
RPC 全称 Remote Procedure Call ，简单地来说，它能让使用者像调用本地方法一样，调用远程的接口，而不需要关注底层的具体细节。
例如车辆违章代办功能，如果车辆因为某种原因违章，只需要通过这个违章代办功能（它也许是个APP），我们就能动动手指，而省去了一些跑腿的工作。

不像微服务背景下大家所说的 RPC 框架，如 Dubbo 之类。这个 RPC 框架不提供过多的关于服务注册、服务发现、服务管理等功能。它针对的是这样的一些场景：在内部网络，或者局域网内，两个属于同个业务的系统之间需要通信，而我们又觉得去设计多一种二进制网络协议过于繁琐并且没有必要，这时候如果给客户端开发者一些明确的接口，让他知道实现什么功能该调用什么接口，那么省去的工作量以及开发效率上的提升不言而喻。

这个 RPC 系统基于 Java 语言实现，需求如下：
- RPC 服务端可以通过一条长连接发布多个接口（Interface），客户端按需生成对应接口的代理。
- RPC 客户端也可以发布接口，以便在必要的时候，服务端可以主动调用客户端的接口实现
- 客户端与服务端之间保持长连接并且维持心跳
- 服务端针对不同的接口实现，可以指定不同的线程池去处理
- 序列化协议支持扩展
- 通信协议与具体编程语言无关
- 支持并发调用，一个RPC客户端实例要求是线程安全的

# 通信协议设计
高效的通信协议一般是二进制格式的，比较常见的还有文本协议比如说HTTP，为了追求效率，这个 RPC 框架就采用二进制格式。
## 协议的基本要素
### 魔数
要了解到，报文是在网络上传输的，安全性比较低，因此有必要采取一些措施使得并不是任何人都可以随随便便往我们的端口上发东西，因此我们对报文要有一个初步的识别功能，这时候“魔数(magic number)”就派上用场了。魔数并不受任何规范约束，没有人可以要求你的魔数应该遵循什么规范，实际上魔数只是我们通信双方都约定的一个“暗号”，不知道这个暗号的人就无法参与进通信中。例如 Java 源文件编译后的 class 文件开头就有一个魔数：0xCAFEBABE，随随便便打开一个class文件用十六进制编辑器查看，就能看到。

![class文件格式](/assets/java-rpc/class.png)

Java 虚拟机加载 class 的时候会先验证魔数。如果不是 CAFEBABE 就认为是不合法的 class 文件，并拒绝加载。
不过魔数起到的安全防范作用是非常有限的，“有心人”可以通过抓取网络包就识别出魔数了。因此魔数这个东西其实是“防君子不防小人”。

### 协议版本
一个协议可能也会有多个版本，例如说 HTTP1.0 和 HTTP1.1，不同版本的协议元素可能发生了改变，解析方式也会发生改变，因此协议设计这一块，需要预留出地方声明协议的版本，通信双方在解析协议或者拼装协议的时候才有迹可循。

### 报文类型
对于RPC框架来说，报文可能有多种类型：心跳类型报文、认证类型报文、请求类型报文、响应类型报文等。

### 上下文 ID
RPC 调用其实是一个“请求-响应”的过程，并且跨物理机器，因此每次请求和响应，都必须带上上下文 ID，通信双方才能把请求和响应对应起来。

### 状态
状态用来标识一次调用时正常结束还是异常结束，通常由被调用方置状态。

### 请求数据
即发送到服务端的调用请求，通常是序列化后的二进制流，长度不定。

### 长度编码字段
收报文的一方怎么知道发报文的那一方发了多少字节呢？因此发送方必须在协议里告诉接收方需要接受多少字节才算一个完整的报文。

### 保留字段
协议一旦被设计，并非一成不变的，日后可能有变动的可能，因此还需要考虑保留一些字节空间作为保留字段，以备日后协议的扩展。

## 协议设计
结合以上的一些设计原则，具体协议设计如下：
```bash
 ------------------------------------------------------------------------
 | magic (2bytes) | version (1byte) |  type (1byte)  | reserved (7bits) | 
 ------------------------------------------------------------------------
 | status (1byte) |    id (8bytes)    |        body length (4bytes)     |
 ------------------------------------------------------------------------
 |                                                                      |
 |                   body ($body_length bytes)                          |
 |                                                                      |
 ------------------------------------------------------------------------
```
# 链路可靠性
客户端与服务端之间的连接采用 TCP 长连接，一个客户端与服务端之间保持至少一条长连接。接口调用请求的发送，在多条连接之间进行负载均衡。
每条连接在空闲的时候，由客户端主动向服务端发送心跳报文，并且客户端在发现连接失效或断开的时候，自动进行重连。
每个客户端向服务端建立连接后，在正式发起接口调用请求之前，都需要进行check in 操作, check in 操作主要是将客户端的身份标识(identifier)和客户端的心跳间隔告诉服务端。利用 netty 的 handler 责任链机制和自带的 IdleStateHandler，自动检测出连接是否空闲，并在空闲时触发心跳报文的发送。而服务端在客户端 checkin 后，根据客户端的心跳频率，在自己的 handler pipeline 上动态加入一个 IdleStateHandler，来检测出客户端是否已经失联，如果是，则主动关闭连接。
同时，客户端本地将会起一个定时执行任务的线程，定期检查连接是否失效，如果失效，则关闭旧连接，并进行连接的重建。

# 编解码

协议的解码我们称为 decode，编码我们成为 encode，下文我们将直接使用 decode 和 encode 术语。

decode 的本质就是讲接收到的一串二进制报文，转化为具体的消息对象，在 Java 中，就是将这串二进制报文所包含的信息，用某种类型的对象存储起来。
encode 则是将存储了信息的对象，转化为具有相同含义的一串二进制报文，然后网络收发模块再将报文发出去。

无论是 rpc 客户端还是服务端，都需要有一个 decode 和 encode 的逻辑。

# 消息类型
rpc 客户端与服务端之间的通信，需要通过发送不同类型的消息来实现，例如：client 向 server 端发送的消息，可能是请求消息，可能是心跳消息，可能是认证消息，而 server 向 client 发送的消息，一般就是响应消息。
利用 Java 中的枚举类型，可以将消息类型进行如下定义：
```java
/**
 * 消息类型
 *
 * @author beanlam
 * @version 1.0
 */
public enum MessageType {

    REQUEST((byte) 0x01), HEARTBEAT((byte) 0x02), CHECKIN((byte) 0x03), RESPONSE(
            (byte) 0x04), UNKNOWN((byte) 0xFF);

    private byte code;

    MessageType(byte code) {
        this.code = code;
    }

    public static MessageType valueOf(byte code) {
        for (MessageType instance : values()) {
            if (instance.code == code) {
                return instance;
            }
        }
        return UNKNOWN;
    }

    public byte getCode() {
        return code;
    }
}
```
在这个类中设计了 valueOf 方法，方便进行具体的 byte 字节与具体的消息枚举类型之间的映射和转换。

# 调用状态设计
client 主动发起的一次 rpc 调用，要么成功，要么失败，server 端有责任告知 client 此次调用的结果，client 也有责任去感知调用失败的原因，因为不一定是 server 端造成的失败，可能是因为 client 端在对消息进行预处理的时候，例如序列化，就已经出错了，这种错误也应该作为一次调用的调用结果返回给 client 调用者。因此引入一个调用状态，与消息类型一样，它也借助了 Java 语言里的枚举类型来实现，并实现了方便的 valueOf 方法：
```java
/**
 * 调用状态
 *
 * @author beanlam
 * @version 1.0
 */
public enum InvocationStatus {

    OK((byte) 0x01), CLIENT_TIMEOUT((byte) 0x02), SERVER_TIMEOUT(
            (byte) 0x03), BAD_REQUEST((byte) 0x04), BAD_RESPONSE(
            (byte) 0x05), SERVICE_NOT_FOUND((byte) 0x06), SERVER_SERIALIZATION_ERROR(
            (byte) 0x07), CLIENT_SERIALIZATION_ERROR((byte) 0x08), CLIENT_CANCELED(
            (byte) 0x09), SERVER_BUSY((byte) 0x0A), CLIENT_BUSY(
            (byte) 0x0B), SERIALIZATION_ERROR((byte) 0x0C), INTERNAL_ERROR(
            (byte) 0x0D), SERVER_METHOD_INVOKE_ERROR((byte) 0x0E), UNKNOWN((byte) 0xFF);

    private byte code;

    InvocationStatus(byte code) {
        this.code = code;
    }

    public static InvocationStatus valueOf(byte code) {
        for (InvocationStatus instance : values()) {
            if (code == instance.code) {
                return instance;
            }
        }

        return UNKNOWN;
    }

    public byte getCode() {
        return code;
    }
}
```
# 消息实体设计
我们将 client 往 server 端发送的统一称为 rpc 请求消息，一个请求对应着一个响应，因此在 client 和 server 端间流动的信息大体上其实就只有两种，即要么是请求，要么是响应。我们将会定义两个类，分别是 RpcRequest 和 RpcResponse 来代表请求消息和响应消息。
另外由于无论是请求消息还是响应消息，它们都有一些共同的属性，例如说“调用上下文ID”，或者消息类型。因此会再定义一个 RpcMessage 类，作为父类。

![消息类关系](/assets/java-rpc/relations.png)

## RpcMessage
```java
/**
 * rpc消息
 *
 * @author beanlam
 * @version 1.0
 */
public class RpcMessage {

    private MessageType type;
    private long contextId;

    private Object data;

    public long getContextId() {
        return this.contextId;
    }

    public void setContextId(long id) {
        this.contextId = id;
    }

    public Object getData() {
        return this.data;
    }

    public void setData(Object data) {
        this.data = data;
    }

    public void setType(byte code) {
        this.type = MessageType.valueOf(code);
    }

    public MessageType getType() {
        return this.type;
    }

    public void setType(MessageType type) {
        this.type = type;
    }

    @Override
    public String toString() {
        return "[messageType=" + type.name() + ", contextId=" + contextId + ", data="
                + data + "]";
    }
}
```
## RpcRequest
```java
import java.util.concurrent.atomic.AtomicLong;

/**
 * rpc请求消息
 *
 * @author beanlam
 * @version 1.0
 */
public class RpcRequest extends RpcMessage {

    private static final AtomicLong ID_GENERATOR = new AtomicLong(0);

    public RpcRequest() {
        this(ID_GENERATOR.incrementAndGet());
    }

    public RpcRequest(long contextId) {
        setContextId(contextId);
        setType(MessageType.REQUEST);
    }
}
```

## RpcResponse
```java
/**
 *
 * rpc响应消息
 *
 * @author beanlam
 * @version 1.0
 */
public class RpcResponse extends RpcMessage {

    private InvocationStatus status = InvocationStatus.OK;

    public RpcResponse(long contextId) {
        setContextId(contextId);
        setType(MessageType.RESPONSE);
    }

    public InvocationStatus getStatus() {
        return this.status;
    }

    public void setStatus(InvocationStatus status) {
        this.status = status;
    }

    @Override
    public String toString() {
        return "RpcResponse[contextId=" + getContextId() + ", status=" + status.name() + "]";
    }
}
```
# netty 编解码介绍
netty 是一个 NIO 框架，应该这么说，netty 是一个有良好设计思想的 NIO 框架。一个 NIO 框架必备的要素就是 reactor 线程模型，目前有一些比较优秀而且开源的小型 NIO 框架，例如分库分表中间件 mycat 实现的一个简易 NIO 框架，可以在[这里](https://github.com/MyCATApache/Mycat-NIO)看到。

netty 的主要特点有：微内核设计、责任链模式的业务逻辑处理、内存和资源泄露的检测等。其中编解码在 netty 中，都被设计成责任链上的一个一个 Handler。
decode 对于 netty 来说，它提供了 ByteToMessageDecoder，它也提供了 MessageToByteEncoder。
借助 netty 来实现协议编解码，实际上就是去在这两个handler里面实现编解码的逻辑。

# decode
在实现 decode 逻辑时需要注意的一个问题是，由于二进制报文是在网络上发送的，因此一个完整的报文可能经过多个分组来发送的，什么意思呢，就是当有报文进来后，要确认报文是否完整，decode逻辑代码不能假设收到的报文就是一个完整报文，一般称这为“TCP半包问题”。同样，报文是连着报文发送的，意味着decode代码逻辑还要负责在一长串二进制序列中，分割出一个一个独立的报文，这称之为“TCP粘包问题”。
netty 本身有提供一些方便的 decoder handler 来处理 TCP 半包和粘包的问题。不过一般情况下我们不会直接去用它，因为我们的协议比较简单，自己在代码里处理一下就可以了。
完整的 decode 代码逻辑如下所示：
```java
import cn.com.agree.ats.rpc.message.*;
import cn.com.agree.ats.util.logfacade.AbstractPuppetLoggerFactory;
import cn.com.agree.ats.util.logfacade.IPuppetLogger;
import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.ByteToMessageDecoder;

import java.util.List;

/**
 * 协议解码器
 *
 * @author beanlam
 * @version 1.0
 */
public class ProtocolDecoder extends ByteToMessageDecoder {

    private static final IPuppetLogger logger = AbstractPuppetLoggerFactory
            .getInstance(ProtocolDecoder.class);

    private boolean magicChecked = false;

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> list)
            throws Exception {

        if (!magicChecked) {
            if (in.readableBytes() < ProtocolMetaData.MAGIC_LENGTH_IN_BYTES) {
                return;
            }
            magicChecked = true;

            if (!(in.getShort(in.readerIndex()) == ProtocolMetaData.MAGIC)) {
                logger.warn(
                        "illegal data received without correct magic number, channel will be close");
                ctx.close();
                magicChecked = false; //this line of code makes no any sense, but it's good for a warning
                return;
            }
        }

        if (in.readableBytes() < ProtocolMetaData.HEADER_LENGTH_IN_BYTES) {
            return;
        }

        int bodyLength = in
                .getInt(in.readerIndex() + ProtocolMetaData.BODY_LENGTH_OFFSET);
        if (in.readableBytes() < bodyLength + ProtocolMetaData.HEADER_LENGTH_IN_BYTES) {
            return;
        }
        magicChecked = false;// so far the whole packet was received

        in.readShort(); // skip the magic
        in.readByte(); // dont care about the protocol version so far
        byte type = in.readByte();
        byte status = in.readByte();
        long contextId = in.readLong();

        byte[] body = new byte[in.readInt()];
        in.readBytes(body);

        RpcMessage message = null;
        MessageType messageType = MessageType.valueOf(type);

        if (messageType == MessageType.RESPONSE) {
            message = new RpcResponse(contextId);
            ((RpcResponse) message).setStatus(InvocationStatus.valueOf(status));
        } else {
            message = new RpcRequest(contextId);
        }
        message.setType(messageType);
        message.setData(body);
        list.add(message);
    }
}
```
可以看到，我们解决半包问题的时候，是判断有没有收到我们期望收到的报文，如果没有，直接在 decode 方法里面 return，等有更多的报文被收到的时候，netty 会自动帮我们调起 decode 方法。而我们解决粘包问题的思路也很清晰，那就是一次只处理一个报文，不去动后面的报文内容。

还需要注意的是，在 netty 中，对于 ByteBuf 的 get 是不会消费掉报文的，而 read 是会消费掉报文的。当不确定报文是否收完整的时候，我们都是用 get开头的方法去试探性地验证报文是否接收完全，当确定报文接收完全后，我们才用 read 开头的方法去消费这段报文。

# encode
直接贴代码，参考前文提到的协议格式阅读以下代码：
```java
/**
 *
 * 协议编码器
 *
 * @author beanlam
 * @version 1.0
 */
public class ProtocolEncoder extends MessageToByteEncoder<RpcMessage> {

    @Override
    protected void encode(ChannelHandlerContext ctx, RpcMessage rpcMessage, ByteBuf out)
            throws Exception {
        byte status;
        byte[] data = (byte[]) rpcMessage.getData();

        if (rpcMessage instanceof RpcRequest) {
            RpcRequest request = (RpcRequest) rpcMessage;
            status = InvocationStatus.OK.getCode();
        } else {
            RpcResponse response = (RpcResponse) rpcMessage;
            status = response.getStatus().getCode();
        }

        out.writeShort(ProtocolMetaData.MAGIC);
        out.writeByte(ProtocolMetaData.VERSION);
        out.writeByte(rpcMessage.getType().getCode());
        out.writeByte(status);
        out.writeLong(rpcMessage.getContextId());
        out.writeInt(data.length);
        out.writeBytes(data);
    }
}
```

# 序列化
由于我们还未谈到具体的 RPC 调用机制，因此暂且认为 encode 就是把一个包含了调用信息的 Java 对象，从 client 经过序列化，变成一串二进制流，发送到了 server 端。
这里需要明确的是，encode 的职责是拼协议，它不负责序列化，同样，decode 只是把整个二进制报文分割，哪部分是报文头，哪部分是报文体，诚然，报文体就是被序列化成二进制流的一个 Java 对象。
对于调用方来说，先将调用信息封装成一个 Java 对象，经过序列化后形成二进制流，再经过 encode 阶段，拼接成一个完整的遵守我们定好的协议的报文。
对于被调用方来说，则是收取完整的报文，在 decode 阶段将报文中的报文头，报文体分割出来，在序列化阶段将报文体反序列化为一个 Java 对象，从而获得调用信息。
本文探讨序列化机制。

# 基于 netty handler
由于这个 RPC 框架基于 netty 实现，因此序列化机制其实体现在了 netty 的 pipeline 上的 handler 上。
例如对于调用方，它需要在 pipeline 上加上一个 序列化 encode handler，用来序列化发出去的请求，同时需要加上一个反序列化的 decode handler, 以便反序列化调用结果。如下所示：
```java
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ChannelPipeline pipeline = ch.pipeline();
                        pipeline.addLast(new ProtocolEncoder())
                                .addLast(new ProtocolDecoder())
                                .addLast(new SerializationHandler(serialization))
                                .addLast(new DeserializationHandler(serialization));

                    }
```
其中的 SerializationHandler 和 DeserializationHandler 就是上文提到的序列化 encode handler 和反序列化 decode handler。
同样，对于被调用方来说，它也需要这两个handler，与调用方的 handler 编排顺序一致。

其中，serialization 这个参数的对象代表具体的序列化机制策略。

# 序列化机制
上文中，SerializationHandler 和 DeserializationHandler 这两个对象都需要一个 serialization 对象作为参数，它是这么定义的：
```java
private ISerialization serialization = SerializationFactory.getSerialization(ServerDefaults.DEFAULT_SERIALIZATION_TYPE);
```
采用工厂模式来创建具体的序列化机制：
```java
/**
 * 序列化工厂
 *
 * @author beanlam
 * @version 1.0
 */
public class SerializationFactory {

    private SerializationFactory() {
    }

    public static ISerialization getSerialization(SerializationType type) {
        if (type == SerializationType.JDK) {
            return new JdkSerialization();
        }
        return new HessianSerialization();
    }
}
```
这里暂时只支持 JDK 原生序列化 和 基于 Hessian 的序列化机制，日后若有其他效率更高更适合的序列化机制，则可以在工厂类中进行添加。
> 这里的 hessian 序列化是从 dubbo 中剥离出来的一块代码，感兴趣可以从 dubbo 的源码中的 com.caucho.hessian 包中获得。

以 HessianSerialization 为例：
```java
/**
 * @author beanlam
 * @version 1.0
 */
public class HessianSerialization implements ISerialization {

    private ISerializer serializer = new HessianSerializer();
    private IDeserializer deserializer = new HessianDeserializer();

    @Override
    public ISerializer getSerializer() {
        return serializer;
    }

    @Override
    public IDeserializer getDeserializer() {
        return deserializer;
    }

    @Override
    public boolean accept(Class<?> clazz) {
        return Serializable.class.isAssignableFrom(clazz);
    }
}
```

根据 Hessian 的 API， 分别返回一个 hessian 的序列化器和反序列化器即可。