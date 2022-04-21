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



## 工厂模式

>工厂模式（Factory Pattern）是 Java 中最常用的设计模式之一。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。
>
>在工厂模式中，我们在创建对象时不会对客户端暴露创建逻辑，并且是通过使用一个共同的接口来指向新创建的对象。

**意图：**定义一个创建对象的接口，让其子类自己决定实例化哪一个工厂类，工厂模式使其创建过程延迟到子类进行。

**主要解决：**主要解决接口选择的问题。

**何时使用：**我们明确地计划不同条件下创建不同实例时。

**如何解决：**让其子类实现工厂接口，返回的也是一个抽象的产品。

**关键代码：**创建过程在其子类执行。

**应用实例：** 1、您需要一辆汽车，可以直接从工厂里面提货，而不用去管这辆汽车是怎么做出来的，以及这个汽车里面的具体实现。 2、Hibernate 换数据库只需换方言和驱动就可以。

我们来对工厂模式做一个简单的实现

我们创建一个接口，三个实现类（一个动物接口，三个小动物，他们都有个功能，会叫）

```Java
public interface Animal {
    void call();
}


public class Cat implements Animal{
    @Override
    public void call() {
        System.out.println("喵");
    }
}

public class Dog implements Animal{
    @Override
    public void call() {
        System.out.println("汪");
    }
}

public class Pig implements Animal{
    @Override
    public void call() {
        System.out.println("哼唧哼唧");
    }
}
```

比如上面这样，这时候我们可以搞一个动物工厂，里面有个getAnimal方法，专门对外提供小动物

```Java
public class AnimalFactory {
    public Animal getAnimal(String type){
        if(type == null){
            return null;
        }
        if(type.equalsIgnoreCase("CAT")){
            return new Cat();
        }else if(type.equalsIgnoreCase("DOG")){
            return new Dog();
        }else if(type.equalsIgnoreCase("Pig")){
            return new Pig();
        }
        return null;
    }
}
```

这时候有个老头想养一只小动物，他就可以调用这个工厂

```JAVA
public class OldMan {
    public static void main(String[] args) {
        AnimalFactory animalFactory = new AnimalFactory();
        //这时候他就拥有了一只小猪
        Animal pig = animalFactory.getAnimal("PIG");
        //然后他从没听过猪叫,想让小猪叫唤两声，于是调用call方法，就可以发出 哼唧哼唧 的声音了
        pig.call();
    }
}
```

这个工厂不仅可以提供给老头，要是有别人想养小猫，也是一样的

> 做个总结的话就是，你想要的只要问工厂要就好了，你无需关心你要的东西是怎么产生的。典型应用就是mybatis或者hibernate这种切换数据库的时候，只需要换驱动即可。还有redisTemplate啊，这种可以接入不同的连接池，都使用了工厂模式



## 抽象工厂模式

>抽象工厂模式（Abstract Factory Pattern）是围绕一个超级工厂创建其他工厂。该超级工厂又称为其他工厂的工厂。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。
>
>在抽象工厂模式中，接口是负责创建一个相关对象的工厂，不需要显式指定它们的类。每个生成的工厂都能按照工厂模式提供对象。

**意图：**提供一个创建一系列相关或相互依赖对象的接口，而无需指定它们具体的类。

**主要解决：**主要解决接口选择的问题。

**何时使用：**系统的产品有多于一个的产品族，而系统只消费其中某一族的产品。

**如何解决：**在一个产品族里面，定义多个产品。

**关键代码：**在一个工厂里聚合多个同类产品。

**应用实例：**工作了，为了参加一些聚会，肯定有两套或多套衣服吧，比如说有商务装（成套，一系列具体产品）、时尚装（成套，一系列具体产品），甚至对于一个家庭来说，可能有商务女装、商务男装、时尚女装、时尚男装，这些也都是成套的，即一系列具体产品。假设一种情况（现实中是不存在的，要不然，没法进入共产主义了，但有利于说明抽象工厂模式），在您的家中，某一个衣柜（具体工厂）只能存放某一种这样的衣服（成套，一系列具体产品），每次拿这种成套的衣服时也自然要从这个衣柜中取出了。用 OOP 的思想去理解，所有的衣柜（具体工厂）都是衣柜类的（抽象工厂）某一个，而每一件成套的衣服又包括具体的上衣（某一具体产品），裤子（某一具体产品），这些具体的上衣其实也都是上衣（抽象产品），具体的裤子也都是裤子（另一个抽象产品）。

