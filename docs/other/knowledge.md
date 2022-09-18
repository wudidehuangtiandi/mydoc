# 概念整理



# 一 抽象类和接口的区别

1.最大的区别就是使用场景不同，抽象类用于抽象一种具体的事物，即对类抽象，而接口是对行为的抽象。

举个例子，比如鸟和飞机，它们都会飞，这时候鸟和飞机就可以被定义成抽象类，而飞则是定义成接口。这样不同的鸟继承鸟的抽象类，而鸟的抽象类和飞机的抽象类都实现飞这个接口，而它们飞的方式不相同，当然还可以实现其它不同的接口。

2.接口可以支持多实现，而抽象类只能单继承。

3.其它都是小区别不重要。



# 二 异常分类及常见异常

所有异常都是`Throwable`的子类，下面分为`Error`和`Exception`。

`Error`下有众多的无法被处理的异常例如`OutOfMemoryError`,`IOError`等，而`Exception`下则分为运行时异常`RuntimeException`和非运行时异常,如`ArrayIndexOutOfBoundsException`,`NullPointerException`等,非运行时异常(也叫编译异常)多的很，一大堆比如`IOException`,`SQLException`等等。

`Exception`下除了`RuntimeException`以外的都是受查异常。这种异常都发生在编译阶段，Java 编译器会强制程序去捕获此类异常。

非受查异常包括`Error`和`RuntimeException`。`RuntimeException`表示编译器不会检查是否对`RuntimeException`进行了处理。

