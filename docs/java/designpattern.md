# 常用设计模式

## 单例模式



>单例模式（Singleton Pattern）是 Java 中最简单的设计模式之一。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。
>
>这种模式涉及到一个单一的类，该类负责创建自己的对象，同时确保只有单个对象被创建。这个类提供了一种访问其唯一的对象的方式，可以直接访问，不需要实例化该类的对象。



**意图：**保证一个类仅有一个实例，并提供一个访问它的全局访问点。

**主要解决：**一个全局使用的类频繁地创建与销毁。

**何时使用：**当您想控制实例数目，节省系统资源的时候。

**如何解决：**判断系统是否已经有这个单例，如果有则返回，如果没有则创建。

**关键代码：**构造函数是私有的。



单例模式的实现有多种方式，总体可分为懒汉式和饿汉式（就是该类被初始化的时候是否就会创建对象，不是的为懒汉是的为饿汉），其中懒汉式又可分为线程安全的和线程不安全的。还有一些其它更为高效的实现。



首先我们看下调用的代码,可以看到目的是获取`Singleton`对象

```java
public static void main(String[] args) {
    Singleton instance = Singleton.getInstance();
}
```

> 下面我们用代码来演示下各种实现

首先是 懒汉式线程不安全,这种方式是最基本的实现方式，这种实现最大的问题就是不支持多线程。因为没有加锁 synchronized，所以严格意义上它并不算单例模式。这种方式 lazy loading 很明显，不要求线程安全，在多线程不能正常工作。（多个线程会获取到不同的对象）

```java
public class Singleton {

    //成员变量(最终需要获取的对象)
    private static Singleton instance;

    //构造
    private Singleton(){}

    //获取方法
    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }

}
```

下面是线程安全的懒汉式,可以看到唯一的区别就是获取的方法加了个锁，从而保证多线程下也能获取到单一的对象，不过其效率十分低下

```java
public class Singleton {

    //成员变量(最终需要获取的对象)
    private static Singleton instance;

    //构造
    private Singleton(){}

    //获取方法加锁
    public static synchronized Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

下面是饿汉式，可以看到，类被装载时就会初始化对象，它基于 `classloader` 机制避免了多线程的同步问题,虽然导致类装载的原因有很多种，在单例模式中大多数都是调用 `getInstance` 方法,这时候初始化 `instance` 显然没有达到 `lazy loading` 的效果。

```java
public class Singleton {
    //成员变量在类初始化时对象就产生
    private static Singleton instance = new Singleton();

    //构造
    private Singleton(){}

    //获取方法
    public static Singleton getInstance() {
        return instance;
    }
    
}
```

> 下面我们来看一下其它几种实现

首先是双检锁（DCL，Double Check Lock），必须在jdk1.5之后才能实现，线程安全，懒加载，且在多线程下能保持高性能，缺点是实现较为复杂

```java
public class Singleton {
    
    //(2)
    private volatile static Singleton instance;

    //构造
    private Singleton (){}

    //获取方法
    public static Singleton getInstance() {
        //(1)
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

这个里面涉及到两个关键点，第一是以上的(1)处，可以看到由于做了在外层先做了 if (instance == null)的判断，如果对象不为空会直接返回，而此处是非同步的，也就是说，大部分请求都不会再进入阻塞代码块。这样看起来十分完美，既减少了阻塞，又避免了竞态条件。不错，但实际上仍然存在一个问题——**当instance不为null时，仍可能指向一个`"被部分初始化的对象"`**。这里就要说到(2)的地方关键词`volatile`的作用。问题实际上出在简单的赋值语句instance = **new** **Singleton**();上。它并不是一个原子操作。事实上，它可以”抽象“为下面几条JVM指令：

```
memory = allocate();    //1：分配对象的内存空间
initInstance(memory);   //2：初始化对象
instance = memory;      //3：设置instance指向刚分配的内存地址
```

上面*操作2依赖于操作1，但是操作3并不依赖于操作2*，所以JVM可以以“优化”为目的对它们进行`重排序`，经过重排序后如下：

```
memory = allocate();    //1：分配对象的内存空间
instance = memory;      //3：设置instance指向刚分配的内存地址（此时对象还未初始化）
ctorInstance(memory);   //2：初始化对象
```

可以看到指令重排之后，操作 3 排在了操作 2 之前，即**引用instance指向内存memory时，这段崭新的内存还没有初始化**——即，引用instance指向了一个"被部分初始化的对象"。此时，如果另一个线程调用`getInstance`方法，*由于instance已经指向了一块内存空间，从而if条件判为false，方法返回instance引用*，用户得到了没有完成初始化的“半个”单例。

此时要解决这个问题只需要将instance声明为volatile变量,即可防止指令重排序

```java
private static volatile Singleton instance;
```

至于如何防止重排序这边涉及比较深入，主要是使用了内存屏障，而`volatile`关键词的作用也不仅限于此，以后将会出个文章解析，可以先看以下两个链接

https://www.cnblogs.com/dolphin0520/p/3920373.html

https://zhuanlan.zhihu.com/p/138819184

!>需要注意的是，**只有在Happens-Before内存模型中才会出现这样的指令重排序问题**，所以synchronized代码块内部，即instance = new Singleton();之后的代码如果调用该对象，虽然也有重排序问题，但重排序之后的所有指令，仍然能够保证在`instance.toString()`之前执行。

下面是静态内部类的实现，可以看到和饿汉式唯一的区别在于，采用静态内部类来获取对象，静态内部类只有当`getInstance`时才会被加载，所以显然是懒加载，且应为`jvm`在初始化类时会去加锁所以又是线程安全的。

```java
public class Singleton {

    private static class SingletonHolder {
        private static final Singleton INSTANCE = new Singleton();
    }

    //构造
    private Singleton (){}

    //获取方法
    public static final Singleton getInstance() {
        return SingletonHolder.INSTANCE;
    }

}
```

最后一种实现，枚举的实现，枚举不能实现懒加载但是是线程绝对安全的。

```java
public enum Singleton {  
    INSTANCE;  
    public void whateverMethod() {  
    }  
}
```



> 一般来说，使用单例模式不建议使用第一和第二种懒汉的方式，建议采用第三种饿汉的方式，如果有明确的懒加载需求可以使用静态内部类的方式。





## 



单例模式，工厂模式，代理模式，建造者模式，适配器模式