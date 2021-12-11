# log4j2 CVE-2021-44228 漏洞分析

*2021年11月24日，阿里云团队向Apache官方报告了 Log4j2远程代码执行漏洞，该漏洞又于2021年12月7日首次在野外发现，同日，阿帕奇发布了2.15.0-rc1版本以修复该漏洞，同月10日，该漏洞补丁被证实毫无卵用，于是当日，Apache又发布了2.15.0-rc2用以修复该漏洞。*

## 漏洞描述

Apache Log4j2是一个基于Java的日志记录工具，该工具重写了Log4j框架，并且引入了大量丰富的特性，Apache log4j-2是Log4j的升级版，这个日志框架被大量用于业务系统开发，用来记录日志信息。在大多数情况下，开发者可能会将用户输入导致的错误信息写入日志中，而攻击者则可以利用此特性通过该漏洞构造特殊的数据请求包，最终触发远程代码执行。



## 漏洞分析

许多人看到刷屏的报告可能纷纷意识到事情的严重性，毕竟该日志依赖广泛运用于各个框架，影响比较广泛，修复的方式也比较简单，除了升级以外对于旧的版本也可以通过修改Java程序的启动参数：`-Dlog4j2.formatMsgNoLookups=true`;或修改Log4j2的配置项`log4j2.formatMsgNoLookups=true`;也可将系统环境变量 `FORMAT_MESSAGES_PATTERN_DISABLE_LOOKUPS `设置为 `true`来避免触发。另外使用新版本`11.0+`的JDK;老版本JDK建议更新为`8u191`、`7u201`或`6u211`，可以在一定程度上限制JNDI等漏洞利用方式。

> 可能网上说了一大堆，大多数人也一知半解，不明白这个漏洞的触发机制及原理，那么我们就对该漏洞进行简单的演示及剖析

既然是漏洞，运行条件应当时比较苛刻的，我们此时先设定以下我们的运行环境

jdk采用 1.8.0_241

log4j2使用 2.14.1

建立如下两个工程

![avatar](https://picture.zhanghong110.top/docsify/16392220217028.png)

![avatar](https://picture.zhanghong110.top/docsify/16392221395586.png)

第一个工程我们用来演示LOG4J的模拟日志写入，第二个工程则为`rmi`的注册中心



我们首先不分析代码，作几个演示，演示该漏洞的该如何被利用

### 演示一

1.首先，我们在第一个项目中导入如下两个依赖

![avatar](https://picture.zhanghong110.top/docsify/16392079449077.png)

2.以下便是我们经常使用的日志打印方式，可见正常情况下打印出hello world没有问题

![avatar](https://picture.zhanghong110.top/docsify/16392223901734.png)

3.但是当我们注入特定的值就会发现，日志没有按照我们所想的输出而是将参数注入了日志中。

![avatar](https://picture.zhanghong110.top/docsify/16392225252772.png)

> 有的同学就说了，这个貌似也没啥大问题，最多就是日志打出来可能乱七八糟，反正我们平时的日志也瞎几把打，但是由此我们可以引入更加严重的问题



### 演示二

1.首先我们启动一个rmi注册中心，用来代理我们的攻击者服务，如下图所示。

![avatar](https://picture.zhanghong110.top/docsify/16392229211787.png)

2.此时我们模拟一个攻击者代码入下图所示

![avatar](https://picture.zhanghong110.top/docsify/1639223078869.png)

3.然后我们启动一个nginx来代理这个攻击类所编译好的字节码文件，监听80端口如下图所示

![avatar](https://picture.zhanghong110.top/docsify/16392232618414.png)

*至此我们完成了攻击前的准备步骤*



4.这时候我们修改下log4j2的代码,我们发现，并没有卵用呀，好像啥都没发生，是不是翻车了呢，其实这是应为我的jdk版本问题，1.8.0_241默认的`com.sun.jndi.rmi.object.trustURLCodebase`及`com.sun.jndi.ldap.object.trustURLCodebase`是`false`。而低版本的jdk则没有考虑这个问题。

![avatar](https://picture.zhanghong110.top/docsify/16392235257493.png)



5.我们模拟下低版本加入这两个选项,可以看到，静态攻击类中的代码成功被目标服务器执行，攻击者可以很顺利的操控该服务器完成任何可以做到的事，由此可见这个漏洞的严重性。

![avatar](https://picture.zhanghong110.top/docsify/16392238156156.png)



### 原因剖析

那么通过以上的两个演示，我们可以得出以下结论

只有较老版本的JDK才会触发该问题，而对于1.8及以上版本则只会出现轻微症状，即日志被写入不明代码。

那么有同学就要问了，为什么log4j2会出现这样的漏洞？

下面我们就来解析下log4j2的源码,在此之前,我们先要了解下两个概念,`jndi`与`rmi`，这两个技术相对而言比较年纪大了，可能现在的开发用的比较少。

`jndi`全称 Java Naming and Directory Interface,Java命名和目录接口，是[SUN公司](https://baike.baidu.com/item/SUN公司)提供的一种标准的Java命名系统接口，JNDI提供统一的[客户端](https://baike.baidu.com/item/客户端/101081)API，通过不同的访问提供者接口JNDI服务供应接口(SPI)的实现，由管理者将JNDI API映射为特定的命名服务和目录系统。这么讲可能会有点晕，我们可以设想下windows系统的注册表，两者比较类似。学习过`mybatis`的朋友应该知道，在数据源注入的时候有一个选项为`jndi`注入。所以简单来说它就是为了更加去耦合及更加易维护易拆分而诞生的产物。

`rmi` Remote Method Invocation,是一种实现RPC的JAVA API，大家所熟知的`dubbo`就是它进化后的产物,简单来说它运行于两个JVM之间，无法实现集群功能。大致的流程就是服务方将服务注册到注册中心，服务方调用后生成代理对象，然后去执行这个代理对象。

我们下面进入`org.apache.logging.log4j.core.lookup.JndiLookup`,可见如下代码

```
   /**
     * Looks up the value of the JNDI resource.
     * @param event The current LogEvent (is ignored by this StrLookup).
     * @param key  the JNDI resource name to be looked up, may be null
     * @return The String value of the JNDI resource.
     */
    @Override
    public String lookup(final LogEvent event, final String key) {
        if (key == null) {
            return null;
        }
        final String jndiName = convertJndiName(key);
        try (final JndiManager jndiManager = JndiManager.getDefaultManager()) {
            return Objects.toString(jndiManager.lookup(jndiName), null);
        } catch (final NamingException e) {
            LOGGER.warn(LOOKUP, "Error looking up JNDI resource [{}].", jndiName, e);
            return null;
        }
    }
```

该类未对传入字符串做出任何处理导致JNDI注入,官方现已经修复具体修复方式可见以下链接

[漏洞修复链接](https://github.com/apache/logging-log4j2/pull/607)

我们可以看到标题未no longer formats lookups in messages by default，字面意思可知现在默认情况下将不对传入的信息进行处理，有兴趣的朋友可以去看下如何修改的，也就改了几十行不是很多。不过造成的影响还是很大的。