![avatar](https://picture.zhanghong110.top/docsify/2019101117003396.png)



!>故而理论上在程序中不必去捕获`RuntimeException`异常。`RuntimeException`发生就说明程序出现了编程错误，应该找出错误去修改，而不是事先去捕获异常或者抛出，我们自己定义的运行时异常，抛出了，在调用的地方在捕获，挺搞笑的。然而实际上，我们还是经常会去捕获它们（笑哭）,比如做容错啊，或者记日志啊这些，可能还要保证发生了，程序还能运行下去等等的。

!>`Spring`的`AOP`即声明式事务管理默认是针对`unchecked exception`回滚。也就是默认对`RuntimeException()`异常或是其子类进行事务回滚。`checked`异常,即`Exception`可`try{}`捕获的不会回滚，因此对于我们自定义异常，通过`rollbackFor`进行设定。



# 三 JAVA集合

分为`Collection`和`Map`的两种体系。这边讲讲太麻烦，贴个图，以下分析都是基于JDK1.8

![avatar](https://picture.zhanghong110.top/docsify/b3fb43166d224f4a5cebf37901f790529822d16e.jpg)

1. 线程安全，除了`Vector`,`HashTable`,`ConcurrentHashMap`，`ConcurrentSkipListMap`，`ConcurrentSkipListSet`等等,线程安全。（至于为什么没有     `ConcurrentArrayList`，原因是无法设计一个通用的而且可以规避`ArrayList`的并发瓶颈的线程安全的集合类，只能锁住整个`list`，这用`Collections`里的包装类就能办到。

​       `CopyOnWriteArrayList`和`CopyOnWriteArraySet`,它们是加了写锁的`ArrayList`和`ArraySet`，锁住的是整个对象，但读操作可以并发执行。

​       其它的集合都是线程不安全的。

​    2.`ArrayList`、`LinkList`、`Vector`

​      2.1 `Vector`线程安全，其他两个不安全。

​      2.2  `ArrayList`和`Vector`都是基于动态数组对象实现。

​      2.3 `ArrayList`构造默认创建空数组`DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {}所以数组容量为0`，当正真添加元素时才会赋值`add`方法得   

​       `ensureCapacityInternal`方法在调用`calculateCapacity`赋予默认`10`得大小。扩容得时候利用移位运算`>> `作用在这为除以`2`，`int newCapacity =                   `      

​       `oldCapacity + (oldCapacity >> 1)`,等于老的集合长度加上一半，更加省内存。

​       2.4 `Vector `会在创建时赋予默认大小`10`，扩容方式直接增加一倍`int newCapacity = oldCapacity + ((capacityIncrement > 0) ?   `

​       `capacityIncrement:oldCapacity)`，其中绝大部分方法都加上了`synchronized`。

​       2.5 `LinkedList`基于双向链表实现，由于`LinkedList`是链表实现的，所以额外提供了在头部和尾部添加/删除元素的方法，也没有`ArrayList`扩容的问题。

```
总而言之

对于随机访问元素，Array获取数据的时间复杂度是O(1),但是要删除数据却是开销很大的，因为这需要重排数组中的所有数据。ArrayList想要get(int index)元素时，直接返回index位置上的元素，而LinkedList需要通过for循环进行查找，虽然LinkedList已经在查找方法上做了优化，比如index < size / 2，则从左边开始查找，反之从右边开始查找，但是还是比ArrayList要慢。

对于添加和删除操作add和remove，LinkedList是更快的。因为LinkedList不像ArrayList一样，不需要改变数组的大小，也不需要在数组装满的时候要将所有的数据重新装入一个新的数组，这是ArrayList最坏的一种情况，时间复杂度是O(n)，而LinkedList中插入或删除的时间复杂度仅为O(1)。ArrayList在插入数据时还需要更新索引（除了插入数组的尾部）。 ArrayList想要在指定位置插入或删除元素时，主要耗时的是System.arraycopy动作，会移动index后面所有的元素；LinkedList主耗时的是要先通过for循环找到index，然后直接插入或删除。这就导致了两者并非一定谁快谁慢,大多数情况下LinkedList更快。
```

​      3.`hashmap`

​       3.1 `jdk1.7`及之前是数组+链表得结构，之后是数组+链表+红黑树(节点达到阈值`TREEIFY_THRESHOLD`,默认是`8`)，当哈希冲突时将会被添加到相同节点中。

​       3.2 `hashmap`线程不安全，`hashtable`线程安全。

​       3.3 默认初始大小为`16`,默认负载因子是`0.75`，当元素达到数量时默认扩容两倍

​       3.4 关于扩容机制

```
2的幂次方：hashmap在确定元素落在数组的位置的时候，计算方法是(n - 1) & hash，n为数组长度也就是初始容量 。hashmap结构是数组，每个数组里面的结构是node（链表或红黑树），正常情况下，如果你想放数据到不同的位置，肯定会想到取余数确定放在那个数组里，计算公式：hash % n，这个是十进制计算。在计算机中， (n - 1) & hash，当n为2次幂时，会满足一个公式：(n - 1) & hash = hash % n，计算更加高效。
奇数不行：在计算hash的时候，确定落在数组的位置的时候，计算方法是(n - 1) & hash，奇数n-1为偶数，偶数2进制的结尾都是0，经过hash值&运算后末尾都是0，那么0001，0011，0101，1001，1011，0111，1101这几个位置永远都不能存放元素了，空间浪费相当大，更糟的是这种情况中，数组可以使用的位置比数组长度小了很多，这样就会造成空间的浪费而且会增加hash冲突。
```



# 四 JDK8的特性

> 函数式接口

四大函数式接口

```java
1.Function 
java.util.function
public interface Function<T,R>表示接受一个参数并产生结果的函数。
这是一个functional interface的功能方法是apply(Object)。
R apply(T t)将此函数应用于给定的参数。  

2.Predicate 
也可以用来可以判断字符串是否为空
@FunctionalInterface
public interface Predicate<T>表示一个参数的谓词（布尔值函数）。
这是一个functional interface的功能方法是test(Object)。
boolean test(T t)在给定的参数上评估这个谓词。
参数
t - 输入参数
结果
true如果输入参数匹配谓词，否则为 false
    
3.Consumer
public interface Consumer<T>
表示接受单个输入参数并且不返回结果的操作。 与大多数其他功能界面不同， Consumer预计将通过副作用进行操作。
这是一个functional interface的功能方法是accept(Object) 。
void accept(T t)对给定的参数执行此操作。
t - 输入参数

4.Supplier
@FunctionalInterface
public interface Supplier<T>
没有要求每次调用供应商时都会返回新的或不同的结果。 这是一个functional interface的功能方法是get()。
T get()获得结果
```



> lambda表达式

Lambda表达式（也称为闭包）是整个Java 8发行版中最受期待的在Java语言层面上的改变，Lambda允许把函数作为一个方法的参数（函数作为参数传递进方法中），或者把代码看成数据

基本语法:
(parameters) -> expression
或
(parameters) ->{ statements; }



> Stream 流

分为`Parallel-Streams` 和`Stream `前者多线程，应该避免在需要线程环境的代码中使用。

终止方法，例如`toList()`会终止流，而其它的非终止方法还是会返回一个流。



> Optional

可以很好的用来解决空指针异常。

构造其对象可以用

```
Optional.empty()： 创建一个空的 Optional 实例

Optional.of(T t)：创建一个 Optional 实例，当 t为null时抛出异常

Optional.ofNullable(T t)：创建一个 Optional 实例，但当 t为null时不会抛出异常，而是返回一个空的实例
```

```
获取：

get()：获取optional实例中的对象，当optional 容器为空时报错

判断：

isPresent()：判断optional是否为空，如果空则返回false，否则返回true

ifPresent(Consumer c)：如果optional不为空，则将optional中的对象传给Comsumer函数

orElse(T other)：如果optional不为空，则返回optional中的对象；如果为null，则返回 other 这个默认值

orElseGet(Supplier other)：如果optional不为空，则返回optional中的对象；如果为null，则使用Supplier函数生成默认值other

orElseThrow(Supplier exception)：如果optional不为空，则返回optional中的对象；如果为null，则抛出Supplier函数生成的异常

过滤：

filter(Predicate p)：如果optional不为空，则执行断言函数p，如果p的结果为true，则返回原本的optional，否则返回空的optional

映射：

map(Function<T, U> mapper)：如果optional不为空，则将optional中的对象 t 映射成另外一个对象 u，并将 u 存放到一个新的optional容器中。

flatMap(Function< T,Optional> mapper)：跟上面一样，在optional不为空的情况下，将对象t映射成另外一个optional

区别：map会自动将u放到optional中，而flatMap则需要手动给u创建一个optional
```



# 五 并发编程

> volatile

volatile可以看作轻量级的synchronized。 作用，保证**可见性**，屏蔽指令重排序(保证**有序性**)

1、 保证变量的可见性：当一个被volatile关键字修饰的变量被一个线程修改的时候，其他线程可以立刻得到修改之后的结果。当一个线程向被volatile关键字修饰的变量写入数据的时候，虚拟机会强制它被值刷新到主内存中。当一个线程用到被volatile关键字修饰的值的时候，虚拟机会强制要求它从主内存中读取。
2、 屏蔽指令重排序：指令重排序是编译器和处理器为了高效对程序进行优化的手段，它只能保证程序执行的结果时正确的，但是无法保证程序的操作顺序与代码顺序一致。这在单线程中不会构成问题，但是在多线程中就会出现问题。非常经典的例子是在单例方法中同时对字段加入volatile，就是为了防止指令重排序。

指令重排序可以参见设计模式单例中的双检索，其实现原理为**内存屏障**。本文档在那边做了解释。但是volatile无法保证**原子性**（比如不同线程读取其修饰的变量进行操作，多个线程读写间原子性无法保证，比如++操作）。



>synchronized

中文意思是同步，也称之为”同步锁“。它保证在同一时刻， 被修饰的代码块或方法只会有一个线程执行，以达到保证并发安全的效果。它是一个重量级锁

其实现原理为监视器锁（monitor），监视器锁依赖底层的操作系统的 Mutex Lock（互斥锁）来实现。

synchronized锁升级过程：

 根据上面内容可以知道，synchronized锁有四种状态：无锁，偏向锁、轻量级锁和重量级锁，下面介绍四种状态和其之间的转换。

2.1 无锁
 当一个对象被创建之后，还没有线程进入，这个时候对象处于无锁状态，其Mark Word中的信息如上表所示。

2.2 偏向锁
 当锁处于无锁状态时，有一个线程A访问同步块并获取锁时，会在对象头和栈帧中的锁记录记录线程ID，以后该线程在进入和退出同步块时不需要进行CAS操作（CAS全称compare and swap,JDK提供的非阻塞原子性操作，它通过硬件保证了更新操作的原子性。它允许多线程非阻塞地对共享资源进行修改，但是同一时刻只有一个线程可以修改）来进行加锁和解锁，只需要简单的测试一下啊对象头中的线程ID和当前线程是否一致。

2.3 轻量级锁
 在偏向锁的基础上，又有另外一个线程B进来，这时判断对象头中存储的线程A的ID和线程B不一致，就会使用CAS竞争锁，并且升级为轻量级锁，会在线程栈中创建一个锁记录(lock Record)，将Mark Word复制到锁记录中，然后线程尝试使用CAS将对象头的Mark Word替换成指向锁记录的指针，如果成功，则当前线程获得锁；失败，表示其他线程竞争锁，当前线程便尝试CAS来获取锁。

2.4 重量级锁
 当线程没有获得轻量级锁时，线程会CAS自旋来获取锁，当一个线程自旋10次之后，仍然未获得锁，那么就会升级成为重量级锁。

 成为重量级锁之后，线程会进入阻塞队列(EntryList)，线程不再自旋获取锁，而是由CPU进行调度，线程串行执行。



> lock

lock锁为`jdk`层面的锁,Lock是同步非阻塞，采用的是乐观并发策略。在大量并发的时候性能更好。更加灵活

`ReentrantLock`,继承lock接口,是可重入的互斥锁，虽然具有与`synchronized`相同功能，但是会比`synchronized`更加灵活（**具有更多的方法**）,其同步器`sync`继承了`AbstractQueuedSynchronizer`类即`AQS`



# 六 多线程

1.线程池的种类

1. `newCachedThreadPool`
2. `newFixedThreadPool`
3. `newScheduledThreadPool`
4. `newSingleThreadExecutor`

2.线程的生命周期

1. 新建、就绪、运行、阻塞(等待阻塞、同步阻塞、其他阻塞)、死亡



# 七 基础补充

> ==和equals的区别

==比较内存地址，equals没有重写前也是比较内存地址。重写后可以比较内容等。

> 两个对象的` hashCode()` 相同，则 equals() 也一定为 true，对吗？

“通话”和“重地”的 hashCode() 相同，然而 equals() 则为 false，因为在散列表中，hashCode() 相等即两个键值对的哈希值相等，然而哈希值相等，并不一定能得出键值对相等。

> final

- final 修饰的类叫最终类，该类不能被继承。
- final 修饰的方法不能被重写。
- final 修饰的变量叫常量，常量必须初始化，初始化之后值就不能被修改。

> Math. round(-1. 5)

等于 -1，因为在数轴上取值时，中间值（0.5）向右取整，所以正 0.5 是往上取整，负 0.5 是直接舍弃

> String 属于基础的数据类型吗

String 不属于基础类型，基础类型有 8 种：byte、boolean、char、short、int、float、long、double，而 String 属于对象。

> Java 中操作字符串都有哪些类？它们之间有什么区别？

操作字符串的类有：String、StringBuffer、StringBuilder。

String 和 StringBuffer、StringBuilder 的区别在于 String 声明的是不可变的对象，每次操作都会生成新的 String 对象，然后将指针指向新的 String 对象，而 StringBuffer、StringBuilder 可以在原有对象的基础上进行操作，所以在经常改变字符串内容的情况下最好不要使用 String。

StringBuffer 和 StringBuilder 最大的区别在于，StringBuffer 是线程安全的，而 StringBuilder 是非线程安全的，但 StringBuilder 的性能却高于 StringBuffer，所以在单线程环境下推荐使用 StringBuilder，多线程环境下推荐使用 StringBuffer。

> String str="i"与 String str=new String("i")一样吗？

不一样，因为内存的分配方式不一样。String str="i"的方式，Java 虚拟机会将其分配到常量池中；而 String str=new String("i") 则会被分到堆内存中。

> 如何将字符串反转

使用 StringBuilder 或者 stringBuffer 的 reverse() 方法。

> String 类的常用方法都有那些

indexOf()：返回指定字符的索引。
charAt()：返回指定索引处的字符。
replace()：字符串替换。
trim()：去除字符串两端空白。
split()：分割字符串，返回一个分割后的字符串数组。
getBytes()：返回字符串的 byte 类型数组。
length()：返回字符串长度。
toLowerCase()：将字符串转成小写字母。
toUpperCase()：将字符串转成大写字符。
substring()：截取字符串。
equals()：字符串比较。

> Java 中 IO 流分为几种

按功能来分：输入流（input）、输出流（output）。

按类型来分：字节流和字符流。

字节流和字符流的区别是：字节流按 8 位传输以字节为单位输入输出数据，字符流按 16 位传输以字符为单位输入输出数据。

> BIO、NIO、AIO 有什么区别？

`BIO`：Block IO 同步阻塞式 IO，就是我们平常使用的传统 IO，它的特点是模式简单使用方便，并发处理能力低。
`NIO`：Non IO 同步非阻塞 IO，是传统 IO 的升级，客户端和服务器端通过 Channel（通道）通讯，实现了多路复用。
`AIO`：Asynchronous IO 是 NIO 的升级，也叫 NIO2，实现了异步非堵塞 IO ，异步 IO 的操作基于事件和回调机制。

> Files的常用方法都有哪些？

Files. exists()：检测文件路径是否存在。
Files. createFile()：创建文件。
Files. createDirectory()：创建文件夹。
Files. delete()：删除一个文件或目录。
Files. copy()：复制文件。
Files. move()：移动文件。
Files. size()：查看文件个数。
Files. read()：读取文件。
Files. write()：写入文件。

> 怎么确保一个集合不能被修改？

可以使用 Collections. unmodifiableCollection(Collection c) 方法来创建一个只读集合，这样改变集合的任何操作都会抛出 Java. lang. UnsupportedOperationException 异常。

> 创建线程有哪几种方式？

创建线程有三种方式：

- 继承 Thread 重写 run 方法；
- 实现 Runnable 接口；
- 实现 Callable 接口。

> 说一下 runnable 和 callable 有什么区别？

runnable 没有返回值，callable 可以拿到有返回值，callable 可以看作是 runnable 的补充。

> sleep() 和 wait() 有什么区别？

- 类的不同：sleep() 来自 Thread，wait() 来自 Object。
- 释放锁：sleep() 不释放锁；wait() 释放锁。
- 用法不同：sleep() 时间到会自动恢复；wait() 可以使用 notify()/notifyAll()直接唤醒。

> notify()和 notifyAll()有什么区别？

notifyAll()会唤醒所有的线程，notify()之后唤醒一个线程。notifyAll() 调用后，会将全部线程由等待池移到锁池，然后参与锁的竞争，竞争成功则继续执行，如果不成功则留在锁池等待锁被释放后再次参与竞争。而 notify()只会唤醒一个线程，具体唤醒哪一个线程由虚拟机控制。

> 线程的 run() 和 start() 有什么区别？

start() 方法用于启动线程，run() 方法用于执行线程的运行时代码。run() 可以重复调用，而 start() 只能调用一次。

> 线程池中 submit() 和 execute() 方法有什么区别？

- execute()：只能执行 Runnable 类型的任务。
- submit()：可以执行 Runnable 和 Callable 类型的任务。

Callable 类型的任务可以获取执行的返回值，而 Runnable 执行无返回值。

> 什么是 Java 序列化？什么情况下需要序列化？

Java 序列化是为了保存各种对象在内存中的状态，并且可以把保存的对象状态再读出来。

以下情况需要使用 Java 序列化：

- 想把的内存中的对象状态保存到一个文件中或者数据库中时候；
- 想用套接字在网络上传送对象的时候；
- 想通过RMI（远程方法调用）传输对象的时候。

> 动态代理是什么？有哪些应用？

动态代理是运行时动态生成代理类。

动态代理的应用有 spring aop、hibernate 数据查询、测试框架的后端 mock、rpc，Java注解对象获取等。

> 怎么实现动态代理？

JDK 原生动态代理和 cglib 动态代理。JDK 原生动态代理是基于接口实现的，而 cglib 是基于继承当前类的子类实现的。

> 如何实现对象克隆？

- 实现 Cloneable 接口并重写 Object 类中的 clone() 方法。
- 实现 Serializable 接口，通过对象的序列化和反序列化实现克隆，可以实现真正的深度克隆。

> 什么是 XSS 攻击，如何避免？

XSS 攻击：即跨站脚本攻击，它是 Web 程序中常见的漏洞。原理是攻击者往 Web 页面里插入恶意的脚本代码（css 代码、Javascript 代码等），当用户浏览该页面时，嵌入其中的脚本代码会被执行，从而达到恶意攻击用户的目的，如盗取用户 cookie、破坏页面结构、重定向到其他网站等。

预防 XSS 的核心是必须对输入的数据做过滤处理。

> 什么是 CSRF 攻击，如何避免？

CSRF：Cross-Site Request Forgery（中文：跨站请求伪造），可以理解为攻击者盗用了你的身份，以你的名义发送恶意请求，比如：以你名义发送邮件、发消息、购买商品，虚拟货币转账等。

防御手段：

验证请求来源地址；
关键操作添加验证码；
在请求地址添加 token 并验证。

> throw 和 throws 的区别？

- throw：是真实抛出一个异常。
- throws：是声明可能会抛出一个异常。

> final、finally、finalize 有什么区别？

final：是修饰符，如果修饰类，此类不能被继承；如果修饰方法和变量，则表示此方法和此变量不能在被改变，只能使用。
finally：是 try{} catch{} finally{} 最后一部分，表示不论发生任何情况都会执行，finally 部分可以省略，但如果 finally 部分存在，则一定会执行 finally 里面的代码。
finalize：是 Object 类的一个方法，在垃圾收集器执行的时候会调用被回收对象的此方法。

> try-catch-finally

try-catch-finally 其中 catch 和 finally 都可以被省略，但是不能同时省略，也就是说有 try 的时候，必须后面跟一个 catch 或者 finally。

> try-catch-finally 中，如果 catch 中 return 了，finally 还会执行吗？

finally 一定会执行，即使是 catch 中 return 了，catch 中的 return 会等 finally 中的代码执行完之后，才会执行。

> http 响应码 301 和 302 代表的是什么？有什么区别？

forward 是转发 和 redirect 是重定向：

地址栏 url 显示：foward url 不会发生改变，redirect url 会发生改变；
数据共享：forward 可以共享 request 里的数据，redirect 不能共享；
效率：forward 比 redirect 效率高。

> 简述 tcp 和 udp的区别？

tcp 和 udp 是 OSI 模型中的运输层中的协议。tcp 提供可靠的通信传输，而 udp 则常被用于让广播和细节控制交给应用的通信传输。

两者的区别大致如下：

tcp 面向连接，udp 面向非连接即发送数据前不需要建立链接；
tcp 提供可靠的服务（数据传输），udp 无法保证；
tcp 面向字节流，udp 面向报文；
tcp 数据传输慢，udp 数据传输快；

> tcp 为什么要三次握手，两次不行吗？为什么？

如果采用两次握手，那么只要服务器发出确认数据包就会建立连接，但由于客户端此时并未响应服务器端的请求，那此时服务器端就会一直在等待客户端，这样服务器端就白白浪费了一定的资源。若采用三次握手，服务器端没有收到来自客户端的再此确认，则就会知道客户端并没有要求建立请求，就不会浪费服务器的资源。

> 说一下 tcp 粘包是怎么产生的？

tcp 粘包可能发生在发送端或者接收端，分别来看两端各种产生粘包的原因：

- 发送端粘包：发送端需要等缓冲区满才发送出去，造成粘包；
- 接收方粘包：接收方不及时接收缓冲区的包，造成多个包接收。

> OSI 的七层模型都有哪些？

物理层：利用传输介质为数据链路层提供物理连接，实现比特流的透明传输。
数据链路层：负责建立和管理节点间的链路。
网络层：通过路由选择算法，为报文或分组通过通信子网选择最适当的路径。
传输层：向用户提供可靠的端到端的差错和流量控制，保证报文的正确传输。
会话层：向两个实体的表示层提供建立和使用连接的方法。
表示层：处理用户信息的表示问题，如编码、数据格式转换和加密解密等。
应用层：直接向用户提供服务，完成用户希望在网络上完成的各种工作。

> get 和 post 请求有哪些区别？

- get 请求会被浏览器主动缓存，而 post 不会。
- get 传递参数有大小限制，而 post 没有。
- post 参数传输更安全，get 的参数会明文限制在 url 上，post 不会。

> 如何实现跨域？

实现跨域有以下几种方案：

- 服务器端运行跨域 设置 CORS 等于 *；
- 在单个接口使用注解 @CrossOrigin 运行跨域；
- 使用 jsonp 跨域；

# 八.框架



> 为什么要使用 spring？

spring 提供 ioc 技术，容器会帮你管理依赖的对象，从而不需要自己创建和管理依赖对象了，更轻松的实现了程序的解耦。
spring 提供了事务支持，使得事务操作变的更加方便。
spring 提供了面向切片编程，这样可以更方便的处理某一类的问题。
更方便的框架集成，spring 可以很方便的集成其他框架，比如 MyBatis、hibernate 等。

> 解释一下什么是 aop？

aop 是面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。

简单来说就是统一处理某一“切面”（类）的问题的编程思想，比如统一处理日志、异常等。

> 解释一下什么是 ioc？

ioc：Inversionof Control（中文：控制反转）是 spring 的核心，对于 spring 框架来说，就是由 spring 来负责控制对象的生命周期和对象间的关系。

简单来说，控制指的是当前对象对内部成员的控制权；控制反转指的是，这种控制权不由当前对象管理了，由其他（类,第三方容器）来管理。

> spring 有哪些主要模块？

spring core：框架的最基础部分，提供 ioc 和依赖注入特性。
spring context：构建于 core 封装包基础上的 context 封装包，提供了一种框架式的对象访问方法。
spring dao：Data Access Object 提供了JDBC的抽象层。
spring aop：提供了面向切面的编程实现，让你可以自定义拦截器、切点等。
spring Web：提供了针对 Web 开发的集成特性，例如文件上传，利用 servlet listeners 进行 ioc 容器初始化和针对 Web 的 ApplicationContext。
spring Web mvc：spring 中的 mvc 封装包提供了 Web 应用的 Model-View-Controller（MVC）的实现。

> spring 常用的注入方式有哪些？

setter 属性注入
构造方法注入
注解方式注入

> spring 中的 bean 是线程安全的吗？

spring 中的 bean 默认是单例模式，spring 框架并没有对单例 bean 进行多线程的封装处理。

实际上大部分时候 spring bean 无状态的（比如 dao 类），所有某种程度上来说 bean 也是安全的，但如果 bean 有状态的话（比如 view model 对象），那就要开发者自己去保证线程安全了，最简单的就是改变 bean 的作用域，把“singleton”变更为“prototype”，这样请求 bean 相当于 new Bean()了，所以就可以保证线程安全了。

有状态就是有数据存储功能。
无状态就是不会保存数据。
> spring 支持几种 bean 的作用域？

spring 支持 5 种作用域，如下：

singleton：spring ioc 容器中只存在一个 bean 实例，bean 以单例模式存在，是系统默认值；
prototype：每次从容器调用 bean 时都会创建一个新的示例，既每次 getBean()相当于执行 new Bean()操作；
Web 环境下的作用域：
request：每次 http 请求都会创建一个 bean；
session：同一个 http session 共享一个 bean 实例；
global-session：用于 portlet 容器，因为每个 portlet 有单独的 session，globalsession 提供一个全局性的 http session。
「注意：」 使用 prototype 作用域需要慎重的思考，因为频繁创建和销毁 bean 会带来很大的性能开销。

> spring 自动装配 bean 有哪些方式？

no：默认值，表示没有自动装配，应使用显式 bean 引用进行装配。
byName：它根据 bean 的名称注入对象依赖项。
byType：它根据类型注入对象依赖项。
构造函数：通过构造函数来注入依赖项，需要设置大量的参数。
autodetect：容器首先通过构造函数使用 autowire 装配，如果不能，则通过 byType 自动装配。

> spring 事务实现方式有哪些？

声明式事务：声明式事务也有两种实现方式，基于 xml 配置文件的方式和注解方式（在类上添加 @Transaction 注解）。
编码方式：提供编码的形式管理和维护事务。

> 说一下 spring 的事务隔离？

spring 有五大隔离级别，默认值为 ISOLATION_DEFAULT（使用数据库的设置），其他四个隔离级别和数据库的隔离级别一致：

ISOLATION_DEFAULT：用底层数据库的设置隔离级别，数据库设置的是什么我就用什么；

ISOLATIONREADUNCOMMITTED：未提交读，最低隔离级别、事务未提交前，就可被其他事务读取（会出现幻读、脏读、不可重复读）；

ISOLATIONREADCOMMITTED：提交读，一个事务提交后才能被其他事务读取到（会造成幻读、不可重复读），SQL server 的默认级别；

ISOLATIONREPEATABLEREAD：可重复读，保证多次读取同一个数据时，其值都和事务开始时候的内容是一致，禁止读取到别的事务未提交的数据（会造成幻读），MySQL 的默认级别；

ISOLATION_SERIALIZABLE：序列化，代价最高最可靠的隔离级别，该隔离级别能防止脏读、不可重复读、幻读。

「脏读」 ：表示一个事务能够读取另一个事务中还未提交的数据。比如，某个事务尝试插入记录 A，此时该事务还未提交，然后另一个事务尝试读取到了记录 A。

「不可重复读」 ：是指在一个事务内，多次读同一数据。

「幻读」 ：指同一个事务内多次查询返回的结果集不一样。比如同一个事务 A 第一次查询时候有 n 条记录，但是第二次同等条件下查询却有 n+1 条记录，这就好像产生了幻觉。发生幻读的原因也是另外一个事务新增或者删除或者修改了第一个事务结果集里面的数据，同一个记录的数据内容被修改了，所有数据行的记录就变多或者变少了。

> 说一下 spring mvc 运行流程？

spring mvc 先将请求发送给 DispatcherServlet。
DispatcherServlet 查询一个或多个 HandlerMapping，找到处理请求的 Controller。
DispatcherServlet 再把请求提交到对应的 Controller。
Controller 进行业务逻辑处理后，会返回一个ModelAndView。
Dispathcher 查询一个或多个 ViewResolver 视图解析器，找到 ModelAndView 对象指定的视图对象。
视图对象负责渲染返回给客户端。

> spring mvc 有哪些组件？

前置控制器 DispatcherServlet。
映射控制器 HandlerMapping。
处理器 Controller。
模型和视图 ModelAndView。
视图解析器 ViewResolver。

> @RequestMapping 的作用是什么？

将 http 请求映射到相应的类/方法上。

> @Autowired 的作用是什么？

@Autowired 它可以对类成员变量、方法及构造函数进行标注，完成自动装配的工作，通过@Autowired 的使用来消除 set/get 方法。


> 什么是 spring boot？

spring boot 是为 spring 服务的，是用来简化新 spring 应用的初始搭建以及开发过程的。

> 为什么要用 spring boot？

配置简单
独立运行
自动装配
无代码生成和 xml 配置
提供应用监控
易上手
提升开发效率

> spring boot 核心配置文件是什么？

spring boot 核心的两个配置文件：

bootstrap (. yml 或者 . properties)：boostrap 由父 ApplicationContext 加载的，比 applicaton 优先加载，且 boostrap 里面的属性不能被覆盖；
application (. yml 或者 . properties)：用于 spring boot 项目的自动化配置。
> spring boot 配置文件有哪几种类型？它们有什么区别？

配置文件有 . properties 格式和 . yml 格式，它们主要的区别是书法风格不同。
yml 格式不支持 @PropertySource 注解导入。

> spring boot 有哪些方式可以实现热部署？

使用 devtools 启动热部署，添加 devtools 库，在配置文件中把 spring. devtools. restart. enabled 设置为 true；
使用 Intellij Idea 编辑器，勾上自动编译或手动重新编译。

> jpa 和 hibernate 有什么区别？

jpa 全称 Java Persistence API，是 Java 持久化接口规范，hibernate 属于 jpa 的具体实现。

> 什么是 spring cloud？

spring cloud 是一系列框架的有序集合。它利用 spring boot 的开发便利性巧妙地简化了分布式系统基础设施的开发，如服务发现注册、配置中心、消息总线、负载均衡、断路器、数据监控等，都可以用 spring boot 的开发风格做到一键启动和部署。

> spring cloud 断路器的作用是什么？

在分布式架构中，断路器模式的作用也是类似的，当某个服务单元发生故障（类似用电器发生短路）之后，通过断路器的故障监控（类似熔断保险丝），向调用方返回一个错误响应，而不是长时间的等待。这样就不会使得线程因调用故障服务被长时间占用不释放，避免了故障在分布式系统中的蔓延。

> spring cloud 的核心组件有哪些？

Eureka：服务注册于发现。
Feign：基于动态代理机制，根据注解和选择的机器，拼接请求 url 地址，发起请求。
Ribbon：实现负载均衡，从一个服务的多台机器中选择一台。
Hystrix：提供线程池，不同的服务走不同的线程池，实现了不同服务调用的隔离，避免了服务雪崩的问题。
Zuul：网关管理，由 Zuul 网关转发请求给对应的服务。

> 为什么要使用 hibernate？

hibernate 是对 jdbc 的封装，大大简化了数据访问层的繁琐的重复性代码。
hibernate 是一个优秀的 ORM 实现，很多程度上简化了 DAO 层的编码功能。
可以很方便的进行数据库的移植工作。
提供了缓存机制，是程序执行更改的高效。

> 什么是 ORM 框架？

ORM（Object Relation Mapping）对象关系映射，是把数据库中的关系数据映射成为程序中的对象。

使用 ORM 的优点：提高了开发效率降低了开发成本、开发更简单更对象化、可移植更强。

> hibernate 中如何在控制台查看打印的 SQL 语句？

在 Config 里面把 hibernate. show_SQL 设置为 true 就可以。但不建议开启，开启之后会降低程序的运行效率。

> hibernate 有几种查询方式？

三种：hql、原生 SQL、条件查询 Criteria。

> hibernate 实体类可以被定义为 final 吗？

实体类可以定义为 final 类，但这样的话就不能使用 hibernate 代理模式下的延迟关联提供性能了，所以不建议定义实体类为 final。

> 在 hibernate 中使用 Integer 和 int 做映射有什么区别？

Integer 类型为对象，它的值允许为 null，而 int 属于基础数据类型，值不能为 null。

> hibernate 是如何工作的？

读取并解析配置文件。
读取并解析映射文件，创建 SessionFactory。
打开 Session。
创建事务。
进行持久化操作。
提交事务。
关闭 Session。
关闭 SessionFactory。

> get()和 load()的区别？

数据查询时，没有 OID 指定的对象，get() 返回 null；load() 返回一个代理对象。
load()支持延迟加载；get() 不支持延迟加载。

> 说一下 hibernate 的缓存机制？

hibernate 常用的缓存有一级缓存和二级缓存：

一级缓存：也叫 Session 缓存，只在 Session 作用范围内有效，不需要用户干涉，由 hibernate 自身维护，可以通过：evict(object)清除 object 的缓存；clear()清除一级缓存中的所有缓存；flush()刷出缓存；

二级缓存：应用级别的缓存，在所有 Session 中都有效，支持配置第三方的缓存，如：EhCache。

> hibernate 对象有哪些状态？

临时/瞬时状态：直接 new 出来的对象，该对象还没被持久化（没保存在数据库中），不受 Session 管理。
持久化状态：当调用 Session 的 save/saveOrupdate/get/load/list 等方法的时候，对象就是持久化状态。
游离状态：Session 关闭之后对象就是游离状态。

> 在 hibernate 中 getCurrentSession 和 openSession 的区别是什么？

getCurrentSession 会绑定当前线程，而 openSession 则不会。
getCurrentSession 事务是 Spring 控制的，并且不需要手动关闭，而 openSession 需要我们自己手动开启和提交事务。



> hibernate 实体类必须要有无参构造函数吗？为什么？

hibernate 中每个实体类必须提供一个无参构造函数，因为 hibernate 框架要使用 reflection api，通过调用 ClassnewInstance() 来创建实体类的实例，如果没有无参的构造函数就会抛出异常。



> MyBatis 中 #{}和 ${}的区别是什么？

#{}是预编译处理，${}是字符替换。在使用 #{}时，MyBatis 会将 SQL 中的 #{}替换成“?”，配合 PreparedStatement 的 set 方法赋值，这样可以有效的防止 SQL 注入，保证程序的运行安全。

> MyBatis 有几种分页方式？

分页方式：逻辑分页和物理分页。

「逻辑分页：」 使用 MyBatis 自带的 RowBounds 进行分页，它是一次性查询很多数据，然后在数据中再进行检索。

「物理分页：」 自己手写 SQL 分页或使用分页插件 PageHelper，去数据库查询指定条数的分页数据的形式。

> RowBounds 是一次性查询全部结果吗？为什么？

RowBounds 表面是在“所有”数据中检索数据，其实并非是一次性查询出所有数据，因为 MyBatis 是对 jdbc 的封装，在 jdbc 驱动中有一个 Fetch Size 的配置，它规定了每次最多从数据库查询多少条数据，假如你要查询更多数据，它会在你执行 next()的时候，去查询更多的数据。就好比你去自动取款机取 10000 元，但取款机每次最多能取 2500 元，所以你要取 4 次才能把钱取完。只是对于 jdbc 来说，当你调用 next()的时候会自动帮你完成查询工作。这样做的好处可以有效的防止内存溢出。

> MyBatis 逻辑分页和物理分页的区别是什么？

逻辑分页是一次性查询很多数据，然后再在结果中检索分页的数据。这样做弊端是需要消耗大量的内存、有内存溢出的风险、对数据库压力较大。
物理分页是从数据库查询指定条数的数据，弥补了一次性全部查出的所有数据的种种缺点，比如需要大量的内存，对数据库查询压力较大等问题。

> MyBatis 是否支持延迟加载？延迟加载的原理是什么？

MyBatis 支持延迟加载，设置 lazyLoadingEnabled=true 即可。

延迟加载的原理的是调用的时候触发加载，而不是在初始化的时候就加载信息。比如调用 a. getB(). getName()，这个时候发现 a. getB() 的值为 null，此时会单独触发事先保存好的关联 B 对象的 SQL，先查询出来 B，然后再调用 a. setB(b)，而这时候再调用 a. getB(). getName() 就有值了，这就是延迟加载的基本原理。

> 说一下 MyBatis 的一级缓存和二级缓存？

一级缓存：基于 PerpetualCache 的 HashMap 本地缓存，它的声明周期是和 SQLSession 一致的，有多个 SQLSession 或者分布式的环境中数据库操作，可能会出现脏数据。当 Session flush 或 close 之后，该 Session 中的所有 Cache 就将清空，默认一级缓存是开启的。
二级缓存：也是基于 PerpetualCache 的 HashMap 本地缓存，不同在于其存储作用域为 Mapper 级别的，如果多个SQLSession之间需要共享缓存，则需要使用到二级缓存，并且二级缓存可自定义存储源，如 Ehcache。默认不打开二级缓存，要开启二级缓存，使用二级缓存属性类需要实现 Serializable 序列化接口(可用来保存对象的状态)。
开启二级缓存数据查询流程：二级缓存 -> 一级缓存 -> 数据库。

缓存更新机制：当某一个作用域(一级缓存 Session/二级缓存 Mapper)进行了C/U/D 操作后，默认该作用域下所有 select 中的缓存将被 clear。

> MyBatis 和 hibernate 的区别有哪些？

灵活性：MyBatis 更加灵活，自己可以写 SQL 语句，使用起来比较方便。
可移植性：MyBatis 有很多自己写的 SQL，因为每个数据库的 SQL 可以不相同，所以可移植性比较差。
学习和使用门槛：MyBatis 入门比较简单，使用门槛也更低。
二级缓存：hibernate 拥有更好的二级缓存，它的二级缓存可以自行更换为第三方的二级缓存。

> MyBatis 有哪些执行器（Executor）？

MyBatis 有三种基本的Executor执行器：

SimpleExecutor：每执行一次 update 或 select 就开启一个 Statement 对象，用完立刻关闭 Statement 对象；
ReuseExecutor：执行 update 或 select，以 SQL 作为 key 查找 Statement 对象，存在就使用，不存在就创建，用完后不关闭 Statement 对象，而是放置于 Map 内供下一次使用。简言之，就是重复使用 Statement 对象；
BatchExecutor：执行 update（没有 select，jdbc 批处理不支持 select），将所有 SQL 都添加到批处理中（addBatch()），等待统一执行（executeBatch()），它缓存了多个 Statement 对象，每个 Statement 对象都是 addBatch()完毕后，等待逐一执行 executeBatch()批处理，与 jdbc 批处理相同。
> MyBatis 分页插件的实现原理是什么？

分页插件的基本原理是使用 MyBatis 提供的插件接口，实现自定义插件，在插件的拦截方法内拦截待执行的 SQL，然后重写 SQL，根据 dialect 方言，添加对应的物理分页语句和物理分页参数。

> MyBatis 如何编写一个自定义插件？

MyBatis 自定义插件针对 MyBatis 四大对象（Executor、StatementHandler、ParameterHandler、ResultSetHandler）进行拦截：

Executor：拦截内部执行器，它负责调用 StatementHandler 操作数据库，并把结果集通过 ResultSetHandler 进行自动映射，另外它还处理了二级缓存的操作；
StatementHandler：拦截 SQL 语法构建的处理，它是 MyBatis 直接和数据库执行 SQL 脚本的对象，另外它也实现了 MyBatis 的一级缓存；
ParameterHandler：拦截参数的处理；
ResultSetHandler：拦截结果集的处理。



> RabbitMQ 的使用场景有哪些？

抢购活动，削峰填谷，防止系统崩塌。
延迟信息处理，比如 10 分钟之后给下单未付款的用户发送邮件提醒。
解耦系统，对于新增的功能可以单独写模块扩展，比如用户确认评价之后，新增了给用户返积分的功能，这个时候不用在业务代码里添加新增积分的功能，只需要把新增积分的接口订阅确认评价的消息队列即可，后面再添加任何功能只需要订阅对应的消息队列即可。

> RabbitMQ 有哪些重要的角色？

RabbitMQ 中重要的角色有：生产者、消费者和代理：

生产者：消息的创建者，负责创建和推送数据到消息服务器；
消费者：消息的接收方，用于处理数据和确认消息；
代理：就是 RabbitMQ 本身，用于扮演“快递”的角色，本身不生产消息，只是扮演“快递”的角色。
> RabbitMQ 有哪些重要的组件？

ConnectionFactory（连接管理器）：应用程序与Rabbit之间建立连接的管理器，程序代码中使用。
Channel（信道）：消息推送使用的通道。
Exchange（交换器）：用于接受、分配消息。
Queue（队列）：用于存储生产者的消息。
RoutingKey（路由键）：用于把生成者的数据分配到交换器上。
BindingKey（绑定键）：用于把交换器的消息绑定到队列上。

> RabbitMQ 中 vhost 的作用是什么？

vhost：每个 RabbitMQ 都能创建很多 vhost，我们称之为虚拟主机，每个虚拟主机其实都是 mini 版的RabbitMQ，它拥有自己的队列，交换器和绑定，拥有自己的权限机制。

> RabbitMQ 的消息是怎么发送的？

首先客户端必须连接到 RabbitMQ 服务器才能发布和消费消息，客户端和 rabbit server 之间会创建一个 tcp 连接，一旦 tcp 打开并通过了认证（认证就是你发送给 rabbit 服务器的用户名和密码），你的客户端和 RabbitMQ 就创建了一条 amqp 信道（channel），信道是创建在“真实” tcp 上的虚拟连接，amqp 命令都是通过信道发送出去的，每个信道都会有一个唯一的 id，不论是发布消息，订阅队列都是通过这个信道完成的。

> RabbitMQ 怎么保证消息的稳定性？

提供了事务的功能。
通过将 channel 设置为 confirm（确认）模式。

> RabbitMQ 怎么避免消息丢失？

把消息持久化磁盘，保证服务器重启消息不丢失。
每个集群中至少有一个物理磁盘，保证消息落入磁盘。

> 要保证消息持久化成功的条件有哪些？

声明队列必须设置持久化 durable 设置为 true.
消息推送投递模式必须设置持久化，deliveryMode 设置为 2（持久）。
消息已经到达持久化交换器。
消息已经到达持久化队列。
以上四个条件都满足才能保证消息持久化成功。

> RabbitMQ 持久化有什么缺点？

持久化的缺地就是降低了服务器的吞吐量，因为使用的是磁盘而非内存存储，从而降低了吞吐量。可尽量使用 ssd 硬盘来缓解吞吐量的问题。

> RabbitMQ 有几种广播类型？

direct（默认方式）：最基础最简单的模式，发送方把消息发送给订阅方，如果有多个订阅者，默认采取轮询的方式进行消息发送。
headers：与 direct 类似，只是性能很差，此类型几乎用不到。
fanout：分发模式，把消费分发给所有订阅者。
topic：匹配订阅模式，使用正则匹配到消息队列，能匹配到的都能接收到。

> RabbitMQ 怎么实现延迟消息队列？

延迟队列的实现有两种方式：

通过消息过期后进入死信交换器，再由交换器转发到延迟消费队列，实现延迟功能；
使用 RabbitMQ-delayed-message-exchange 插件实现延迟功能。
> RabbitMQ 集群有什么用？

集群主要有以下两个用途：

高可用：某个服务器出现问题，整个 RabbitMQ 还可以继续使用；
高容量：集群可以承载更多的消息量。

> RabbitMQ 节点的类型有哪些？

磁盘节点：消息会存储到磁盘。
内存节点：消息都存储在内存中，重启服务器消息丢失，性能高于磁盘类型。

> RabbitMQ 集群搭建需要注意哪些问题？

各节点之间使用“--link”连接，此属性不能忽略。
各节点使用的 erlang cookie 值必须相同，此值相当于“秘钥”的功能，用于各节点的认证。
整个集群中必须包含一个磁盘节点。

> RabbitMQ 每个节点是其他节点的完整拷贝吗？为什么？

不是，原因有以下两个：

存储空间的考虑：如果每个节点都拥有所有队列的完全拷贝，这样新增节点不但没有新增存储空间，反而增加了更多的冗余数据；
性能的考虑：如果每条消息都需要完整拷贝到每一个集群节点，那新增节点并没有提升处理消息的能力，最多是保持和单节点相同的性能甚至是更糟。

> RabbitMQ 集群中唯一一个磁盘节点崩溃了会发生什么情况？

如果唯一磁盘的磁盘节点崩溃了，不能进行以下操作：

不能创建队列
不能创建交换器
不能创建绑定
不能添加用户
不能更改权限
不能添加和删除集群节点
唯一磁盘节点崩溃了，集群是可以保持运行的，但你不能更改任何东西。

> RabbitMQ 对集群节点停止顺序有要求吗？

RabbitMQ 对集群的停止的顺序是有要求的，应该先关闭内存节点，最后再关闭磁盘节点。如果顺序恰好相反的话，可能会造成消息的丢失。

Kafka
> kafka 可以脱离 zookeeper 单独使用吗？为什么？

kafka 不能脱离 zookeeper 单独使用，因为 kafka 使用 zookeeper 管理和协调 kafka 的节点服务器。

> kafka 有几种数据保留的策略？

kafka 有两种数据保存策略：按照过期时间保留和按照存储的消息大小保留。

> kafka 同时设置了 7 天和 10G 清除数据，到第五天的时候消息达到了 10G，这个时候 kafka 将如何处理？

这个时候 kafka 会执行数据清除工作，时间和大小不论那个满足条件，都会清空数据。

> 使用 kafka 集群需要注意什么？

集群的数量不是越多越好，最好不要超过 7 个，因为节点越多，消息复制需要的时间就越长，整个群组的吞吐量就越低。
集群数量最好是单数，因为超过一半故障集群就不能用了，设置为单数容错率更高。

> zookeeper 是什么？

zookeeper 是一个分布式的，开放源码的分布式应用程序协调服务，是 google chubby 的开源实现，是 hadoop 和 hbase 的重要组件。它是一个为分布式应用提供一致性服务的软件，提供的功能包括：配置维护、域名服务、分布式同步、组服务等。

> zookeeper 都有哪些功能？

集群管理：监控节点存活状态、运行请求等。
主节点选举：主节点挂掉了之后可以从备用的节点开始新一轮选主，主节点选举说的就是这个选举的过程，使用 zookeeper 可以协助完成这个过程。
分布式锁：zookeeper 提供两种锁：独占锁、共享锁。独占锁即一次只能有一个线程使用资源，共享锁是读锁共享，读写互斥，即可以有多线线程同时读同一个资源，如果要使用写锁也只能有一个线程使用。zookeeper可以对分布式锁进行控制。
命名服务：在分布式系统中，通过使用命名服务，客户端应用能够根据指定名字来获取资源或服务的地址，提供者等信息。

> zookeeper 有几种部署模式？

zookeeper 有三种部署模式：

单机部署：一台集群上运行；
集群部署：多台集群运行；
伪集群部署：一台集群启动多个 zookeeper 实例运行。

> zookeeper 怎么保证主从节点的状态同步？

zookeeper 的核心是原子广播，这个机制保证了各个 server 之间的同步。实现这个机制的协议叫做 zab 协议。zab 协议有两种模式，分别是恢复模式（选主）和广播模式（同步）。当服务启动或者在领导者崩溃后，zab 就进入了恢复模式，当领导者被选举出来，且大多数 server 完成了和 leader 的状态同步以后，恢复模式就结束了。状态同步保证了 leader 和 server 具有相同的系统状态。

> 集群中为什么要有主节点？

在分布式环境中，有些业务逻辑只需要集群中的某一台机器进行执行，其他的机器可以共享这个结果，这样可以大大减少重复计算，提高性能，所以就需要主节点。

> 集群中有 3 台服务器，其中一个节点宕机，这个时候 zookeeper 还可以使用吗？

可以继续使用，单数服务器只要没超过一半的服务器宕机就可以继续使用。

> 说一下 zookeeper 的通知机制？

客户端端会对某个 znode 建立一个 watcher 事件，当该 znode 发生变化时，这些客户端会收到 zookeeper 的通知，然后客户端可以根据 znode 变化来做出业务上的改变。


> 数据库的三范式是什么？

第一范式：即数据库表的每一列都是不可分割的原子数据项。
第二范式：消除非主属性对主属性的依赖。
第三范式：消除传递依赖。

> 一张自增表里面总共有 7 条数据，删除了最后 2 条数据，重启 MySQL 数据库，又插入了一条数据，此时 id 是几？

表类型如果是 MyISAM ，那 id 就是 8。
表类型如果是 InnoDB，那 id 就是 6。
InnoDB 表只会把自增主键的最大 id 记录在内存中，所以重启之后会导致最大 id 丢失。

> 如何获取当前数据库版本？

使用 select version() 获取当前 MySQL 数据库版本。

> 说一下 ACID 是什么？

Atomicity（原子性）：一个事务（transaction）中的所有操作，或者全部完成，或者全部不完成，不会结束在中间某个环节。事务在执行过程中发生错误，会被恢复（Rollback）到事务开始前的状态，就像这个事务从来没有执行过一样。即，事务不可分割、不可约简。
Consistency（一致性）：在事务开始之前和事务结束以后，数据库的完整性没有被破坏。这表示写入的资料必须完全符合所有的预设约束、触发器、级联回滚等。
Isolation（隔离性）：数据库允许多个并发事务同时对其数据进行读写和修改的能力，隔离性可以防止多个事务并发执行时由于交叉执行而导致数据的不一致。事务隔离分为不同级别，包括读未提交（Read uncommitted）、读提交（read committed）、可重复读（repeatable read）和串行化（Serializable）。
Durability（持久性）：事务处理结束后，对数据的修改就是永久的，即便系统故障也不会丢失。

> MySQL 的内连接、左连接、右连接有什么区别？

内连接关键字：inner join；左连接：left join；右连接：right join。

内连接是把匹配的关联数据显示出来；左连接是左边的表全部显示出来，右边的表显示出符合条件的数据；右连接正好相反。

> MySQL 索引是怎么实现的？

索引是满足某种特定查找算法的数据结构，而这些数据结构会以某种方式指向数据，从而实现高效查找数据。

具体来说 MySQL 中的索引，不同的数据引擎实现有所不同，但目前主流的数据库引擎的索引都是 B+ 树实现的，B+ 树的搜索效率，可以到达二分法的性能，找到数据区域之后就找到了完整的数据结构了，所有索引的性能也是更好的。

> 怎么验证 MySQL 的索引是否满足需求？

使用 explain 查看 SQL 是如何执行查询语句的，从而分析你的索引是否满足需求。

explain 语法：explain select * from table where type=1。

> 说一下数据库的事务隔离？

MySQL 的事务隔离是在 MySQL. ini 配置文件里添加的，在文件的最后添加：

❝transaction-isolation = REPEATABLE-READ
❞
可用的配置值：READ-UNCOMMITTED、READ-COMMITTED、REPEATABLE-READ、SERIALIZABLE。

READ-UNCOMMITTED：未提交读，最低隔离级别、事务未提交前，就可被其他事务读取（会出现幻读、脏读、不可重复读）。
READ-COMMITTED：提交读，一个事务提交后才能被其他事务读取到（会造成幻读、不可重复读）。
REPEATABLE-READ：可重复读，默认级别，保证多次读取同一个数据时，其值都和事务开始时候的内容是一致，禁止读取到别的事务未提交的数据（会造成幻读）。
SERIALIZABLE：序列化，代价最高最可靠的隔离级别，该隔离级别能防止脏读、不可重复读、幻读。
「脏读」 ：表示一个事务能够读取另一个事务中还未提交的数据。比如，某个事务尝试插入记录 A，此时该事务还未提交，然后另一个事务尝试读取到了记录 A。

「不可重复读」 ：是指在一个事务内，多次读同一数据。

「幻读」 ：指同一个事务内多次查询返回的结果集不一样。比如同一个事务 A 第一次查询时候有 n 条记录，但是第二次同等条件下查询却有 n+1 条记录，这就好像产生了幻觉。发生幻读的原因也是另外一个事务新增或者删除或者修改了第一个事务结果集里面的数据，同一个记录的数据内容被修改了，所有数据行的记录就变多或者变少了。

> 说一下 MySQL 常用的引擎？

InnoDB 引擎：mysql 5.1 后默认的数据库引擎，提供了对数据库 acid 事务的支持，并且还提供了行级锁和外键的约束，它的设计的目标就是处理大数据容量的数据库系统。MySQL 运行的时候，InnoDB 会在内存中建立缓冲池，用于缓冲数据和索引。但是该引擎是不支持全文搜索，同时启动也比较的慢，它是不会保存表的行数的，所以当进行 select count() from table 指令的时候，需要进行扫描全表。由于锁的粒度小，写操作是不会锁定全表的,所以在并发度较高的场景下使用会提升效率的。
MyIASM 引擎：不提供事务的支持，也不支持行级锁和外键。因此当执行插入和更新语句时，即执行写操作的时候需要锁定这个表，所以会导致效率会降低。不过和 InnoDB 不同的是，MyIASM 引擎是保存了表的行数，于是当进行 select count() from table 语句时，可以直接的读取已经保存的值而不需要进行扫描全表。所以，如果表的读操作远远多于写操作时，并且不需要事务的支持的，可以将 MyIASM 作为数据库引擎的首选。

> 说一下 MySQL 的行锁和表锁？

MyISAM 只支持表锁，InnoDB 支持表锁和行锁，默认为行锁。

表级锁：开销小，加锁快，不会出现死锁。锁定粒度大，发生锁冲突的概率最高，并发量最低。
行级锁：开销大，加锁慢，会出现死锁。锁力度小，发生锁冲突的概率小，并发度最高。
> 说一下乐观锁和悲观锁？

乐观锁：每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在提交更新的时候会判断一下在此期间别人有没有去更新这个数据。
悲观锁：每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会阻止，直到这个锁被释放。
数据库的乐观锁需要自己实现，在表里面添加一个 version 字段，每次修改成功值加 1，这样每次修改的时候先对比一下，自己拥有的 version 和数据库现在的 version 是否一致，如果不一致就不修改，这样就实现了乐观锁。

> MySQL 问题排查都有哪些手段？

使用 show processlist 命令查看当前所有连接信息。
使用 explain 命令查询 SQL 语句执行计划。
开启慢查询日志，查看慢查询的 SQL。

> 如何做 MySQL 的性能优化？

为搜索字段创建索引。
避免使用 select *，列出需要查询的字段。
垂直分割分表。
选择正确的存储引擎。

> Redis 是什么？都有哪些使用场景？

Redis 是一个使用 C 语言开发的高速缓存数据库。

Redis 使用场景：

记录帖子点赞数、点击数、评论数；
缓存近期热帖；
缓存文章详情信息；
记录用户会话信息。
> Redis 有哪些功能？

数据缓存功能
分布式锁的功能
支持数据持久化
支持事务
支持消息队列

> Redis 为什么是单线程的？

因为 cpu 不是 Redis 的瓶颈，Redis 的瓶颈最有可能是机器内存或者网络带宽。既然单线程容易实现，而且 cpu 又不会成为瓶颈，那就顺理成章地采用单线程的方案了。

关于 Redis 的性能，官方网站也有，普通笔记本轻松处理每秒几十万的请求。

而且单线程并不代表就慢 nginx 和 nodejs 也都是高性能单线程的代表。

> 什么是缓存穿透？怎么解决？

缓存穿透：指查询一个一定不存在的数据，由于缓存是不命中时需要从数据库查询，查不到数据则不写入缓存，这将导致这个不存在的数据每次请求都要到数据库去查询，造成缓存穿透。

解决方案：最简单粗暴的方法如果一个查询返回的数据为空（不管是数据不存在，还是系统故障），我们就把这个空结果进行缓存，但它的过期时间会很短，最长不超过五分钟。

> Redis 支持的数据类型有哪些？

Redis 支持的数据类型：string（字符串）、list（列表）、hash（字典）、set（集合）、zset（有序集合）。

> Redis 支持的 Java 客户端都有哪些？

支持的 Java 客户端有 Redisson、jedis、lettuce 等。

> jedis 和 Redisson 有哪些区别？

jedis：提供了比较全面的 Redis 命令的支持。
Redisson：实现了分布式和可扩展的 Java 数据结构，与 jedis 相比 Redisson 的功能相对简单，不支持排序、事务、管道、分区等 Redis 特性。

> 怎么保证缓存和数据库数据的一致性？

合理设置缓存的过期时间。
新增、更改、删除数据库操作时同步更新 Redis，可以使用事物机制来保证数据的一致性。

> Redis 持久化有几种方式？

Redis 的持久化有两种方式，或者说有两种策略：

RDB（Redis Database）：指定的时间间隔能对你的数据进行快照存储。
AOF（Append Only File）：每一个收到的写命令都通过write函数追加到文件中。
> Redis 怎么实现分布式锁？

Redis 分布式锁其实就是在系统里面占一个“坑”，其他程序也要占“坑”的时候，占用成功了就可以继续执行，失败了就只能放弃或稍后重试。

占坑一般使用 setnx(set if not exists)指令，只允许被一个程序占有，使用完调用 del 释放锁。

> Redis 分布式锁有什么缺陷？

Redis 分布式锁不能解决超时的问题，分布式锁有一个超时时间，程序的执行如果超出了锁的超时时间就会出现问题。

> Redis 如何做内存优化？

尽量使用 Redis 的散列表，把相关的信息放到散列表里面存储，而不是把每个字段单独存储，这样可以有效的减少内存使用。比如将 Web 系统的用户对象，应该放到散列表里面再整体存储到 Redis，而不是把用户的姓名、年龄、密码、邮箱等字段分别设置 key 进行存储。

> Redis 淘汰策略有哪些？

volatile-lru：从已设置过期时间的数据集（server. db[i]. expires）中挑选最近最少使用的数据淘汰。
volatile-ttl：从已设置过期时间的数据集（server. db[i]. expires）中挑选将要过期的数据淘汰。
volatile-random：从已设置过期时间的数据集（server. db[i]. expires）中任意选择数据淘汰。
allkeys-lru：从数据集（server. db[i]. dict）中挑选最近最少使用的数据淘汰。
allkeys-random：从数据集（server. db[i]. dict）中任意选择数据淘汰。
no-enviction（驱逐）：禁止驱逐数据。

> Redis 常见的性能问题有哪些？该如何解决？

主服务器写内存快照，会阻塞主线程的工作，当快照比较大时对性能影响是非常大的，会间断性暂停服务，所以主服务器最好不要写内存快照。
Redis 主从复制的性能问题，为了主从复制的速度和连接的稳定性，主从库最好在同一个局域网内。

> 说一下 JVM 的主要组成部分？及其作用？

类加载器（ClassLoader）
运行时数据区（Runtime Data Area）
执行引擎（Execution Engine）
本地库接口（Native Interface）
「组件的作用：」 首先通过类加载器（ClassLoader）会把 Java 代码转换成字节码，运行时数据区（Runtime Data Area）再把字节码加载到内存中，而字节码文件只是 JVM 的一套指令集规范，并不能直接交给底层操作系统去执行，因此需要特定的命令解析器执行引擎（Execution Engine），将字节码翻译成底层系统指令，再交由 CPU 去执行，而这个过程中需要调用其他语言的本地库接口（Native Interface）来实现整个程序的功能。

> 说一下 JVM 运行时数据区？

不同虚拟机的运行时数据区可能略微有所不同，但都会遵从 Java 虚拟机规范， Java 虚拟机规范规定的区域分为以下 5 个部分：

程序计数器（Program Counter Register）：当前线程所执行的字节码的行号指示器，字节码解析器的工作是通过改变这个计数器的值，来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等基础功能，都需要依赖这个计数器来完成；
Java 虚拟机栈（Java Virtual Machine Stacks）：用于存储局部变量表、操作数栈、动态链接、方法出口等信息；
本地方法栈（Native Method Stack）：与虚拟机栈的作用是一样的，只不过虚拟机栈是服务 Java 方法的，而本地方法栈是为虚拟机调用 Native 方法服务的；
Java 堆（Java Heap）：Java 虚拟机中内存最大的一块，是被所有线程共享的，几乎所有的对象实例都在这里分配内存；
方法区（Methed Area）：用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译后的代码等数据。
> 说一下堆栈的区别？

功能方面：堆是用来存放对象的，栈是用来执行程序的。
共享性：堆是线程共享的，栈是线程私有的。
空间大小：堆大小远远大于栈。

> 队列和栈是什么？有什么区别？

队列和栈都是被用来预存储数据的。

队列允许先进先出检索元素，但也有例外的情况，Deque 接口允许从两端检索元素。

栈和队列很相似，但它运行对元素进行后进先出进行检索。

> 什么是双亲委派模型？

在介绍双亲委派模型之前先说下类加载器。对于任意一个类，都需要由加载它的类加载器和这个类本身一同确立在 JVM 中的唯一性，每一个类加载器，都有一个独立的类名称空间。类加载器就是根据指定全限定名称将 class 文件加载到 JVM 内存，然后再转化为 class 对象。

类加载器分类：

启动类加载器（Bootstrap ClassLoader），是虚拟机自身的一部分，用来加载Java_HOME/lib/目录中的，或者被 -Xbootclasspath 参数所指定的路径中并且被虚拟机识别的类库；
其他类加载器：
扩展类加载器（Extension ClassLoader）：负责加载<java_home style="box-sizing: border-box; outline: 0px !important;">libext目录或Java. ext. dirs系统变量指定的路径中的所有类库；
应用程序类加载器（Application ClassLoader）。负责加载用户类路径（classpath）上的指定类库，我们可以直接使用这个类加载器。一般情况，如果我们没有自定义类加载器默认就是用这个加载器。
双亲委派模型：如果一个类加载器收到了类加载的请求，它首先不会自己去加载这个类，而是把这个请求委派给父类加载器去完成，每一层的类加载器都是如此，这样所有的加载请求都会被传送到顶层的启动类加载器中，只有当父加载无法完成加载请求（它的搜索范围中没找到所需的类）时，子加载器才会尝试去加载类。

> 说一下类装载的执行过程？

类装载分为以下 5 个步骤：

加载：根据查找路径找到相应的 class 文件然后导入；
检查：检查加载的 class 文件的正确性；
准备：给类中的静态变量分配内存空间；
解析：虚拟机将常量池中的符号引用替换成直接引用的过程。符号引用就理解为一个标示，而在直接引用直接指向内存中的地址；
初始化：对静态变量和静态代码块执行初始化工作。
> 怎么判断对象是否可以被回收？

一般有两种方法来判断：

引用计数器：为每个对象创建一个引用计数，有对象引用时计数器 +1，引用被释放时计数 -1，当计数器为 0 时就可以被回收。它有一个缺点不能解决循环引用的问题；
可达性分析：从 GC Roots 开始向下搜索，搜索所走过的路径称为引用链。当一个对象到 GC Roots 没有任何引用链相连时，则证明此对象是可以被回收的。
> Java 中都有哪些引用类型？

强引用：发生 gc 的时候不会被回收。
软引用：有用但不是必须的对象，在发生内存溢出之前会被回收。
弱引用：有用但不是必须的对象，在下一次GC时会被回收。
虚引用（幽灵引用/幻影引用）：无法通过虚引用获得对象，用 PhantomReference 实现虚引用，虚引用的用途是在 gc 时返回一个通知。

> 说一下 JVM 有哪些垃圾回收算法？

标记-清除算法：标记无用对象，然后进行清除回收。缺点：效率不高，无法清除垃圾碎片。
标记-整理算法：标记无用对象，让所有存活的对象都向一端移动，然后直接清除掉端边界以外的内存。
复制算法：按照容量划分二个大小相等的内存区域，当一块用完的时候将活着的对象复制到另一块上，然后再把已使用的内存空间一次清理掉。缺点：内存使用率不高，只有原来的一半。
分代算法：根据对象存活周期的不同将内存划分为几块，一般是新生代和老年代，新生代基本采用复制算法，老年代采用标记整理算法。

> 说一下 JVM 有哪些垃圾回收器？

Serial：最早的单线程串行垃圾回收器。
Serial Old：Serial 垃圾回收器的老年版本，同样也是单线程的，可以作为 CMS 垃圾回收器的备选预案。
ParNew：是 Serial 的多线程版本。
Parallel 和 ParNew 收集器类似是多线程的，但 Parallel 是吞吐量优先的收集器，可以牺牲等待时间换取系统的吞吐量。
Parallel Old 是 Parallel 老生代版本，Parallel 使用的是复制的内存回收算法，Parallel Old 使用的是标记-整理的内存回收算法。
CMS：一种以获得最短停顿时间为目标的收集器，非常适用 B/S 系统。
G1：一种兼顾吞吐量和停顿时间的 GC 实现，是 JDK 9 以后的默认 GC 选项。

> 详细介绍一下 CMS 垃圾回收器？

CMS 是英文 Concurrent Mark-Sweep 的简称，是以牺牲吞吐量为代价来获得最短回收停顿时间的垃圾回收器。对于要求服务器响应速度的应用上，这种垃圾回收器非常适合。在启动 JVM 的参数加上“-XX:+UseConcMarkSweepGC”来指定使用 CMS 垃圾回收器。

CMS 使用的是标记-清除的算法实现的，所以在 gc 的时候回产生大量的内存碎片，当剩余内存不能满足程序运行要求时，系统将会出现 Concurrent Mode Failure，临时 CMS 会采用 Serial Old 回收器进行垃圾清除，此时的性能将会被降低。

> 新生代垃圾回收器和老生代垃圾回收器都有哪些？有什么区别？

新生代回收器：Serial、ParNew、Parallel Scavenge
老年代回收器：Serial Old、Parallel Old、CMS
整堆回收器：G1
新生代垃圾回收器一般采用的是复制算法，复制算法的优点是效率高，缺点是内存利用率低；老年代回收器一般采用的是标记-整理的算法进行垃圾回收。

> 简述分代垃圾回收器是怎么工作的？

分代回收器有两个分区：老生代和新生代，新生代默认的空间占比总空间的 1/3，老生代的默认占比是 2/3。

新生代使用的是复制算法，新生代里有 3 个分区：Eden、To Survivor、From Survivor，它们的默认占比是 8:1:1，它的执行流程如下：

把 Eden + From Survivor 存活的对象放入 To Survivor 区；
清空 Eden 和 From Survivor 分区；
From Survivor 和 To Survivor 分区交换，From Survivor 变 To Survivor，To Survivor 变 From Survivor。
每次在 From Survivor 到 To Survivor 移动时都存活的对象，年龄就 +1，当年龄到达 15（默认配置是 15）时，升级为老生代。大对象也会直接进入老生代。

老生代当空间占用到达某个值之后就会触发全局垃圾收回，一般使用标记整理的执行算法。以上这些循环往复就构成了整个分代垃圾回收的整体执行流程。

> 说一下 JVM 调优的工具？

JDK 自带了很多监控工具，都位于 JDK 的 bin 目录下，其中最常用的是 jconsole 和 jvisualvm 这两款视图监控工具。

jconsole：用于对 JVM 中的内存、线程和类等进行监控；
jvisualvm：JDK 自带的全能分析工具，可以分析：内存快照、线程快照、程序死锁、监控内存的变化、gc 变化等。
> 常用的 JVM 调优的参数都有哪些？

-Xms2g：初始化推大小为 2g；
-Xmx2g：堆最大内存为 2g；
-XX:NewRatio=4：设置年轻的和老年代的内存比例为 1:4；
-XX:SurvivorRatio=8：设置新生代 Eden 和 Survivor 比例为 8:2；
–XX:+UseParNewGC：指定使用 ParNew + Serial Old 垃圾回收器组合；
-XX:+UseParallelOldGC：指定使用 ParNew + ParNew Old 垃圾回收器组合；
-XX:+UseConcMarkSweepGC：指定使用 CMS + Serial Old 垃圾回收器组合；
-XX:+PrintGC：开启打印 gc 信息；
-XX:+PrintGCDetails：打印 gc 详细信息。

# 九 增强

> mysql增强

一.为什么使用B+树

1.相比红黑树和二叉树，高度更低，增加读取效率。

2.b+树相对B树，由于只有叶子节点存在数据，非叶子节点只存在KEY，使得磁盘IO次数减少，能够提高查询速度。叶子节点由双向链表链接，批量查询或者顺序查询性能更高。



二.回表操作

```sql
select * from t1 where name =‘tao’
select id from t1 where name =‘tao’
```

二者区别就是第一条查询所有字段，第二条只查询id，这二条语句的区别就再于是否会触发回表操作，第一条就会触发回表操作，是因为我们根据name这个字段的B+树只能得到对应name的id值，需要知道其他的字段还需要通过主键id维护的B+树回表进行查找，否则无法查找其他字段，而第二条SQL就只要id，那么无需查找其他字段，所以不会触发回表操作，那么个这个第二条不会触发回表操作的过程就叫做索引覆盖

避免回表：索引覆盖。



三.最左匹配

这个最左匹配原则是发生在建立多字段的组合索引是参数的，如我们有表t1字段有id，name，age，sex 当我们查询条件通常为name和age时，我们给以给name和age建立组合索引(name，age)，此时我们如果跨国name 直接where age是不会使用该联合索引得。

我们写几个sql理解下

```sql
-- 当前组合索引(name,age)
select * from t1 where name =‘tao’
select * from t1 where name =‘tao’ and age=10
select * from t1 where age=10
select * from t1 where age=10 and name =‘tao’
```

以上能用到索引的有第一条，第二条，第四条。第一条name能匹配到最左边的索引name所以是可以的，第二条，既能匹配到name也能匹配到age，第三条则是直接跨过了name那么违背了最左匹配原则的，所以不行，第四条，这个有人会质疑了，这里先查询的是age后查询的是name，顺序对不上呀，这里我们需要知道MySQL的逻辑架构，MySQL逻辑架构，MySQL逻辑架构中有一个优化器，这个优化器会帮我们优化SQL，所以这里优化后是name在前，age在后。这个我们可以创建索引使用explain看看。这里不单是在WHERE条件时会优化，在表关联的时候也会优化。所以结论是除了第三条都会用到联合索引。



四. InnoDB得特性

1.在不通过索引条件查询的时候，InnoDB使用的确实是表锁（锁的是整张表）

2.由于 MySQL 的行锁是针对索引加的锁,不是针对记录加的锁,所以虽然是访问不同行的记录,但是如果是使用相同的索引键,是会出现锁冲突的。

3.当表有多个索引的时候,不同的事务可以使用不同的索引锁定不同的行,另外,不论 是使用主键索引、唯一索引或普通索引,InnoDB都会使用行锁来对数据加锁。

4.即便在条件中使用了索引字段,但是否使用索引来检索数据是由 MySQL 通过判断不同 执行计划的代价来决定的,如果 MySQL 认为全表扫效率更高,比如对一些很小的表,它 就不会使用索引,这种情况下 InnoDB 将使用表锁,而不是行锁。因此,在分析锁冲突时, 别忘了检查SQL 的执行计划（explain查看）,以确认是否真正使用 了索引。



五. InnoDB如何实现可重复读

1. InnoDB 的行数据有多个版本，每个版本都有 row trx_id。
2. 事务根据 undo log 和 trx_id 构建出满足当前隔离级别的一致性视图。
3. 可重复读的核心是一致性读，而事务更新数据的时候，只能使用当前读，如果当前记录的行锁被其他事务占用，就需要进入锁等待。



六. explain得各种type

   eq_ref：主键或唯一键索引被连接使用，最多只会返回一条符合条件的记录。简单的select查询不会出现这种type。

   ref：相比eq_ref，不使用唯一索引，而是使用普通索引或者唯一索引的部分前缀，索引和某个值比较，会找到多个符合条件的行。       

   range：通常出现在范    围查询中，比如in、between、大于、小于等。使用索引来检索给定范围的行。

   index：扫描全索引拿到结果，一般是扫描某个二级索引，二级索引一般比较少，所以通常比ALL快一点。

   ALL：全表扫描，扫描聚簇索引的所有叶子节点。



七.mvcc(多版本并发控制)（实现事物的隔离性）

MVCC(Multi-Version Concurrency Control)，多版本并发控制。MVCC是一种并发控制方法，通俗点就是MVCC通过保存数据的历史版本，根据比较版本号来处理数据的是否显示，从而达到读取数据的时候不需要加锁就可以保证事务隔离性的效果。mysql中的innoDB中就是使用这种方法来提高读写事务控制的、他大大提高了读写事务的并发性能，原因是MVCC是一种不采用锁来控制事物的方式，是一种非堵塞、同时还可以解决脏读，幻读，不可重复读等事务隔离问题，但不能解决更新丢失问题。
快照读和当前读

**当前读**读取的是数据库记录，都是当前最新的版本，会对当前读取的数据进行加锁，防止其他事务修改数据。

**快照读**的实现就是基于多版本并发控制，即MVCC,既然是多版本，那么快照读读到的数据不一定是当前的最新的数据，有可能是之前历史版本的数据。

事务版本号：
每次事务开启前都会从数据库获得一个自增长的事务ID，可以从事务ID判断事务的执行先后顺序。
表格隐藏列
trx_id ：每次一个事务对某条聚簇索引记录进行改动时，都会把该事务的 事务id 赋值给 trx_id 隐藏列。
roll_pointer ：每次对某条聚簇索引记录进行改动时，都会把旧的版本写入到 undo日志 中，然后这个隐藏列就相当于一个指针，可以通过它来找到该记录修改前的信息。
Undo log
Undo log主要用于记录数据被修改之前的日志，在表信息修改之前先会把数据拷贝到undo log中，当事务进行回滚操作是可以通过undo log里的日志进行数据还原。
用于MVCC快照读的数据，在MVCC多版本控制中，通过读取undo log的历史版本数据可以实现不同事务版本号都拥有自己独立的快照数据版本。
每次对记录进行改动，都会记录一条 undo日志 ，每条 undo日志 也都有一个 roll_pointer 属性（ INSERT 操作对应的 undo日志 没有该属性，因为该记录并没有更早的版本），可以将这些 undo日志 都连起来，串成一个链表



八.二级索引(辅助索引) 聚簇索引

1. **聚簇索引**: 一般建表时的主键就会被mysql作为聚簇索引，如果没有主键则选择非空唯一的索引作为聚簇索引，都没有则隐式创建一个索引作为聚簇索引
2. **辅助索引**: 也就是非聚簇索引或二级索引，平时我们添加的索引就是辅助索引；

创建一张表时默认会为主键创建聚簇索引，聚簇(主键)索引的叶子节点存的是整行数据。

除了聚簇(主键)索引以外的所有索引都称为二级索引也就是非主键索引，二级索引的叶子节点内容是主键的值，主键长度越小，二级索引的叶子节点就越小，占用的空间也就越小；

二级索引在查询需要多扫描一棵索引树，也就是回表，通过覆盖索引和默认的索引下推机制可以避免回表


九.行锁（记录锁） 间隙锁 临键锁(next-key lock)

这两锁是innodb 在 可重复读下为了满足解决幻读问题引入的锁机制

行锁列必须为位移索引或主键列且必须是精准匹配（=） 某则会退化成 临键锁



在非唯一索引的情况下

间隙锁 锁定的是记录与记录之间的空隙，间隙锁只阻塞插入操作，解决幻读问题

在可重复读隔离级别下，临键锁 nextkeylock 是行锁与间隙锁的并集。



总结: 

只使用唯一索引查询，并且只锁定一条记录时，innoDB会使用行锁。
只使用唯一索引查询，但是检索条件是范围检索，或者是唯一检索但检索结果不存在（试图锁住不存在的数据）时，会产生 Next-Key Lock（临键锁）。
使用普通索引检索时，不管是何种查询，只要加锁，都会产生间隙锁（Gap Lock）。
同时使用唯一索引和普通索引时，由于数据行是优先根据普通索引排序，再根据唯一索引排序，所以也会产生间隙锁。

ps: 查询不加锁的情况下情况下，rr隔离级别 MVCC和锁一起解决了幻读问题。



> spring增强

一:bean的生命周期

singleton时 当容器（ApplicationContext）创建时对象出生，容器销毁则对象消亡，单列对象的生命周期和容器相同

prototype时 当我们使用时才创建，对象使用时一直活着，当对象长时间不用且没有引用时等待GC回收。



二:ioc的流程(bean初始化)

无论是注解还是XML,最终都会使用`AbstractApplicationContext`中的refresh()方法这个方法完成了整个spring生命周期的初始化

简单形容下：

1.根据配置生成BeanDefinition

2.通过BeanFactoryPostProcessor后置处理器 扩展BeanFactory 对BeanDefinition中的bean 进行类似 配置文件的属性注入 这种操作（比如JDBC 从括号换成具体地址）及一些注解的处理比如@Bean等

3.实例化bean。

4.填充对象属性 （可能会出现循环依赖）， 判断是否实现aware接口（如果实现了会进行方法回调,可以在自定义对象获取一些系统属性比如beanFactory）

5.执行BeanPostProcessor后置处理器，重写postProcessBeforeInitialization（可以对实列化后的对象进行修改）和postProcessAfterInitialization（产生代理对象等操作）方法 before执行后  判断对象是否实现initializingBean的afterpropertiesSet 有的话执行bean的init-method,再执行after 



```Java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 准备，记录容器的启动时间startupDate, 标记容器为激活，初始化上下文环境如文件路径信息，验证必填属性是否填写
        prepareRefresh();
        // 获取新的beanFactory，销毁原有beanFactory、为每个bean生成BeanDefinition等
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        // 初始化beanfactory的各种属性
        prepareBeanFactory(beanFactory);
        try {
            // 模板方法，此时，所有的beanDefinition已经加载，但是还没有实例化。
            //允许在子类中对beanFactory进行扩展处理。比如添加aware相关接口自动装配设置，添加后置处理器等，是子类扩展prepareBeanFactory(beanFactory)的方法
            postProcessBeanFactory(beanFactory);
            // 实例化并调用所有注册的beanFactory后置处理器（实现接口BeanFactoryPostProcessor的bean，在beanFactory标准初始化之后执行）
            invokeBeanFactoryPostProcessors(beanFactory);
            // 实例化和注册beanFactory中扩展了BeanPostProcessor的bean
            //例如：
            // AutowiredAnnotationBeanPostProcessor(处理被@Autowired注解修饰的bean并注入)
            // RequiredAnnotationBeanPostProcessor(处理被@Required注解修饰的方法)
            // CommonAnnotationBeanPostProcessor(处理@PreDestroy、@PostConstruct、@Resource等多个注解的作用)等。
            registerBeanPostProcessors(beanFactory);
            // Initialize message source for this context.
            initMessageSource();
            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();
            // 模板方法，在容器刷新的时候可以自定义逻辑，不同的Spring容器做不同的事情。
            onRefresh();
            // 注册监听器，广播early application events
            registerListeners();
            // 实例化所有剩余的（非懒加载）单例
            // 比如invokeBeanFactoryPostProcessors方法中根据各种注解解析出来的类，在这个时候都会被初始化。
            // 实例化的过程各种BeanPostProcessor开始起作用。
            finishBeanFactoryInitialization(beanFactory);
            // refresh做完之后需要做的其他事情。
            // 清除上下文资源缓存（如扫描中的ASM元数据）
            // 初始化上下文的生命周期处理器，并刷新（找出Spring容器中实现了Lifecycle接口的bean并执行start()方法）。
            // 发布ContextRefreshedEvent事件告知对应的ApplicationListener进行响应的操作
            finishRefresh();
        } catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }
            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();
            // Reset 'active' flag.
            cancelRefresh(ex);
            // Propagate exception to caller.
            throw ex;
        } finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
```



三:循环依赖

spring如何解决循环依赖：

在refish()方法中 finishBeanFactoryInitialization生成单例对象会存到一个Map中被称为一级缓存。这里面的对象可以直接被用户使用

在对象初始化但是并没有赋值的时候也会存入一个Map，这里面的bean不完整，不能被用户世界使用，这是二级缓存。

当用户的对象需要代理生成，但是又不是立马需要生成，spring设计了三级依赖存放代理工厂，这个map称为三级缓存



A创建过程中需要B，于是**A将自己放到三级缓存**里面，去**实例化B*,*B实例化的时候发现需要A，于是B先查一级缓存，没有，再查二级缓存，还是没有，再查三级缓存，找到了A然后把三级缓存里面的这个**A放到二级缓存里面，并删除三级缓存里面的A，B顺利初始化完毕**，将自己放到一级缓存里面（**此时B里面的A依然是创建中状态**）然后回来接着创建A，此时B已经创建结束，直接从一级缓存里面拿到B，然后完成创建，并**将A放到一级缓存中。



spring解决不了我们如何解决：

首先考录修改结构消除循环依赖，不行的情况下可以使用@Lazy,setter注入的等方式使bean被懒加载，从而解决问题。



> 基础

一:线程池得参数，拒绝策略

```java
   public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
```

**corePoolSize** – 池中要保留的线程数，即使它们处于空闲状态，除非设置了 `allowCoreThreadTimeOut`

**maximumPoolSize**– 池中允许的最大线程数

**keepAliveTime** – 当线程数大于核心时，这是多余空闲线程在终止前等待新任务的最长时间。

**unit** – `keepAliveTime` 参数的时间单位

**workQueue** – 用于在执行任务之前保存任务的队列。这个队列将只保存 execute 方法提交的 Runnable 任务。

```java
ThreadFactory threadFactory
RejectedExecutionHandler handler
```

**一个是线程工厂，用来创建线程**，主要是用来给线程起名字

**还有一个是拒绝策略**，当线程池被耗尽或者队列满了的时候会调用,默认的实现是**AbortPolicy**,如果线程池队列满了丢掉这个任务并且抛出`RejectedExecutionException`异常。



二:lock的底层实现

lock接口的实现类有读写锁 ，可重入锁（ReentrantLock）等

lock接口的实现类核心同步器`sync` ,它是一个抽象静态内部类，继承`AbstractQueuedSynchronizer`抽象类即aqs



三: AQS(*抽象队列同步器*)和可重入锁

aqs 主要实现了共享锁和排他锁

aqs 维护了一个由volatile修饰的int字段 state，表示加锁的次数。

排他锁：

当一个线程过来时加锁时然后尝试获取锁，state初始时0，获取锁时则++。这个尝试方法由对应的lock实现类 的核心同步器`sync`实现。或如锁的方式如下。

```java
             * 1. If read count nonzero or write count nonzero
             *    and owner is a different thread, fail.
             * 2. If count would saturate, fail. (This can only
             *    happen if count is already nonzero.)
             * 3. Otherwise, this thread is eligible for lock if
             *    it is either a reentrant acquire or
             *    queue policy allows it. If so, update state
             *    and set owner.
```

翻译下：

1.如果state不为0或者线程所有者不是当前线程则失败

2.当计数器达到上限时则失败

3.如果是可重入或者队列允许的情况下该线程可以获取锁并更新状态设置所有者

没有获取到锁的线程将会被加到等待队列中。



共享锁：

state初始时是构造实现类时传入的值。获取锁时则--。

尝试获取资源复数表示失败，0表示成功，但是可能没有可用资源，正数表示成功且有可用资源

如果发现自己是第二个节点，会再次去尝试获得锁资源,没有获取到锁的线程则加入到等待队列中



可重入锁（ReentrantLock）为排他锁

当一个线程过来时加锁时会判断是否当前锁是否是当前线程，是的化还能获取锁state+1

这里还有公平锁还是非公平锁（lock实现类构造方法决定）的区分，非公平锁将不会判断等待队列里是否有值。



AQS中还提供了一个内部类ConditionObject，它实现了Condition接口，可以用于await/signal。采用CLH队列的算法，唤醒当前线程的下一个节点对应的线程，而signalAll唤醒所有线程。



三:JVM垃圾回收算法

有哪几种，工作原理。

JVM调优。



四：三次握手 四次挥手

握手：

pc1 向2 发送连接请求

2回复1 ，同时2向1发起链接请求

1收到2回复，向2回复确认信息



挥手：

pc2 向1 发送报文请求关闭

1接收到，处于半关闭状态并且返回2，2收到以后进入FIN-WAIT-2阶段

1过段时间后再次向客户端发出报文，此时1进入LAST-ACK阶段，停止向2发送信息，但是任然可以收到信息

2收到1的第二次信息后进入TIME-WAIT阶段并且发送1一段信息，并且在TIME-WAIT阶段等待2MSL，1收到后正式关闭，2等待完也关闭。



五: RSA加解密

RSA算法可以总结为四句话：**公钥加密、私钥解密、私钥签名、公钥验签**。

JWT 包含头部，载荷，签名三部分，签名需要用私钥签名， 公钥解析。载荷为明文

> redis

一: `redis`基础数据类型 string（字符串），hash（哈希），list（列表），set（集合）及sortset（有序集合）



二: `redis`的事物如何处理

- MULTI：用来组装一个事务；
- EXEC：用来执行一个事务；
- DISCARD：用来取消一个事务；
- WATCH：用来监视一些key，一旦这些key在事务执行之前被改变，则取消事务的执行

这些入队的命令会根据顺序在事务执行后，一个一个执行

`redis` 在事务失败时不进行回滚，而是继续执行余下的命令