# Spring源码解析

> 源码构建参考本文springmvc源码解析

无图版，主要是代码的快速解析，图以后再补啦



## **以ContextLoaderListener（上下文加载器监听器）为入口**

- ContextLoaderListener是配置在web.xml中的Spring环境加载监听器

- 它既继承了ContextLoader类，也实现了ServletContextListener接口

- 它的大部分方法都是通过父类ContextLoader实现的， 他可以算作一个适配器，本身只是为了实现Servlet的监听器接口，以便在WEB应用启动时加载容器（ApplicationContext类）

- 当Web应用启动时，也就是ServletContext启动时; 它的contextInitialized()方法监听到该事件， 将执行其父类ContextLoader的initWebApplicationContext()方法， 并以ServletContext作为参数

- 此外，它也有销毁方法，也就是在Servlet销毁时触发

- ServletContext是Servlet的最高级别范围，其优先级如下:

- - page -> request -> session -> application(servletContext)
  - 注意：ServletContext在jsp中为application,这两个是同一个东西



## **ContextLoader类（上下文加载器） : ContextLoaderListener的父类**

- 该类的静态代码块中，从一个Spring内部的属性文件中读取了默认的Application类，为XMLWebApplicationContext类。当没有指定web容器的时候，默认使用它。

- initWebApplicationContext()方法，初始化容器

- - 如果ServletContext中，已经设置了容器名.ROOT属性名的属性，表示容器已经加载，抛出异常

  - 否则，调用createWebApplicationContext()方法创建容器

  - - 创建容器方法中，调用determineContextClass()方法确认容器的类对象

    - - 如果在web.xml中指定了contextClass参数，则使用自定义的容器类（该自定义容器类必须实现ConfigurableWebApplicationContext接口）
      - 否则会从Spring的一个默认策略文件中读取默认的容器类(WebApplicationContext)。默认也就是该类静态代码块中加载的属性文件，默认类为XmlWebApplication类



- - - 确认完容器的类对象后，会调用BeanUtils.instantiateClass()实例化该容器(该容器必须有无参构造方法)(注意，该容器类必须实现ConfigurableWebApplication接口)



- - 创建完容器实例后，如果该容器是自定义容器（即实现了ConfigurableWebApplicationContext接口，只要进入了上面这步createWebApplicationContext()方法的，都是实现了接口的）则

  - - 强转实例为ConfigurableWebApplication类,并使用isActive()方法判断是否激活，如果不是激活的：

    - - 判断getParent()方法是否为空，如果为空，使用loadParentContext()和setParent()加载并设置父容器，但目前loadParentContext()方法默认为空

      - 调用configureAndRefreshWebApplicationContext()配置并刷新该容器

      - - 配置刷新方法中，如果容器id是默认的,也就是没修改过(容器类全名（例如：java.util.XXXClassName）@16进制hashcode) 如果web.xml中配置了contextId参数，就使用该参数值作为id， 否则使用 容器类全名:contextPath（项目路径名,ServletContext的getContextPath()方法返回值） 作为id
        - 并且将ServletContext放到容器中，并从web.xml的contextConfigLocation属性中读取Spring配置文件路径值,放到容器中
        - 并获取容器的ConfigurableEnvironment属性，如果该属性实现了ConfigurableWebEnvironment接口， 就调用该属性的initPropertySources()方法；
        - 在StandardServletEnvironment类中，该方法调用了WebApplicationContextUtils.initServletPropertySources()方法； 把ServletContext或ServletConfig中的web.xml中配置的servletContextInitParams和servletConfigInitParams参数,加载到环境类中的propertySources属性中去
        - 这个环境类存储的东西大致是jvm的一些启动参数、当前主机系统变量、还有上面两个参数等



- - - - - 然后调用customizeContext()方法
        - 在该方法中，首先调用determineContextInitializerClasses()方法,确认容器的初始化类对象集合（List<Class<ApplicationContextInitializer>>）：
        - 根据web.xml中的全局配置参数（globalInitializerClasses）和非全局配置参数（contextInitializerClasses）， 确认容器初始化器的类对象集合（可能有多个，用INIT_PARAM_DELIMITERS常量规定的字符分割；也可能没有配置，则list.size为0）



- - - - - 然后遍历这些容器初始化器，并使用GenericTypeResolver.resolveTypeArgument()方法判断这个 容器初始化器的泛型参数(该参数表示该容器初始化器对应的容器类), 如果对应的容器类不是 ConfigurableWebApplicationContext，也就是不能初始化自定义容器，就抛出异常
        - 然后将符合要求的容器初始化器类对象实例化 加入到 ContextLoader类的 容器初始化器集合中
        - 然后将该集合排序，并遍历该集合，调用每个容器初始化器的 initialize()方法 对自定义容器进行初始化操作 (ApplicationContextInitializer接口没有已有的实现类，需要自行实现，来对容器进行一些操作)



- - - - - 调用WebApplicationContext的refresh()方法
        - 该方法首先执行了prepareRefresh()方法
        - 设置容器的一些状态属性为激活，未关闭，启动时间为当前毫秒等
        - 然后执行之前提到的initPropertySources()方法；
        - 接着执行validateRequiredProperties()方法校验一些必须的参数不能为空（哪些参数为必须参数，参考ConfigurablePropertyResolver#setRequiredProperties()方法）
        - 最后新建一个LinkedHashSet，存放early ApplicationEvents



- - - - - 然后执行obtainFreshBeanFactory()方法，告诉子类刷新内部的bean工厂
        - 执行refreshBeanFactory()方法
        - 如果容器的beanFactory不为空，则销毁它
        - 然后创建一个新的beanFactory
        - 然后执行loadBeanDefinitions(beanFactory)方法
        - 创建一个XmlBeanDefinitionReader类
        - 将当前容器的Environment传给该读取器类
        - 设置该读取器类的资源加载器为该容器
        - 执行initBeanDefinitionReader()方法，默认实现为空
        - 执行loadBeanDefinitions()方法
        - 从容器中读取到之前存着的web.xml中配置的spring配置文件string[]
        - 然后一次调用reader.loadBeanDefinitions(configLocation)方法加载每个配置文件中的bean（此方法留后补充，总而言之就是加载文件中的bean）
        - 如果只有一个spring配置文件，resourceLoader实现的应该是ResourceLoader接口
        - 如果是有多个spring配置文件，则resourceLoader实现ResourcePatternResolver接口
        - 使用getResources()方法获取Resource[]（但是貌似只有一个文件，并且数组len为1）
        - 接着执行prepareBeanFactory(beanFactory)方法给beanFactory设置了一系列预备属性 // TODO

​                              https://blog.csdn.net/wzngzaixiaomantou/article/details/122400554)