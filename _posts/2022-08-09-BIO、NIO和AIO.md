---
redirect_from: /_posts/2022-08-09-网络IO模型/
title: BIO、NIO和AIO
tags:
  - Netty
  - I/O模型
---

## BIO
### BIO通讯模型
采用BIO通讯模型的服务端，通常会有一个Acceptor的线程去监听客户端的连接。当Acceptor接收到客户端的请求时会单独创建一个新的线程去进行链路处理，当处理完成之后通过输出流的方式应答客户端，然后销毁线程。

此模型最大的问题就是缺乏弹性伸缩能力，一旦有大量请求访问服务器时，就需要创建大量线程去进行链路处理。当线程数急剧膨胀，而且仍有大量请求访问服务器时就会引发堆栈溢出、线程无法创建等问题。最终导致服务僵死或直接宕机。

我们来模拟一下BIO方式:
```java
ServerSocket server = null;
try {
    server = new ServerSocket(port);
    System.out.println("The time server is start in port : " + port);
    Socket socket = null;
    while (true) {
        //当启动服务端后，我们可以发现线程在此处阻塞住了。直到有请求连接进来才进行下面的处理
        socket = server.accept();
        //这里就是我们每收到客户端一个请求就去创建一个新的线程去做链路处理。当大量请求过来，服务器资源就会被很快用尽。
        new Thread(new TimeServerHandler(socket)).start();
    }
} finally {
    if (null != server) {
        System.out.println("The time server close");
        server.close();
        server = null;
    }
}
```
其实上面这段代码最大的问题就是你每一个连接过来就会创建一个线程太耗资源，但是我们可以通过使用线程池来进行管 理，这样就可以控制创建的线程数量不至于过于膨胀导致内存溢出，也就是创建一个伪异步I/O。

伪异步I/O也只是优化了阻塞问题，但没有解决根本。当通讯长时间未应答时还是会引发级联故障:
1. 响应缓慢，平时只需要10ms，大量请求下可能会达到10s甚至更多
2. 伪异步I/O读取故障节点，由于读取输入流是阻塞的，所以连接会被同步阻塞
3. 如果所有可用线程都被阻塞，后续的I/O只能在队列里排队
4. 线程池采用阻塞队列实现，当队列满了后，后续入队操作会被阻塞
5. 由于前端只有一个Acceptor线程接入客户端，当队列阻塞后，会拒绝其他新的客户端请求并造成大量消息连接超时
6. 当大量请求都超时，客户端认为此时服务器已宕机，无法接收消息

## NIO
NIO提供了高速的面向块的I/O。它可以通过使用包装数据的类以及通过块的形式处理这些数据，这让NIO可以不使用本机代码就可以利用低级优化，这是原来的I/O做不到的。

### 缓冲区(Buffer)
在NIO中所有数据都是在缓冲区处理的，读取和写入都是直接操作的缓冲区(Buffer)。缓冲区本质上就是一个**数组**，通常是byte数组(除了Boolean类型，其他每种基本类型都有对应的一个数组)。Buffer中也不仅仅只含有一个数组，他还提供了对数据结构化的访问以及维护读写位置的limit信息。

---

### 通道(channel)
channel和stream最大的区别是，channel是双向的而stream是单向的，所以channel可以同时进行读写操作。并且channel是**全双工**，这也更好的映射了操作系统底层的API，因为操作系统底层的API也是全双工特性的。

channel主要分为2类: 用于网络读写的SelectableChannel和用于文件操作的FileChannel

---

### 多路复用器(selector)
selector提供了选择已经就绪任务的能力。selector会轮询channel，并筛选出已经就绪状态的channel，然后就可以通过selectionKey获取已经就绪的channel集合。

因为selector使用了epoll替代以前的poll，所以没有了最大链接句柄1024/2048的限制。使得系统理论上可以链接无穷无尽的客户端，大幅提升了服务器性能。

--- 

### 流程介绍
#### 服务端流程

