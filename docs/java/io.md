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

## NIO

`NioSocket`（即new io socket）是一种同步非阻塞的I/O，其使用`buffer`来缓存数据和`channel`来传输数据，使用select来分拣消息。其使用的`ServerSocketChannel`和`SocketChannel`对应于之前学习的`ServerSocket`和`Socket`。

> 我们看下区别在哪

数据传输方面：
 `socket`是直接使用输入输出流的方式直接读，当然也可以选择性的放在缓冲数据区中。
`nioSocket`只能用buffer来进行信息的获取与发出！

异步特性上面:
 `socket` 在连接和读写是都是堵塞状态的，即所在线程必须停住等待连接和数据的读入及写出，如果想在通信的时候可以开辟线程，一个线程对应一个socket连接，这样不仅资源消耗太高，而且线程安全问题也不容小觑！
 `niosocket`可以再连接、读入、写出等待时执行其他代码。

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

> 做个解读，主要就是先初始化一个管道和一个选择器，然后单个线程选择器不断去轮询各种状态的管道（读个管道），根据返回结果给与不同的处理

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

!>可以看到NIO实现起来是比较复杂的，同时许多细节设计不好会引起严重的问题，尽量不要尝试实现自己的nio框架，除非有经验丰富的工程师，尽量使用经过广泛实践的开源NIO框架Mina、Netty3、xSocket，还有可以看出NIO的实现是单线程的，所以会有线程安全问题。

## AIO

待续