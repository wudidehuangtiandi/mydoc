# BIO NIO AIO实例及解析

## BIO

> BIO 同步阻塞IO，服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器就需要启动一个线程进行处理。

*在JDK1.4出来之前，我们建立网络连接的时候采用BIO模式，需要先在服务端启动一个`ServerSocket`，然后在客户端启动`Socket`来对服务端进行通信，默认情况下服务端需要对每个请求建立一堆线程等待请求，而客户端发送请求后，先咨询服务端是否有线程相应，如果没有则会一直等待或者遭到拒绝请求，如果有的话，客户端会线程会等待请求结束后才继续执行。*

> 在创建`demo`之前我们先要明白一些概念

首先我们回顾一下io流

![avatar](https://picture.zhanghong110.top/docsify/16413428556657.png)



> 然后是socket

- Java中的`Socket`分为普通的`Socket`和`NioSocket`。

普通`Socket`的用法:

创建`ServerSocket`。`ServerSocket`的构造方法有5个，其中最方便的是`ServerSocket`(int port)，只需要一个port就可以了。　　Java中的网络通信时通过`Socket`实现的，`Socket`分为`ServerSocket`和`Socket`两大类，`ServerSocket`用于服务器端，可以通过accept方法监听请求，监听请求后返回`Socket`，`Socket`用于完成具体数据传输，客户端也可以使用`Socket`发起请求并传输数据。`ServerSocket`的使用可以分为三步：

- 调用创建出来的`ServerSocket`的`accept`方法进行监听。`accept`方法是阻塞方法，也就是说调用`accept`方法后程序会停下来等待连接请求，在接受请求之前程序将不会继续执行，当接收到请求后`accept`方法返回一个`Socket`。
- 使用`accept`方法返回的`Socket`与客户端进行通信
- 客户端的Socket代码，Socket的使用也是一样，首先创建一个Socket，Socket的构造方法非常多，把目标主机的地址和端口号传入即可，Socket创建的过程就会跟服务器端建立连接，创建完Socket后，再创建Writer和Reader来传输数据，数据传输完成后释放资源关闭连接。

对应实现是BIO

我们看下如下demo

服务端：

```
public class PlainEchoServer {
    //线程池
    private static final ExecutorService executorPool = Executors.newFixedThreadPool(5);

    //静态内部类,处理具体的请求
    private static class Handler implements Runnable {

        private Socket clientSocket;

        public Handler(Socket clientSocket){
            this.clientSocket = clientSocket;
        }

        @Override
        public void run() {
            try {
                //获取字节输入流，创建字符转换流，创建字符输入缓冲流
                BufferedReader reader = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
                //获取字节输出流，创建字符输出打印流
                PrintWriter writer = new PrintWriter(clientSocket.getOutputStream(), true);
                char chars[] = new char[64];
                int len = reader.read(chars);
                //字符串缓冲区（线程安全）
                StringBuffer sb = new StringBuffer();
                sb.append(new String(chars, 0, len));
                System.out.println("From client: " + sb);
                //输出
                writer.write(sb.toString());
                writer.flush();
            } catch (IOException e) {
                e.printStackTrace();
                try {
                    clientSocket.close();
                } catch (IOException ex) {
                    // ignore on close
                }
            }
        }
    }

    public void serve(int port) throws IOException {
        //实例化ServerSocket
        final ServerSocket socket = new ServerSocket(port);
        try {
            //不停的循环
            while (true) {
                //纳秒，System.nanoTime()这个方法一个比较显著的应用是用来提供高精度的计时，不过两次调用的间隔不能超过2^63纳秒（大概292年），目前来看，应该暂时没有人有这么长的需求
                long beforeTime = System.nanoTime();
                //accept阻塞，当有请求进入时才接收成功，处理完继续循环
                final Socket clientSocket = socket.accept();

                System.out.println("Establish connection time: " + (System.nanoTime() - beforeTime) + " ns");

                //Handler一个新的线程，调用静态内部类构造，执行run方法，处理请求
                executorPool.execute(new Handler(clientSocket));
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws IOException {
        //实例化服务类
        PlainEchoServer server = new PlainEchoServer();
        //调用serve方法传入端口号
        server.serve(8080);
    }
}
```

客户端:

```
public class PlainEchoClient {

    public static void main(String args[]) throws Exception {
        for (int i = 0; i < 5; i++) {// i,20
            startClientThread(i + "name");
        }
    }

    private static void startClientThread(String name) throws UnknownHostException, IOException {
        Thread t = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    startClient();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }, name);
        t.start();
    }

    private static void startClient() throws UnknownHostException, IOException {
        long beforeTime = System.nanoTime();
        String host = "127.0.0.1";
        int port = 8080;
        Socket client = new Socket(host, port);
        // 建立连接后就可以往服务端写数据了
        // 获取输出流
        Writer writer = new OutputStreamWriter(client.getOutputStream());
        // 写出内容,标注线程名
        writer.write("Hello Server. from: " + Thread.currentThread().getName());
        writer.flush();
        // 写完以后进行读操作（服务端读完会写出来）
        Reader reader = new InputStreamReader(client.getInputStream());
        char chars[] = new char[64];// 假设所接收字符不超过64位，just for demo
        int len = reader.read(chars);
        StringBuffer sb = new StringBuffer();
        sb.append(new String(chars, 0, len));
        System.out.println("From server: " + sb);
        writer.close();
//        reader.close();
        client.close();
        System.out.println("Client use time: " + (System.nanoTime() - beforeTime) + " ns");
    }
}
```

可以看到实现了一个简单的BIO模型，同步，阻塞，进来一个请求启动一个线程处理,代码比较简单明晰，就不讲解了。

!>BIO的问题在于

1.每个请求都需要创建一个线程,与对应的客户端进行数据read,业务处理，write
2.当并发量较大时，需要创建大量的线程来处理连接，系统资源占用较大
3.连接建立后，如果线程暂时没有数据可读，则线程就阻塞在Read上，造成线程资源的浪费,如果没有用户通讯，线程就会堵塞在accept上。

## NIO

> 从JDK1.4开始，增加NIO支持，tomcat从8.0之后也完全移除了BIO的实现，默认采用NIO

`NioSocket`（即new io socket）是一种同步非阻塞的I/O，其使用`buffer`来缓存数据和`channel`来传输数据，使用select来分拣消息。其使用的`ServerSocketChannel`和`SocketChannel`对应于之前学习的`ServerSocket`和`Socket`。

> 我们看下区别在哪

数据传输方面：
 `socket`是直接使用输入输出流的方式直接读，当然也可以选择性的放在缓冲数据区中。
`nioSocket`只能用buffer来进行信息的获取与发出！

异步特性上面:
 `socket` 在连接和读写是都是堵塞状态的，即所在线程必须停住等待连接和数据的读入及写出，如果想在通信的时候可以开辟线程，一个线程对应一个socket连接，这样不仅资源消耗太高，而且线程安全问题也不容小觑！
 `niosocket`可以在连接、读入、写出等待时执行其他代码，简单来说就是无限等待变成了无限循环轮询管道。

对应的实现是`nio`我们看下实现的demo

服务端:

```
public class NIOServer {

    private ByteBuffer      readBuffer;
    // 一般称为选择器，也可以翻译为多路复用器，是Java NIO核心组件之一，
    // 主要功能是用于检查一个或者多个NIO Channel（通道）的状态是否处于可读、可写。
    // 如此可以实现单线程管理多个Channel（通道），当然也可以管理多个网络连接。
    // 使用Selector的好处在于，可以使用更少的线程来处理更多的通道，相比使用更多的线程，避免了线程上下文切换带来的开销等。
    private Selector        selector;

    public static void main(String[] args) {
        NIOServer server = new NIOServer();
        //初始化通道和选择器
        server.init();
        //监听处理
        server.listen();
    }

    private void init() {
        //初始化字节缓冲区
        readBuffer = ByteBuffer.allocate(1024);
        //使用通道
        ServerSocketChannel servSocketChannel;

        try {
            //创建管道
            servSocketChannel = ServerSocketChannel.open();
            //这个通道非常高效，所以要非阻塞
            servSocketChannel.configureBlocking(false);
            //绑定端口
            servSocketChannel.socket().bind(new InetSocketAddress(8383));
            //通过调用静态工厂方法Selector.open()方法创建一个Selector对象。
            selector = Selector.open();
            //Channel注册到Selector中
            //着重讲一下第二个参数，它是一个“interest集合”，意思是在通过Selector监听Channel时对什么事件感兴趣。可以监听四种不同类型的事件：Connect、Accept、Read、Write。
            // Connect：成功连接到另一个服务器称为“连接就绪”；
            // Accept：ServerSocketChannel准备好接收新进入的连接称为“接收就绪”；
            // Read：有数据可读的通道称为“读就绪”；
            // Write：等待写数据的通道称为“写就绪”；
            servSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void listen() {
        //开启一个无限循环
        while (true) {
            try {
                //通过Selector的select()方法可以选择已经准备就绪的通道（这些通道包含你感兴趣的事件）。比如你对读就绪的通道感兴趣，那么select()方法就会返回读事件已经就绪的那些通道。
                //此方法执行阻塞选择操作。只有在至少选择了一个通道、调用此选择器的唤醒方法或当前线程被中断（以先到者为准）后才返回。
                selector.select();

                Iterator<SelectionKey> ite = selector.selectedKeys().iterator();

                while (ite.hasNext()) {

                    SelectionKey key = (SelectionKey) ite.next();
                    ite.remove();// 确保不重复处理
                    //具体的处理手法
                    handleKey(key);
                }

            } catch (Throwable t) {
                t.printStackTrace();
            }
        }
    }

    private void handleKey(SelectionKey key) throws IOException, ClosedChannelException {
        SocketChannel channel = null;

        try {
            //这个key是否准备好接收套接字，即是否处于Accept状态
            if (key.isAcceptable()) {

                //获取ServerSocketChannel
                ServerSocketChannel serverChannel = (ServerSocketChannel) key.channel();
                // 接受连接请求
                channel = serverChannel.accept();
                //非阻塞
                channel.configureBlocking(false);
                //注册事件用于读取
                channel.register(selector, SelectionKey.OP_READ);

            }
            //处于读取状态
            else if (key.isReadable()) {

                channel = (SocketChannel) key.channel();

                //清空缓冲区
                readBuffer.clear();
                /*
                 * 当客户端channel关闭后，会不断收到read事件，但没有消息
                 * 即read方法返回-1 所以这时服务器端也需要关闭channel，避免无限无效的处理
                 */
                int count = channel.read(readBuffer);

                if (count > 0) {
                    // 一定需要调用flip函数，否则读取错误数据
                    //flip方法将Buffer从写模式切换到读模式。调用flip()方法会将position设回0，并将limit设置成之前position的值。
                    //换句话说，position现在用于标记读的位置，limit表示之前写进了多少个byte、char等 —— 现在能读取多少个byte、char等。
                    readBuffer.flip();
                    /*
                     * 使用CharBuffer配合取出正确的数据 String question = new String(readBuffer.array());
                     * 可能会出错，因为前面readBuffer.clear();并未真正清理数据 只是重置缓冲区的position, limit, mark，
                     * 而readBuffer.array()会返回整个缓冲区的内容。 decode方法只取readBuffer的position到limit数据。 例如，上一次读取到缓冲区的是"where",
                     * clear后position为0，limit为 1024， 再次读取“bye"到缓冲区后，position为3，limit不变，
                     * flip后position为0，limit为3，前三个字符被覆盖了，但"re"还存在缓冲区中， 所以 new String(readBuffer.array()) 返回 "byere",
                     * 而decode(readBuffer)返回"bye"。
                     */
                    CharBuffer charBuffer = CharsetHelper.decode(readBuffer);
                    String question = charBuffer.toString();
                    String answer = getAnswer(question);
                    //读取了以后再写出去
                    channel.write(CharsetHelper.encode(CharBuffer.wrap(answer)));
                } else {
                    // 这里关闭channel，因为客户端已经关闭channel或者异常了
                    channel.close();
                }
            }
        } catch (Throwable t) {
            t.printStackTrace();
            if (channel != null) {
                channel.close();
            }
        }
    }

    private String getAnswer(String question) {
        String answer = null;

        switch (question) {
            case "who":
                answer = "我是小娜\n";
                break;
            case "what":
                answer = "我是来帮你解闷的\n";
                break;
            case "where":
                answer = "我来自外太空\n";
                break;
            case "hi":
                answer = "hello\n";
                break;
            case "bye":
                answer = "88\n";
                break;
            default:
                answer = "请输入 who， 或者what， 或者where";
        }

        return answer;
    }
}
```

> 做个解读，主要就是先初始化一个管道和一个选择器，然后单个线程选择器不断去轮询各种状态的管道（读管道），根据返回结果给与不同的处理。仅用单个线程来处理多个Channels的好处是，只需要更少的线程来处理通道。事实上，可以只用一个线程处理所有的通道。因为对于操作系统来说，线程之间上下文切换的开销很大，而且每个线程都要占用系统的一些资源。因此，使用的线程越少越好。

客户端：

```
public class NIOClient implements Runnable {

    private BlockingQueue<String> words;
    private Random                random;

    public static void main(String[] args) {
        // 种多个线程发起Socket客户端连接请求
        for (int i = 0; i < 1; i++) {
            NIOClient c = new NIOClient();
            //初始化队列
            c.init();

            //执行run方法
            new Thread(c).start();
        }
    }

    @Override
    public void run() {
        SocketChannel channel = null;
        Selector selector = null;
        try {
            //创建SocketChannel
            channel = SocketChannel.open();
            //非阻塞
            channel.configureBlocking(false);
            // 请求连接
            channel.connect(new InetSocketAddress("localhost", 8383));

            //创建选择器
            selector = Selector.open();

            //管道注册到选择器，连接成功状态
            channel.register(selector, SelectionKey.OP_CONNECT);

            boolean isOver = false;

            while (!isOver) {
                selector.select();
                Iterator<SelectionKey> ite = selector.selectedKeys().iterator();
                while (ite.hasNext()) {
                    SelectionKey key = (SelectionKey) ite.next();
                    ite.remove();
                    //连接成功
                    if (key.isConnectable()) {
                        if (channel.isConnectionPending()) {
                            if (channel.finishConnect()) {
                                // 只有当连接成功后才能注册OP_READ事件
                                key.interestOps(SelectionKey.OP_READ);
                                //参数返回一个ByteBuffer
                                channel.write(CharsetHelper.encode(CharBuffer.wrap(getWord())));
                                //休眠
                                sleep();
                            } else {
                                key.cancel();
                            }
                        }
                        //待阅读状态
                    } else if (key.isReadable()) {
                        ByteBuffer byteBuffer = ByteBuffer.allocate(128);
                        channel.read(byteBuffer);
                        byteBuffer.flip();
                        CharBuffer charBuffer = CharsetHelper.decode(byteBuffer);
                        String answer = charBuffer.toString();
                        System.out.println(Thread.currentThread().getId() + "---" + answer);

                        String word = getWord();
                        if (word != null) {
                            //写入管道
                            channel.write(CharsetHelper.encode(CharBuffer.wrap(word)));
                        } else {
                            isOver = true;
                        }
                        sleep();
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (channel != null) {
                try {
                    channel.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }

            if (selector != null) {
                try {
                    selector.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    //初始化队列
    private void init() {
        //有界队列
        words = new ArrayBlockingQueue<String>(5);
        try {
            words.put("hi");
            words.put("who");
            words.put("what");
            words.put("where");
            words.put("bye");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        random = new Random();
    }


    private String getWord() {
        //检索并移除此队列的头部，如果此队列为空，则返回 null。返回： 此队列的头部，如果此队列为空，则返回 null
        return words.poll();
    }

    //模拟休眠
    private void sleep() {
        try {
            TimeUnit.SECONDS.sleep(random.nextInt(3));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

> 解读下客户端，首先初始化一个队列，然后通道链接某套接字，将队列中的内容写入通道，开个循环的作用是为了方便读取回复内容，重点在于两边都是不断的在轮询通道。

放一下工具类

```
public class CharsetHelper {
    private static final String UTF_8 = "UTF-8";
    private static CharsetEncoder encoder = Charset.forName(UTF_8).newEncoder();
    private static CharsetDecoder decoder = Charset.forName(UTF_8).newDecoder();

    public static ByteBuffer encode(CharBuffer in) throws CharacterCodingException{
        return encoder.encode(in);
    }

    public static CharBuffer decode(ByteBuffer in) throws CharacterCodingException{
        return decoder.decode(in);
    }
}
```



!>NIO存在的问题

在NIO的处理方式中，当一个请求来的话，开启线程进行处理 ( **这里demo可能没有体现，实际上多路复用的IO并不是一直单线程，demo中其实到`handleKey`这步就可以加入线程池处理了，看多tomcat源码就知道了选择器应该是单独线程，否则完全就阻塞了就没啥意义了。**)，所以这里有链接需要处理就会开启线程，这个线程可能会等待后端应用的资源(JDBC连接等)，其实这个线程就被阻塞了，当并发上来的话，还是会有BIO一样的问题。而选择器非阻塞虽然不用等待但是一直占用CPU。

## AIO

> 从JDK1.7开始新增的异步非阻塞IO，这部分内容被称作NIO2

与NIO不同，当进行读写操作时，AIO只须直接调用API的read或write方法即可。 两种方法均为异步的：

* 对于读操作而言，当有流可读取时，操作系统会将可读的流传入read方法的缓冲区，并通知应用程序； 

* 对于写操作而言，当操作系统将write方法传递的流写入完毕时，操作系统主动通知应用程序。  	

  

  即可以理解为，read/write方法都是异步的，完成后会主动调用回调函数。  

  在JDK1.7中，这部分内容被称作NIO2，主要在`Java.nio.channels`包下增加了下面四个异步通道： 	

  `AsynchronousSocketChannel `	

  `AsynchronousServerSocketChannel `	

  AsynchronousFileChannel `	

  `AsynchronousDatagramChannel ` 

  在AIO socket编程中，服务端通道是 `AsynchronousServerSocketChannel`： 	

  open()静态工厂：  

  ```
        	public static AsynchronousServerSocketChannel open(AsynchronousChannelGroup group)        
  
        	public static AsynchronousServerSocketChannel open()        
  ```

  ​	

  如果参数是null，则由系统默认提供程序创建resulting channel，并且绑定到默认组 	bind()方法用于绑定服务端IP地址（还有端口号）。 	

  accept()用于接收用户连接请求。 

  ```
  	AsynchronousServerSocketChannel server = AsynchronousServerSocketChannel.open().bind(new InetSocketAddress(PORT); 
  
  	public abstract <A> void accept(A attachment,CompletionHandler<AsynchronousSocketChannel,? super A> handler); 
  
  	public abstract Future<AsynchronousSocketChannel> accept();
  ```

  

  在客户端使用的通道是`AsynchronousSocketChannel`： 	

  这个通道处理提供open静态工厂方法外，还提供了read和write方法。 	

  ```
  public abstract Future<Void> connect(SocketAddress remote);
  ```

   Future对象的get()方法会阻塞该线程，所以这种方式是阻塞式的异步IO 

  ```
  public abstract <A> void connect(SocketAddress remote,  A attachment,CompletionHandler<Void,? super A> handler);
  ```



在AIO编程中，发出一个事件（accept read write等）之后要指定事件处理类（回调函数），AIO中的事件处理类是 `CompletionHandler<V,A>`，接口定义了如下两个方法，分别在异步操作成功和失败时被回调： 	

`void completed(V result, A attachment)`;    *//第一个参数代表IO操作返回的对象，第二个参数代表发起IO操作时传入的附加参数* 	

`void failed(Throwable exc, A attachment)`;    *//第一个参数代表IO操作失败引发的异常或错误*  

异步channel API提供了两种方式监控/控制异步操作(connect,accept, read，write等)： 

第一种方式是返回`java.util.concurrent.Future`对象，  检查Future的状态可以得到操作是否完成还是失败，还是进行中（`future.get()`阻塞当前进程以判断IO操作完成） 	

第二种方式为操作提供一个回调参数`java.nio.channels.CompletionHandler` 这个回调类包含completed,failed两个方法。

**Future方式（异步阻塞）**

服务端

```
public class ServerOnFuture {
    static final int PORT = 8088;
    static final String IP = "localhost";
    static ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

    public static void main(String[] args) {
        try (AsynchronousServerSocketChannel serverSocketChannel = AsynchronousServerSocketChannel.open()) {
            serverSocketChannel.bind(new InetSocketAddress(IP, PORT));
            while (true) {
                Future<AsynchronousSocketChannel> channelFuture = serverSocketChannel.accept();
                try (AsynchronousSocketChannel socketChannel = channelFuture.get()) {
                    while (socketChannel.read(buffer).get() != -1) {
                        buffer.flip();
                        /**
                         * 此处要注意：千万不能直接操作buffer（因为write要用到buffer），否则客户端会阻塞并报错
                         *     “java.util.concurrent.ExecutionException: java.io.IOException: 指定的网络名不再可用。”
                         *
                         * 缓冲区的复制有分两种：
                         *      1、完全复制：调用duplicate()函数或者asReadOnlyBuffer()函数
                         *      2、部分复制：调用slice函数
                         *
                         * duplicate()函数创建了一个与原始缓冲区相似的新缓冲区。
                         *      每个缓冲区有自己的位置信息，但对缓冲区的修改都会映射到同一个底层数组上。
                         */
                        //复制一个缓冲区会创建一个新的 Buffer 对象，但并不复制数据。原始缓冲区和副本都会操作同样的数据元素。
                        ByteBuffer duplicate = buffer.duplicate();
                        CharBuffer decode = Charset.defaultCharset().decode(duplicate);
                        System.out.println("收到客户端数据：" + decode);

                        /**
                         * 写回数据(get()会阻塞以等待io操作完成),实际上还是future里获得的
                         */
                        socketChannel.write(buffer).get();

                        /**
                         * 清理buffer，准备下一次read
                         */
                        if (buffer.hasRemaining()) {
                            /**
                             * 如果未写完，表示buffer还有数据，则只清理写过的数据
                             * compact()方法只会清除已经读过的数据。
                             * 任何未读的数据都被移到缓冲区的起始处，新写入的数据将放到缓冲区未读数据的后面。
                             */
                            buffer.compact();
                        } else {
                            buffer.clear();
                        }
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```



服务端Future方式实现多客户端并发

```
public class ServerOnFuture {
    static final int PORT = 10000;
    static final String IP = "localhost";
    //无界线程池
    static ExecutorService taskExecutorService = Executors.newCachedThreadPool();
    static ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

    public static void main(String[] args) {
        try (AsynchronousServerSocketChannel serverSocketChannel = AsynchronousServerSocketChannel.open()) {
            serverSocketChannel.bind(new InetSocketAddress(IP, PORT));
            while (true) {
                Future<AsynchronousSocketChannel> socketChannelFuture = serverSocketChannel.accept();
                try {
                    final AsynchronousSocketChannel socketChannel = socketChannelFuture.get();
                    /**
                     * 创建一个具有回调的线程
                     */
                    Callable<String> worker = new Callable<String>() {
                        @Override
                        public String call() throws Exception {
                            while (socketChannel.read(buffer).get() != -1) {
                                buffer.flip();
                                ByteBuffer duplicate = buffer.duplicate();
                                CharBuffer decode = Charset.defaultCharset().decode(duplicate);
                                System.out.println(decode.toString());
                                socketChannel.write(buffer).get();
                                if (buffer.hasRemaining()) {
                                    buffer.compact();
                                } else {
                                    buffer.clear();
                                }
                            }
                            socketChannel.close();
                            return "服务端反馈信息：收到";
                        }
                    };
                    /**
                     * 将线程提交到线程池
                     */
                    taskExecutorService.submit(worker);
                    //获取线程数
                    System.out.println(((ThreadPoolExecutor) taskExecutorService).getActiveCount());
                } catch (InterruptedException | ExecutionException e) {
                    /**
                     * 出现异常，关闭线程池
                     */
                    taskExecutorService.shutdown();
                    /**
                     * boolean isTerminated()
                     *      若关闭后所有任务都已完成，则返回true。
                     *      注意除非首先调用shutdown或shutdownNow，否则isTerminated永不为true。
                     */
                    while (!taskExecutorService.isTerminated()) {
                    }
                    //跳出循环，结束程序
                    break;
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

> 可以看到，以上两个客户端虽然是异步的，但是都是返回future对象使用get等待异步结果。本质上还是阻塞的

客户端

```
public class ClientOnFuture {
    static final int PORT = 8088;
    static final String IP = "localhost";
    static ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

    public static void main(String[] args) {
        //尝试创建AsynchronousSocketChannel
        try (AsynchronousSocketChannel socketChannel = AsynchronousSocketChannel.open()) {
            //获取连接
            Future<Void> connect = socketChannel.connect(new InetSocketAddress(IP, PORT));
            //返回连接状态
            Void aVoid = connect.get();
            //返回null表示连接成功
            if (aVoid == null) {
                /**
                 * 向服务端发送数据
                 */
                Future<Integer> write = socketChannel.write(ByteBuffer.wrap("客户端说：我连接成功了！".getBytes()));
                Integer integer = write.get();
                System.out.println("服务端接收的字节长度：" + integer);
                /**
                 * 接收服务端数据
                 */
                while (socketChannel.read(buffer).get() != -1) {
                    buffer.flip();
                    CharBuffer decode = Charset.defaultCharset().decode(buffer);
                    System.out.println(decode.toString());
                    if (buffer.hasRemaining()) {
                        buffer.compact();
                    } else {
                        buffer.clear();
                    }
                    int r = new Random().nextInt(10);
                    if (r == 5) {
                        System.out.println("客户端关闭！");
                        break;
                    } else {
                        /**
                         * 如果在频繁调用write()的时候，在上一个操作没有写完的情况下，
                         * 调用write会触发WritePendingException异常
                         *
                         * 应此此处最好在调用write()之后调用get()阻塞以便确认io操作完成
                         */
                        socketChannel.write(ByteBuffer.wrap(("客户端发送的数据:" + r).getBytes())).get();
                    }
                }
            } else {
                System.out.println("无法建立连接!");
            }
        } catch (Exception e) {
            System.out.println("出错了！");
        }
    }
}
```

客户端没啥需要讲的，主要也还是用到了`AsynchronousSocketChannel`，实现异步。

> 下面我们来看下异步非阻塞，也就是基于回调的AIO实现方式

服务端:

```
public class ServerOnCompletionHandler {
    static final int PORT = 8088;
    static final String IP = "localhost";

    public static void main(String[] args) {
        //打开通道
        try (final AsynchronousServerSocketChannel serverSocketChannel = AsynchronousServerSocketChannel.open()) {
            //创建服务
            serverSocketChannel.bind(new InetSocketAddress(IP, PORT));
            //接收客户端连接
            serverSocketChannel.accept(null, new CompletionHandler<AsynchronousSocketChannel, Void>() {
                final ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

                @Override
                public void completed(AsynchronousSocketChannel socketChannel, Void attachment) {
                    /**
                     * 注意接收一个连接之后，紧接着可以接收下一个连接，所以必须再次调用accept方法
                     * AsynchronousSocketChannel就代表该CompletionHandler处理器在处理连接成功时的result（AsynchronousSocketChannel的实例）
                     */
                    serverSocketChannel.accept(null, this);
                    try {
                        while (socketChannel.read(buffer).get() != -1) {
                            buffer.flip();
                            final ByteBuffer duplicate = buffer.duplicate();
                            final CharBuffer decode = Charset.defaultCharset().decode(duplicate);
                            System.out.println(decode.toString());
                            socketChannel.write(buffer).get();  //get()用于阻塞使IO操作完成
                            if (buffer.hasRemaining()) {
                                buffer.compact();
                            } else {
                                buffer.clear();
                            }
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    } finally {
                        try {
                            socketChannel.close();
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }
                }

                @Override
                public void failed(Throwable exc, Void attachment) {
                    /**
                     * 失败后也需要接收下一个连接
                     */
                    serverSocketChannel.accept(null, this);
                    System.out.println("连接失败！");
                }
            });

            //主要是阻塞作用，因为AIO是异步的，所以此处不阻塞的话，主线程很快执行完毕，并会关闭通道
            System.in.read();

        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

可以看到重写了`completed`和`failed`方法。当被调用时IO操作已经完成了。

客户端:

```
/**
 * 客户端
 */
public class ClientOnCompletionHandler {
    static final int PORT = 8088;
    static final String IP = "localhost";

    public static void main(String[] args) {
        try (final AsynchronousSocketChannel socketChannel = AsynchronousSocketChannel.open()) {
            socketChannel.connect(new InetSocketAddress(IP, PORT), null, new CompletionHandler<Void, Void>() {
                final ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

                @Override
                public void completed(Void result, Void attachment) {
                    try {
                        socketChannel.write(ByteBuffer.wrap("Hello Server！".getBytes())).get();
                        while (socketChannel.read(buffer).get() != -1) {
                            buffer.flip();
                            ByteBuffer duplicate = buffer.duplicate();
                            CharBuffer decode = Charset.defaultCharset().decode(duplicate);
                            System.out.println(decode.toString());
                            buffer.clear();
                            int r = new Random().nextInt(10);
                            socketChannel.write(ByteBuffer.wrap("客户端消息：".concat(String.valueOf(r)).getBytes())).get();
                            Thread.sleep(3000);
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    } finally {
                        try {
                            socketChannel.close();
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }
                }

                @Override
                public void failed(Throwable exc, Void attachment) {
                    System.out.println("连接失败！");
                }
            });

            //主要是阻塞作用，因为AIO是异步的，所以此处不阻塞的话，主线程很快执行完毕，并会关闭通道
            System.in.read();

        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

主线程需要`  System.in.read()`无限阻塞,除非读取到键盘输入。以等待IO内容不停的被调用。以上两个都是单线程

> 下面我们来看下自定义group完成多线程处理

`AsynchronousChannelGroup`是异步`Channel`的分组管理器，它可以实现资源共享。

创建`AsynchronousChannelGroup`时，需要传入一个`ExecutorService`，也就是绑定一个线程池。
该线程池负责两个任务：`处理IO事件`和`触发CompletionHandler回调接口`。

每个异步通道都必须关联一个组，要么是`系统默认组`，要么是`用户创建的组`。
如果不使用group参数，java使用一个默认的系统范围的组对象。

异步IO模型中，用户线程直接使用`内核提供的异步IO API`发起read请求。
发起后`立即返回`，`继续执行用户线程代码`。

此时用户线程已经将调用的`AsynchronousOperation`和`CompletionHandler`注册到内核，然后`操作系统开启独立的内核线程去处理IO操作`。
当read请求的数据到达时，由`内核负责读取socket中的数据`，并`写入用户指定的缓冲区`中。
最后内核将read的数据和用户线程注册的`CompletionHandler`分发给`内部Proactor`，`Proactor`将IO完成的信息`通知给用户线程`（一般通过调用用户线程注册的完成事件处理函数），完成异步IO。

```
public class ServerOnReaderAndWriterForMultiClients {
    static final int PORT = 8088;
    static final String IP = "localhost";
    static AsynchronousChannelGroup threadGroup = null;
    static ExecutorService executorService = Executors.newCachedThreadPool();

    public static void main(String[] args) {
        try {
            threadGroup = AsynchronousChannelGroup.withCachedThreadPool(executorService, 5);
            //或者使用指定数量的线程池
            //threadGroup = AsynchronousChannelGroup.withFixedThreadPool(5, Executors.defaultThreadFactory());
        } catch (IOException e) {
            e.printStackTrace();
        }

        try (AsynchronousServerSocketChannel serverSocketChannel = AsynchronousServerSocketChannel.open(threadGroup)) {
            serverSocketChannel.bind(new InetSocketAddress(IP, PORT));
            serverSocketChannel.accept(serverSocketChannel, new CompletionHandler<AsynchronousSocketChannel, AsynchronousServerSocketChannel>() {
                final ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

                @Override
                public void completed(AsynchronousSocketChannel socketChannel, AsynchronousServerSocketChannel attachment) {
                    serverSocketChannel.accept(null, this);
                    try {
                        while (socketChannel.read(buffer).get() != -1) {
                            buffer.flip();
                            final ByteBuffer duplicate = buffer.duplicate();
                            final CharBuffer decode = Charset.defaultCharset().decode(duplicate);
                            System.out.println(decode.toString());
                            socketChannel.write(buffer).get();  //get()用于阻塞使IO操作完成
                            if (buffer.hasRemaining()) {
                                buffer.compact();
                            } else {
                                buffer.clear();
                            }
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    } finally {
                        try {
                            socketChannel.close();
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }
                }

                @Override
                public void failed(Throwable exc, AsynchronousServerSocketChannel attachment) {
                    serverSocketChannel.accept(null, this);
                    System.out.println("连接失败！");
                }
            });

            //此方法一直阻塞，直到组终止、超时或当前线程中断
            threadGroup.awaitTermination(Long.MAX_VALUE, TimeUnit.SECONDS);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

