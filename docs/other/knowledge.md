# 知识概念整理

# 1.java基础

> 抽象类和接口的区别

1.最大的区别就是使用场景不同，抽象类用于抽象一种具体的事物，即对类抽象，而接口是对行为的抽象。

举个例子，比如鸟和飞机，它们都会飞，这时候鸟和飞机就可以被定义成抽象类，而飞则是定义成接口。这样不同的鸟继承鸟的抽象类，而鸟的抽象类和飞机的抽象类都实现飞这个接口，而它们飞的方式不相同，当然还可以实现其它不同的接口。

2.接口可以支持多实现，而抽象类只能单继承。

3.其它都是小区别不重要。



> java中的异常分类及常见异常

所有异常都是`Throwable`的子类，下面分为`Error`和`Exception`。

`Error`下有众多的无法被处理的异常例如`OutOfMemoryError`,`IOError`等，而`Exception`下则分为运行时异常`RuntimeException`和非运行时异常,如`ArrayIndexOutOfBoundsException`,`NullPointerException`等,非运行时异常(也叫编译异常)多的很，一大堆比如`IOException`,`SQLException`等等。

`Exception`下除了`RuntimeException`以外的都是受查异常。这种异常都发生在编译阶段，Java 编译器会强制程序去捕获此类异常。

非受查异常包括`Error`和`RuntimeException`。`RuntimeException`表示编译器不会检查是否对`RuntimeException`进行了处理。

![avatar](https://picture.zhanghong110.top/docsify/2019101117003396.png)



!>故而理论上在程序中不必去捕获`RuntimeException`异常。`RuntimeException`发生就说明程序出现了编程错误，应该找出错误去修改，而不是事先去捕获异常或者抛出，我们自己定义的运行时异常，抛出了，在调用的地方在捕获，挺搞笑的。然而实际上，我们还是经常会去捕获它们（笑哭）,比如做容错啊，或者记日志啊这些，可能还要保证发生了，程序还能运行下去等等的。

!>`Spring`的`AOP`即声明式事务管理默认是针对`unchecked exception`回滚。也就是默认对`RuntimeException()`异常或是其子类进行事务回滚。`checked`异常,即`Exception`可`try{}`捕获的不会回滚，因此对于我们自定义异常，通过`rollbackFor`进行设定。



>JAVA集合

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

​     3.`hashmap`

​       3.1 `jdk1.7`及之前是数组+链表得结构，之后是数组+链表+红黑树(节点达到阈值`TREEIFY_THRESHOLD`,默认是`8`)，当哈希冲突时将会被添加到相同节点中。

​       3.2 `hashmap`线程不安全，`hashtable`线程安全。

​       3.3 默认初始大小为`16`,默认负载因子是`0.75`，当元素达到数量时默认扩容两倍

​       3.4 关于扩容机制

```
2的幂次方：hashmap在确定元素落在数组的位置的时候，计算方法是(n - 1) & hash，n为数组长度也就是初始容量 。hashmap结构是数组，每个数组里面的结构是node（链表或红黑树），正常情况下，如果你想放数据到不同的位置，肯定会想到取余数确定放在那个数组里，计算公式：hash % n，这个是十进制计算。在计算机中， (n - 1) & hash，当n为2次幂时，会满足一个公式：(n - 1) & hash = hash % n，计算更加高效。
奇数不行：在计算hash的时候，确定落在数组的位置的时候，计算方法是(n - 1) & hash，奇数n-1为偶数，偶数2进制的结尾都是0，经过hash值&运算后末尾都是0，那么0001，0011，0101，1001，1011，0111，1101这几个位置永远都不能存放元素了，空间浪费相当大，更糟的是这种情况中，数组可以使用的位置比数组长度小了很多，这样就会造成空间的浪费而且会增加hash冲突。
```

....发现内容太多了。深入进去不是一时半活搞得定的，待续吧



​      



   