1. 打开通讯管道(ServerSocketChannel)，用于监听客户端的链接
```java
ServerSocketChannel socketChannel = ServerSocketChannel.open();
```
2. 绑定监听端口，设置链接为非阻塞模式
```java
socketChannel.socket().bind(new InetSocketAddress(InetAddress.getByName("IP"), port));
socketChannel.configureBlocking(false);
```
3. 创建Reactor线程，创建多路复用器并启动线程
```java
Selector selector = Selector.open();
new Thread(new ReactorTask()).start();
```
4. 将ServerSocketChannel注册到Reactor线程的多路复用器上，监听ACCEPT事件
```java
acceptorServer.register(selector, SelectionKey.OP_ACCEPT, ioHandler);
```
5. 多路复用器在轮询准备就绪的Key
```java
int num = selector.select();
Set selectedKeys = selector.selectedKeys();
Iterator it = selectedKeys.iterator();
while(it.hasNext()) {
    SelectionKey key = it.next();
    //处理I/O事件
}
```
6. 多路复用器监听到有新的客户端接入，处理新的接入请求，完成TCP三次握手后建立物理链路
```java
SocketChannel channel = serverChannel.accept();
```
7. 设置客户端链路为非阻塞模式
```java
channel.configureBlocking(false);
channel.socket().setReuseAddress(true);
```
8. 将新接入的客户端连接注册到Reactor线程的多路复用器上进行监听
```java
SelectionKey key = socketChannel.register(selector, SelectionKey.OP_READ, ioHandler);
```
9.  异步读取客户端请求到缓冲区
```java
int readNumber = channel.read(receivedBuffer);
```
10. 对缓冲区数据进行编解码，如果有半包数据指针reset，继续读取后续的报文。将解码成功的消息封装成Task，投递到业务线程池中进行业务逻辑编排
```java
Object msg = null;
while(buffer.hasRemain()) {
    byteBuffer.mark();
    msg = decode(byteBuffer);
    if (null == msg) {
        byteBuffer.reset();
        break;
    }
    messageList.add(msg);
}
if (!byteBuffer.hasRemain()) {
    byteBuffer.clear();
} else {
    byteBuffer.compact();
}
if (null != messageList && !messagelist.isEmpty()) {
    for (Object obj : messageList) {
        handlerTask(obj);
    }
}
```
11. 将POJO对象encode到缓冲区，调用SocketChannel的异步写接口，将消息异步发送给客户端。
```java
socketChannel.write(buffer);
```
值得注意的一点就是，当写的过程中缓存区满了就会导致写半包，这时需要注册监听写操作位，然后不断轮询直到整包都写入缓存区。

#### 服务端代码分析
我们创建一个Server作为请求接入的接口
```java
public class NIOServer {

    public static void main(String[] args) throws IOException {
        int port = 8080;
        if (null != args && args.length > 0) {
            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {

            }
        }
        //多路复用器
        MultiplexerTimeServer timeServer = new MultiplexerTimeServer(port);
        new Thread(timeServer, "NIO-MultiplexerTimeServer-001").start();
    }

}
```
然后我们需要去实现多路复用器的功能
```java
public class MultiplexerTimeServer implements Runnable {

    private Selector selector;
    private ServerSocketChannel socketChannel;
    private volatile boolean stop;

    /**
     * 初始化多路复用器，并绑定指定端口
     * @param port
     */
    public MultiplexerTimeServer(int port) {
        try {
            selector = Selector.open();
            socketChannel = ServerSocketChannel.open();
            socketChannel.configureBlocking(false);
            socketChannel.socket().bind(new InetSocketAddress(port), 1024);
            socketChannel.register(selector, SelectionKey.OP_ACCEPT);
            System.out.println("The time server is start in port : " + port);
        } catch (IOException e) {
            e.printStackTrace();
            System.exit(1);
        }
    }

    public void stop() {
        this.stop = true;
    }

    @Override
    public void run() {
        while (!stop) {
            try {
                selector.select(1000);
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> it = selectionKeys.iterator();
                while (it.hasNext()) {
                    SelectionKey key = it.next();
                    it.remove();
                    try {
                        handleInput(key);
                    } catch (Exception e) {
                        if (null != key) {
                            key.cancel();
                            if (null != key.channel()) key.channel().close();
                        }
                    }
                }
            } catch (Throwable t) {
                t.printStackTrace();
            }
            if (null != selector) {
                try {
                    selector.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    private void handleInput(SelectionKey key) throws IOException {
        if (key.isValid()) {
            if (key.isAcceptable()) {
                ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
                SocketChannel sc = ssc.accept();
                sc.configureBlocking(false);
                sc.register(selector, SelectionKey.OP_READ);
            }
            if (key.isReadable()) {
                SocketChannel sc = (SocketChannel) key.channel();
                ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                int readBytes = sc.read(readBuffer);
                if (readBytes > 0) {
                    readBuffer.flip();
                    byte[] bytes = new byte[readBuffer.remaining()];
                    readBuffer.get(bytes);
                    String body = new String(bytes, "UTF-8");
                    System.out.println("The time server receive order : " + body);
                    String currentTime = "QUERY TIME ORDER".equalsIgnoreCase(body) ? new Date(System.currentTimeMillis()).toString() : "BAD ORDER";
                    doWrite(sc, currentTime);
                } else if (readBytes < 0) {
                    key.cancel();
                    sc.close();
                } else {
                    //ignore
                }
            }
        }
    }

    private void doWrite(SocketChannel channel, String response) throws IOException {
        if (null != response && response.trim().length() > 0) {
            byte[] bytes = response.getBytes();
            ByteBuffer writeBuffer = ByteBuffer.allocate(bytes.length);
            writeBuffer.put(bytes);
            writeBuffer.flip();
            channel.write(writeBuffer);
        }
    }

}
```
#### 客户端流程

