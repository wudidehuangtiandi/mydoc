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