抽象工厂的话，其实就是将诸多工厂抽离成一个抽象工厂，可以自行选择使用哪个工厂

我们来做一下简单的实现

这里我们借用本工厂模式的几个类，加入这时候这个老头不但想养小动物，还想去买一辆新车，体验一下速度与激情

我们增加一个接口，三个实现类

```java
public interface Car {
    //油门到底
    void Hong();
}

//卡罗拉
public class Corolla implements Car {
    @Override
    public void Hong() {
        System.out.println("时速60");
    }
}

//大宝马
public class Bmw implements Car {
    @Override
    public void Hong() {
        System.out.println("时速100");
    }
}

//布加迪
public class Bugatti implements Car{
    @Override
    public void Hong() {
        System.out.println("时速400");
    }
}
```

这时候我们搞一个工厂，啥车都能造的那种

```Java
public class CarFactory extends AbstractFactory{

    @Override
    public Animal getAnimal(String animal) {
        return null;
    }

    @Override
    public Car getCar(String carType){
        if(carType==null){
            return null;
        }
        if(carType.equalsIgnoreCase("COROLLA")){
            return new Corolla();
        }else if(carType.equalsIgnoreCase("BMW")){
            return new Bmw();
        }else if(carType.equalsIgnoreCase("BUGATTI")){
            return new Bugatti();
        }
        return null;
    }
```

对于上面那个宠物制造厂我们也改造下

```Java
public class AnimalFactory extends AbstractFactory {
    public Animal getAnimal(String type){
        if(type == null){
            return null;
        }
        if(type.equalsIgnoreCase("CAT")){
            return new Cat();
        }else if(type.equalsIgnoreCase("DOG")){
            return new Dog();
        }else if(type.equalsIgnoreCase("Pig")){
            return new Pig();
        }
        return null;
    }

    @Override
    public Car getCar(String car) {
        return null;
    }
}
```

这时候，有个单位吊炸天，专门为老年人提供各种业务，它拥有造车厂，还有宠物制造厂。

```java
public abstract class AbstractFactory {
    public abstract Animal getAnimal(String animal);
    public abstract Car getCar(String car);
}
```

但是由于经济不景气，这些工厂并不是一直开着的，老头需要什么就开什么，如下所示

```java
public class FactoryProducer {
    public static AbstractFactory getFactory(String choice){
        if(choice.equalsIgnoreCase("CAR")){
            return new CarFactory();
        } else if(choice.equalsIgnoreCase("ANIMAL")){
            return new AnimalFactory();
        }
        return null;
    }
}
```

这时候老头来了，他突然觉得不想养猪了，想开布加迪

```java
public class OldMan {
    public static void main(String[] args) {
        //高速公司需求，公司激活汽车生产线
        AbstractFactory car = FactoryProducer.getFactory("CAR");
        //根据老头的需求生产布加迪
        Car bugatti = car.getCar("bugatti");
        //轰油门，输出时速400，老头爽到了
        bugatti.Hong();
    }
}
```

> 可见抽象工厂就是将工厂类进行抽离，根部不同的内容激活不同的工厂来生产不同的东西。



## 代理模式

> 代理模式作为java开发应该非常熟悉了，用于对类进行增强或者变更，可以用一个内存中产生的类代表另一个类的功能。

**意图：**为其他对象提供一种代理以控制对这个对象的访问。

**主要解决：**在直接访问对象时带来的问题，比如说：要访问的对象在远程的机器上。在面向对象系统中，有些对象由于某些原因（比如对象创建开销很大，或者某些操作需要安全控制，或者需要进程外的访问），直接访问会给使用者或者系统结构带来很多麻烦，我们可以在访问此对象时加上一个对此对象的访问层。

**何时使用：**想在访问一个类时做一些控制。

**如何解决：**增加中间层。

**关键代码：**实现与被代理类组合。

**应用实例：** spring aop。



待续




单例模式，工厂模式，代理模式，建造者模式，适配器模式