1. 打开SocketChannel，绑定客户端本地地址(默认系统会随机分配一个可用地址)
```java
SocketChannel clientChannel = SocketChannel.open();
```
2. 设置SocketChannel为非阻塞模式，设置客户端连接的TCP参数
```java
clientChannel.configureBlocking(false);
socket.setReuseAddress(true);
socket.setReceiveBufferSize(BUFFER_SIZE);
socket.setSendBufferSize(BUFFER_SIZE);
```
3. 异步连接服务端
```java
boolean connected = clientChannel.connect(new InetSocketAddress("127.0.0.1", 8080));
```
4. 判断是否连接成功，如果连接成功就直接注册读状态到多路复用器中，如果返回
false说明客户端已经发送了sync包，服务端还没有返回ack包，物理链路还没有建立
```java
if (connected) {
    clientChannel.register(selector, SelectionKey.OP_READ, ioHandler);
} else {
    clientChannel.register(selector, SelectionKey.OP_CONNECT, ioHandler);
}
```

5. 创建Reactor线程，创建多路复用器并启动线程
```java
Selector selector = Selector.open();
new Thread(new ReactorTask()).start();
```
6. 多路复用器轮询准备就绪的Key
```java
int num = selector.select();
Set selectedKeys = selector.selectedKeys();
Iterator it = selectedKeys.iterator();
while(it.hasNext()) {
    SelectionKey key = (SelectionKey) it.next();
    //处理I/O事件
}
```
7. 接收connect事件进行处理
```java
if (key.isConnectable()) {
    ...
}
```
8.  判断连接成功，如果连接成功就注册读事件到多路复用器
```java
if (channel.finishConnect()) registerRead();
```
9. 注册读事件到多路复用器
```java
clientChannel.register(selector, SelectionKey.OP_READ, ioHandler);
```
10. 异步读客户端的请求到缓冲区
```java
int readNumber = channel.read(receivedBuffer);
```
11. 对缓冲区进行编解码，如果有半包消息接收到缓冲区Reset，继续读取后续报文，将解码成功的消息封装为Task，投递到业务线程池中进行处理
```java
Object msg = null;
while(buffer.hasRemain()) {
    byteBuffer.mark();
    msg = deode(byteBuffer);
    if (null == msg) {
        byteBuffer.reset();
        break;
    }
    messageList.add(msg);
}
if(!byteBuffer.hasRemain()) {
    byteBuffer.clear();
} else {
    byteBuffer.compact();
}
if(null != messageList && messageList.isEmpty()) {
    for(Object obj : messageList) {
        handlerTask(obj);
    }
}
```
12. 将POJO对象encode成ByteBuffer，调用SocketChannel的异步write接口，将消息异步发送给客户端
```java
socketChannel.write(buffer);
```
---
#### 客户端代码分析
客户端大体和服务端差不多，我们主要看一下有区别的地方。
```java
private void handleInput(SelectionKey key) throws IOException {
    if (key.isValid()) {
        SocketChannel sc = (SocketChannel) key.channel();
        if (key.isConnectable()) {
            //此时服务端已经返回ACK报文
            if (sc.finishConnect()) {
                //此时说明客户端连接成功
                sc.register(selector, SelectionKey.OP_READ);
                doWrite(sc);
            } else {
                System.exit(1);
            }
        }
        if (key.isReadable()) {
            ByteBuffer readBuffer = ByteBuffer.allocate(1024);
            int readBytes = sc.read(readBuffer);
            if (readBytes > 0) {
                readBuffer.flip();
                byte[] bytes = new byte[readBuffer.remaining()];
                readBuffer.get(bytes);
                String body = new String(bytes, "UTF-8");
                this.stop = true;
            } else if (readBytes < 0) {
                key.cancel();
                sc.close();
            } else {

            }
        }
    }
}

private void doConnect() throws IOException {
    //判断当前是否连接成功，因为是异步的所以没有返回应答消息不一定代表连接失败
    if (socketChannel.connect(new InetSocketAddress(host, port))) {
        //连接成功，将SocketChannel注册到多路复用器上
        socketChannel.register(selector, SelectionKey.OP_READ);
        doWrite(socketChannel);
    } else {
        //没有应答消息，我们需要将SocketChannel注册到多路复用器上，并标记为OP_CONNECT
        //这样当服务端返回syn-ack报文时，selector就能够轮询到这个SocketChannel为就绪状态
        socketChannel.register(selector, SelectionKey.OP_CONNECT);
    }
}
```
---

