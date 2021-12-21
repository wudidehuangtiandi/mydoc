# Tomcat解析

> 在学习javaweb的时候可能大家都学习到了Servlet，但是大家会发现servlet相关实现是由tomcat提供的，那么tomcat是如何封装请求和相应，前端http请求是如何转变成 Servlet 的请求和相应的参数的，相信大家都有疑问。

## 一.Tomcat的源码构建

> 想要解决以上问题，我们需要分析阅读tomcat的源码，首先我们通过github上的源码构建一个tomcat

[github地址](https://github.com/apache/tomcat)

注意，不通的版本需要的构建环境可能有所区别，在此只记录此时的版本及相应的环境,我这里所运用到的环境,对应的tomcat版本为10.0.1dev

idea 2021.3

jdk17

1.首先pull下整个项目,可见如下目录，默认会去使用`build.properties.default`,可以在项目目录下增加如图所示文件做自己的配置，根据官网描述，主要修改内容为`base.path=E:/antrepository/tomcat-build-libs`，本路径将保存构建过程中所需的依赖。使用idea打开后会idea会询问是否使用ant，允许后即出现右侧目录，tomcat是使用ant来完成项目的构建的。



![avatar](https://picture.zhanghong110.top/docsify/16397307035954.png)

2.构建所需的依赖,可在如下文件中查询，由下图可见tomcat10.0.1最少需要jdk11的支持

![avatar](https://picture.zhanghong110.top/docsify/16397310901375.png)

3.点击deploy即可打包出基本的tomcat。在build目录下包含了整个可以使用各种脚本运行的tomcat，如图所示，配置正确的JAVA_HOME,通过bin目录下的`start.bat`改下输出日志为GBK，就可以正确启动tomcat及打印日志了。

![avatar](https://picture.zhanghong110.top/docsify/16397313545559.png)

## 二.结构设计

### 1.目录结构

首先我们先要了解下各个包的作用，如下图所示

![avatar](https://picture.zhanghong110.top/docsify/16397877877962.png)

![avatar](https://picture.zhanghong110.top/docsify/1639787942973.png)

源码中有`jakarta`及`org`两个文件夹

> `jakarta`这个文件夹下面保存的是新的 Java EE 规范，现在的 Java EE 也不这么叫了，要改名叫 Jakarta EE。详见[这里](https://blogs.oracle.com/javamagazine/transition-from-java-ee-to-jakarta-ee)。



![avatar](https://picture.zhanghong110.top/docsify/16397882704110.png)

下面我们对源码文件夹的各个包做下说明

> jakarta

`annotation`这个模块的作用是定义了一些公用的注解，避免在不同的规范中定义相同的注解。

`ejb`是开发和部署基于组件的企业级应用的架构。EJB 是一个架构。

`el`就是JSP中的el表达式。

`mail`邮件相关。

`persistence`持久化相关

`security`安全相关

`servlet`这个是重点，tomcat 就是一个实现了 servlet 规范的一个容器。servlet 定义了服务端处理 Http 请求和响应的规范。

`transaction`事务相关

`websocket`定义了使用 websocket 协议的服务端和客户端 API

`xml`定义了基于 SOAP 协议的 xml 方式的 web 服务。

> org.apache

`catalina `tomcat 的核心代码，可以理解为一个 servlet 容器。

`coyote`tomcat 的核心代码，负责将网络请求转化后和 Catalina 进行通信。

`el`上面的 Jakarta EE 中 el 的实现。

`jasper`负责将JSP转换为java代码

`juli`日志相关实现，使得不通的web应用可以使用各自独立的配置文件

`naming`命名空间相关和JNDI服务

`tomcat`各种辅助工具，包括 websocket 的实现。



> tomcat的运行原理如下图所示，不过光看这张图可能也不太清楚，但是我们带着这张图去看下面的文档会清晰很多

![avatar](https://picture.zhanghong110.top/docsify/16397891766470.png) 





*当我们使用浏览器向某一个网站发起一个 HTTP 格式的请求，那么作为 HTTP 服务器接收到这个请求之后，会调用具体的程序（Java类）进行处理，往往不同的请求由不同的 Java 类完成处理。那么问题来了，HTTP 服务器怎么知道要调用哪个 Java 类的哪个方法呢。当然我们可以在 HTTP 服务器代码里直接进行处理，也就是根据请求路径等信息进行一堆的 if else 逻辑判断，但是这样做缺陷很明显，因为 HTTP 服务器的代码跟后端业务处理逻辑耦合在一起了，不符合我们软件开发设计的原则，并且也非常不利于系统的扩展性。*

*针对这种耦合问题最好的解决办法自然是面向接口编程，于是我们可以定义一个 Servlet 接口，所有的业务类都必须实现这个接口。到这里就完了吗？当然没有，因为我们还是没能解决 Servlet 定位问题，即一个请求到来时 HTTP 服务器如何知道该由哪个 Servlet 来处理呢，并且这些自定义的 Servlet  也需要进行加载和管理。于是 Servlet 容器就被发明出来了，Servlet 容器作为 HTTP 服务器和具体业务类进行交互的桥梁，HTTP 服务器将请求交由 Servlet 容器去处理，而 Servlet 容器则负责将请求转发到具体的 Servlet，并且调用 Servlet 的方法进行业务处理，它们之间的调用通过 Servlet 接口进行解耦。*

*其实 Servlet 接口和 Servlet 容器并不是 Tomcat 发明的，而是在 JAVAEE API 中定义的，我们也把这一整套内容叫做 Servlet 规范。有了这套规范之后，如果我们要实现新的业务功能，只需要实现一个 Servlet，并把它注册到  Servlet 容器中，剩下的事情就需要由 Tomcat 帮我们处理了。因此对于 Tomcat 而言就需要按照 Serlvet 规范的要求去实现一个 Servlet 容器。*



### 2.Tomcat Servlet 容器工作流程

当用户请求某个URL资源时：

1. HTTP 服务器会把请求信息使用 ServletRequest 对象封装起来
2. 进一步去调用 Servlet 容器中某个具体的 Servlet
3. 在第二步中当 Servlet 容器拿到请求后，会根据 URL 和 Servlet 的映射关系，找到相应的 Servlet
4. 如果 Servlet 还没有被加载，就使用反射机制创建这个 Servlet，并调用 Servlet 的 init 方法来完成初始化
5. 接着调用这个具体 Servlet 的 service 方法来处理请求，请求处理结果使用 ServletResponse 对象封装
6. 把 ServletResponse 对象返回给 HTTP 服务器，HTTP 服务器会把响应发送给客户端

![avatar](https://picture.zhanghong110.top/docsify/16397906523836.png)

### 3.Web 服务器

根据上述分析，我们知道了 Tomcat 要实现成 “HTTP 服务器 + Servlet 容器”，也就是所谓的 Web 服务器。
 而作为一个 Web 服务器，Tomcat 要实现两个非常核心的功能：

- **Http 服务器功能：**进行 Socket 通信(基于 TCP/IP)，解析 HTTP 报文

- **Servlet 容器功能：**加载和管理 Servlet，由 Servlet 具体负责处理 Request 请求

  

![avatar](https://picture.zhanghong110.top/docsify/16397907619446.png)

### 4.连接器和容器

为了完成上述两个功能，tomcat设计了两个核心组件连接器（Connector）和容器（Container）来分别做这两件事情。连接器负责对外交流（完成 Http 服务器功能），容器负责内部处理（完成 Servlet 容器功能）。

连接器既然负责对外交流那就免不了进行 socket 通信，说到 socket 通信，就涉及到了网络IO模型，那么网络IO模型是可变的、多种多样的，因此一个容器可能对接多个连接器，这就好比一个房间有多个门。但是单独的连接器或者容器都不能对外提供服务，需要把它们组装起来才能工作，组装后这个整体我们将其命名为 Service 组件。

Service 设计一个还是多个？考虑一下这种场景，当我们有2个或以上网站都需要能够部署和管理自己的应用并且彼此不会相互影响(端口隔离)，而此时又不想安装多个tomcat避免造成资源的浪费，那么就需要多个 Service 了。因此我们在一个 Tomcat 中让它可以配置多个 Service，这种设计也是出于系统的灵活性考虑，虽然现在大多数情况下我们不会用到。

至此我们画出了如下的架构图：

![avatar](https://picture.zhanghong110.top/docsify/16397909459504.png)

`Server`
 这里的 Server 就代表了一个 Tomcat 实例，包含了 Servlet 容器以及其他组件，负责组装并启动 Servlet 引擎、Tomcat 连接器

`Service`
 服务是 Server 内部的组件，一个Server包括多个Service。它将若干个 Connector 组件绑定到一个 Container

`Container`
 容器，负责处理用户的 servlet 请求，并返回对象给 web 用户的模块



> 这里还需注意的是连接器和容器两者之间是通过标准的 ServletRequest 和 ServletResponse 通信，这样连接器对 Servlet 容器就屏蔽了网络协议以及 I/O 模型等的区别



下面我们用两张图来看下连接器和容器的具体设计



**连接器的设计**：

![avatar](https://picture.zhanghong110.top/docsify/16397915719260.png)

连接器主要需要完成以下三个核心功能：

- socket 通信，也就是网络编程
- 解析处理应用层协议，封装成一个 Request 对象
- 将 Request 转换为 ServletRequest，将 Response 转换为 ServletResponse

tomcat设计了三个组件 EndPoint、Processor、Adapter 来对应完成上述三项功能。这三个组件之间通过抽象接口进行交互。从一个请求的正向流程来看， Endpoint 负责提供请求字节流给 Processor，Processor 负责提供 Tomcat 定义的 Request 对象给 Adapter，Adapter 负责提供标准的 ServletRequest 对象给 Servlet 容器。

首先来说一下 Adapter 组件，连接器需要对接的是标准的 Servlet 容器，既然是 Servlet 容器，那就应该遵循 Servlet 规范，也就是说在 Servlet 的 service方法中只能接收标准的 ServletRequest 对象和 ServletResponse对象，而连接器负责对外交流，只能将基础的请求信息封装成一个 Request 对象，这时候就需要一个转换器，遇到这种需求通常会采用适配器模式。因此tomcat设计了一个 CoyoteAdapter 类，并提供一个 service 方法供连接器调用，内部则调用容器的 service 方法。

然后是 EndPoint 组件 和 Processor 组件，这两者一个负责对接 I/O 模型，一个负责对接应用层协议，都是会变化的，并且可以自由组合。Tomcat 能够支持的 I/O 模型和应用层协议，如下图所示。

![avatar](https://picture.zhanghong110.top/docsify/16397919054009.png)



针对这样一个组合的场景可以这么来设计，首先这两者其实是可以作为一个整体的，最终目的就是将请求信息转为一个统一的 Request 对象，当然了还要负责响应信息的输出。因此tomcat设计一个 ProtocolHandler 的接口来封装这两种变化点。

除了这些变化点，这两个组件也存在一些相对稳定的部分或者是一些通用的处理逻辑，这些稳定的部分我们通常使用抽象类来封装，可以定义一个抽象基类 AbstractProtocol 让它实现 ProtocolHandler 接口，然后针对每一种应用层协议也定义一个自己的抽象类，它们继承自 AbstractProtocol 抽象基类，扩展了具体协议相关的内容。

最后tomcat针对具体的应用层协议和 I/O 模型组合定义具体的实现类，例如：Http11NioProtocol 就是对 HTTP/1.1 协议 和 NIO 模型的实现。它们的类关系图如下：

![avatar](https://picture.zhanghong110.top/docsify/16397921165372.png)







>在这里可能大家会有一个疑问，不是说 ProtocolHandler 是对 EndPoint 组件 和 Processor 组件的封装吗？为什么从源码中完全看不出来，很好的一个问题，有关于 EndPoint 组件和 Processor 组件的设计细节以及它们的交互过程我们会在源码部分给出答案。











**容器部分的设计:**

![avatar](https://picture.zhanghong110.top/docsify/16397916916459.png)

- Engine
   表示整个 Catalina 的 Servlet 引擎，用来管理多个虚拟站点，一个 Service 最多只能有一个 Engine，但是一个引擎可包含多个 Host
- Host
   代表一个虚拟主机，或者说一个站点，可以给 Tomcat 配置多个虚拟主机地址，而一个虚拟主机下可包含多个 Context
- Context
   表示一个 Web 应用程序，一个Web应用可包含多个 Wrapper
- Wrapper
   表示一个Servlet，负责管理整个 Servlet 的生命周期，包括装载、初始化、资源回收等

通过这种分层的架构设计，使得 Servlet 容器具有很好的灵活性，同时功能也更加强大。

### 5.配置文件与 Catalina 组件

我们可以看出为了实现两项功能，Tomcat 进行了很多的封装设计，封装出了很多的组件，而这些组件之间呈现出了明显的层级关系，一层套着一层，这就是经典的套娃式架构设计。其实这些组件的设计更多是为了使用者能够灵活的进行 web 项目部署配置，因此我们将其抽取成一个配置文件，名为 server.xml，如下图所示，在配置文件中你也能很清晰的对应上这些层级关系。



![avatar](https://picture.zhanghong110.top/docsify/16397928572123.png)



### 6.整体架构图

分层结构

![avatar](https://picture.zhanghong110.top/docsify/16397933554974.png)

总体架构

![avatar](https://picture.zhanghong110.top/docsify/16397934041889.png)

部分组件没有讲到，这里说明下

`Listener `组件
 可以在 Tomcat 生命周期中完成某些容器相关的监听器

`JNDI`
 JNDI是 Java 命名与目录接口，是属于 J2EE 规范的，Tomcat 对其进行了实现。JNDI 在 J2EE 中的角色就是“交换机”，即 J2EE 组件在运行时间接地查找其他组件、资源或服务的通用机制（你可以简单理解为给资源取个名字，再根据名字来找资源）

`Cluster `组件
 提供了集群功能，可以将对应容器需要共享的数据同步到集群中的其他 Tomcat 实例中

`Realm` 组件
 提供了容器级别的用户-密码-权限的数据对象，配合资源认证模块使用

`Loader `组件
 Web 应用加载器，用于加载 Web 应用的资源，它要保证不同 Web 应用之间的资源隔离

`Manager` 组件
 Servlet 映射器，它属于 Context 内部的路由映射器，只负责该 Context 容器的路由导航



## 三.源码解析

### 1、tomcat的初始化流程

> 首先既然tomcat是java构建的，那启动肯定是由Main方法引导。

![avatar](https://picture.zhanghong110.top/docsify/16397942104432.png)

从启动脚本中可以看出来，最后是去执行了catalina.bat

![avatar](https://picture.zhanghong110.top/docsify/16397944266473.png)

我们可以从这个脚本中找到启动文件

![avatar](https://picture.zhanghong110.top/docsify/16397948569468.png)

我们可以看到，首先它调用了自身的init方法

![avatar](https://picture.zhanghong110.top/docsify/16397954099433.png)

可以看到init中主要初始化了配置文件`conf/server.xml`,配置文件运行时才知道，所以使用了反射调用

![avatar](https://picture.zhanghong110.top/docsify/16397959946749.png)

回到main方法继续往下看我们可以发现，后面调用了load及start方法，这里有一句忘记提了main中初始化了自身。等配置文件初始化后会将自身赋值到daemon `daemon = bootstrap`;所以是调用本身的load及start。

![avatar](https://picture.zhanghong110.top/docsify/16397963211455.png)

我们先来看load方法，上图可见实际上调用的是`Catalina.load()` 方法

![avatar](https://picture.zhanghong110.top/docsify/16397966764167.png)

进入该方法，代码有点多，前面是读取和解析配置文件的过程，分别对该对象设置了Catalina 对象、catalinaHome 和 catalinaBase 属性，catalinahome 是咱们 tomcat 的安装目录，catalinabase 是工作目录，其实这两个都是我们在构建源码时配置的 tomcat 源码下的 source 目录，最后关键的一步是调用了 server 的 init 方法，即 server 组件的初始化。

![avatar](https://picture.zhanghong110.top/docsify/16397971094271.png)

进入init方法我们可见实际调用的是 `LifecycleBase `的 init 方法，该类是一个抽象基类，实现了 Lifecycle 接口，方法主要逻辑如上图

> 这里如果采用debug模式则可见实际上getserver返回的是一个 StandardServer 对象

下一步进入到了 LifecycleBase 的子类，也就是 StandardServer 的 initInternal() 方法，该方法除了对自身的一些初始化操作之外，最后进行了 service 组件的初始化。

![avatar](https://picture.zhanghong110.top/docsify/16397976466139.png)

> 至此,load方法的核心作用就体现出来了

我们进入init,方法发现又来到了 LifecycleBase 的 init 方法,下一步将来到 StandardService 的 initInternal() 方法，该方法的重点依然是对各个子组件的初始化，包括`engin`组件的初始化及`executor`组件的初始化。其实到这你会发现这些组件都有一个共性，就是都继承了 LifecycleBase 抽象类并实现了其中的抽象方法。

![avatar](https://picture.zhanghong110.top/docsify/16399785107275.png)

> 最后看一个 connector 组件的初始化

我们进入该类发现最后方法来到了Connector类的initInternal()方法，如下图所示，我们进入该方法,发现方法来到了AbstractHttp11Protocol类

![avatar](https://picture.zhanghong110.top/docsify/16399791355441.png)

可以看到最终调用了AbstractHttp11Protocol 父类的init方法。

![avatar](https://picture.zhanghong110.top/docsify/16399793174720.png)

我们进入父类的init方法，可以发现最终它完成了endpoint组件的初始化，我们看一下endpoint的初始化过程

![avatar](https://picture.zhanghong110.top/docsify/16399794549001.png)

进入AbstractEndpoint 的 init 方法，下图可见关键方法是AbstractEndpoint 的 init 方法进入里卖弄可以发现执行了bind()方法

![avatar](https://picture.zhanghong110.top/docsify/16399799797503.png)

进入 NioEndpoint 的 bind 方法，其实使用了 java `nio` 的 API 创建了相关的对象及对象初始化工作。关于NIO的相关内容可以参见后端NIO文档(等待建设) 如下图所示

![avatar](https://picture.zhanghong110.top/docsify/16399802604255.png)

> 到这里各个组件初始化的过程就分析完毕了

<hr/>

接下来我们回到

回到 Bootstrap 类的 main 方法，分析一下 start 方法的执行流程。该方法最终也是调用的 Catalina 类 的 start 方法。

![avatar](https://picture.zhanghong110.top/docsify/16399820411139.png)



![avatar](https://picture.zhanghong110.top/docsify/16399844854793.png)

之后调用的方法基本同load,除了实现的抽象方法不通为`startInternal()`,我们来看一下大体流程图

首先还是来到 LifecycleBase 的 start 方法，设置组件的状态，然后调用子类的 startInternal 方法。

![avatar](https://picture.zhanghong110.top/docsify/16399848983557.png)

来到 StandardServer 的 startInternal() 方法，该方法最后启动了 service 组件。

![avatar](https://picture.zhanghong110.top/docsify/16399844854793.png)

> 看到这后面的步骤大家都能知道，和 load 方法的执行流程一样，也是一个逐级启动的过程，我们最后看一下 endpoint 组件启动时完成的工作。来到 NioEndpoint 的 startInternal 方法。

![avatar](https://picture.zhanghong110.top/docsify/16399877308835.png)

Acceptor 线程是负责服务监听端口的请求接入的,我们可以看下 Accepter种复写的run方法是如何执行的

![avatar](https://picture.zhanghong110.top/docsify/16399886695184.png)



方法较长，我们截取一段

![avatar](https://picture.zhanghong110.top/docsify/16399902302549.png)

可以看到endpoint接收传入链接

> 我们用一张图来总结上述调用过程，分为两大部分，逐级初始化和逐级启动

![avatar](https://picture.zhanghong110.top/docsify/16399892513245.png)





*我们不难发现，整个tomcat启动过程都是围绕Lifecycle 接口定义的，这些组件通过实现init和start方法来实现初始化及启动。*

设计其实就是要找到系统的变化点和不变点，然后遵循设计原则，选择合适的设计模式进行具体的设计。

对于这些组件来说不变点就是每个组件的生命周期是一致的，即它们都要经历创建、初始化、启动、停止和销毁这几个过程，在这个过程中组件的状态和状态之间的转化也是不变的。而其中的变化点则是某个具体的组件在执行某个过程时是有所差异的。

因此针对这些组件的生命周期，我们抽取出一个接口，这个接口就是 Lifecycle。在该接口中，我们需要定义这么几个方法：init、start、stop 和 destroy，让每个组件去实现这些方法。

在 tomcat 中组件是有层级关系的，因此父组件需要在自己的生命周期方法比如 init 方法里创建子组件并且调用子组件的 init 方法，最终像一个链条一样层层调用下去，这其实就是设计模式中**组合模式**的经典实用。最终实现的效果就是在启动入口我们只需要调用最顶层组件的 init 方法 和 start 方法，整个 tomcat 就被启动起来了。

tomcat 作为一个框架，尤其是作为一个 Web 容器框架，监听机制和过滤机制是我们设计时必须要考虑的一个点，也就是我们通常说的监听器和过滤器。由于在这里并没有涉及到请求的处理，因此我们只需要考虑监听器的设计。

在启动阶段各个组件会涉及到生命周期方法的组合调用，因此我们可以把组件的生命周期定义成相应的状态，把状态的转变看作是一个事件。有了事件就需要有监听器，在监听器里可以实现一些内部逻辑，并且监听器也可以方便的添加和删除，这就是典型的**观察者模式**的应用。

在设计了 Lifecycle 接口之后，我们通常需要做这么一个考虑，在接口的这些众多方法中，不同实现类实现它们是否包含一些通用的逻辑或者流程，如果有我们就需要定义一个抽象基类来实现这部分共同的逻辑，而其中有差异的地方我们将其抽取成抽象方法，供子类实现。

因此我们可以定义一个抽象基类 LifecycleBase 让它实现 Lifecycle 接口并重写所有的方法，针对方法中的通用逻辑我们可以定义私有的方法自行实现，例如状态的转换和事件监听的处理我们可以定义了 setStateInternal 方法，而非通用逻辑定义相应的抽象方法交给具体子类去实现，例如子类的初始化逻辑定义抽象方法 initInternal，这就是典型的**模板设计模式**的使用，其中的抽象方法也称为模版方法。

//待续，后续准备剖析下tomcat的请求处理流程，这也是当时想看tomcat源码的初衷

部分内容及debug思路引用自[渃汐湲简书](https://www.jianshu.com/p/7c9401b85704)

### 2、tomcat的请求封装流程

> 这边插播下如何debug启动tomcat源码,需要把ant下载下来的依赖加入到Libs或者采用网上那种写pom.xml然后用maven导入的方式。这边要注意，要使用ANT中的ide-intellij才会把所有依赖拉下来,使用deploy只会拉下运行时依赖。

我们采用maven构建tomcat，步骤分为如下几步

1.去github拉下源码

2.在根目录下新建pom.xml，和自己的版本一致即可，内容为空。

3.新建lib,将使用ant拉下来的依赖全部放进去并且设置为libs，这边要注意，ant相关的依赖需要手动从maven仓库导入

4.在ContextConfig类下的configureStart 增加代码context.addServletContainerInitializer(new JasperInitializer(), null); 用来初始化jsp引擎

5.将启动类设置为bootstrap即可，启动后访问8080发现可以访问就成功了，如下图所示

![avatar](https://picture.zhanghong110.top/docsify/16400777952313.png)

*debug启动后我们来分析下tomcat请求封装的过程*

### 