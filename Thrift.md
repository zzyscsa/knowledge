# Thrift

Thrift是一种接口描述语言和二进制通讯协议，它被用来定义和创建跨语言的服务。它被当作一个远程过程调用（RPC）框架来使用

thrift 通过一个中间语言IDL(接口定义语言)来定义RPC的数据类型和接口，以.thrift 结尾，**然后可以根据需求编译成想要的文件。**比如生成java代码，生成go代码

一个rpc框架

![img](https://img-blog.csdn.net/20170207141803075?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGloYW8yMQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

这个Stub可以理解为桩

1、客户过程以正常方式调用客户桩（client stub，一段代码）； 

2、客户桩生成一个消息，然后调用本地操作系统；

3、客户端操作系统将消息发送给远程操作系统； 

4、远程操作系统将消息交给服务器桩（server stub，一段代码）；

5、服务器桩将参数提取出来，然后调用服务器过程； 

6、服务器执行要求的操作，操作完成后将结果返回给服务器桩； 

7、服务器桩将结果打包成一个消息，然后调用本地操作系统； 

8、服务器操作系统将含有结果的消息发送回客户端操作系统； 

9、客户端操作系统将消息交给客户桩； 

10、客户桩将结果从从消息中提取出来，返回给调用它的客户过程；

## Thrift协议栈

thrift主要用于各个服务之间的RPC通信，支持**<font color='red'>跨语言</font>**。thrift是一个**<font color='red'>典型的C/S结构</font>**，客户端和服务端可以使用不同的语言开发，thrift通过IDL(Interface Description Language)来关联客户端和服务端。

![img](https://img-blog.csdn.net/2018081919583516?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3prcF9qYXZh/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

图中`Your Code`是用户实现的业务逻辑，接下来的 `FooService.Client`和 `Foo.write()/read()`是<font color='red'>thrift根据IDL生成的客户端和服务端的代码</font>，**对应于RPC中Client stub和Server stub**。`TProtocol `用来对数据进行序列化与反序列化，具体方法包括二进制，JSON 或者 Apache Thrift 定义的格式。`TTransport `提供数据传输功能，使用 Apache Thrift 可以方便地定义一个服务并选择不同的传输协议。

![img](https://img-blog.csdn.net/20180820225105991?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3prcF9qYXZh/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

thirft使用socket进行数据传输，数据以特定的格式发送，接收方进行解析。我们定义好thrift的IDL文件后，就可以使用thrift的编译器来生成双方语言的接口、model，在生成的model以及接口代码中会有解码编码的代码。

### TTransport层

代表thrift的数据传输方式，thrift定义了如下几种常用数据传输方式

- `TSocket`: 阻塞式socket；
- `TFramedTransport`: 以frame为单位进行传输，非阻塞式服务中使用；
- `TFileTransport`: 以文件形式进行传输；

### TProtocol层

代表thrift客户端和服务端之间传输数据的协议，通俗来讲就是客户端和服务端之间传输数据的格式(例如json等)，thrift定义了如下几种常见的格式

- `TBinaryProtocol`: 二进制格式；
- `TCompactProtocol`: 压缩格式；
- `TJSONProtocol`: JSON格式；
- `TSimpleJSONProtocol`: 提供只写的JSON协议；

### 底层IO

底层IO负责实际的数据传输，包括socket、文件和压缩数据流等。

## Thrift支持的Server模型

thrift主要支持以下几种服务模型

- `TSimpleServer`: 简单的单线程服务模型，常用于测试；
- `TThreadPoolServer`: 多线程服务模型，使用标准的阻塞式IO；
- `TNonBlockingServer`: 多线程服务模型，使用非阻塞式IO(需要使用`TFramedTransport`数据传输方式);
- `THsHaServer`: `THsHa`引入了线程池去处理，其模型读写任务放到线程池去处理，`Half-sync/Half-async`处理模式，`Half-async`是在处理IO事件上(accept/read/write io)，`Half-sync`用于handler对rpc的同步处理；

## Thrift与Dubbo的区别

#### 透明化服务调用

Thrift：**通过IDL（接口定义语言）实现**，Thrift是支持跨语言的，而要实现跨语言机制，就需要一种中间语言来完成，那就是IDL。首先定义好了IDL之后（即定义好了一个service），由于server和Client都需要持有这个与之语言相对应的服务接口，那就需要thrift来将IDL编译成与之语言相对应的接口类，比如server端是java，client端是C#，则server端需要编译成java类，client编译成C#类。然后server端负责接口的具体实现，client只需要持有这个对象接口，进行调用即可，然后通过网络通信传给server。server负责解析client传递过来的数据，由于服务对象接口统一为IDL（thrift格式），所以统一了解析形式，只和本地的开发语言有关。

Dubbo：**通过java反射和动态代理实现这一功能**，由于只支持Java，所以Client和Server端开发语言机制一样，所以它们能够通过spring框架进行依赖，server端一般将对象方法接口注册发布到Zookeper中，然后Client可以直接从Zookeper中获取server端发布的对象方法接口，这样Client可以通过Spring进行自动装配获得server端发布的对象方法服务接口。

#### 网络通信

**<font color='red'>Thrift实际上是实现了C/S模式</font>**，通过代码生成工具将thrift文生成服务器端和客户端代码（可以为不同语言），从而实现服务端和客户端跨语言的支持。用户在Thirft文件中声明自己的服务，这些服务经过编译后会生成相应语言的代码文件，然后客户端调用服务，服务器端提服务便可以了。

**<font color='red'>dubbo的网络通信基于NIO框架</font>**，一般基于事件的NIO网络框架都涉及到 channel , channelHandle核心概念，网络数据buffer, 网络数据编解码器，dubbo为了能够适配多种NIO框架，将以上概念全部又抽象了一层接口。

#### 序列化

thrift只支持对thrift协议描述的IDL进行序列化

dubbo支持各种协议的序列化（实际上，dubbo没有IDL这一机制，因为实际上他就是通过java服务对象接口进行交互的）例如他也可支持thrift协议，只需要thrift将IDL转换为一个java服务对象接口，那么dubbo就可以使用了