## AIO
NIO 2.0引入了新的异步通道概念，提供了异步文件通道和异步套接字通道的实现。异步套接字通道实现了真正的异步非阻塞I/O，对应于UNIX网络编程中的事件驱动I/O(AIO)。最大的区别就是它不需要通过多路复用器对注册的通道进行轮询。

### 服务端
#### 服务端代码分析
先提供一个请求的接入入口，这里直接展示异步线程的实现，Server客户端的创建和NIO一样
```java
public class AsyncTimeServerHandler implements Runnable {

    private int port;
    CountDownLatch latch;
    AsynchronousServerSocketChannel asynchronousServerSocketChannel;

    public AsyncTimeServerHandler(int port) {
        this.port = port;
        try {
            asynchronousServerSocketChannel = AsynchronousServerSocketChannel.open();
            asynchronousServerSocketChannel.bind(new InetSocketAddress(port));
            System.out.println("The time server is start in port : " + port);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {
        //计数器，让一组正在执行的动作完成之前当前线程一直处于阻塞状态，这里是模拟使用，正式不会有这一块
        latch = new CountDownLatch(1);
        doAccept();
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    private void doAccept() {
        //用来接收客户端的链接
        asynchronousServerSocketChannel.accept(this, new AcceptCompletionHandler());
    }

}
```

---

