Socket：

![img](https://upload-images.jianshu.io/upload_images/11345047-261171b0e575255c.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

套接字使用TCP提供了两台计算机之间的通信机制。 客户端程序创建一个套接字，并尝试连接服务器的套接字。

当连接建立时，服务器会创建一个 Socket 对象。客户端和服务器现在可以通过对 Socket 对象的写入和读取来进行通信。

如果一个程序创建了一个socket，并让其监听80端口，其实是向TCP/IP协议栈声明了其对80端口的占有。以后，所有目标是80端口的TCP数据包都会转发给该程序（这里的程序，因为使用的是Socket编程接口，所以首先由Socket层来处理）。所谓accept函数，其实抽象的是TCP的连接建立过程。accept函数返回的新socket其实指代的是本次创建的连接，而一个连接是包括两部分信息的，一个是源IP和源端口，另一个是宿IP和宿端口。所以，accept可以产生多个不同的socket，而这些socket里包含的宿IP和宿端口是不变的，变化的只是源IP和源端口。这样的话，这些socket宿端口就可以都是80，而Socket层还是能根据源/宿对来准确地分辨出IP包和socket的归属关系，从而完成对TCP/IP协议的操作封装。

```java
// sun.nio.ch.SelectorProvider
public SocketChannel openSocketChannel() throws IOException {
    // 调用SocketChannelImpl的构造器
    return new SocketChannelImpl(this);
}

// sun.nio.ch.SocketChannelImpl
SocketChannelImpl(SelectorProvider sp) throws IOException {
    super(sp);
    // 创建socket fd
    this.fd = Net.socket(true);
    // 获取socket fd的值
    this.fdVal = IOUtil.fdVal(fd);
    // 初始化SocketChannel状态, 状态不多，总共就6个
    // 未初始化，未连接，正在连接，已连接，断开连接中，已断开
    this.state = ST_UNCONNECTED;
}

// sun.nio.ch.Net
static FileDescriptor socket(ProtocolFamily family, boolean stream)
    throws IOException {
    boolean preferIPv6 = isIPv6Available() &&
        (family != StandardProtocolFamily.INET);
    // 最后调用的是socket0
    return IOUtil.newFD(socket0(preferIPv6, stream, false));
}

// Due to oddities SO_REUSEADDR on windows reuse is ignored
private static native int socket0(boolean preferIPv6, boolean stream, boolean reuse);
```



可以看到，最后还是靠一个native方法socket0来创建socket fd,打开jdk/src/solaris/native/sun/nio/ch/Net.c

```C
JNIEXPORT int JNICALL
Java_sun_nio_ch_Net_socket0(JNIEnv *env, jclass cl, jboolean preferIPv6,
                            jboolean stream, jboolean reuse)
{
    int fd;
    int type = (stream ? SOCK_STREAM : SOCK_DGRAM);

    // 老朋友socket函数
    fd = socket(domain, type, 0);
    if (fd < 0) {
        return handleSocketError(env, errno);
    }

    ....省略非关键代码

    // 设置是否重用地址，如果打开的是ServerSocketChannel
    // 默认是重用的，其他普通SocketChannel默认不重用
    // 重用和不重用的区别在于，就算你关掉了程序，你绑定的
    // 本地端口也在一定时间内是已使用的(address already in use)
    if (reuse) {
        int arg = 1;
        if (setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, (char*)&arg,
                       sizeof(arg)) < 0) {
            JNU_ThrowByNameWithLastError(env,
                                         JNU_JAVANETPKG "SocketException",
                                         "Unable to set SO_REUSEADDR");
            close(fd);
            return -1;
        }
    }
    ...
    return fd;
}
```



JVM里，在SocketChannel上有一个configureBlocking函数，这个函数是设置当前SocketChannel是否是阻塞的，和selector一起用的时候一定要设置成非阻塞才有意义, 阻塞的话就不需要IO多路复用的事件通知了。

```Java
// java.nio.channels.spi.AbstractSelectableChannel
public final SelectableChannel configureBlocking(boolean block)
    throws IOException
{
    ...
    // 模板方法模式，调用子类的实现
    implConfigureBlocking(block);
    ...
    return this;
}
```

然后去SocketChannelImpl里看

```java
protected void implConfigureBlocking(boolean block) throws IOException {
    IOUtil.configureBlocking(fd, block);
}
```

将这个操作又交给了IOUtil的configureBlocking, 同时还传入了我们上面创建的socket fd. 打开IOUtil一看,

```java
public static native void configureBlocking(FileDescriptor fd,
                                            boolean blocking)
    throws IOException;
```

打开IOUtil.c

```cpp
JNIEXPORT void JNICALL
Java_sun_nio_ch_IOUtil_configureBlocking(JNIEnv *env, jclass clazz,
                                         jobject fdo, jboolean blocking)
{
    if (configureBlocking(fdval(env, fdo), blocking) < 0)
        JNU_ThrowIOExceptionWithLastError(env, "Configure blocking failed");
}

static int
configureBlocking(int fd, jboolean blocking)
{
    // 所以还是靠file control
    int flags = fcntl(fd, F_GETFL);
    int newflags = blocking ? (flags & ~O_NONBLOCK) : (flags | O_NONBLOCK);

    return (flags == newflags) ? 0 : fcntl(fd, F_SETFL, newflags);
}
```





Netty是一个异步的、基于事件驱动的网络应用框架，用以高速开发高性能、高可靠性的网络IO程序。主要针对在TCP协议下，面向clients端的高并发应用，或者Peer-to-Peer场景下的大量数据持续传输的应用。

IO模型：用什么样的通道进行数据的发送和接受，很大程度上决定了程序通信的性能。

## BIO -- Blocking IO

同步阻塞，服务器实现模式为一个连接一个线程，如果这个连接不做任何事情会造成不必要的线程开销，可以通过线程池机制改善。

编程流程：

1. 服务器端启动一个server/socket
2. 客户端启动socket对服务器进行通信，默认情况下服务器端需要对每个客户建立一个线程与之通信
3. 客户端发出请求后，先咨询服务器是否有线程响应，如果没有则会等待，或者被拒绝
4. 如果有响应，客户端线程会等待请求结束后，会继续执行。

## NIO Non-Blocking IO

是指JDK提供的新API，java提供了一系列改进的输入、输出的新特性，被统称为NIO new IO。

NIO有三大核心部分：Channel（通道），Buffer（缓冲区），Selector（选择器）

client端把数据写入Buffer，然后Buffer和Channel之间双向交互，最终由selector来选择管道来服务。一个线程管理一个Selector。

NIO 面向缓冲区，或者面向块编程的。数据读取到一个它稍后处理的缓冲区，需要时可在缓冲区中前后移动，这就增加了处理过程中的灵活性，使用它可以提供非阻塞式的高伸缩性网络。

- BIO以流的方式处理数据，而NIO以块的方式处理数据，块I/O比流I/O高很多。
- BIO是阻塞的，NIO是非阻塞的
- BIO基于字节流和字符流进行操作，而NIO基于Channel和Buffer进行操作，数据总是从管道读取到缓冲区中，或者从缓冲区写入到通道中。Selector用于监听多个通道时间（比如：连接请求，数据到达等），因此使用单个线程就可以监听多个客户端通道。
- 程序切换到哪个channel是由Event决定的。
- Selector会根据不同时间，在各个通道上切换。
- Buffer就是一个内存块，底层是一个数组。
- 数据的读取写入是通过Buffer，跟BIO不同。BIO中要么是输入流，或者是输出流，不能双向，但是NIO的buffer是可以读也可以写，需要flip方法切换。

![Asyncdb（三）：Java NIO - ScalaCool](https://scala.cool/images/2017/11/java-nio.png)

### Buffer

Buffer底层就是数据块。可以读入也可以写入。

![img](https://user-gold-cdn.xitu.io/2019/11/15/16e6d7431265cbcb?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

0 `<=` *mark* `<=` *position* `<=` *limit* `<=` *capacity*

- capacity

> 作为一个内存块，Buffer有一个固定的大小值，也叫“capacity”.你只能往里写capacity个byte、long，char等类型。一旦Buffer满了，需要将其清空（通过读数据或者清除数据）才能继续写数据往里写数据。

- position

> 当你写数据到Buffer中时，position表示当前的位置。初始的position值为0.当一个byte、long等数据写到Buffer后， position会向前移动到下一个可插入数据的Buffer单元。position最大可为capacity – 1.

当读取数据时，也是从某个特定位置读。当将Buffer从写模式切换到读模式，position会被重置为0. 当从Buffer的position处读取数据时，position向前移动到下一个可读的位置。

- limit

> 在写模式下，Buffer的limit表示你最多能往Buffer里写多少数据。 写模式下，limit等于Buffer的capacity。



### Channel

BIO中的stram是单向的，FileInputStream对象只能进行读取数据的操作，而NIO中的通道Channel是双向的，可以读操作，也可以写操作。

Channel在NIO中是一个接口，public interface Channel extends Closeable{}

常见方法：read(ByteBuffer dst) 从管道读取数据并放到缓冲区中

write(ByteBuffer src), 把缓冲区的数据写到通道中。

- 通道可以同时进行读写，而流只能读或者写。
- 通道可以实现异步读写数据
- 通道可以从缓冲读数据，也可以写数据到缓冲。

#### FileChannel类

FileChannel主要用来对本地文件进行IO操作，主要方法有 read，write， transferFrom(从目标通道中复制数据到当前通道)，tranferTo(从当前通道复制给目标通道)

ByteBuffer支持类型读写。

#### ServerSocketChannel

ServerSocketChannel在服务器端监听新的客户端socket连接。生成新的SocketChannel。

#### SocketChannel

socketChannel，网络IO通道，具体负责进行读写操作。NIO把缓冲区的数据写入通道，或者把通道里的数据读到缓冲区。相关方法：

open() ：得到一个SocketChannel通道，

connect(SocketAddress) :  连接服务器

finishConnect()：如果上面的方法连接失败，接下来就要通过该方法完成连接操作。

write(ByteBuffer src): 往通道里写数据

read(ByteBuffer dist): 网通道里读数据

register(Selector, ops, att): 注册一个选择器并且设置监听事件，最后一个参数可以设置共享数据。

close(): 关闭通道



A `Socket` is a blocking input/output device. It makes the `Thread` that is using it to block on reads and potentially also block on writes if the underlying buffer is full. Therefore, you have to create a bunch of different threads if your server has a bunch of open `Socket`s.

A `SocketChannel` is a non-blocking way to read from sockets, so that you can have one thread communicate with a bunch of open connections at once. This works by adding a bunch of `SocketChannel`s to a `Selector`, then looping on the selector's `select()` method, which can notify you if sockets have been accepted, received data, or closed. This allows you to communicate with multiple clients in one thread and not have the overhead of multiple threads and synchronization.

`Buffer`s are another feature of NIO that allows you to access the underlying data from reads and writes to avoid the overhead of copying data into new arrays.

### Selector

处理个客户端连接。Selector能检测多个注册通道上是否有事件发生(注意：多个channel以事件的方式可以注册到同一个Selector)，如果有事件发生，便获取事件然后针对每个事件进行相应的处理。这样就可以只用一个单线程去管理多个通道，也就是管理多个连接和请求。

只有在连接真正有读写事件发生时，才会进行读写，减少了系统开销，并且不必为每个连接都创建一个线程，不用去维护多个线程。

#### 相关方法

open() : 得到一个选择器对象

select(long timeout) : 监控所有注册的通道，当有IO操作可以进行时，将对应的SelectionKey加入到内部集合中并返回。

selectedKeys() : 从内部集合中得到所有的SelectionKey

![image-20200805144941937](..\typora-user-images\image-20200805144941937.png)

1. 当客户端连接时，会通过ServerSocketChannel创建得到相应的SocketChannel
2. Selector进行监听，将得到的socketChannel注册到selector上，register(Selector sel, int ops), 一个selector上可以注册多个Channel
3. 注册后返回一个selectionkey，会和该Selector关联。
4. select()方法返回有事件发生的通道的个数。
5. 进一步得到各个selectionKey
   6. 再通过SelectionKey反向获取对应的 ()
7. 通过得到channel，完成业务处理。

### NIO 与 零拷贝

![img](https://pic3.zhimg.com/80/v2-471a7563187e4486f1596debd74d1d32_720w.jpg)

Linux 2.1

![img](https://pic1.zhimg.com/80/v2-63e52971d182791d3c4b1368e20585cf_720w.jpg)

Linux 2.4

![img](https://pic4.zhimg.com/80/v2-1a80c8fd81a635abb12313a7714c23d8_720w.jpg)

**零拷贝是没有CPU拷贝**

从操作系统的角度来说，因为内核缓冲区之间，没有数据是重复的（只有kernel buffer有一份数据）。

零拷贝不仅仅带来更少的数据复制，还能带来其他的性能优势，例如更少的上下问切换，更少的CPU缓存伪共享以及无CPU校验和计算。

```java
File file = new File("a.txt");
RandomAccessFile raf = new RandomAccessFile(file, "rw");
byte[] arr = new byte[(int) file.length()];
raf.read(arr);
Socket socket = new ServerSocket(8080).accept();
socket.getOutputStream().write(arr);
```

这段代码在cpu中：

先从Hard drive里DMA 拷贝内存（Direct Memory Access 直接内存拷贝不使用CPU）到Kernel buffer，然后切换回用户态，通过CPU，然后CPU再次拷贝到socket Buffer，最后DMA copy到protocol engine（协议栈）。

一共四次拷贝 三次切换状态

![什么是零拷贝？mmap与sendFile的区别是什么？_weixin_37782390的博客 ...](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy80MjM2NTUzLWM1ZWEwMGI3OGUxYjkzZmQucG5n?x-oss-process=image/format,png)

#### mmap优化



mmap通过内存映射，将文件映射到内核缓冲区，同时，用户控件可以共享内核空间的数据。这样，在进行网络传输时，就可以减少内核空间到用户空间的拷贝次数。

![什么是零拷贝？mmap与sendFile的区别是什么？_weixin_37782390的博客 ...](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy80MjM2NTUzLWM1ZWEwMGI3OGUxYjkzZmQucG5n?x-oss-process=image/format,png)

Kernel buffer和user buffer可以共享数据，不必拷贝。

#### sendFile 优化

Linux2.1 版本提供了sendFile函数，原理：数据根本不经过用户态，直接从内核缓冲区进入到Socket Buffer，同时，由于和用户态完全无关，就减少了一次上下文切换。

![snedFile 2.1 版本](https://upload-images.jianshu.io/upload_images/4236553-695904d24edc2fd5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在2.4版本中，做了一些修改，避免了从内核缓冲区拷贝到Socket buffer的操作，直接拷贝到协议栈，从而再一次减少了数据拷贝

![ sendFile 在 2.4 版本的再一次优化](https://upload-images.jianshu.io/upload_images/4236553-00c3e47936ecea25.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其实还是存在CPU拷贝的，kernel buffer -> Socket buffer，但是拷贝的信息很少，比如length，offset，消耗低，可以忽略。

区别：

**mmap适合小数据量读写，需要4次上下文切换，3次数据拷贝；sendFile适合大文件传输，需要3次上下文切换，最少2次数据拷贝。sendFile可以利用DMA方式，减少CPU拷贝，mmap则不能（必须从内核拷贝到Socket缓冲区）**





## Netty

原生NIO存在的问题：

1. NIO的类库和API繁杂，使用麻烦：需要掌握Selector、ServerSocketChannel、SocketChannel、ByteBuff等。
2. 需要具备其他的额外技能：熟悉java多线程，因为NIO编程设计到Reactor模式，需要对多线程和网络编程

Netty Core:

- Extensible Event Model
- Universal Communication API
- Zero-Copy-Capable Rich Byte Buffer

帮助你快速简单地开发网络应用，相当于简化和流程化了NIO的开发过程。

优点：

1. 设计优雅：适用于各种传输类型的统一 API阻塞和非阻塞Socket；基于灵活且可扩展的事件模型，可以清晰地分离关注点；高度可定制的线程模型。
2. 使用方便。依赖度低，只依赖JDK
3. 社区活跃。

### Netty的零拷贝

Netty的零拷贝主要体现在五个方面

1. Netty的接收和发送ByteBuffer使用直接内存进行Socket读写，不需要进行字节缓冲区的二次拷贝。如果使用JVM的堆内存进行Socket读写，JVM会将堆内存Buffer拷贝一份到直接内存中，然后才写入Socket中。相比于使用直接内存，消息在发送过程中多了一次缓冲区的内存拷贝。
2. Netty的文件传输调用FileRegion包装的transferTo方法，可以直接将文件缓冲区的数据发送到目标Channel，避免通过循环write方式导致的内存拷贝问题。
3. Netty提供CompositeByteBuf类, 可以将多个ByteBuf合并为一个逻辑上的ByteBuf, 避免了各个ByteBuf之间的拷贝。
4. 通过wrap操作, 我们可以将byte[]数组、ByteBuf、ByteBuffer等包装成一个Netty ByteBuf对象, 进而避免拷贝操作。
5. ByteBuf支持slice操作，可以将ByteBuf分解为多个共享同一个存储区域的ByteBuf, 避免内存的拷贝。

### Netty线程模型

![io-diff](https://user-gold-cdn.xitu.io/2019/2/24/1691ec1919cbe0ac?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

现有的线程模型：

- 产痛阻塞I/O服务模型
- Reactor模式

根据Reactor的数量和处理资源池线程的数量不同，有3种典型的实现：

- 单Reactor单线程
- 单Reactor多线程
- 主从Reactor多线程

Netty线程模式主要基于主从Reactor多线程模型做了一定的改进。

#### 阻塞IO模型

模型特点：1. 才采用阻塞IO模式获取输入的数据；2. 每个连接都需要独立的线程完成数据的输入，业务处理，数据返回。

问题：1. 当并发数很大时会创建大量的线程，占用很大系统资源；2.连接创建后，如果当前线程暂时没有数据可读，该线程会阻塞在read操作造成资源浪费。

![image-20200807151500354](..\typora-user-images\image-20200807151500354.png)

#### Reactor模式

针对传统阻塞IO服务模型的两个缺点，解决方案：

1. 基于I/O复用模型：多个连接公用一个阻塞对象，应用程序只需要在一个阻塞对象等待，无需阻塞等待所有连接。当某个连接有新的数据可以处理时，操作系统通知应用程序，线程从阻塞状态返回，开始进行业务处理。
2. 基于线程池复用线程资源：不必再为每个连接创建线程，将连接完成后的业务处理任务分配给线程进行处理，一个线程可以处理多个连接的业务。

Reactor: 分发者模式，通知者模式，反应器模式

![image-20200807154109771](..\typora-user-images\image-20200807154109771.png)

1. Reactor模式，通过一个或多个输入同时传递给服务处理器的模式（基于事件驱动）
2. 服务器端程序处理传入的多个请求，并将它们同步分派到相应的处理线程。
3. Reactor模式使用IO复用监听事件，收到事件后，分发给某个线程（进程），这点就是网路服务高并发处理的关键。

Reactor模式中核心组成：

1. Reactor：reactor在一个单独的线程中运行，负责监听和分发事件，分发给适当的处理程序来对IO事件作出反应。它就像公司的电话接线员，它接听来自客户的电话并将线路转移到适当的联系人。
2. handler: 处理程序执行I/O事件要完成的实际事件，类似于客户想要与之交谈的实际官员。

#### 单Reactor单线程

以群聊来说，单线程一个reactor的情况，如果reactor分发之后线程阻塞在了handler read上，就很有可能造成后续的请求无法处理。

方案说明：

1. Select是前面I/O复用模型介绍的标准网络编程API，可以实现程序通过一个阻塞对象监听多路连接请求
2. Reactor对象通过select监控客户端请求事件，收到事件后通过Dispatch进行分发
3. 如果是建立连接的饿请求事件，则有Acceptor通过Accept处理连接请求，然后创建一个Handler对象处理连接完成后的后续业务处理
4. 如果不是建立连接时间，则Reactor会分发调用连接对应的Handler来相应
5. Handler会完成Read->业务处理->send的完成业务流程

服务器端用一个线程通过多路复用搞定所有的IO操作（包括连接，读、写等），编码简单，清晰明了，但是如果客户端连接数量较多，将无法支撑。

优点：模型简单，没有多线程、进程通信、竞争的问题，全部都在一个线程中完成。

缺点：性能问题，只有一个线程，无法完全发挥多核CPU的性能。Handler在处理某个连接上的业务时，整个进程无法处理其他连接时间，很容易导致性能瓶颈。并且，如果线程意外终止或者进入死循环，会导致整个系统通信模块不可用，造成节点故障。

![image-20200807154753270](..\typora-user-images\image-20200807154753270.png)

#### 单reactor多线程

方案说明：

1. Reactor对象通过select监控客户端请求事件，收到事件后，通过dispatch进行分发。
2. 如果建立连接请求，则有Acceptor通过accept处理连接请求，然后创建一个Handler对象处理完成连接后的各种事件。
3. 如果不是连接请求，则由reactor分发调用连接对应的handler来响应。
4. handler只负责响应事件，不做具体的业务处理，通过read读取数据后，会分发给后面的worker线程池的某个线程处理业务。
5. worker线程池会分配独立线程完成真正的业务，并将结果返回给handler。
6. handler收到响应后，通过send将结果返回给client。

优点：可以充分利用多核CPU的处理能力

缺点：多线程数据共享和访问比较复杂，reactor处理所有的事件的监听和响应。在单线程运行，在高并发场景容易出现性能瓶颈。

![image-20200807161501756](..\typora-user-images\image-20200807161501756.png)

#### 主从Reactor多线程

方案说明：

1. Reactor主线程 MainReactor通过select监听连接时间，收到事件后，通过Accept处理连接事件
2. 当Acceptor处理连接事件后，MainReactor将连接分配给SubReactor
3. SubReactor将连接加入到连接队列进行监听，并创建handler进行各种事件处理。
4. 当有新事件发生时，subreactor就会调用对应的handler处理
5. handler通过read读取数据，分发给后面的worker线程处理
6. worker线程池分配独立的worker线程进行业务处理并返回结果。
7. handler收到相应的结果后，再通过send方法返回给client。
8. Reactor主线程可以对应多个Reactor子线程，即MainReactor可以关联多个SubReactor。

优点：父线程与子线程的数据交互简单职责明确，父线程只需要接受新连接，子线程完成后续的业务处理；父线程与子线程数据交互简单，reactor主线程主需要把心连接传给子线程，子线程无需返回数据。

缺点：编程复杂度较高。

![image-20200807164449823](..\typora-user-images\image-20200807164449823.png)

### Netty 模型

```
@Value(("${edge.south.tcp.reconnect-delay}"))
private int RECONNECT_DELAY;
```

示意图：（简单版）

![image-20200807171555329](..\typora-user-images\image-20200807171555329.png)

1. BossGroup线程维护Selector，只关注Acceptor。
2. 当接收到Accept事件，获取到对应的SocketChannel，封装成NIOSocketChannel并注册到Worker线程（事件循环），并进行维护
3. 当Worker线程监听到Selector中的通道发生自己感兴趣的事件后，就进行处理（就由handler来完成），注意handler已经加入到通道。

（进阶版）

![image-20200807172301312](..\typora-user-images\image-20200807172301312.png)

（详细版）

![image-20200807172551219](..\typora-user-images\image-20200807172551219.png)

1. Netty抽象出两组线程池BossGroup专门负责接收客户端的连接，WorkerGroup专门负责网络的读写
2. BossGroup和WorkerGroup类型都是NIOEventLoopGroup
3. NioEventLoopGroup相当于一个事件循环组，这个组中含有多个时间循环，每一个事件循环是NioEventLoop
4. NioEventLoop表示已个不断循环的处理任务的线程，每个NioEventLoop都有一个Selector，用于监听绑定在其上的socket的网络通讯。
5. NioEventLoopGroup可以有多个线程，即可以含有多个NioEventLoop
6. 每个Boss NioEventLoop执行的步骤有三步
   1. 轮询accept事件
   2. 处理accept事件，与client建立连接，生成NioSocketChannel，并将其注册到某个worker NIOEventLoop上的selector
   3. 处理任务队列的任务，即runAllTask
7. 每个worker NIOEventLoop循环执行的步骤
   1. 轮询 read，write事件
   2. 处理I/O时间，即read、write事件，在对应NioSocketChannel处理
   3. 处理任务队列的任务，即runAllTasks
8. 每个worker NIOEventLoop处理业务时，会使用到pipeline，pipeline中包含了channel，即通过pipeline可以获取到对应的channel。pipeline中维护了很多处理器。





#### Pipeline

在Netty中，ChannelEvent是数据或者状态的载体，例如传输的数据对应MessageEvent，状态的改变对应ChannelStateEvent。当对Channel进行操作时，会产生一个ChannelEvent，并发送到ChannelPipeline。ChannelPipeline会选择一个ChannelHandler进行处理。这个ChannelHandler处理之后，可能会产生新的ChannelEvent，并流转到下一个ChannelHandler。

![输入图片说明](https://static.oschina.net/uploads/img/201609/11120254_iSgD.png)

例如，一个数据最开始是一个MessageEvent，它附带了一个未解码的原始二进制消息ChannelBuffer，然后某个Handler将其解码成了一个数据对象，并生成了一个新的MessageEvent，并传递给下一步进行处理。





### taskQueue

把花费时间必要长的任务提交到NioEventLoop里的taskQueue。taskQueue跟Channel有绑定关系。任务队列中的Task有3种典型使用场景

1. 用户程序自定义的普通任务

   1.

   ```java
           ctx.channel().eventLoop().execute(new Runnable() {
               @Override
               public void run() {
                   try {
                       Thread.sleep(5000);
                       ctx.writeAndFlush(Unpooled.copiedBuffer("读完成了", CharsetUtil.UTF_8));
                   } catch (Exception e) {
                       System.out.println("发生异常");
                       e.printStackTrace();
                   }
               }
           });		
   ```

2. 用户自定义定时任务

3. 非当前Reactor线程调用Channel的各种方法

   例如在推送系统的业务线程里面，根据用户标识，找到对应的Channel引用，然后调用Write类方法向该用户推送消息，就会进入这样的场景。最终的Write会提交到任务队列中后被异步消费。

### Netty异步模型

1. 异步的概念和同步相对，当一个异步过程调用发出后，调用者不能立刻得到结果。时机处理这个调用的组件在完成，通过状态、通知和回调来地通知调用者。
2. Netty中的I/O操作是异步的，包括Bind、Write、Connect等操作会简单的返回一个ChannelFuture。
3. 调用者并不能立刻获得结果，而是通过Future-Listener机制，用户可以方便地主动获取或者通过通知机制获得IO操作结果。
4. Netty的异步模型是建立在future和callback之上的。核心思想是：假设一个方法func，计算过程可能非常耗时，等待func返回显然不合适，name可以再调用func的时候，立马返回一个Future，后续可以通过Future去监控方法fun的处理过程。（即：Future-Listener机制）

Future 说明：

1. 表示异步执行结果，可以通过它提供的方法来检测执行是否完成，比如检索计算等等。

2. ChannelFuture是一个接口.

   我们可以添加监听器，当监听的事件发生时，就会通知监听器。

工作原理：

![image-20200810132947823](C:\Users\xdu4sgh\AppData\Roaming\Typora\typora-user-images\image-20200810132947823.png)

1. 在使用netty编程时，连接操作和转换出入站数据只需要你提供callback或利用future即可。这使得链式操作简单、搞笑，并有利于编写可重用的、通用的代码
2. Netty框架的目的就是让你的业务逻辑从网络基础应用编码中分离出来、解脱出来。

#### Future-Listener 机制

1. 当future对象刚刚创建时，处于非完成状态，调用者可以通过返回的ChannelFuture来获取操作执行的状态，注册监听函数来执行完成后的操作。
2. 常见操作：
   - isDone: 判断当前操作是否完成
   - isSuccess: 判断已完成操作是否成功
   - getCause: 获取已完成的当前操作失败的原因
   - isCancelled: 判断已完成的当前操作是否被取消
   - addListener: 注册监听器，当操作已完成，将会通知指定的监听器；如果Future对象已完成，则通知指定的监听器。

相比传统阻塞I/O，执行IO操作后线程会被阻塞住，知道操作完成；异步处理的好处是不会造成线程阻塞，线程在I/O操作期间可以作兴别的程序，在高并发清醒下会更稳定和更高的吞吐量。

### Netty 核心组件

#### Bootstrap, ServerBootstrap

Bootstrap 的意思是引导，一个Netty应用通常由一个BootStrap开始，主要作用是配置整个netty程序，串联各个组件，Netty中Bootstrap类是客户端程序的启动引导类，ServerBootstrap是服务端启动引导类。

常见的方法：

group：用来设置两个EventLoop(ServerBootstrap); 用来设置一个EventLoopGroup(Bootstrap)

channel：用来设置一个服务器端的通道实现类

option：用来给serverChannel添加配置

childOption：用来给接收到的通道添加配置

childHandler：用来设置业务处理类

bind：占用端口号

#### Future, ChannelFuture

Netty中所有IO操作都是异步的，不能立刻得知消息是否被正确处理。但是可以过一会等它执行完成或者直接注册一个监听，具体实现就是通过Future和ChannelFutures，他们可以注册一个监听，当操作执行成功或失败时监听会自动触发注册的监听事件

常用的方法：

channel(): 返回当前正在进行IO操作的通道

sync(): 等待异步操作执行完毕

#### Channel

Netty网络通信的组件，能够用于执行网络I/O操作。

通过Channel可以获得当前网络连接的通道的状态和网络连接的配置参数（例如接受缓冲区的大小）

#### Selector

Netty基于Selector对象实现I/O多路复用，通过Selector一个线程可以监听多个连接的Channel事件。

当想一个Selector中注册Channel后，Selector内部的机制就可以自动不断地查询（select）这些注册的Channel是否有已就绪的I/O事件（例如：可读，可写，网络连接完成等），这样程序就可以很简单地使用一个线程高效地管理多个Channel。

#### ChannelHandler

1. ChannelHandler是一个接口，处理I/O事件或者拦截I/O操作，并将其转发到其他ChannelPipeline（业务处理链）中的下一个处理程序。

2. ChannelHandler本身并没有提供很多方法，因为这个接口有许多方法需要事件，方便使用期间，可以继承它的子类

   ![img](https://img-blog.csdn.net/20160504160757707?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

3. ChannelPipeline提供了ChannelHandler链的容器，以客户端应用为例，如果事件运动方向是从客户端到服务端的，那么我们称这些事件为出站，即了护短发送给服务端的数据会通过pipeline中的以西系列ChannelOutboundHandler，并被这些Handler处理，反之则成为入站的。

#### Pipeline和ChannelPipeline

**重点**

1. ChannelPipeline是一个Handler的集合，它负责处理和拦截inbound或者outbound的事件和操作，相当于一个贯穿Netty的链。（也可以理解为：ChannelPipeline是保存Channelhandler的list，用于处理或拦截Channel的入站事件和出站操作）
2. ChannelPipeline实现了一种高级形式的拦截过滤器模式，使用户可以完全控制事件的处理方式，以及Channel中各个的ChannelHandler如何相互交互。

3. Netty中每个Channel都有且仅有一个ChannelPipeline与之对应

   ![image-20200810164419611](C:\Users\xdu4sgh\AppData\Roaming\Typora\typora-user-images\image-20200810164419611.png)

   - 一个Channel包含乐意i个ChannelPipeline，而ChannelPipeline中又维护了一个由ChannelHandlerContext组成的双向链表，并且每个ChannelHandlerContext中又关联着一个ChannelHandler
   - 入站事件和出站事件在一个双向链表中，入站事件会从链表head往后 传递到最后一个入站的handler，出站事件会从链表tail往前传递到最前一个出站的handler，两种类型的handler互不干扰。

#### ChannelHandlerContext组件

**Pipeline链表里的节点类型**

1. 保存Channel相关的所有上下文信息，同事关联一个Channelhandler对象
2. 即ChannelHandlerContext中包含一个具体的事件处理器ChannelHandler，同事ChannelHandlerContext中也绑定了对应的pipeline和Channel的信息，方便对Channelhandler进行调用
3. 常用方法：
   - close（），关闭通道
   - flush（），刷新
   - writeAndFlush（Object msg），将数据写到ChannelPipeline中当前ChannelHandler的下一个ChannelHandler开始处理（出站）

#### ChannelOption

ChannelOption.SO_BACKLOG

对应TCP/IP协议listen函数中的backlog参数，用来初始化服务器可连接队列大小。服务端处理客户端连接请求是顺序处理的，所以同一时间只能处理一个客户端连接。多个客户端来的时候，服务端将不能处理客户端连接请求放在队列中等待处理，backlog参数指定了队列的大小。

#### EventLoopGroup和其实现类NioEventLoopGroup

1. EventLoopGroup是一组EventLoop的抽象，Netty为了更好的利用多核CPU资源，一般会有多个EventLoop同时工作，每个EventLoop维护着一个Selector实例。

2. EventLoopGroup提供next接口，可以从组里面按照一定规则获取其中一个EventLoop来处理任务。在Netty服务器端编程中，我们一般都需要提供两个EventLoopGroup。

3.  通常一个服务器端口即一个ServerSocketChannel对应一个Selector和一个EventLoop线程。BossEventLoop负责接收客户端的连接并将SocketChannel交给WorkerEventLoopGroup来进行IO处理。

   ![image-20200810213437630](C:\Users\xdu4sgh\AppData\Roaming\Typora\typora-user-images\image-20200810213437630.png)

4. 常用方法：

   NioEventLoopGroup（）：构造方法

   shutdownGracefully（）：断开连接，关闭线程

#### Unpooled类

1. Netty提供一个专门用来操作缓冲区（即Netty的数据容器）的工具类

2. 常用方法：

   public static ByteBuf copiedBuffer(CharSequence string, Charset charset): 通过给定的数据和字符编码返回一个ByteBuf对象

### Netty 心跳处理

```java
                            // 加入一个Netty提供的IdleStateHandler
                            // 1. IdleStateHandler是netty提供的处理空闲状态的处理器
                            // 2. 变量分别为: 多长时间没有读，多长时间没有写，多长时间没有读写 就会发送一个心跳检测包，检测是否还是连接的状态
                            // 3. 文档说明
                            // * Triggers an {@link IdleStateEvent} when a {@link Channel} has not performed
                            // * read, write, or both operation for a while.
                            // 如果IdleStateEvent被触发，就会传递给管道的下一个，通过调用下一个handler的userEventTriggerd()方法
                            pipeline.addLast(new IdleStateHandler(5,10,15, TimeUnit.SECONDS));
                            // 加入一个对空闲检测进一步处理的自定义的handler
                            pipeline.addLast(new HeartBeatServerHandler());
```

在HeartBeatServerHandler里面做业务处理：

```java
@Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        IdleStateEvent event = (IdleStateEvent) evt;
        IdleState state = event.state();
        ...logic code...
    }
```

## WebSocket

- 在传统的b/s架构中，要实现服务器向client进行实时消息推送功能，市场上常用的解决方案大致分为三类：

| 定时轮询   | 客户端以一定的时间间隔向服务端发出请求                       |
| ---------- | ------------------------------------------------------------ |
| **长轮询** | **如果服务器没有可以立即返回给客户端的数据，则不会立刻返回一个空结果** |
| **流技术** | **客户端隐藏的窗口向服务端发出一个长轮询的请求**             |

长轮询机制：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530183427446.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d3eDY5MjM3MTcwMA==,size_16,color_FFFFFF,t_70)

综合这几种方案，您会发现这些目前我们所使用的所谓的实时技术并不是真正的实时技术，它们只是在用 Ajax 方式来模拟实时的效果，定时轮询需要实时获取取服务端信息的应用时, client需要频繁轮询server，为了拿到最新信息, client一直这样循环下去server如果一直没有新的消息, client的大多请求都是属于无效请求, 导致会带来很多无谓的网络传输，所以这是一种非常低效的实时方案。长轮询对服务器造成的压力非常大，并且如果服务端的数据变更非常频繁的话，这种方式无异于定时轮询。所以为了解决传统http请求的实际问题，WebSocket技术应运而生！下面博主给张图让大家生动的理解传统HTTP和WebSocket的差异化：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530190931986.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d3eDY5MjM3MTcwMA==,size_16,color_FFFFFF,t_70)

## 编码、解码

1. 编写网络应用程序时，因为数据在网络中传输的都是二进制字节码数据，在发送数据时就需要编码，接受数据时就需要解码
2. codec(编码器)的组成部分有两个：decoder和encoder，encoder负责把业务数据转换成字节码数据，decoder负责把字节码数据转换成业务数据

#### Protobuf

可以用于结构化数据串行化，或者说序列化。它适合做数据存储或RPC(远程过程调用 remote procedure call)

// todo

### 出站入站

1. Netty的组件设计：channel, EventLoop, ChannelFutrue, ChannelHandler, ChannelPipe等
2. ChannelHandler充当了处理入站和出站数据的应用程序逻辑的容器。例如，实现ChannelInboundHandler接口，你就可以接受入站事件和数据，这些数据会被业务逻辑处理。当要给客户端发送响应时，也可以从ChannelInboundHandler冲刷数据。业务逻辑通常卸载一个或者多个ChannelInboundHandler中。CHannelOutboundHandler原理一样，只不过它是用来处理出站数据的。

### 编码解码器

1. 当Netty发送或者接受一个消息的时候，就将会发生一次数据转换。入站消息会被解码：从字节转换为另一种格式（比如java对象）；如果是出站消息，它会被编码成字节。
2. Netty提供一系列使用的编解码器，他们都实现了ChannelInboundHandler或者ChannelOutboundHandler接口。在这些类中，channelRead方法已经被重写了，以入站为例，对于每个从入站Channel读取的消息，这个方法会被调用。随后，它将调用由解码器所提供的decode()方法进行解码，并将已经解码的字节转发给ChannelPipeline中的下一个ChannelInboundHandler。

#### ByteToMessageDecode 解码器

write方法里会判断你的message是否是你在解码器中定义的类型：acceptOutboundMessage(msg)

如果不是则不会处理

 ```java
@Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        ByteBuf buf = null;
        try {
            // 判断是否为定义的类型
            if (acceptOutboundMessage(msg)) {
                @SuppressWarnings("unchecked")
                I cast = (I) msg;
                buf = allocateBuffer(ctx, cast, preferDirect);
                try {
                    // 如果是 就encode
                    encode(ctx, cast, buf);
                } finally {
                    ReferenceCountUtil.release(cast);
                }

                if (buf.isReadable()) {
                    ctx.write(buf, promise);
                } else {
                    buf.release();
                    ctx.write(Unpooled.EMPTY_BUFFER, promise);
                }
                buf = null;
            } else {
                // 如果不是就直接写出去
                ctx.write(msg, promise);
            }
        } catch (EncoderException e) {
            throw e;
        } catch (Throwable e) {
            throw new EncoderException(e);
        } finally {
            if (buf != null) {
                buf.release();
            }
        }
    }
 ```

因此我们在编写encoder的时候要注意传入的数据类型和处理的数据类型一致。

**不论解码器handler还是编码器handler即接收的消息类型必须与待处理的消息类型一致，否则该handler不会被执行**

在解码器进行数据解码时，需要判断缓存区（byteBuf）的数据是否足够，否则接收到的结果和期望结果可能不一致。

##### ReplayingDecode 的局限性：

- 不是所有的ByteBuf操作都被支持，如果调用了一个不被支持的方法，将会抛出一个UnsupportedOperationException。
- ReplayingDecoder在某些情况下可能稍慢于ByteToMessageDecoder，例如网络缓慢并且消息格式复杂时，消息会被拆成了多个碎片，速度变慢。

##### LineBasedFrameDecoder:

这个类在Netty内部也有使用，它使用行尾控制字符（\n 或者\r\n）作为分隔符来解析数据。

##### DelimiterBasedFrameDecoder：

使用自定义的特殊字符作为消息的分隔符。

##### HttpObjectDecoder：

一个Http数据的解码器

##### LengthFieldBasedFrameDecoder：

通过制定长度来标识整包消息，这样就可以自动的处理粘包和半包消息。

### TCP粘包和拆包基本介绍

1. TCP是面向连接的，面向流的，提供高可靠性服务。收发两端都要有一一承兑的socket，因此，发送单为了将多个发给接收端的包，更有效地发给对方，使用了优化方法（Nagle算法），将多次间隔较小且数据量小的数据，合并成一个大的数据块，然后进行封包。这样做虽然提高了效率，但是接受端就难于分辨出完整的数据包了，因为面向流的通信是无消息保护边界的。
2. 由于TCP无消息保护边界，需要再接收端处理消息边界问题，也就是我们所说的粘包、拆包问题。

![image-20200813162434489](C:\Users\xdu4sgh\AppData\Roaming\Typora\typora-user-images\image-20200813162434489.png)

保护消息边界，就是指传输协议把数据当作一条独立的消息在网上传输，接收端只能接收独立的消息。也就是说存在保护消息边界，接收端一次只能接收发送端发出的一个数据包。而面向流则是指无保护**消息保护边界**的，如果发送端连续发送数据，接收端有可能在一次接收动作中，会接收两个或者更多的数据包。

##### 粘包发生的原因：

1发送端需要等缓冲区满才发送出去，造成粘包

2接收方不及时接收缓冲区的包，造成多个包接收

具体点：

（1）发送方引起的粘包是由TCP协议本身造成的，TCP为提高传输效率，发送方往往要收集到足够多的数据后才发送一包数据。若连续几次发送的数据都很少，通常TCP会根据优化算法把这些数据合成一包后一次发送出去，这样接收方就收到了粘包数据。

（2）接收方引起的粘包是由于接收方用户进程不及时接收数据，从而导致粘包现象。这是因为接收方先把收到的数据放在系统接收缓冲区，用户进程从该缓冲区取数据，若下一包数据到达时前一包数据尚未被用户进程取走，则下一包数据放到系统接收缓冲区时就接到前一包数据之后，而用户进程根据预先设定的缓冲区大小从系统接收缓冲区取数据，这样就一次取到了多包数据。

#### 解决方案：

1. 使用自定义协议 + 编解码器
2. 关键就是要解决服务端每次读取数据长度的问题，这个问题解决，就不会出现服务器多读或少读数据的问题，从而避免TCP粘包、拆包

### Netty 源码

#### Netty启动过程源码剖析

1. 需要婆媳到Netty调用doBind方法，最终到NioSerSocketChannel的doBind
2. Debug程序到NioEventLoop的run代码，无限循环，在服务器端运行

创建group：

```java
    /**
     * Set the {@link EventLoopGroup} for the parent (acceptor) and the child (client). These
     * {@link EventLoopGroup}'s are used to handle all the events and IO for {@link ServerChannel} and
     * {@link Channel}'s.
     */
    public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
        // 交给父类处理
        super.group(parentGroup);
        if (this.childGroup != null) {
            throw new IllegalStateException("childGroup set already");
        }
        this.childGroup = ObjectUtil.checkNotNull(childGroup, "childGroup");
        return this;
    }
```

然后添加Channel，其中参数一个Class对象，引导类将通过这个Class反射创建ChannelFactory。然后添加了一些TCP的参数。【说明：Channel的创建在bind方法，可以Debugbind，会找到channel = channelFactory.newChannel();】

handler交给bossgroup，把childHandler交给workerGroup

**链式调用**：

1. group方法，将boss和worker传入，boss赋值给parentGroup属性，worker赋值给childGroup属性
2. channel方法传入NioServerSocketChannel class对象，根据这个class创建channel对象
3. option方法传入TCP参数，放在一个LinkedHashMap中
4. handler方法传入一一个handler中，这个handler只专属于ServerSocketChannel而不是SocketChannel
5. childHandler传入一个handler，这个handler将会在每个客户端连接的时候调用。供SocketChannel使用



在dobind的时候，创建channel

通过ServerBootstrap的通道工程反射创建一个NioServerSocketChannel

**channel = channelFactory.newChannel();**

追踪代码后发现：

1. 通过NIO的SelectorProvider的openServerSocketChannel方法得到JDK的channel。目的是让Netty包装JDK的channel。
2. 创建了一个唯一的ChannelID，创建了一个NioMessageUnsafe，用于操作消息，创建了一个DefaultChannelPipeline管道，是个双向链表结构，用于过滤所有进出消息。
3. 创建了一个NioServerSocketChannelConfig对象，用于对外展示一些配置。

** init(channel);**

1. init，是抽象方法（AbstractBootstrap类的，由ServerBootstrap实现（可以追一下源码 // setChannelOptions(channel, options,logger);),
2. 设置NioServerSocketChannel的TCP属性。
3. 由于LinkedHashMap是非线程安全的，使用同步进行处理
4. 对NioServerSocketChannel的ChannelPipeline添加ChannelInitializer处理器。
5. 可以看出，init的方法的核心作用在和ChannelPipeline相关。
6. 从NioServerSocketChannel的初始化过程中，我们知道，pipeline是一个双向链表，并且，他本身就初始化了head和tail，这里调用了他的addLast方法，也就是将整个handler插入到tail的前面，因为tail永远会在后面，需要做一些系统的固定工作。



```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        // 通过ServerBootstrap的通道工程反射创建一个NioServerSocketChannel:
        channel = channelFactory.newChannel();
        // 此处很重要
        init(channel);
    } catch (Throwable t) {
        if (channel != null) {
            // channel can be null if newChannel crashed (eg SocketException("too many open files"))
            channel.unsafe().closeForcibly();
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }
        // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
        return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
    }

    ChannelFuture regFuture = config().group().register(channel);
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }

    // If we are here and the promise is not failed, it's one of the following cases:
    // 1) If we attempted registration from the event loop, the registration has been completed at this point.
    //    i.e. It's safe to attempt bind() or connect() now because the channel has been registered.
    // 2) If we attempted registration from the other thread, the registration request has been successfully
    //    added to the event loop's task queue for later execution.
    //    i.e. It's safe to attempt bind() or connect() now:
    //         because bind() or connect() will be executed *after* the scheduled registration task is executed
    //         because register(), bind(), and connect() are all bound to the same thread.

    return regFuture;
}
```

1. initAndRegister()初始化NioServerSocketChannel通道并出则各个handler，返回一个future
2. 通过ServerBootstrap的通道工程反射创建一个NioServerSocketChannel。
3. init初始化这个NioServerSocketChannel。
4. config().group().rigister(channel)通过ServerBootstrap的bossGroup注册NioServerSocketChannel。
5. 最后，返回这个异步执行的展位符即regFuture

**addLast()**

1. 在DefaultChannelPipeline类中
2. addLast方法就是pipeline方法的核心
3. 价差该handler是否符合标准
4. 创建一个AbstractChannelHandlerContext对象，这里说一下，ChannelHandlerContext对象是ChannelHandler和ChannelPipeline之间的关联，**每当有ChannelHandler添加到Pipeline中时，都会创建Context。**Context的主要功能是管理他所管理的Handler和同一个Pipeline中的其他Handler之间的交互。
5. 将Context添加到链表中，也就是追加到tail节点的前面。
6. 最后同步或异步或者晚点异步调用callHandlerAdded()方法。

```java
 @Override
    public final ChannelPipeline addLast(String name, ChannelHandler handler) {
        //  调用了下一个addLast
        return addLast(null, name, handler);
    }

    @Override
    public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
        final AbstractChannelHandlerContext newCtx;
        synchronized (this) {
            checkMultiplicity(handler);

            newCtx = newContext(group, filterName(name, handler), handler);

            addLast0(newCtx);

            // If the registered is false it means that the channel was not registered on an eventLoop yet.
            // In this case we add the context to the pipeline and add a task that will call
            // ChannelHandler.handlerAdded(...) once the channel is registered.
            if (!registered) {
                newCtx.setAddPending();
                callHandlerCallbackLater(newCtx, true);
                return this;
            }

            EventExecutor executor = newCtx.executor();
            if (!executor.inEventLoop()) {
                callHandlerAddedInEventLoop(newCtx, executor);
                return this;
            }
        }
        callHandlerAdded0(newCtx);
        return this;
    }

    private void addLast0(AbstractChannelHandlerContext newCtx) {
        AbstractChannelHandlerContext prev = tail.prev;
        // 将新的context插入到tail的前面一个
        newCtx.prev = prev;
        newCtx.next = tail;
        prev.next = newCtx;
        tail.prev = newCtx;
    }
```



#### EventLoop

```java
public abstract class MultithreadEventExecutorGroup extends AbstractEventExecutorGroup
```

// Todo





## RPC

1. Remote procedure Call 远程过程调用，是一个计算机通信协议。该协议允许运行于一台计算机的程序调用另一台计算机的子程序，而程序员无需额外地为这个交互作用编程。

2. 两个 多个应用程序都分布在不同的服务器上，他们之间的调用都像是本地方法调用一样。

![image-20200814160908770](C:\Users\xdu4sgh\AppData\Roaming\Typora\typora-user-images\image-20200814160908770.png)

3. 常见RPC框架有：阿里的Dubbo，google的gRPC，Go语言的rpcx，Apache的thrift，Spring旗下的Spring cloud

![image-20200814171558891](C:\Users\xdu4sgh\AppData\Roaming\Typora\typora-user-images\image-20200814171558891.png)

RPC中，client叫服务消费者，server叫服务提供者。

1、客户端（Client）:服务调用方（服务消费者）

2、客户端存根（Client Stub）:存放服务端地址信息，将客户端的请求参数数据信息打包成网络消息，再通过网络传输发送给服务端

3、服务端存根（Server Stub）:接收客户端发送过来的请求消息并进行解包，然后再调用本地服务进行处理4、服务端（Server）:服务的真正提供者

![在这里插入图片描述](https://user-gold-cdn.xitu.io/2020/4/14/1717841c300aefca?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)


1. 服务消费方（client）以本地调用方式调用服务
2. client stub接收到调用后服务将方法，参数等封装成能够进行网络传输的消息体
3. client stub将消息进行编码并发送到服务端
4. server stub收到消息后进行解码
5. server stub根据解码结构调用本地的服务
6. 本地服务执行将返回导入结果进行编码并发送至消费方
7. client stub接收到消息并进行解码
8. 服务消费方（client）得到结果

RPC的目标就是将2-8这些步骤都封装起来，用户无需关心这些细节，可以像调用本地方法一样既可完成远程服务调用。

自己实现dubbo RPC

模仿dubbo，消费者和提供者约定接口和协议，消费者远程调用地宫这，提供者返回一个字符串，消费者打印提供者返回的数据。底层网络通信使用Netty4.x。

设计说明：

1. 创建一一个借口，定义抽象方法。用于消费者和提供者之间的约定
2. 创建一个提供者，该类需要监听消费者的请求，并按照约定返回数据。
3. 创建一个消费者，该类需要透明的调用自己不存在的方法，内部需要使用Netty请求提供者返回数据。

![image-20200815013958816](..\typora-user-images\image-20200815013958816.png)
