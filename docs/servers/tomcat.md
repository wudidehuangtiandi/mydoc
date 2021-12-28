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

下面终于进行到`Connector`的分析阶段了，这也是Tomcat里面最复杂的一块功能了。`Connector`中文名为`连接器`，既然是连接器，它肯定会连接某些东西，连接些什么呢？`Connector`用于接受请求并将请求封装成Request和Response，然后交给`Container`进行处理，`Container`处理完之后再交给`Connector`返回给客户端。

要理解`Connector`，我们需要问自己4个问题。

- （1）`Connector`如何接受请求的？
- （2）如何将请求封装成Request和Response的？
- （3）封装完之后的Request和Response如何交给`Container`进行处理的？
- （4）`Container`处理完之后如何交给`Connector`并返回给客户端的？



先上一张图

![avatar](https://picture.zhanghong110.top/docsify/5bf1bf5c1e4663382700d3f42eabfa96_1168971-20190805194836476-1284200425.png)

可以看到`ProtocolHandler`的类继承层级，ajp和http11是两种不同的协议nio和nio2是不通的通讯方式，协议和通讯方式可以相互结合

![avatar](https://picture.zhanghong110.top/docsify/16406587679869.png)

`ProtocolHandler`包含三个部件：`Endpoint`、`Processor`、`Adapter`。

在分析之前我们先来看结论，以便更好地理解源码。

1. `Endpoint`用来处理底层Socket的网络连接，`Processor`用于将`Endpoint`接收到的Socket封装成Request，`Adapter`用于将Request交给Container进行具体的处理。
2. `Endpoint`由于是处理底层的Socket网络连接，因此`Endpoint`是用来实现`TCP/IP协议`的，而`Processor`用来实现`HTTP协议`的，`Adapter`将请求适配到Servlet容器进行具体的处理。
3. `Endpoint`的抽象实现类AbstractEndpoint里面定义了`Acceptor`和`AsyncTimeout`两个内部类和一个`Handler接口`。`Acceptor`用于监听请求，`AsyncTimeout`用于检查异步Request的超时，`Handler`用于处理接收到的Socket，在内部调用`Processor`进行处理。
4. 在我们分析完源码后就明白了

我们在`Service`标准实现`StandardService`的源码中发现，其`init()`、`start()`、`stop()`和`destroy()`方法分别会对Connectors的同名方法进行调用。而一个`Service`对应着多个`Connector`。

![avatar](https://picture.zhanghong110.top/docsify/16406602414068.png)



![avatar](https://picture.zhanghong110.top/docsify/16406603116234.png)

![avatar](https://picture.zhanghong110.top/docsify/16406607282947.png)

在分析之前，我们看看`server.xml`,在这个文件中，我们看到一个`Connector`有几个关键属性，`port`和`protocol`是其中的两个。`server.xml`默认支持两种协议：`HTTP/1.1`和`AJP/1.3`。其中`HTTP/1.1`用于支持http1.1协议，而`AJP/1.3`用于支持对apache服务器的通信。

> 截图太麻烦了，下面我们采用代码粘贴的方式展示

我们来看下Connector的构造方法，有如下三个构造方法

```
   public Connector() {
        this("HTTP/1.1");  默认无参构造会传入HTTP/1.1
    }


    public Connector(String protocol) {
        configuredProtocol = protocol;
        ProtocolHandler p = null;
        try {
            p = ProtocolHandler.create(protocol);
        } catch (Exception e) {
            log.error(sm.getString(
                    "coyoteConnector.protocolHandlerInstantiationFailed"), e);
        }
        if (p != null) {
            protocolHandler = p;
            protocolHandlerClassName = protocolHandler.getClass().getName();
        } else {
            protocolHandler = null;
            protocolHandlerClassName = protocol;
        }
        // Default for Connector depends on this system property
        setThrowOnFailure(Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE"));
    }


    public Connector(ProtocolHandler protocolHandler) {
        protocolHandlerClassName = protocolHandler.getClass().getName();
        configuredProtocol = protocolHandlerClassName;
        this.protocolHandler = protocolHandler;
        // Default for Connector depends on this system property
        setThrowOnFailure(Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE"));
    }

```

由上述代码可知，当调用无参构造时会调用第二个构造方法，其核心create方法如下

```
 public static ProtocolHandler create(String protocol)
            throws ClassNotFoundException, InstantiationException, IllegalAccessException,
            IllegalArgumentException, InvocationTargetException, NoSuchMethodException, SecurityException {
        if (protocol == null || "HTTP/1.1".equals(protocol)
                || org.apache.coyote.http11.Http11NioProtocol.class.getName().equals(protocol)) {
            return new org.apache.coyote.http11.Http11NioProtocol();
        } else if ("AJP/1.3".equals(protocol)
                || org.apache.coyote.ajp.AjpNioProtocol.class.getName().equals(protocol)) {
            return new org.apache.coyote.ajp.AjpNioProtocol();
        } else {
            // Instantiate protocol handler
            Class<?> clazz = Class.forName(protocol);
            return (ProtocolHandler) clazz.getConstructor().newInstance();
        }
    }
```

可知当protocol为空或者是HTTP/1.1时会使用`Http11NioProtocol`当protocol是AJP/1.3时则是`AjpNioProtocol`都不是的情况下则是反射调用类名。

下面我们来分析下Connector.init()方法

```
@Override
    protected void initInternal() throws LifecycleException {

        super.initInternal();

        if (protocolHandler == null) {
            throw new LifecycleException(
                    sm.getString("coyoteConnector.protocolHandlerInstantiationFailed"));
        }

        // Initialize adapter
        adapter = new CoyoteAdapter(this);
        protocolHandler.setAdapter(adapter);
        if (service != null) {
            protocolHandler.setUtilityExecutor(service.getServer().getUtilityExecutor());
        }

        // Make sure parseBodyMethodsSet has a default
        if (null == parseBodyMethodsSet) {
            setParseBodyMethods(getParseBodyMethods());
        }

        if (AprStatus.isAprAvailable() && AprStatus.getUseOpenSSL() &&
                protocolHandler instanceof AbstractHttp11JsseProtocol) {
            AbstractHttp11JsseProtocol<?> jsseProtocolHandler =
                    (AbstractHttp11JsseProtocol<?>) protocolHandler;
            if (jsseProtocolHandler.isSSLEnabled() &&
                    jsseProtocolHandler.getSslImplementationName() == null) {
                // OpenSSL is compatible with the JSSE configuration, so use it if APR is available
                jsseProtocolHandler.setSslImplementationName(OpenSSLImplementation.class.getName());
            }
        }

        try {
            protocolHandler.init();
        } catch (Exception e) {
            throw new LifecycleException(
                    sm.getString("coyoteConnector.protocolHandlerInitializationFailed"), e);
        }
    }
```

由上面的代码可知，其主要做了三件事，第一，初始化adapter，第二设置body的method列表，默认为POST，第三，初始化protocolHandler。

从`ProtocolHandler类继承层级`我们知道`ProtocolHandler`的子类都必须实现`AbstractProtocol`抽象类，而`protocolHandler.init();`的逻辑代码正是在这个抽象类里面。我们来分析一下。、

我们追踪到AbstractHttp11Protocol的init，发现它调用了父类的init，其代码如下

```
  @Override
    public void init() throws Exception {
        if (getLog().isInfoEnabled()) {
            getLog().info(sm.getString("abstractProtocolHandler.init", getName()));
            logPortOffset();
        }

        if (oname == null) {
            // Component not pre-registered so register it
            oname = createObjectName();
            if (oname != null) {
                Registry.getRegistry(null, null).registerComponent(this, oname, null);
            }
        }

        if (this.domain != null) {
            ObjectName rgOname = new ObjectName(domain + ":type=GlobalRequestProcessor,name=" + getName());
            this.rgOname = rgOname;
            Registry.getRegistry(null, null).registerComponent(
                    getHandler().getGlobal(), rgOname, null);
        }

        String endpointName = getName();
        endpoint.setName(endpointName.substring(1, endpointName.length()-1));
        endpoint.setDomain(domain);

        endpoint.init();
    }
```

我们由上述代码不难看出，核心是初始化了endpoint,我们进去  endpoint.init();发现该方法位于抽象类AbstractEndpoint，该类是基于模板方法模式实现的，主要调用了子类的`bindWithCleanup()`方法，里面直接执行了`bind()`方法。代码如下

```
public final void init() throws Exception {
        if (bindOnInit) {
            bindWithCleanup();
            bindState = BindState.BOUND_ON_INIT;
        }
        if (this.domain != null) {
            // Register endpoint (as ThreadPool - historical name)
            oname = new ObjectName(domain + ":type=ThreadPool,name=\"" + getName() + "\"");
            Registry.getRegistry(null, null).registerComponent(this, oname, null);

            ObjectName socketPropertiesOname = new ObjectName(domain +
                    ":type=SocketProperties,name=\"" + getName() + "\"");
            socketProperties.setObjectName(socketPropertiesOname);
            Registry.getRegistry(null, null).registerComponent(socketProperties, socketPropertiesOname, null);

            for (SSLHostConfig sslHostConfig : findSslHostConfigs()) {
                registerJmx(sslHostConfig);
            }
        }
    }
```

bind的实现由下面两个类提供，NIO2Endpoint，它跟NIOEndpoint的区别就是它实现了IO异步非阻塞。

![avatar](https://picture.zhanghong110.top/docsify/16406717289283.png)

我们进入NIO2Endpoint的bind方法代码如下

```
    @Override
    public void bind() throws Exception {

        // Create worker collection
        if (getExecutor() == null) {
            createExecutor();
        }
        if (getExecutor() instanceof ExecutorService) {
            threadGroup = AsynchronousChannelGroup.withThreadPool((ExecutorService) getExecutor());
        }
        // AsynchronousChannelGroup needs exclusive access to its executor service
        if (!internalExecutor) {
            log.warn(sm.getString("endpoint.nio2.exclusiveExecutor"));
        }

        serverSock = AsynchronousServerSocketChannel.open(threadGroup);
        socketProperties.setProperties(serverSock);
        InetSocketAddress addr = new InetSocketAddress(getAddress(), getPortWithOffset());
        serverSock.bind(addr, getAcceptCount());

        // Initialize SSL if needed
        initialiseSsl();
    }
```

我们可以看到核心代码serverSock.bind(addr, getAcceptCount());，用于绑定`ServerSocket`到指定的IP和端口。

接下来我们分析下Connector.start()方法

```
  @Override
    protected void startInternal() throws LifecycleException {

        // Validate settings before starting
        String id = (protocolHandler != null) ? protocolHandler.getId() : null;
        if (id == null && getPortWithOffset() < 0) {
            throw new LifecycleException(sm.getString(
                    "coyoteConnector.invalidPort", Integer.valueOf(getPortWithOffset())));
        }

        setState(LifecycleState.STARTING);

        try {
            protocolHandler.start();
        } catch (Exception e) {
            throw new LifecycleException(
                    sm.getString("coyoteConnector.protocolHandlerStartFailed"), e);
        }
    }
```

关键代码就一行protocolHandler.start();由抽象类AbstractAjpProtocol及AbstractProtocol提供实现，我们进入AbstractProtocol的start方法

```
@Override
public void start() throws Exception {
    if (getLog().isInfoEnabled()) {
        getLog().info(sm.getString("abstractProtocolHandler.start", getName()));
        logPortOffset();
    }

    endpoint.start();
    monitorFuture = getUtilityExecutor().scheduleWithFixedDelay(
            () -> {
                if (!isPaused()) {
                    startAsyncTimeout();
                }
            }, 0, 60, TimeUnit.SECONDS);
}
```

发现它调用endpoint.start(),我们进入后发现又回到了刚才那两个bind方法的实现。我们已经分析过了这边掠过

```
    public final void start() throws Exception {
        if (bindState == BindState.UNBOUND) {
            bindWithCleanup();
            bindState = BindState.BOUND_ON_START;
        }
        startInternal();
    }
```

之后进入startInternal，它也是由NIO2Endpoint及NIOEndpoint提供实现，我们进入NIO2Endpoint的startInternal方法如下所示

```
  @Override
    public void startInternal() throws Exception {

        if (!running) {
            allClosed = false;
            running = true;
            paused = false;

            if (socketProperties.getProcessorCache() != 0) {
                processorCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                        socketProperties.getProcessorCache());
            }
            int actualBufferPool =
                    socketProperties.getActualBufferPool(isSSLEnabled() ? getSniParseLimit() * 2 : 0);
            if (actualBufferPool != 0) {
                nioChannels = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                        actualBufferPool);
            }
            // Create worker collection
            if (getExecutor() == null) {
                createExecutor();
            }

            initializeConnectionLatch();
            startAcceptorThread();
        }
    }
```

发现它主要的任务如下

1.createExecutor 创建线程池

2.initializeConnectionLatch 初始化链接latch 用于限制请求的并发量

3.startAcceptorThread 开启acceptor线程

!>注意这里的区别由于进入的是nio2其过程与nio略有区别。nio会多个poller = new Poller();开启poller线程。poller用于对接受者线程生产的消息（或事件）进行处理，poller最终调用的是Handler的代码

> 到这里我们可以看到其实核心内容在初始化的流程中就有所涉及

> 分析完了`Connector`的启动逻辑之后，我们就需要进一步分析一下http的请求逻辑，当请求从客户端发起之后，需要经过哪些操作才能真正地得到执行？

我们用两张图来先大致了解下NIO和NIO2两种实现的区别

nio:

![avatar](https://picture.zhanghong110.top/docsify/16406769174094.png)

    Java NIO提供了选择器组件（Selector）用于同时检测多个通道的事件以实现异步I/O。我们将感兴趣的事件注册到Selector上，当事件发生时可以通过Selector获得事件发生的通道，并进行相关的操作。
    
    异步I/O的一个优势在于，他允许你同时根据大量的输入、输出执行I/O操作。同步I/O一般要借助于轮询，或者创建许许多多的线程以处理大量的链接。使用异步I/O，你可以监听任意数量的通道事件，不必轮询，也不必启动额外的线程。
    
    由于Selector.select()方法是阻塞的，因此Tomcat采用轮询的方式进行处理，轮询线程称为Poller。每个Poller维护了一个Selector实例以及一个PollerEvent事件队列。每当接收到新的链接时，会将获得的SocketChannel对象封装为org.apache.tomcat.util.net.NioChannel，并且将其注册到Poller（创建一个PollerEvent实例，添加到事件队列）。
    
    Poller运行时，首先将新添加到队列中的PollerEvent取出，并将SocketChannel的读事件（OP_READ）注册到Poller持有的Selector上，然后执行Selector.select。当捕获到读事件时，构造SocketProcessor，并提交到线程池进行请求处理。
    
    为了提升对象的利用率，NioEndpoint分别为NioChannel和PollerEvent对象创建了缓存队列。当需要NioChannel和PollerEvent对象时，会检测缓存队列中是否存在可用对象，如果存在则从队列中取出对象并且重置，如果不存在则新建。


nio2:

![avatar](https://picture.zhanghong110.top/docsify/16406771061244.png)

> 我们在这只分析Nio的实现，注意此处低版本的tomcat实现可能不通，Acceptor之前是景天抽象类现在是普通类,想要看懂下列流程我们首先要在心里大致知晓NIO的处理方式，也就是图nio所表述的东西，简单来说tomcat请求数据处理有两种线程，一个接收，一个处理，请求进入后先需要构造pollerEvent添加到事件队列，然后才会被处理线程处理。

从startInternal中的startAcceptorThread方法进入，我们可以看到如下代码

```
protected void startAcceptorThread() {
        acceptor = new Acceptor<>(this);
        String threadName = getName() + "-Acceptor";
        acceptor.setThreadName(threadName);
        Thread t = new Thread(acceptor, threadName);
        t.setPriority(getAcceptorThreadPriority());
        t.setDaemon(getDaemon());
        t.start();
    }
```

最终执行了start方法，所以哦我们进入Acceptor查看重写的run方法如下

```
@Override
    public void run() {

        int errorDelay = 0;
        long pauseStart = 0;

        try {
            // Loop until we receive a shutdown command
            while (!stopCalled) {

                // Loop if endpoint is paused.
                // There are two likely scenarios here.
                // The first scenario is that Tomcat is shutting down. In this
                // case - and particularly for the unit tests - we want to exit
                // this loop as quickly as possible. The second scenario is a
                // genuine pause of the connector. In this case we want to avoid
                // excessive CPU usage.
                // Therefore, we start with a tight loop but if there isn't a
                // rapid transition to stop then sleeps are introduced.
                // < 1ms       - tight loop
                // 1ms to 10ms - 1ms sleep
                // > 10ms      - 10ms sleep
                while (endpoint.isPaused() && !stopCalled) {
                    if (state != AcceptorState.PAUSED) {
                        pauseStart = System.nanoTime();
                        // Entered pause state
                        state = AcceptorState.PAUSED;
                    }
                    if ((System.nanoTime() - pauseStart) > 1_000_000) {
                        // Paused for more than 1ms
                        try {
                            if ((System.nanoTime() - pauseStart) > 10_000_000) {
                                Thread.sleep(10);
                            } else {
                                Thread.sleep(1);
                            }
                        } catch (InterruptedException e) {
                            // Ignore
                        }
                    }
                }

                if (stopCalled) {
                    break;
                }
                state = AcceptorState.RUNNING;

                try {
                    //if we have reached max connections, wait
                    //如果请求达到了最大连接数，则wait直到连接数降下来
                    endpoint.countUpOrAwaitConnection();

                    // Endpoint might have been paused while waiting for latch
                    // If that is the case, don't accept new connections
                    if (endpoint.isPaused()) {
                        continue;
                    }

                    U socket = null;
                    try {
                        // Accept the next incoming connection from the server
                        // socket
                        //接受下一次连接的socket,区分实现
                        socket = endpoint.serverSocketAccept();
                    } catch (Exception ioe) {
                        // We didn't get a socket
                        endpoint.countDownConnection();
                        if (endpoint.isRunning()) {
                            // Introduce delay if necessary
                            errorDelay = handleExceptionWithDelay(errorDelay);
                            // re-throw
                            throw ioe;
                        } else {
                            break;
                        }
                    }
                    // Successful accept, reset the error delay
                    errorDelay = 0;

                    // Configure the socket
                    if (!stopCalled && !endpoint.isPaused()) {
                        // setSocketOptions() will hand the socket off to
                        // an appropriate processor if successful
                        //`setSocketOptions()`这儿是关键，会将socket以事件的方式传递给poller
                        if (!endpoint.setSocketOptions(socket)) {
                            endpoint.closeSocket(socket);
                        }
                    } else {
                        endpoint.destroySocket(socket);
                    }
                } catch (Throwable t) {
                    ExceptionUtils.handleThrowable(t);
                    String msg = sm.getString("endpoint.accept.fail");
                    log.error(msg, t);
                }
            }
        } finally {
            stopLatch.countDown();
        }
        state = AcceptorState.ENDED;
    }
```

我们来翻译一下注释

*如果endpoint暂停，则循环。这里有两种可能的情况。第一种情况是 Tomcat 正在关闭。在这种情况下 - 特别是对于单元测试 - 我们希望尽快退出这个循环。第二种情况是连接器的真正暂停。在这种情况下，我们希望避免过度使用 CPU。因此，我们从一个紧密的循环开始，但如果没有快速过渡到停止，则引入睡眠。 < 1ms - 紧密循环 1ms 到 10ms - 1ms 睡眠 > 10ms - 10ms 睡眠*



- countUpOrAwaitConnection函数检查当前最大连接数，若未达到maxConnections则加一，否则等待；
- socket = endpoint.serverSocketAccept();区分了nio与nio2的实现
- setSocketOptions函数调用上的注释表明该函数将已连接套接字交给Poller线程处理，区分了nio与nio2的实现。



下面我们进入setSocketOptions查看处理方式，我们先进入NioEndpoint的实现代码如下：

```
protected boolean setSocketOptions(SocketChannel socket) {
        NioSocketWrapper socketWrapper = null;
        try {
            // Allocate channel and wrapper
            NioChannel channel = null;
            if (nioChannels != null) {
                channel = nioChannels.pop();
            }
            if (channel == null) {
                SocketBufferHandler bufhandler = new SocketBufferHandler(
                        socketProperties.getAppReadBufSize(),
                        socketProperties.getAppWriteBufSize(),
                        socketProperties.getDirectBuffer());
                if (isSSLEnabled()) {
                    channel = new SecureNioChannel(bufhandler, this);
                } else {
                    channel = new NioChannel(bufhandler);
                }
            }
            //封装channel
            NioSocketWrapper newWrapper = new NioSocketWrapper(channel, this);
            channel.reset(socket, newWrapper);
            connections.put(socket, newWrapper);
            socketWrapper = newWrapper;

            // Set socket properties
            // Disable blocking, polling will be used
            socket.configureBlocking(false);
            if (getUnixDomainSocketPath() == null) {
                socketProperties.setProperties(socket.socket());
            }

            socketWrapper.setReadTimeout(getConnectionTimeout());
            socketWrapper.setWriteTimeout(getConnectionTimeout());
            socketWrapper.setKeepAliveLeft(NioEndpoint.this.getMaxKeepAliveRequests());
            //注册到poller
            poller.register(socketWrapper);
            return true;
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            try {
                log.error(sm.getString("endpoint.socketOptionsError"), t);
            } catch (Throwable tt) {
                ExceptionUtils.handleThrowable(tt);
            }
            if (socketWrapper == null) {
                destroySocket(socket);
            }
        }
        // Tell to close the socket if needed
        return false;
    }
```

从以上代码可以看出

- 从NioChannel栈中出栈一个，若能重用（即不为null）则重用对象，否则新建一个NioChannel对象；
- 封装channel到NioSocketWrapper
- 将NioSocketWrapper注册到poller上
- 若成功转给Poller线程该函数返回true，否则返回false。返回false后，Acceptor类的closeSocket函数会关闭通道和底层Socket连接并将当前最大连接数减一。

poller:NioEndpoint的一个内部类实现了runable接口

> Poller线程主要用于以较少的资源轮询已连接套接字以保持连接，当数据可用时转给工作线程。

Poller线程数由NioEndPoint的pollerThreadCount成员变量控制，默认值为2与可用处理器数二者之间的较小值。

Poller实现了Runnable接口，可以看到构造函数为每个Poller打开了一个新的Selector。

```
//构造函数
public Poller() throws IOException {
    this.selector = Selector.open();
}
```

这个类有两个比较关键的方法Poller.register(),我们来看一下

```
  public void register(final NioSocketWrapper socketWrapper) {
            socketWrapper.interestOps(SelectionKey.OP_READ);//this is what OP_REGISTER turns into.
            PollerEvent event = null;
            if (eventCache != null) {
                event = eventCache.pop();
            }
            if (event == null) {
                event = new PollerEvent(socketWrapper, OP_REGISTER);
            } else {
                event.reset(socketWrapper, OP_REGISTER);
            }
            addEvent(event);
        }
  private void addEvent(PollerEvent event) {
            events.offer(event);
            if (wakeupCounter.incrementAndGet() == 0) {
                selector.wakeup();
            }
        }      
```

因为`Poller`维持了一个`events同步队列`，所以`Acceptor`接受到的channel会放在这个队列里面，放置的代码为`events.offer(event)`

接下来我们来看下run函数的功能

```
 @Override
        public void run() {
            // Loop until destroy() is called
            while (true) {

                boolean hasEvents = false;

                try {
                    if (!close) {
                        hasEvents = events();
                        if (wakeupCounter.getAndSet(-1) > 0) {
                            // If we are here, means we have other stuff to do
                            // Do a non blocking select
                            keyCount = selector.selectNow();
                        } else {
                            keyCount = selector.select(selectorTimeout);
                        }
                        wakeupCounter.set(0);
                    }
                    if (close) {
                        events();
                        timeout(0, false);
                        try {
                            selector.close();
                        } catch (IOException ioe) {
                            log.error(sm.getString("endpoint.nio.selectorCloseFail"), ioe);
                        }
                        break;
                    }
                    // Either we timed out or we woke up, process events first
                    if (keyCount == 0) {
                        hasEvents = (hasEvents | events());
                    }
                } catch (Throwable x) {
                    ExceptionUtils.handleThrowable(x);
                    log.error(sm.getString("endpoint.nio.selectorLoopError"), x);
                    continue;
                }

                // 获取当前选择器中所有注册的“选择键(已就绪的监听事件)”
                Iterator<SelectionKey> iterator =
                    keyCount > 0 ? selector.selectedKeys().iterator() : null;
                // Walk through the collection of ready keys and dispatch
                // any active event.
                // 对已经准备好的key进行处理
                while (iterator != null && iterator.hasNext()) {
                    SelectionKey sk = iterator.next();
                    iterator.remove();
                    NioSocketWrapper socketWrapper = (NioSocketWrapper) sk.attachment();
                    // Attachment may be null if another thread has called
                    // cancelledKey()
                    if (socketWrapper != null) {
                        // 真正处理key的地方
                        processKey(sk, socketWrapper);
                    }
                }

                // Process timeouts
                timeout(keyCount,hasEvents);
            }

            getStopLatch().countDown();
        }
```

- 若队列里有元素则会先把队列里的事件均执行一遍，PollerEvent的run方法会将通道注册到Poller的Selector上；
- 对select返回的SelectionKey进行处理，用SelectionKey的attachment方法得到，接着调用processKey去处理已连接套接字通道。

接下去我们分析processKey方法代码如下

```
protected void processKey(SelectionKey sk, NioSocketWrapper socketWrapper) {
            try {
                if (close) {
                    socketWrapper.close();
                } else if (sk.isValid()) {
                    if (sk.isReadable() || sk.isWritable()) {
                        if (socketWrapper.getSendfileData() != null) {
                            processSendfile(sk, socketWrapper, false);
                        } else {
                            unreg(sk, socketWrapper, sk.readyOps());
                            boolean closeSocket = false;
                             // 1. 处理读事件，比如生成Request对象
                            // Read goes before write
                            if (sk.isReadable()) {
                                if (socketWrapper.readOperation != null) {
                                    if (!socketWrapper.readOperation.process()) {
                                        closeSocket = true;
                                    }
                                } else if (socketWrapper.readBlocking) {
                                    synchronized (socketWrapper.readLock) {
                                        socketWrapper.readBlocking = false;
                                        socketWrapper.readLock.notify();
                                    }
                                } else if (!processSocket(socketWrapper, SocketEvent.OPEN_READ, true)) {
                                    closeSocket = true;
                                }
                            }
                             // 2. 处理写事件，比如将生成的Response对象通过socket写回客户端
                            if (!closeSocket && sk.isWritable()) {
                                if (socketWrapper.writeOperation != null) {
                                    if (!socketWrapper.writeOperation.process()) {
                                        closeSocket = true;
                                    }
                                } else if (socketWrapper.writeBlocking) {
                                    synchronized (socketWrapper.writeLock) {
                                        socketWrapper.writeBlocking = false;
                                        socketWrapper.writeLock.notify();
                                    }
                                } else if (!processSocket(socketWrapper, SocketEvent.OPEN_WRITE, true)) {
                                    closeSocket = true;
                                }
                            }
                            if (closeSocket) {
                                socketWrapper.close();
                            }
                        }
                    }
                } else {
                    // Invalid key
                    socketWrapper.close();
                }
            } catch (CancelledKeyException ckx) {
                socketWrapper.close();
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                log.error(sm.getString("endpoint.nio.keyProcessingError"), t);
            }
        }
```

它分别处理了读事件和写事件

1. 处理读事件，比如生成Request对象

2. 处理写事件，比如将生成的Response对象通过socket写回客户端

   

由上面的方法可以看出，处理关键方法为processSocket方法，我们进入它，位于AbstractEndpoint，代码如下

```
public boolean processSocket(SocketWrapperBase<S> socketWrapper,
            SocketEvent event, boolean dispatch) {
        try {
            if (socketWrapper == null) {
                return false;
            }
            SocketProcessorBase<S> sc = null;
            if (processorCache != null) {
            // 1. 从`processorCache`里面拿一个`Processor`来处理socket，`Processor`的实现为`SocketProcessor`
                sc = processorCache.pop();
            }
            if (sc == null) {
                sc = createSocketProcessor(socketWrapper, event);
            } else {
                sc.reset(socketWrapper, event);
            }
            // 2. 将`Processor`放到工作线程池中执行
            Executor executor = getExecutor();
            if (dispatch && executor != null) {
                executor.execute(sc);
            } else {
                sc.run();
            }
        } catch (RejectedExecutionException ree) {
            getLog().warn(sm.getString("endpoint.executor.fail", socketWrapper) , ree);
            return false;
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            // This means we got an OOM or similar creating a thread, or that
            // the pool and its queue are full
            getLog().error(sm.getString("endpoint.process.fail"), t);
            return false;
        }
        return true;
    }

```

dispatch参数表示是否要在另外的线程中处理，上文processKey各处传递的参数都是true。

- dispatch为true且工作线程池存在时会执行executor.execute(sc)，之后是由工作线程池处理已连接套接字；
- 否则继续由Poller线程自己处理已连接套接字。

我们看一下createSocketProcessor方法

```
 @Override
    protected SocketProcessorBase<NioChannel> createSocketProcessor(
            SocketWrapperBase<NioChannel> socketWrapper, SocketEvent event) {
        return new SocketProcessor(socketWrapper, event);
    }
```

是由于NioEndPoint自己实现的

我们回到NioEndPoint，看SocketProcessor的run方法最终做了什么，代码如下

```
 @Override
        protected void doRun() {
            /*
             * Do not cache and re-use the value of socketWrapper.getSocket() in
             * this method. If the socket closes the value will be updated to
             * CLOSED_NIO_CHANNEL and the previous value potentially re-used for
             * a new connection. That can result in a stale cached value which
             * in turn can result in unintentionally closing currently active
             * connections.
             */
            Poller poller = NioEndpoint.this.poller;
            if (poller == null) {
                socketWrapper.close();
                return;
            }

            try {
                int handshake = -1;
                try {
                    if (socketWrapper.getSocket().isHandshakeComplete()) {
                        // No TLS handshaking required. Let the handler
                        // process this socket / event combination.
                        handshake = 0;
                    } else if (event == SocketEvent.STOP || event == SocketEvent.DISCONNECT ||
                            event == SocketEvent.ERROR) {
                        // Unable to complete the TLS handshake. Treat it as
                        // if the handshake failed.
                        handshake = -1;
                    } else {
                        handshake = socketWrapper.getSocket().handshake(event == SocketEvent.OPEN_READ, event == SocketEvent.OPEN_WRITE);
                        // The handshake process reads/writes from/to the
                        // socket. status may therefore be OPEN_WRITE once
                        // the handshake completes. However, the handshake
                        // happens when the socket is opened so the status
                        // must always be OPEN_READ after it completes. It
                        // is OK to always set this as it is only used if
                        // the handshake completes.
                        event = SocketEvent.OPEN_READ;
                    }
                } catch (IOException x) {
                    handshake = -1;
                    if (log.isDebugEnabled()) {
                        log.debug("Error during SSL handshake",x);
                    }
                } catch (CancelledKeyException ckx) {
                    handshake = -1;
                }
                if (handshake == 0) {
                    SocketState state = SocketState.OPEN;
                    // Process the request from this socket
                    // 将处理逻辑交给`Handler`处理，当event为null时，则表明是一个`OPEN_READ`事件
                    if (event == null) {
                        state = getHandler().process(socketWrapper, SocketEvent.OPEN_READ);
                    } else {
                        state = getHandler().process(socketWrapper, event);
                    }
                    if (state == SocketState.CLOSED) {
                        socketWrapper.close();
                    }
                } else if (handshake == -1 ) {
                    getHandler().process(socketWrapper, SocketEvent.CONNECT_FAIL);
                    socketWrapper.close();
                } else if (handshake == SelectionKey.OP_READ){
                    socketWrapper.registerReadInterest();
                } else if (handshake == SelectionKey.OP_WRITE){
                    socketWrapper.registerWriteInterest();
                }
            } catch (CancelledKeyException cx) {
                socketWrapper.close();
            } catch (VirtualMachineError vme) {
                ExceptionUtils.handleThrowable(vme);
            } catch (Throwable t) {
                log.error(sm.getString("endpoint.processing.fail"), t);
                socketWrapper.close();
            } finally {
                socketWrapper = null;
                event = null;
                //return to cache
                if (running && !paused && processorCache != null) {
                    processorCache.push(this);
                }
            }
        }

```

首先该类有如下注释

```
/**
 * This class is the equivalent of the Worker, but will simply use in an
 * external Executor thread pool.
 */
```

翻译可知SocketProcessor与Worker的作用等价。Handler`的关键方法是`process(),虽然这个方法有很多条件分支，但是逻辑却非常清楚，主要是调用Processor.process()方法我们跟进可以进入AbstractProtocol的process方法，这个方法有点长就不贴了，其核心代码为下面这句

```
  state = processor.process(wrapper, status);
```

> 至此endpoint的任务交给了processor

processor:

下面我们开始分析processor如何交接

在process方法种还有一句代码如下所示，由所调用的`AbstractHttp11Protocol`和`AbstractAjpProtocol`来实现

```
processor = getProtocol().createProcessor();
```

我们由此可以进入这个方法，发现调用了下面的构造方法，设置了一些配置属性

```
public Http11Processor(AbstractHttp11Protocol<?> protocol, Adapter adapter) {
        super(adapter);
        this.protocol = protocol;

        httpParser = new HttpParser(protocol.getRelaxedPathChars(),
                protocol.getRelaxedQueryChars());

        inputBuffer = new Http11InputBuffer(request, protocol.getMaxHttpHeaderSize(),
                protocol.getRejectIllegalHeader(), httpParser);
        request.setInputBuffer(inputBuffer);

        outputBuffer = new Http11OutputBuffer(response, protocol.getMaxHttpHeaderSize());
        response.setOutputBuffer(outputBuffer);

        // Create and add the identity filters.
        inputBuffer.addFilter(new IdentityInputFilter(protocol.getMaxSwallowSize()));
        outputBuffer.addFilter(new IdentityOutputFilter());

        // Create and add the chunked filters.
        inputBuffer.addFilter(new ChunkedInputFilter(protocol.getMaxTrailerSize(),
                protocol.getAllowedTrailerHeadersInternal(), protocol.getMaxExtensionSize(),
                protocol.getMaxSwallowSize()));
        outputBuffer.addFilter(new ChunkedOutputFilter());

        // Create and add the void filters.
        inputBuffer.addFilter(new VoidInputFilter());
        outputBuffer.addFilter(new VoidOutputFilter());

        // Create and add buffered input filter
        inputBuffer.addFilter(new BufferedInputFilter());

        // Create and add the gzip filters.
        //inputBuffer.addFilter(new GzipInputFilter());
        outputBuffer.addFilter(new GzipOutputFilter());

        pluggableFilterIndex = inputBuffer.getFilters().length;
    }
```

下面我们进入核心代码processor.process，主要关注其对读的操作，也只有一行代码。调用`service()`方法。

```
 @Override
    public SocketState process(SocketWrapperBase<?> socketWrapper, SocketEvent status)
            throws IOException {

        SocketState state = SocketState.CLOSED;
        Iterator<DispatchType> dispatches = null;
        do {
            if (dispatches != null) {
                DispatchType nextDispatch = dispatches.next();
                if (getLog().isDebugEnabled()) {
                    getLog().debug("Processing dispatch type: [" + nextDispatch + "]");
                }
                state = dispatch(nextDispatch.getSocketStatus());
                if (!dispatches.hasNext()) {
                    state = checkForPipelinedData(state, socketWrapper);
                }
            } else if (status == SocketEvent.DISCONNECT) {
                // Do nothing here, just wait for it to get recycled
            } else if (isAsync() || isUpgrade() || state == SocketState.ASYNC_END) {
                state = dispatch(status);
                state = checkForPipelinedData(state, socketWrapper);
            } else if (status == SocketEvent.OPEN_WRITE) {
                // Extra write event likely after async, ignore
                state = SocketState.LONG;
            } else if (status == SocketEvent.OPEN_READ) {
            //这里
                state = service(socketWrapper);
            } else if (status == SocketEvent.CONNECT_FAIL) {
                logAccess(socketWrapper);
            } else {
                // Default to closing the socket if the SocketEvent passed in
                // is not consistent with the current state of the Processor
                state = SocketState.CLOSED;
            }

            if (getLog().isDebugEnabled()) {
                getLog().debug("Socket: [" + socketWrapper +
                        "], Status in: [" + status +
                        "], State out: [" + state + "]");
            }

            if (isAsync()) {
                state = asyncPostProcess();
                if (getLog().isDebugEnabled()) {
                    getLog().debug("Socket: [" + socketWrapper +
                            "], State after async post processing: [" + state + "]");
                }
            }

            if (dispatches == null || !dispatches.hasNext()) {
                // Only returns non-null iterator if there are
                // dispatches to process.
                dispatches = getIteratorAndClearDispatches();
            }
        } while (state == SocketState.ASYNC_END ||
                dispatches != null && state != SocketState.CLOSED);

        return state;
    }
```

我们进入对应的实现，由Http11Processor类实现service方法，这个类也很长我们不全贴了，它的主要操作就是

1. 生成Request和Response对象
2. 调用`Adapter.service()`方法，将生成的Request和Response对象传进去

```
  rp.setStage(org.apache.coyote.Constants.STAGE_SERVICE);
  //传入生成的Request和Response对象传进去
  getAdapter().service(request, response);
```

adapter:

`Adapter`用于连接`Connector`和`Container`，起到承上启下的作用。`Processor`会调用`Adapter.service()`方法。我们来分析一下，主要做了下面几件事情：

1. *根据coyote框架的request和response对象，生成connector的request和response对象（是HttpServletRequest和HttpServletResponse的封装）*
2. *补充header*
3. *解析请求，该方法会出现代理服务器、设置必要的header等操作*
4. *真正进入容器的地方，调用Engine容器下pipeline的阀门*
5. *通过request.finishRequest 与 response.finishResponse(刷OutputBuffer中的数据到浏览器) 来完成整个请求*

代码如下，由CoyoteAdapter类提供

```
@Override
    public void service(org.apache.coyote.Request req, org.apache.coyote.Response res)
            throws Exception {
// 1. 根据coyote框架的request和response对象，生成connector的request和response对象（是HttpServletRequest和HttpServletResponse的封装）
        Request request = (Request) req.getNote(ADAPTER_NOTES);
        Response response = (Response) res.getNote(ADAPTER_NOTES);

        if (request == null) {
            // Create objects
            request = connector.createRequest();
            request.setCoyoteRequest(req);
            response = connector.createResponse();
            response.setCoyoteResponse(res);

            // Link objects
            request.setResponse(response);
            response.setRequest(request);

            // Set as notes
            req.setNote(ADAPTER_NOTES, request);
            res.setNote(ADAPTER_NOTES, response);

            // Set query string encoding
            req.getParameters().setQueryStringCharset(connector.getURICharset());
        }

       // 2. 补充header
        if (connector.getXpoweredBy()) {
            response.addHeader("X-Powered-By", POWERED_BY);
        }

        boolean async = false;
        boolean postParseSuccess = false;

        req.getRequestProcessor().setWorkerThreadName(THREAD_NAME.get());

        try {
            // Parse and set Catalina and configuration specific
            // request parameters
            // 3. 解析请求，该方法会出现代理服务器、设置必要的header等操作
            // 用来处理请求映射 (获取 host, context, wrapper, URI 后面的参数的解析, sessionId )
            postParseSuccess = postParseRequest(req, request, res, response);
            if (postParseSuccess) {
                //check valves if we support async
                request.setAsyncSupported(
                        connector.getService().getContainer().getPipeline().isAsyncSupported());
                // Calling the container
                 // 4. 真正进入容器的地方，调用Engine容器下pipeline的阀门
                connector.getService().getContainer().getPipeline().getFirst().invoke(
                        request, response);
            }
            if (request.isAsync()) {
                async = true;
                ReadListener readListener = req.getReadListener();
                if (readListener != null && request.isFinished()) {
                    // Possible the all data may have been read during service()
                    // method so this needs to be checked here
                    ClassLoader oldCL = null;
                    try {
                        oldCL = request.getContext().bind(false, null);
                        if (req.sendAllDataReadEvent()) {
                            req.getReadListener().onAllDataRead();
                        }
                    } finally {
                        request.getContext().unbind(false, oldCL);
                    }
                }

                Throwable throwable =
                        (Throwable) request.getAttribute(RequestDispatcher.ERROR_EXCEPTION);

                // If an async request was started, is not going to end once
                // this container thread finishes and an error occurred, trigger
                // the async error process
                if (!request.isAsyncCompleting() && throwable != null) {
                    request.getAsyncContextInternal().setErrorState(throwable, true);
                }
            } else {
                //5. 通过request.finishRequest 与 response.finishResponse(刷OutputBuffer中的数据到浏览器) 来完成整个请求
                request.finishRequest();
                //将 org.apache.catalina.connector.Response对应的 OutputBuffer 中的数据 刷到 org.apache.coyote.Response 对应的 InternalOutputBuffer 中, 并且最终调用 socket对应的 outputStream 将数据刷出去( 这里会组装 Http Response 中的 header 与 body 里面的数据, 并且刷到远端
                response.finishResponse();
            }

        } catch (IOException e) {
            // Ignore
        } finally {
            AtomicBoolean error = new AtomicBoolean(false);
            res.action(ActionCode.IS_ERROR, error);

            if (request.isAsyncCompleting() && error.get()) {
                // Connection will be forcibly closed which will prevent
                // completion happening at the usual point. Need to trigger
                // call to onComplete() here.
                res.action(ActionCode.ASYNC_POST_PROCESS,  null);
                async = false;
            }

            // Access log
            if (!async && postParseSuccess) {
                // Log only if processing was invoked.
                // If postParseRequest() failed, it has already logged it.
                Context context = request.getContext();
                Host host = request.getHost();
                // If the context is null, it is likely that the endpoint was
                // shutdown, this connection closed and the request recycled in
                // a different thread. That thread will have updated the access
                // log so it is OK not to update the access log here in that
                // case.
                // The other possibility is that an error occurred early in
                // processing and the request could not be mapped to a Context.
                // Log via the host or engine in that case.
                long time = System.nanoTime() - req.getStartTimeNanos();
                if (context != null) {
                    context.logAccess(request, response, time, false);
                } else if (response.isError()) {
                    if (host != null) {
                        host.logAccess(request, response, time, false);
                    } else {
                        connector.getService().getContainer().logAccess(
                                request, response, time, false);
                    }
                }
            }

            req.getRequestProcessor().setWorkerThreadName(null);

            // Recycle the wrapper request and response
            if (!async) {
                updateWrapperErrorCount(request, response);
                request.recycle();
                response.recycle();
            }
        }
    }
```

请求预处理，上面代码的postParseRequest方法，如下所示

```
protected boolean postParseRequest(org.apache.coyote.Request req, Request request,
        org.apache.coyote.Response res, Response response) throws IOException, ServletException {

    // If the processor has set the scheme (AJP does this, HTTP does this if
    // SSL is enabled) use this to set the secure flag as well. If the
    // processor hasn't set it, use the settings from the connector
    if (req.scheme().isNull()) {
        // Use connector scheme and secure configuration, (defaults to
        // "http" and false respectively)
        req.scheme().setString(connector.getScheme());
        request.setSecure(connector.getSecure());
    } else {
        // Use processor specified scheme to determine secure state
        request.setSecure(req.scheme().equals("https"));
    }

    // At this point the Host header has been processed.
    // Override if the proxyPort/proxyHost are set
    String proxyName = connector.getProxyName();
    int proxyPort = connector.getProxyPort();
    if (proxyPort != 0) {
        req.setServerPort(proxyPort);
    } else if (req.getServerPort() == -1) {
        // Not explicitly set. Use default ports based on the scheme
        if (req.scheme().equals("https")) {
            req.setServerPort(443);
        } else {
            req.setServerPort(80);
        }
    }
    if (proxyName != null) {
        req.serverName().setString(proxyName);
    }

    MessageBytes undecodedURI = req.requestURI();

    // Check for ping OPTIONS * request
    if (undecodedURI.equals("*")) {
        if (req.method().equalsIgnoreCase("OPTIONS")) {
            StringBuilder allow = new StringBuilder();
            allow.append("GET, HEAD, POST, PUT, DELETE, OPTIONS");
            // Trace if allowed
            if (connector.getAllowTrace()) {
                allow.append(", TRACE");
            }
            res.setHeader("Allow", allow.toString());
            // Access log entry as processing won't reach AccessLogValve
            connector.getService().getContainer().logAccess(request, response, 0, true);
            return false;
        } else {
            response.sendError(400, "Invalid URI");
        }
    }

    MessageBytes decodedURI = req.decodedURI();

    if (undecodedURI.getType() == MessageBytes.T_BYTES) {
        if (connector.getRejectSuspiciousURIs()) {
            if (checkSuspiciousURIs(undecodedURI.getByteChunk())) {
                response.sendError(400, "Invalid URI");
            }
        }

        // Copy the raw URI to the decodedURI
        decodedURI.duplicate(undecodedURI);

        // Parse (and strip out) the path parameters
        parsePathParameters(req, request);

        // URI decoding
        // %xx decoding of the URL
        try {
            req.getURLDecoder().convert(decodedURI.getByteChunk(), connector.getEncodedSolidusHandlingInternal());
        } catch (IOException ioe) {
            response.sendError(400, "Invalid URI: " + ioe.getMessage());
        }
        // Normalization
        if (normalize(req.decodedURI(), connector.getAllowBackslash())) {
            // Character decoding
            convertURI(decodedURI, request);
            // URIEncoding values are limited to US-ASCII supersets.
            // Therefore it is not necessary to check that the URI remains
            // normalized after character decoding
        } else {
            response.sendError(400, "Invalid URI");
        }
    } else {
        /* The URI is chars or String, and has been sent using an in-memory
         * protocol handler. The following assumptions are made:
         * - req.requestURI() has been set to the 'original' non-decoded,
         *   non-normalized URI
         * - req.decodedURI() has been set to the decoded, normalized form
         *   of req.requestURI()
         * - 'suspicious' URI filtering - if required - has already been
         *   performed
         */
        decodedURI.toChars();
        // Remove all path parameters; any needed path parameter should be set
        // using the request object rather than passing it in the URL
        CharChunk uriCC = decodedURI.getCharChunk();
        int semicolon = uriCC.indexOf(';');
        if (semicolon > 0) {
            decodedURI.setChars(uriCC.getBuffer(), uriCC.getStart(), semicolon);
        }
    }

    // Request mapping.
    MessageBytes serverName;
    if (connector.getUseIPVHosts()) {
        serverName = req.localName();
        if (serverName.isNull()) {
            // well, they did ask for it
            res.action(ActionCode.REQ_LOCAL_NAME_ATTRIBUTE, null);
        }
    } else {
        serverName = req.serverName();
    }

    // Version for the second mapping loop and
    // Context that we expect to get for that version
    String version = null;
    Context versionContext = null;
    boolean mapRequired = true;

    if (response.isError()) {
        // An error this early means the URI is invalid. Ensure invalid data
        // is not passed to the mapper. Note we still want the mapper to
        // find the correct host.
        decodedURI.recycle();
    }

    while (mapRequired) {
        // This will map the the latest version by default
        connector.getService().getMapper().map(serverName, decodedURI,
                version, request.getMappingData());

        // If there is no context at this point, either this is a 404
        // because no ROOT context has been deployed or the URI was invalid
        // so no context could be mapped.
        if (request.getContext() == null) {
            // Allow processing to continue.
            // If present, the rewrite Valve may rewrite this to a valid
            // request.
            // The StandardEngineValve will handle the case of a missing
            // Host and the StandardHostValve the case of a missing Context.
            // If present, the error reporting valve will provide a response
            // body.
            return true;
        }

        // Now we have the context, we can parse the session ID from the URL
        // (if any). Need to do this before we redirect in case we need to
        // include the session id in the redirect
        String sessionID;
        if (request.getServletContext().getEffectiveSessionTrackingModes()
                .contains(SessionTrackingMode.URL)) {

            // Get the session ID if there was one
            sessionID = request.getPathParameter(
                    SessionConfig.getSessionUriParamName(
                            request.getContext()));
            if (sessionID != null) {
                request.setRequestedSessionId(sessionID);
                request.setRequestedSessionURL(true);
            }
        }

        // Look for session ID in cookies and SSL session
        try {
            parseSessionCookiesId(request);
        } catch (IllegalArgumentException e) {
            // Too many cookies
            if (!response.isError()) {
                response.setError();
                response.sendError(400);
            }
            return true;
        }
        parseSessionSslId(request);

        sessionID = request.getRequestedSessionId();

        mapRequired = false;
        if (version != null && request.getContext() == versionContext) {
            // We got the version that we asked for. That is it.
        } else {
            version = null;
            versionContext = null;

            Context[] contexts = request.getMappingData().contexts;
            // Single contextVersion means no need to remap
            // No session ID means no possibility of remap
            if (contexts != null && sessionID != null) {
                // Find the context associated with the session
                for (int i = contexts.length; i > 0; i--) {
                    Context ctxt = contexts[i - 1];
                    if (ctxt.getManager().findSession(sessionID) != null) {
                        // We found a context. Is it the one that has
                        // already been mapped?
                        if (!ctxt.equals(request.getMappingData().context)) {
                            // Set version so second time through mapping
                            // the correct context is found
                            version = ctxt.getWebappVersion();
                            versionContext = ctxt;
                            // Reset mapping
                            request.getMappingData().recycle();
                            mapRequired = true;
                            // Recycle cookies and session info in case the
                            // correct context is configured with different
                            // settings
                            request.recycleSessionInfo();
                            request.recycleCookieInfo(true);
                        }
                        break;
                    }
                }
            }
        }

        if (!mapRequired && request.getContext().getPaused()) {
            // Found a matching context but it is paused. Mapping data will
            // be wrong since some Wrappers may not be registered at this
            // point.
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                // Should never happen
            }
            // Reset mapping
            request.getMappingData().recycle();
            mapRequired = true;
        }
    }

    // Possible redirect
    MessageBytes redirectPathMB = request.getMappingData().redirectPath;
    if (!redirectPathMB.isNull()) {
        String redirectPath = URLEncoder.DEFAULT.encode(
                redirectPathMB.toString(), StandardCharsets.UTF_8);
        String query = request.getQueryString();
        if (request.isRequestedSessionIdFromURL()) {
            // This is not optimal, but as this is not very common, it
            // shouldn't matter
            redirectPath = redirectPath + ";" +
                    SessionConfig.getSessionUriParamName(
                        request.getContext()) +
                "=" + request.getRequestedSessionId();
        }
        if (query != null) {
            // This is not optimal, but as this is not very common, it
            // shouldn't matter
            redirectPath = redirectPath + "?" + query;
        }
        response.sendRedirect(redirectPath);
        request.getContext().logAccess(request, response, 0, true);
        return false;
    }

    // Filter trace method
    if (!connector.getAllowTrace()
            && req.method().equalsIgnoreCase("TRACE")) {
        Wrapper wrapper = request.getWrapper();
        String header = null;
        if (wrapper != null) {
            String[] methods = wrapper.getServletMethods();
            if (methods != null) {
                for (String method : methods) {
                    if ("TRACE".equals(method)) {
                        continue;
                    }
                    if (header == null) {
                        header = method;
                    } else {
                        header += ", " + method;
                    }
                }
            }
        }
        if (header != null) {
            res.addHeader("Allow", header);
        }
        response.sendError(405, "TRACE method is not allowed");
        // Safe to skip the remainder of this method.
        return true;
    }

    doConnectorAuthenticationAuthorization(req, request);

    return true;
}
```

以MessageBytes的类型是T_BYTES为例：

- parsePathParameters方法去除URI中分号表示的路径参数；
- req.getURLDecoder()得到一个UDecoder实例，它的convert方法对URI解码，这里的解码只是移除百分号，计算百分号后两位的十六进制数字值以替代原来的三位百分号编码；
- normalize方法规格化URI，解释路径中的“.”和“..”；
- convertURI方法利用Connector的uriEncoding属性将URI的字节转换为字符表示；
- 注意connector.getService().getMapper().map(serverName, decodedURI, version, request.getMappingData()) 这行，之前Service启动时MapperListener注册了该Service内的各Host和Context。根据URI选择Context时，Mapper的map方法采用的是convertURI方法解码后的URI与每个Context的路径去比较

由容器来处理（自己处理）的话

如果请求可以被传给容器的Pipeline即当postParseRequest方法返回true时，则由容器继续处理，在service方法中有connector.getService().getContainer().getPipeline().getFirst().invoke(request, response)这一行：

- Connector调用getService返回StandardService；
- StandardService调用getContainer返回StandardEngine；
- StandardEngine调用getPipeline返回与其关联的StandardPipeline；

> 至此我们应该就很清楚tomcat如何封装请求的了