```java
public class AcceptCompletionHandler implements CompletionHandler<AsynchronousSocketChannel, AsyncTimeServerHandler> {
    @Override
    public void completed(AsynchronousSocketChannel result, AsyncTimeServerHandler attachment) {
        //这里又调用了accept方法是因为一个Channel可以接收很多客户端，所以当新客户端连接过来时需要继续调用accept方法
        attachment.asynchronousServerSocketChannel.accept(attachment, this);
        //链路建立成功之后，服务端要接收消息就需要创建一个缓冲区，然后进行异步读取
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        result.read(buffer, buffer, new ReadCompletionHandler(result));
    }

    @Override
    public void failed(Throwable exc, AsyncTimeServerHandler attachment) {
        exc.printStackTrace();
        attachment.latch.countDown();
    }
}
```
然后就是数据从缓冲区获取数据进行处理并应答客户端
```java
public class ReadCompletionHandler implements CompletionHandler<Integer, ByteBuffer> {

    private AsynchronousSocketChannel channel;

    public ReadCompletionHandler(AsynchronousSocketChannel channel) {
        //channel作为参数传递进来，这样就可以读取半包消息和发送应答
        if (this.channel == null) this.channel = channel;
    }

    @Override
    public void completed(Integer result, ByteBuffer attachment) {
        //flip方法会让读写指针指向缓存头部，并只能读取之前写入的数据长度
        attachment.flip();
        byte[] body = new byte[attachment.remaining()];
        attachment.get(body);
        try {
            String req = new String(body, "UTF-8");
            System.out.println("The time server receive order : " + req);
            String currentTime = "QUERY TIME ORDER".equalsIgnoreCase(req) ? new Date(System.currentTimeMillis()).toString() : "BAD ORDER";
            doWrite(currentTime);
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void failed(Throwable exc, ByteBuffer attachment) {
        try {
            this.channel.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void doWrite(String currentTime) {
        if (null != currentTime && currentTime.trim().length() > 0) {
            byte[] bytes = currentTime.getBytes();
            ByteBuffer writeBuffer = ByteBuffer.allocate(bytes.length);
            writeBuffer.put(bytes);
            writeBuffer.flip();
            channel.write(writeBuffer, writeBuffer, new CompletionHandler<Integer, ByteBuffer>() {
                @Override
                public void completed(Integer result, ByteBuffer buffer) {
                    if (buffer.hasRemaining()) {
                        //如果还有剩余数据没有发送就继续发送，知道全部发送完成
                        channel.write(buffer, buffer, this);
                    }
                }

                @Override
                public void failed(Throwable exc, ByteBuffer attachment) {
                    try {
                        channel.close();
                    } catch (IOException e) {
                        //ignore
                    }
                }
            });
        }
    }

}
```

### 客户端
#### 客户端代码分析
客户端最重要的部分就是completed()方法，这里会处理服务器的应答消息

```java
public class AsyncTimeClientHandler implements CompletionHandler<Void, AsyncTimeClientHandler>, Runnable {

    private AsynchronousSocketChannel client;
    private String host;
    private int port;
    private CountDownLatch latch;

    public AsyncTimeClientHandler(String host, int port) {
        this.host = host;
        this.port = port;
        try {
            client = AsynchronousSocketChannel.open();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {
        latch = new CountDownLatch(1);
        client.connect(new InetSocketAddress(host, port), this, this);
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        try {
            client.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 异步连接后会触发此回调
     * @param result
     * @param attachment
     */
    @Override
    public void completed(Void result, AsyncTimeClientHandler attachment) {
        byte[] req = "QUERY TIME ORDER".getBytes();
        ByteBuffer writeBuffer = ByteBuffer.allocate(1024);
        writeBuffer.put(req);
        writeBuffer.flip();
        client.write(writeBuffer, writeBuffer, new CompletionHandler<Integer, ByteBuffer>() {
            @Override
            public void completed(Integer result, ByteBuffer buffer) {
                if (buffer.hasRemaining()) {
                    client.write(buffer, buffer, this);
                } else {
                    ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                    //异步读取服务器应答消息处理
                    client.read(readBuffer, readBuffer, new CompletionHandler<Integer, ByteBuffer>() {
                        @Override
                        public void completed(Integer result, ByteBuffer buffer) {
                            buffer.flip();
                            byte[] bytes = new byte[buffer.remaining()];
                            buffer.get(bytes);
                            try {
                                String body = new String(bytes, "UTF-8");
                                System.out.println("Now is : " + body);
                                latch.countDown();
                            } catch (UnsupportedEncodingException e) {
                                e.printStackTrace();
                            }
                        }

                        @Override
                        public void failed(Throwable exc, ByteBuffer attachment) {
                            try {
                                client.close();
                                latch.countDown();
                            } catch (IOException e) {
                                //ignore
                            }
                        }
                    });
                }
            }

            @Override
            public void failed(Throwable exc, ByteBuffer attachment) {
                try {
                    client.close();
                    latch.countDown();
                } catch (IOException e) {
                    //ignore
                }
            }
        });
    }

    @Override
    public void failed(Throwable exc, AsyncTimeClientHandler attachment) {
        try {
            client.close();
            latch.countDown();
        } catch (IOException e) {
            //ignore
        }
    }
}
```

JDK底层通过线程池ThreadPoolExecutor来执行回调通知，最终回调completed方法。因此我们不需要像NIO那样去创建一个独立的I/O线程处理读写操作。