# SpringMVC解析

*MVC已经是现代Web开发中的一个很重要的部分，而springmvc又是我们现在常用框架，无论是使用springboot还是SSM,SSH架构都需要使用，服务器通过ServerSocket获取到Http请求然后对其解析并封装成Request和Response对象，然后将其交给Servlet容器进行处理,选择对应的Servlet处理请求,返回结果(实际上是很复杂,作为一个web程序员,这个真的是应该好好了解研究的)。那么为什么tomcat和springmvc可以结合起来呢,最最核心的原因是他们都是基于Servlet规范的,
由于Servlet规范,他们可以互相通信(服务器和SpringMVC的结合在debug时将会简单体现)。下面我们就来研究一下springmvc框架的一些逻辑原理*

## 一.构建源码

首先拉取源码 

[github地址](https://github.com/spring-projects/spring-framework)

拉个全家桶，spring-framework的阅读模式构建相对比较简单，我们这里拉取的版本及环境如下所示

spring-framework      v5.3.14

idea 2021.3

jdk17

我们采用自带的`gradle`构建（本地的版本落后了），目前自带的为7.3版本

导入后构建，如图所示

![avatar](https://picture.zhanghong110.top/docsify/16407413746525.png)

## 二.springmvc源码分析

> 在开始之前，我们先引用一张图来详细了解全部内容

![avatar](https://picture.zhanghong110.top/docsify/b92f9bd5eded1599182c459883ff20f0_411512-20170729103203243-1447701989.png)



```java
  <servlet>
    <servlet-name>dispatcherServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>classpath:springmvc.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
  </servlet>
  <servlet-mapping>
    <servlet-name>dispatcherServlet</servlet-name>
    <url-pattern>/</url-pattern>
  </servlet-mapping>
```

虽然使用SpringMVC不需要我们写Servlet，但SpringMVC是封装了Servlet，提供 `DispatcherServlet` 来帮我们处理的。
 所以需要在 `web.xml` 配置 `DispatcherServlet`。

> 初始化

我们知道，Servlet初始化时，Servlet的 `init()`方法会被调用。我们进入 `DispatcherServlet`中，发现并没有该方法，那么肯定在它集成的父类上。
 `DispatcherServlet` 继承于 `FrameworkServlet`，结果还是没找到，继续找它的父类 `HttpServletBean`。可以看到如下`init`方法

```java
@Override
	public final void init() throws ServletException {

		// Set bean properties from init parameters.
		// 获取配置web.xml中的参数
		PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
		if (!pvs.isEmpty()) {
			try {
				BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
				ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
				bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
				initBeanWrapper(bw);
				bw.setPropertyValues(pvs, true);
			}
			catch (BeansException ex) {
				if (logger.isErrorEnabled()) {
					logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
				}
				throw ex;
			}
		}

		// Let subclasses do whatever initialization they like.
		// 空方法，由子类实现
		initServletBean();
	}
```

`getServletConfig`由`servlet-api`的`GenericServlet`提供

我们主要看`initServletBean`方法，回到`FrameworkServlet`类中

```java
@Override
	protected final void initServletBean() throws ServletException {
		getServletContext().log("Initializing Spring " + getClass().getSimpleName() + " '" + getServletName() + "'");
		if (logger.isInfoEnabled()) {
			logger.info("Initializing Servlet '" + getServletName() + "'");
		}
		long startTime = System.currentTimeMillis();

		try {
		    //重点：初始化WebApplicationContext
			this.webApplicationContext = initWebApplicationContext();
			// 空方法由子类实现，做一些初始化的事情，暂时没有被复写
			initFrameworkServlet();
		}
		catch (ServletException | RuntimeException ex) {
			logger.error("Context initialization failed", ex);
			throw ex;
		}

		if (logger.isDebugEnabled()) {
			String value = this.enableLoggingRequestDetails ?
					"shown which may lead to unsafe logging of potentially sensitive data" :
					"masked to prevent unsafe logging of potentially sensitive data";
			logger.debug("enableLoggingRequestDetails='" + this.enableLoggingRequestDetails +
					"': request parameters and headers will be " + value);
		}

		if (logger.isInfoEnabled()) {
			logger.info("Completed initialization in " + (System.currentTimeMillis() - startTime) + " ms");
		}
	}
```

> 为啥会执行`FrameworkServlet`中的`initServletBean`方法要清楚这边的执行逻辑调用孙类的`init`,最后找到了爷类的`init`,爷类的`init`调用自身的`initServletBean`方法，由于这边父类`FrameworkServlet`重写了`initServletBean`故而根据子类方法优先的规则优先调用父类中的实现，`initFrameworkServlet`同理

我们重点看下`initWebApplicationContext`方法，代码如下

```java
protected WebApplicationContext initWebApplicationContext() {
		WebApplicationContext rootContext =
				WebApplicationContextUtils.getWebApplicationContext(getServletContext());
		WebApplicationContext wac = null;
        //有参数构造方法，传入webApplicationContext对象，就会进入该判断
		if (this.webApplicationContext != null) {
			// A context instance was injected at construction time -> use it
			wac = this.webApplicationContext;
			//还没初始化过，容器的refresh()还没有调用
			if (wac instanceof ConfigurableWebApplicationContext cwac && !cwac.isActive()) {
				// The context has not yet been refreshed -> provide services such as
				// setting the parent context, setting the application context id, etc
				if (cwac.getParent() == null) {
					// The context instance was injected without an explicit parent -> set
					// the root application context (if any; may be null) as the parent
					//设置父容器
					cwac.setParent(rootContext);
				}
				configureAndRefreshWebApplicationContext(cwac);
			}
		}
		if (wac == null) {
			// No context instance was injected at construction time -> see if one
			// has been registered in the servlet context. If one exists, it is assumed
			// that the parent context (if any) has already been set and that the
			// user has performed any initialization such as setting the context id
			//获取ServletContext，之前通过setAttribute设置到了ServletContext中，现在通过getAttribute获取到
			wac = findWebApplicationContext();
		}
		if (wac == null) {
			// No context instance is defined for this servlet -> create a local one
			//创建WebApplicationContext，设置环境environment、父容器，本地资源文件
			wac = createWebApplicationContext(rootContext);
		}

		if (!this.refreshEventReceived) {
			// Either the context is not a ConfigurableApplicationContext with refresh
			// support or the context injected at construction time had already been
			// refreshed -> trigger initial onRefresh manually here.
			synchronized (this.onRefreshMonitor) {
			//刷新，也是模板模式，空方法，让子类重写进行逻辑处理，而子类DispatcherServlet重写了它
				onRefresh(wac);
			}
		}
        //用setAttribute()，将容器设置到ServletContext中
		if (this.publishContext) {
			// Publish the context as a servlet context attribute.
			String attrName = getServletContextAttributeName();
			getServletContext().setAttribute(attrName, wac);
		}

		return wac;
	}
```

可见其主要作用为将 `Spring` 和 `Servler` 进行一个关联。

接来下，我们看子类 `DispatcherServlet` 复写的 `onRefresh()`方法。方法如下所示

```java
@Override
	protected void onRefresh(ApplicationContext context) {
		initStrategies(context);
	}

	/**
	 * Initialize the strategy objects that this servlet uses.
	 * <p>May be overridden in subclasses in order to initialize further strategy objects.
	 */
	protected void initStrategies(ApplicationContext context) {
	    //解析请求
		initMultipartResolver(context);
		//国际化
		initLocaleResolver(context);
		//主题
		initThemeResolver(context);
		//处理Controller的方法和url映射关系
		initHandlerMappings(context);
		//初始化适配器，多样写法Controller的适配处理，实现最后返回都是ModelAndView
		initHandlerAdapters(context);
		//初始化异常处理器
		initHandlerExceptionResolvers(context);
		//初始化视图转发
		initRequestToViewNameTranslator(context);
		//初始化视图解析器，将ModelAndView保存的视图信息，转换为一个视图，输出数据
		initViewResolvers(context);
		//初始化映射处理器
		initFlashMapManager(context);
	}
```

可见做了一系列的初始化,这些都是`DispatcherServlet`的私有方法

下面我们总结下这三个servlet的职责和作用

- HttpServletBean

主要做一些初始化工作，解析 `web.xml` 中配置的参数到Servlet上，比如`init-param`中配置的参数。提供 `initServletBean()` 模板方法，给子类 `FrameworkServlet`实现。

- FrameworkServlet

将Servlet和SpringIoC容器关联。主要是初始化其中的 `WebApplicationContext`，它代表SpringMVC的上下文，它有一个父上下文，就是 `web.xml` 配置文件中配置的 `ContextLoaderListener` 监听器，监听器进行初始化容器上下文。（ssm配置时spring配置文件的监听器）
 提供了 `onRefresh()` 模板方法，给子类 `DispatcherServlet` 实现，作为初始化入口方法。

- DispatcherServlet

最后的子类，作为前端控制器，初始化各种组件，比如请求映射、视图解析、异常处理、请求处理等。



> 下面我们开始分析各个组件的初始化流程

首先`HandlerMapping` 处理器映射，我们可以看到`DispatcherServlet` 有许多成员变量，我们把相关的成员变量也带上

```java
//处理器映射集合
private List<HandlerMapping> handlerMappings;
//一个开关，标识是否获取所有的处理器映射，如果为false，则搜寻指定名为的 handlerMapping 的 Bean实例
private boolean detectAllHandlerMappings = true;
//指定的Bean的名称
public static final String HANDLER_MAPPING_BEAN_NAME = "handlerMapping";

private void initHandlerMappings(ApplicationContext context) {
      //清空集合
		this.handlerMappings = null;
        //一个开关，默认为true，设置为false，才走else的逻辑
		if (this.detectAllHandlerMappings) {
			// Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
			//重点：在容器中找到所有HandlerMapping
			Map<String, HandlerMapping> matchingBeans =
					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
					//找到了，进行排序，保证顺序
			if (!matchingBeans.isEmpty()) {
				this.handlerMappings = new ArrayList<>(matchingBeans.values());
				// We keep HandlerMappings in sorted order.
				AnnotationAwareOrderComparator.sort(this.handlerMappings);
			}
		}
		else {
			try {
			   //指定搜寻指定名为 handlerMapping 的 HandlerMapping 实例
				HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
				this.handlerMappings = Collections.singletonList(hm);
			}
			catch (NoSuchBeanDefinitionException ex) {
				// Ignore, we'll add a default HandlerMapping later.
			}
		}

		// Ensure we have at least one HandlerMapping, by registering
		// a default HandlerMapping if no other mappings are found.
		 //也找不到映射关系，设置一个默认的
		if (this.handlerMappings == null) {
		 //从配置文件中获取配置的组件，其他组件找不到时，也是调用这个方法进行默认配置
			this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
			if (logger.isTraceEnabled()) {
				logger.trace("No HandlerMappings declared for servlet '" + getServletName() +
						"': using default strategies from DispatcherServlet.properties");
			}
		}

		for (HandlerMapping mapping : this.handlerMappings) {
			if (mapping.usesPathPatterns()) {
				this.parseRequestPath = true;
				break;
			}
		}
	}
```



如果找不到任何一个映射关系，会通过 `getDefaultStrategies` 方法，从配置文件中获取默认配置。其他组件找不到时，也是调用这个方法进行默认配置

配置文件名：`DispatcherServlet.properties`。会加入2个默认的映射关系类 `BeanNameUrlHandlerMapping` 、 `RequestMappingHandlerMapping`。

下面看下`HandlerAdapter` 处理器适配器的初始化流程

`HandlerAdapter` 的初始化逻辑和上面的 `HandlerMapping` 基本一样。从容器中搜寻所有 `HandlerAdapter` 的实例。
 如果找不到，则从配置文件中获取`默认` 的 `HandlerAdapter`。

```java
private List<HandlerAdapter> handlerAdapters;
//和上面HandlerMapping一样，一个开关，是否搜寻容器中所有的HandlerAdapter，如果为false，则搜寻指定名为 handlerAdapter 的Bean
private boolean detectAllHandlerAdapters = true;
//指定的HandlerAdapter实例
public static final String HANDLER_ADAPTER_BEAN_NAME = "handlerAdapter";

private void initHandlerAdapters(ApplicationContext context) {
    //清空集合
    this.handlerAdapters = null;

    //也是一个开关，默认true，搜寻容器中所有的HandlerAdapter
    if (this.detectAllHandlerAdapters) {
        Map<String, HandlerAdapter> matchingBeans =
                BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerAdapter.class, true, false);
        //找到了，进行排序，保证HandlerAdapter是有序的
        if (!matchingBeans.isEmpty()) {
            this.handlerAdapters = new ArrayList<>(matchingBeans.values());
            AnnotationAwareOrderComparator.sort(this.handlerAdapters);
        }
    }
    else {
        //指定找名为 handlerAdapter 的HandlerAdapter
        try {
            HandlerAdapter ha = context.getBean(HANDLER_ADAPTER_BEAN_NAME, HandlerAdapter.class);
            this.handlerAdapters = Collections.singletonList(ha);
        }
        catch (NoSuchBeanDefinitionException ex) {
            // Ignore, we'll add a default HandlerAdapter later.
        }
    }

    //没有找一个HandlerAdapter，从配置文件中获取默认的HandlerAdapter
    if (this.handlerAdapters == null) {
        this.handlerAdapters = getDefaultStrategies(context, HandlerAdapter.class);
        if (logger.isTraceEnabled()) {
            logger.trace("No HandlerAdapters declared for servlet '" + getServletName() +
                    "': using default strategies from DispatcherServlet.properties");
        }
    }
}
```

HandlerExceptionResolver 异常处理器

和上面的一样，从容器中搜寻所有的异常处理器的实例，也有一个开关去搜索指定名称的异常处理器。

```java
private List<HandlerExceptionResolver> handlerExceptionResolvers;
//开关，是否搜索所有的异常处理器，设置为false，就会找下面名为 handlerExceptionResolver 的Bean实例
private boolean detectAllHandlerExceptionResolvers = true;
//指定名为 handlerExceptionResolver 的实例
public static final String HANDLER_EXCEPTION_RESOLVER_BEAN_NAME = "handlerExceptionResolver";

private void initHandlerExceptionResolvers(ApplicationContext context) {
    //清空集合
    this.handlerExceptionResolvers = null;

    //开关，默认true
    if (this.detectAllHandlerExceptionResolvers) {
        //搜寻所有的异常处理器
        Map<String, HandlerExceptionResolver> matchingBeans = BeanFactoryUtils
                .beansOfTypeIncludingAncestors(context, HandlerExceptionResolver.class, true, false);
        //搜寻到了
        if (!matchingBeans.isEmpty()) {
            this.handlerExceptionResolvers = new ArrayList<>(matchingBeans.values());
            //排序
            AnnotationAwareOrderComparator.sort(this.handlerExceptionResolvers);
        }
    }
    else {
        try {
            HandlerExceptionResolver her =
                    context.getBean(HANDLER_EXCEPTION_RESOLVER_BEAN_NAME, HandlerExceptionResolver.class);
            this.handlerExceptionResolvers = Collections.singletonList(her);
        }
        catch (NoSuchBeanDefinitionException ex) {
            // Ignore, no HandlerExceptionResolver is fine too.
        }
    }

    //一个异常处理器都没有，从配置文件中获取默认的
    if (this.handlerExceptionResolvers == null) {
        this.handlerExceptionResolvers = getDefaultStrategies(context, HandlerExceptionResolver.class);
        if (logger.isTraceEnabled()) {
            logger.trace("No HandlerExceptionResolvers declared in servlet '" + getServletName() +
                    "': using default strategies from DispatcherServlet.properties");
        }
    }
}
```

ViewResolver 视图解析器

视图解析器和上面的解析器逻辑一样，先有开关决定是搜寻容器中所有的，还是搜寻指定名称的。

```java
private List<ViewResolver> viewResolvers;
//开关
private boolean detectAllViewResolvers = true;
//指定名称
public static final String VIEW_RESOLVER_BEAN_NAME = "viewResolver";

private void initViewResolvers(ApplicationContext context) {
    //清空集合
    this.viewResolvers = null;

    if (this.detectAllViewResolvers) {
        //搜寻所有视图解析器
        Map<String, ViewResolver> matchingBeans =
                BeanFactoryUtils.beansOfTypeIncludingAncestors(context, ViewResolver.class, true, false);
        if (!matchingBeans.isEmpty()) {
            this.viewResolvers = new ArrayList<>(matchingBeans.values());
            //排序
            AnnotationAwareOrderComparator.sort(this.viewResolvers);
        }
    }
    else {
        try {
            //搜寻指定名为 viewResolver 的视图解析器Bean
            ViewResolver vr = context.getBean(VIEW_RESOLVER_BEAN_NAME, ViewResolver.class);
            this.viewResolvers = Collections.singletonList(vr);
        }
        catch (NoSuchBeanDefinitionException ex) {
            // Ignore, we'll add a default ViewResolver later.
        }
    }

    //没有找到任何一个视图解析器，从配置文件中读取
    if (this.viewResolvers == null) {
        this.viewResolvers = getDefaultStrategies(context, ViewResolver.class);
        if (logger.isTraceEnabled()) {
            logger.trace("No ViewResolvers declared for servlet '" + getServletName() +
                    "': using default strategies from DispatcherServlet.properties");
        }
    }
```

> 可见初始化的流程都十分相似

下面我们分析请求流程

当请求进入时，我们都知道会调用Servlet的 `service()` 方法，我们试着去 `DispatchServlet` 中搜索，发现没有。我们去到父类 `FrameworkServlet` 找到了。我们看下代码

```java
	@Override
	protected void service(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

        //获取请求方式
		HttpMethod httpMethod = HttpMethod.valueOf(request.getMethod());
		//如果是PATCH或获取不到，则走自己的processRequest()方法
		if (HttpMethod.PATCH.equals(httpMethod)) {
			processRequest(request, response);
		}
		else {
		//其它情况走父类的HttpServlet
			super.service(request, response);
		}
	}
```

我们进入父类HttpServlet，确切的说是父类的父类HttpServletBean的父类HttpServlet区分

```java
protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String method = req.getMethod();

        if (method.equals(METHOD_GET)) {
            long lastModified = getLastModified(req);
            if (lastModified == -1) {
                // servlet doesn't support if-modified-since, no reason
                // to go through further expensive logic
                doGet(req, resp);
            } else {
                long ifModifiedSince = req.getDateHeader(HEADER_IFMODSINCE);
                if (ifModifiedSince < lastModified) {
                    // If the servlet mod time is later, call doGet()
                    // Round down to the nearest second for a proper compare
                    // A ifModifiedSince of -1 will always be less
                    maybeSetLastModified(resp, lastModified);
                    doGet(req, resp);
                } else {
                    resp.setStatus(HttpServletResponse.SC_NOT_MODIFIED);
                }
            }

        } else if (method.equals(METHOD_HEAD)) {
            long lastModified = getLastModified(req);
            maybeSetLastModified(resp, lastModified);
            doHead(req, resp);

        } else if (method.equals(METHOD_POST)) {
            doPost(req, resp);

        } else if (method.equals(METHOD_PUT)) {
            doPut(req, resp);

        } else if (method.equals(METHOD_DELETE)) {
            doDelete(req, resp);

        } else if (method.equals(METHOD_OPTIONS)) {
            doOptions(req, resp);

        } else if (method.equals(METHOD_TRACE)) {
            doTrace(req, resp);

        } else {
            //
            // Note that this means NO servlet supports whatever
            // method was requested, anywhere on this server.
            //

            String errMsg = lStrings.getString("http.method_not_implemented");
            Object[] errArgs = new Object[1];
            errArgs[0] = method;
            errMsg = MessageFormat.format(errMsg, errArgs);

            resp.sendError(HttpServletResponse.SC_NOT_IMPLEMENTED, errMsg);
        }
    }
```

以上这段是servlet-api的实现，用来区分请求，但是springmvc在`FrameworkServlet`中重写了这些方法（如下所示），所以父类的仅仅是用来区分请求

```java
/**
	 * Delegate GET requests to processRequest/doService.
	 * <p>Will also be invoked by HttpServlet's default implementation of {@code doHead},
	 * with a {@code NoBodyResponse} that just captures the content length.
	 * @see #doService
	 * @see #doHead
	 */
	@Override
	protected final void doGet(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		processRequest(request, response);
	}

	/**
	 * Delegate POST requests to {@link #processRequest}.
	 * @see #doService
	 */
	@Override
	protected final void doPost(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		processRequest(request, response);
	}

	/**
	 * Delegate PUT requests to {@link #processRequest}.
	 * @see #doService
	 */
	@Override
	protected final void doPut(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		processRequest(request, response);
	}

	/**
	 * Delegate DELETE requests to {@link #processRequest}.
	 * @see #doService
	 */
	@Override
	protected final void doDelete(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		processRequest(request, response);
	}

	/**
	 * Delegate OPTIONS requests to {@link #processRequest}, if desired.
	 * <p>Applies HttpServlet's standard OPTIONS processing otherwise,
	 * and also if there is still no 'Allow' header set after dispatching.
	 * @see #doService
	 */
	@Override
	protected void doOptions(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		if (this.dispatchOptionsRequest || CorsUtils.isPreFlightRequest(request)) {
			processRequest(request, response);
			if (response.containsHeader(HttpHeaders.ALLOW)) {
				// Proper OPTIONS response coming from a handler - we're done.
				return;
			}
		}

		// Use response wrapper in order to always add PATCH to the allowed methods
		super.doOptions(request, new HttpServletResponseWrapper(response) {
			@Override
			public void setHeader(String name, String value) {
				if (HttpHeaders.ALLOW.equals(name)) {
					value = (StringUtils.hasLength(value) ? value + ", " : "") + HttpMethod.PATCH.name();
				}
				super.setHeader(name, value);
			}
		});
	}

	/**
	 * Delegate TRACE requests to {@link #processRequest}, if desired.
	 * <p>Applies HttpServlet's standard TRACE processing otherwise.
	 * @see #doService
	 */
	@Override
	protected void doTrace(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		if (this.dispatchTraceRequest) {
			processRequest(request, response);
			if ("message/http".equals(response.getContentType())) {
				// Proper TRACE response coming from a handler - we're done.
				return;
			}
		}
		super.doTrace(request, response);
	}
```

我们从以上重写的代码可知最终都调用了`processRequest`方法我们看下代码

```java
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		long startTime = System.currentTimeMillis();
		Throwable failureCause = null;

		LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
		LocaleContext localeContext = buildLocaleContext(request);

		RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
		ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
		asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());

		initContextHolders(request, localeContext, requestAttributes);

		try {
		    //重点，doService()是一个抽象方法，强制让子类进行复写
			doService(request, response);
		}
		catch (ServletException | IOException ex) {
			failureCause = ex;
			throw ex;
		}
		catch (Throwable ex) {
			failureCause = ex;
			throw new NestedServletException("Request processing failed", ex);
		}

		finally {
			resetContextHolders(request, previousLocaleContext, previousAttributes);
			if (requestAttributes != null) {
				requestAttributes.requestCompleted();
			}
			logResult(request, response, failureCause, asyncManager);
			publishRequestHandledEvent(request, response, startTime, failureCause);
		}
	}
```

一大片的处理和设置，不是我们的重点，主要是 `doService()` 这个方法，它是一个抽象方法，强制让子类进行复写。
 所以最终子类 `DispatcherServlet` 肯定会复写 `doService()` 方法。

我们进入`DispatcherServlet` 的`doService`方法

```java
@Override
	protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
		//打印请求
		logRequest(request);

		// Keep a snapshot of the request attributes in case of an include,
		// to be able to restore the original attributes after the include.
		Map<String, Object> attributesSnapshot = null;
		if (WebUtils.isIncludeRequest(request)) {
			attributesSnapshot = new HashMap<>();
			Enumeration<?> attrNames = request.getAttributeNames();
			while (attrNames.hasMoreElements()) {
				String attrName = (String) attrNames.nextElement();
				if (this.cleanupAfterInclude || attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)) {
					attributesSnapshot.put(attrName, request.getAttribute(attrName));
				}
			}
		}

		// Make framework objects available to handlers and view objects.
		 //设置组件到请求域中，给后续的其他组件可以获取到
		request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
		request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
		request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
		request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

		if (this.flashMapManager != null) {
			FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
			if (inputFlashMap != null) {
				request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
			}
			request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
			request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);
		}

		RequestPath previousRequestPath = null;
		if (this.parseRequestPath) {
			previousRequestPath = (RequestPath) request.getAttribute(ServletRequestPathUtils.PATH_ATTRIBUTE);
			ServletRequestPathUtils.parseAndCache(request);
		}

		try {
		    //重点：主要的组件分发处理逻辑在 doDispatch() 方法
			doDispatch(request, response);
		}
		finally {
			if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
				// Restore the original attribute snapshot, in case of an include.
				if (attributesSnapshot != null) {
					restoreAttributesAfterInclude(request, attributesSnapshot);
				}
			}
			if (this.parseRequestPath) {
				ServletRequestPathUtils.setParsedRequestPath(previousRequestPath, request);
			}
		}
	}
```

doDispatch() 分发请求给各个组件处理，这个是请求分发的的关键。

在开始这段分析之前，我们还是老样子，先写结论，带着结论去看源码会比较清晰

分发步骤：

1. `getHandler()`，获取本次请求的处理器执行链，包括Controller和拦截器，它们组合成一个执行链 `HandlerExecutionChain`。
2. `getHandlerAdapter()`，获取处理器的适配器，因为有很多种处理器的实现方式，例如直接是Servlet作为处理器、实现Controller接口、使用Controller注解等，每个接口方法的返回值各式各样，所以这里使用了适配器模式，让适配器对处理器的返回值统一输出为ModelAndView。
3. `mappedHandler.applyPreHandle`，责任链模式调用处理器链中的拦截器的 `preHandle()` 方法，代表请求准备进行处理。拦截器可拦截处理。如果拦截器拦截了，则继续往下走。
4. `ha.handle()`，调用适配器的处理方法，传入处理器，调用处理器接口方法，并适配处理器的结果为ModelAndView。
5. `mappedHandler.applyPostHandle`，遍历调用处理器执行链中的拦截器的 `postHandle()` 后置处理方法，代表请求以被处理，但视图还未渲染
6. `processDispatchResult()`，处理视图和结果，调用视图处理器，将真正的视图创建，并对视图数据进行渲染。以及渲染完毕，调用拦截器的 `afterCompletion()`方法，代表视图渲染完毕。
7. `mappedHandler.applyAfterConcurrentHandlingStarted`，调用处理器执行链中的拦截器，无论当前请求处理成功，还是失败，都处理。

下面我们进入`doDispatch`，我们先看下这个方法自带的注释的翻译

*处理对处理程序的实际调度。 <p>将通过依次应用 servlet 的 HandlerMappings 获得处理程序。 HandlerAdapter 将通过查询 servlet 安装的 HandlerAdapters 来找到第一个支持处理程序类的。 <p>所有 HTTP 方法都由该方法处理。由 HandlerAdapters 或处理程序自己决定哪些方法是可接受的。*

> 我们还需要知道一个概念springmvc是基于拦截器实现的

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		//本次请求的处理器以及拦截器，它们组合成一个执行链
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
			   //检查是否是文件上传请求。是的话，做一些处理
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);

				// Determine handler for the current request.
				//重点：找到本次请求的处理器以及拦截器
				mappedHandler = getHandler(processedRequest);
				//找不到处理器处理，响应404
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

				// Determine handler adapter for the current request.
				//重点：找到本次请求中，处理器的适配器
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				// Process last-modified header, if supported by the handler.
				String method = request.getMethod();
				boolean isGet = HttpMethod.GET.matches(method);
				if (isGet || HttpMethod.HEAD.matches(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}

                 //重点：处理前，责任链模式 回调拦截器的 preHandle() 方法，如果拦截了，则不继续往下走了
                 //返回true代表放心，false为拦截
				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// Actually invoke the handler.
				//重点：调用适配器的处理方法，传入处理器，让适配器将处理器的结果转换成统一的ModelAndView
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}
                //如果找不到默认的视图，则设置默认的视图
				applyDefaultViewName(processedRequest, mv);
				//重点：处理完成，调用拦截器的 postHandle() 后置处理方法
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}
			catch (Throwable err) {
				// As of 4.3, we're processing Errors thrown from handler methods as well,
				// making them available for @ExceptionHandler methods and other scenarios.
				dispatchException = new NestedServletException("Handler dispatch failed", err);
			}
			//重点：分发结果，让视图解析器解析视图，渲染视图和数据
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}
		catch (Throwable err) {
			triggerAfterCompletion(processedRequest, response, mappedHandler,
					new NestedServletException("Handler processing failed", err));
		}
		finally {
			if (asyncManager.isConcurrentHandlingStarted()) {
				// Instead of postHandle and afterCompletion
				if (mappedHandler != null) {
				  //重点：视图渲染完成，调用拦截器的 afterConcurrentHandlingStarted() 方法
					mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
				}
			}
			else {
				// Clean up any resources used by a multipart request.
				if (multipartRequestParsed) {
					cleanupMultipart(processedRequest);
				}
			}
		}
	}
```

下面就对上面的每个步骤，进行分析。

> 首先getHandler() 搜寻本次请求的处理器对象,

责任链模式，遍历handlerMappings集合，找到处理器和拦截器，会调用到 `AbstractHandlerMapping`的 `getHandler()`方法。
 最后将处理器和拦截器都封装到 `HandlerExecutionChain` 这个处理器执行链对象中。

我们看下`AbstractHandlerMapping`的 `getHandler()`方法

```java
	@Override
	@Nullable
	public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		//模板方法，获取处理器，具体子类进行实现
		Object handler = getHandlerInternal(request);
		 //如果没有获取到，则使用默认的处理器
		if (handler == null) {
			handler = getDefaultHandler();
		}
		//默认的也没有，那就返回null了
		if (handler == null) {
			return null;
		}
		// Bean name or resolved handler?
		//如果处理器是字符串类型，则在IoC容器中搜寻实例
		if (handler instanceof String handlerName) {
			handler = obtainApplicationContext().getBean(handlerName);
		}

		// Ensure presence of cached lookupPath for interceptors and others
		if (!ServletRequestPathUtils.hasCachedPath(request)) {
			initLookupPath(request);
		}
         //构成处理器执行链，主要是添加拦截器
		HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);

		if (logger.isTraceEnabled()) {
			logger.trace("Mapped to " + handler);
		}
		else if (logger.isDebugEnabled() && !DispatcherType.ASYNC.equals(request.getDispatcherType())) {
			logger.debug("Mapped to " + executionChain.getHandler());
		}

		if (hasCorsConfigurationSource(handler) || CorsUtils.isPreFlightRequest(request)) {
			CorsConfiguration config = getCorsConfiguration(handler, request);
			if (getCorsConfigurationSource() != null) {
				CorsConfiguration globalConfig = getCorsConfigurationSource().getCorsConfiguration(request);
				config = (globalConfig != null ? globalConfig.combine(config) : config);
			}
			if (config != null) {
				config.validateAllowCredentials();
			}
			executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
		}

		return executionChain;
	}
```

可以看到调用`getHandlerInternal`获取handler,它是`AbstractHandlerMapping`的抽象方法，主要有以下实现

![avatar](https://picture.zhanghong110.top/docsify/16408283985578.png)

主要实现有2个，第一个是AbstractUrlHandlerMapping，一般用它的子类SimpleUrlHandlerMapping，这种方式需要在xml配置文件中配置，已经很少用了。第二个是AbstractHandlerMethodMapping，就是处理我们@Controller和@RequestMapping的。

AbstractHandlerMethodMapping查询注解的方式，我们简单追踪下可以看它的如下调用链

```java
	
	//AbstractHandlerMethodMapping中
	@Override
	public void afterPropertiesSet() {
		initHandlerMethods();
	}
	
	/**
	 * Scan beans in the ApplicationContext, detect and register handler methods.
	 * @see #getCandidateBeanNames()
	 * @see #processCandidateBean
	 * @see #handlerMethodsInitialized
	 */
	protected void initHandlerMethods() {
		for (String beanName : getCandidateBeanNames()) {
			if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
				processCandidateBean(beanName);
			}
		}
		handlerMethodsInitialized(getHandlerMethods());
	}
	
	protected void processCandidateBean(String beanName) {
		Class<?> beanType = null;
		try {
			beanType = obtainApplicationContext().getType(beanName);
		}
		catch (Throwable ex) {
			// An unresolvable bean type, probably from a lazy bean - let's ignore it.
			if (logger.isTraceEnabled()) {
				logger.trace("Could not resolve type for bean '" + beanName + "'", ex);
			}
		}
		//这里关键
		if (beanType != null && isHandler(beanType)) {
			detectHandlerMethods(beanName);
		}
	}
	
	//isHandler一般实现RequestMappingHandlerMapping如下
	
		@Override
	protected boolean isHandler(Class<?> beanType) {
		return AnnotatedElementUtils.hasAnnotation(beanType, Controller.class);
	}
	
	
	//最终调用了AnnotatedElementUtils中的工具类查询注解
		public static boolean hasAnnotation(AnnotatedElement element, Class<? extends Annotation> annotationType) {
		// Shortcut: directly present on the element, with no merging needed?
		if (AnnotationFilter.PLAIN.matches(annotationType) ||
				AnnotationsScanner.hasPlainJavaAnnotationsOnly(element)) {
			return element.isAnnotationPresent(annotationType);
		}
		// Exhaustive retrieval of merged annotations...
		return findAnnotations(element).isPresent(annotationType);
	}
	
```

!>这一步的话实际的处理比较复杂的,没有上面分析的那么直接，包括一些具体的处理，会在补充里面说明



> 接下来分析适配组件，`getHandlerAdapter()`

getHandlerAdapter() 获取处理器对应的适配器,我们返回`doDispatch`方法进入getHandlerAdapter()代码如下

```java
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
		if (this.handlerAdapters != null) {
		    //责任链模式，遍历调用适配器集合，调用supports()方法，询问每个适配器，是否支持当前的处理器
            //如果返回true，则代表找到了，停止遍历，返回适配器
			for (HandlerAdapter adapter : this.handlerAdapters) {
				if (adapter.supports(handler)) {
					return adapter;
				}
			}
		}
		throw new ServletException("No adapter for handler [" + handler +
				"]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
	}
```

我们来看看适配器接口，以及它的部分子类

![avatar](https://picture.zhanghong110.top/docsify/16408320519825.png)

`HttpRequestHandlerAdapter`，适配 `HttpRequestHandler` 作为handler的适配器。

```java
public class HttpRequestHandlerAdapter implements HandlerAdapter {

	@Override
	public boolean supports(Object handler) {
		return (handler instanceof HttpRequestHandler);
	}

	@Override
	@Nullable
	public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		((HttpRequestHandler) handler).handleRequest(request, response);
		return null;
	}

	@Override
	@SuppressWarnings("deprecation")
	public long getLastModified(HttpServletRequest request, Object handler) {
		if (handler instanceof LastModified) {
			return ((LastModified) handler).getLastModified(request);
		}
		return -1L;
	}

}
```

`SimpleServletHandlerAdapter`，适配 `Servlet` 作为handler的适配器

```java
public class SimpleServletHandlerAdapter implements HandlerAdapter {

	@Override
	public boolean supports(Object handler) {
		return (handler instanceof Servlet);
	}

	@Override
	@Nullable
	public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		((Servlet) handler).service(request, response);
		return null;
	}

	@Override
	@SuppressWarnings("deprecation")
	public long getLastModified(HttpServletRequest request, Object handler) {
		return -1;
	}

}
```

`SimpleControllerHandlerAdapter`，适配 `Controller` 接口 作为`handler`的适配器

```java
public class SimpleControllerHandlerAdapter implements HandlerAdapter {

	@Override
	public boolean supports(Object handler) {
		return (handler instanceof Controller);
	}

	@Override
	@Nullable
	public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		return ((Controller) handler).handleRequest(request, response);
	}

	@Override
	@SuppressWarnings("deprecation")
	public long getLastModified(HttpServletRequest request, Object handler) {
		if (handler instanceof LastModified) {
			return ((LastModified) handler).getLastModified(request);
		}
		return -1L;
	}

}
```

根据流程，获取到对应的适配器后，就可以通知拦截器了（比如我们找到了controller的适配器，就可以返回调用具体的controller去处理了）

我们回到 `DispatcherServletdo`的`Dispatch`方法看下`applyPreHandle`方法，来到`HandlerExecutionChain`的`applyPreHandle`

```java
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
		for (int i = 0; i < this.interceptorList.size(); i++) {
			HandlerInterceptor interceptor = this.interceptorList.get(i);
			if (!interceptor.preHandle(request, response, this.handler)) {
				triggerAfterCompletion(request, response, null);
				return false;
			}
			this.interceptorIndex = i;
		}
		return true;
	}
```

通知拦截器进行请求处理前的拦截和附加处理。
如果有一个拦截器返回false，代表拦截，则处理流程被中断，就是拦截了。

回到 `DispatcherServletdo`的`Dispatch`方法，

和前置通知不同，后置通知没有拦截功能，只能是增强。逻辑还是遍历拦截器链，调用拦截器的 `postHandle()` 方法。

```java
void applyPostHandle(HttpServletRequest request, HttpServletResponse response, @Nullable ModelAndView mv)
      throws Exception {

   for (int i = this.interceptorList.size() - 1; i >= 0; i--) {
      HandlerInterceptor interceptor = this.interceptorList.get(i);
      interceptor.postHandle(request, response, this.handler, mv);
   }
}
```

> 之后我们就进入了结果处理流程

因为 `doDispatch()` 的处理流程，SpringMVC都帮我们try-catch了，所以能捕获到异常，并传入该方法。

接着首先判断处理过程中，是否产生了异常，有则用异常处理器处理。
没有异常，则继续往下走，判断是否需要渲染，需要渲染，则进行渲染，最后回调拦截器进行通知。

我们回到 `DispatcherServletdo`的`Dispatch`方法，看下`processDispatchResult`方法

```java
	private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
			@Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
			@Nullable Exception exception) throws Exception {
        //是否显示错误页面
		boolean errorView = false;
        //处理异常
		if (exception != null) {
			if (exception instanceof ModelAndViewDefiningException) {
				logger.debug("ModelAndViewDefiningException encountered", exception);
				mv = ((ModelAndViewDefiningException) exception).getModelAndView();
			}
			else {
				Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
				//处理异常
				mv = processHandlerException(request, response, handler, exception);
				errorView = (mv != null);
			}
		}

		// Did the handler return a view to render?
		//判断处理器是否需要返回视图
		if (mv != null && !mv.wasCleared()) {
		    //重点：渲染
			render(mv, request, response);
			if (errorView) {
				WebUtils.clearErrorRequestAttributes(request);
			}
		}
		else {
			if (logger.isTraceEnabled()) {
				logger.trace("No view rendering, null ModelAndView returned.");
			}
		}

		if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
			// Concurrent handling started during a forward
			return;
		}
        //视图渲染完成，回调拦截器
		if (mappedHandler != null) {
			// Exception (if any) is already handled..
			mappedHandler.triggerAfterCompletion(request, response, null);
		}
	}
	
	//拦截器回调，通知拦截器，视图已被渲染，拦截器可以再做点事情
   void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, @Nullable Exception ex) {
		for (int i = this.interceptorIndex; i >= 0; i--) {
			HandlerInterceptor interceptor = this.interceptorList.get(i);
			try {
				interceptor.afterCompletion(request, response, this.handler, ex);
			}
			catch (Throwable ex2) {
				logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
			}
		}
	}	
```

> 视图解析及渲染

先判断是否需要视图解析器进行视图解析，最后调用解析出来的视图的 `render()` 方法进行渲染操作。

我们进入`processDispatchResult`的`render()` 方法看下

```java
protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
   // Determine locale for request and apply it to the response.
   Locale locale =
         (this.localeResolver != null ? this.localeResolver.resolveLocale(request) : request.getLocale());
   response.setLocale(locale);

  //真正的视图对象
   View view;
   String viewName = mv.getViewName();
   if (viewName != null) {
      // We need to resolve the view name.
       //重点：使用视图解析器，生成正真的视图
      view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
      if (view == null) {
         throw new ServletException("Could not resolve view with name '" + mv.getViewName() +
               "' in servlet with name '" + getServletName() + "'");
      }
   }
   else {
      // No need to lookup: the ModelAndView object contains the actual View object.
      //不需要查找，ModelAndView中已经包含了真正的视图
      view = mv.getView();
      if (view == null) {
         throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a " +
               "View object in servlet with name '" + getServletName() + "'");
      }
   }

   // Delegate to the View object for rendering.
   if (logger.isTraceEnabled()) {
      logger.trace("Rendering view [" + view + "] ");
   }
   try {
      if (mv.getStatus() != null) {
         request.setAttribute(View.RESPONSE_STATUS_ATTRIBUTE, mv.getStatus());
         response.setStatus(mv.getStatus().value());
      }
      //重点：开始渲染
      view.render(mv.getModelInternal(), request, response);
   }
   catch (Exception ex) {
      if (logger.isDebugEnabled()) {
         logger.debug("Error rendering view [" + view + "]", ex);
      }
      throw ex;
   }
}
```

我们进入render方法的`resolveViewName()`方法查看如何进行视图解析

```java
@Nullable
	protected View resolveViewName(String viewName, @Nullable Map<String, Object> model,
			Locale locale, HttpServletRequest request) throws Exception {

		if (this.viewResolvers != null) {
		//遍历视图解析器集合，不同的视图需要不同的解析器进行处理
			for (ViewResolver viewResolver : this.viewResolvers) {
				View view = viewResolver.resolveViewName(viewName, locale);
				if (view != null) {
					return view;
				}
			}
		}
		return null;
	}

```

可以看到，便利了不通的视图解析器，不同的视图用不通的解析器解析

ViewResolver解析器是一个接口，他有几个实现类，对应支持的视图技术。

```java
public interface ViewResolver {

	/**
	 * Resolve the given view by name.
	 * <p>Note: To allow for ViewResolver chaining, a ViewResolver should
	 * return {@code null} if a view with the given name is not defined in it.
	 * However, this is not required: Some ViewResolvers will always attempt
	 * to build View objects with the given name, unable to return {@code null}
	 * (rather throwing an exception when View creation failed).
	 * @param viewName name of the view to resolve
	 * @param locale the Locale in which to resolve the view.
	 * ViewResolvers that support internationalization should respect this.
	 * @return the View object, or {@code null} if not found
	 * (optional, to allow for ViewResolver chaining)
	 * @throws Exception if the view cannot be resolved
	 * (typically in case of problems creating an actual View object)
	 */
	@Nullable
	View resolveViewName(String viewName, Locale locale) throws Exception;

}
```



1. `AbstractCachingViewResolver`，抽象类，支持缓存视图，所有的解析器都继承它，它内部有一个Map，缓存解析过的视图对象，解决效率问题。
2. `UrlBasedViewResolver`，继承于 `AbstractCachingViewResolver`，当我们Controller返回一个字符串，例如success，它就会从我们的xml配置文件中，找到prefix前缀和suffix后缀，和url进行拼接，输出一个完成的视图地址。还有一种就是我们返回 `redirect:`前缀的字符串时，会解析为重定向视图View，进行重定向操作。
3. `InternalResourceViewResolver`，内部资源解析器，继承于上面的 `UrlBasedViewResolver`，所以 `UrlBasedViewResolver` 有的功能呢，它都有，主要用于加载 `/WEB-INF/` 目录下的资源。
4. 还有一些其他不太常用的解析器，这里就不介绍了

> 我们看一下View对象

```java


/**
 * MVC View for a web interaction. Implementations are responsible for rendering
 * content, and exposing the model. A single view exposes multiple model attributes.
 *
 * <p>This class and the MVC approach associated with it is discussed in Chapter 12 of
 * <a href="https://www.amazon.com/exec/obidos/tg/detail/-/0764543857/">Expert One-On-One J2EE Design and Development</a>
 * by Rod Johnson (Wrox, 2002).
 *
 * <p>View implementations may differ widely. An obvious implementation would be
 * JSP-based. Other implementations might be XSLT-based, or use an HTML generation library.
 * This interface is designed to avoid restricting the range of possible implementations.
 *
 * <p>Views should be beans. They are likely to be instantiated as beans by a ViewResolver.
 * As this interface is stateless, view implementations should be thread-safe.
 *
 * @author Rod Johnson
 * @author Arjen Poutsma
 * @author Rossen Stoyanchev
 * @see org.springframework.web.servlet.view.AbstractView
 * @see org.springframework.web.servlet.view.InternalResourceView
 */
public interface View {

	/**
	 * Name of the {@link HttpServletRequest} attribute that contains the response status code.
	 * <p>Note: This attribute is not required to be supported by all View implementations.
	 * @since 3.0
	 */
	String RESPONSE_STATUS_ATTRIBUTE = View.class.getName() + ".responseStatus";

	/**
	 * Name of the {@link HttpServletRequest} attribute that contains a Map with path variables.
	 * The map consists of String-based URI template variable names as keys and their corresponding
	 * Object-based values -- extracted from segments of the URL and type converted.
	 * <p>Note: This attribute is not required to be supported by all View implementations.
	 * @since 3.1
	 */
	String PATH_VARIABLES = View.class.getName() + ".pathVariables";

	/**
	 * The {@link org.springframework.http.MediaType} selected during content negotiation,
	 * which may be more specific than the one the View is configured with. For example:
	 * "application/vnd.example-v1+xml" vs "application/*+xml".
	 * @since 3.2
	 */
	String SELECTED_CONTENT_TYPE = View.class.getName() + ".selectedContentType";


	/**
	 * Return the content type of the view, if predetermined.
	 * <p>Can be used to check the view's content type upfront,
	 * i.e. before an actual rendering attempt.
	 * @return the content type String (optionally including a character set),
	 * or {@code null} if not predetermined
	 */
	@Nullable
	default String getContentType() {
		return null;
	}

	/**
	 * Render the view given the specified model.
	 * <p>The first step will be preparing the request: In the JSP case, this would mean
	 * setting model objects as request attributes. The second step will be the actual
	 * rendering of the view, for example including the JSP via a RequestDispatcher.
	 * @param model a Map with name Strings as keys and corresponding model
	 * objects as values (Map can also be {@code null} in case of empty model)
	 * @param request current HTTP request
	 * @param response he HTTP response we are building
	 * @throws Exception if rendering failed
	 */
	void render(@Nullable Map<String, ?> model, HttpServletRequest request, HttpServletResponse response)
			throws Exception;

}

```

视图解析完成后将会生成view视图对象，而View也是一个接口，他有以下实现类

1.AbstractView，View的抽象类，定义了渲染流程，抽象了一些抽象方法，子类做特殊处理即可，大部分的实现类都继承于它

2.VelocityView，支持Velocity框架生成的页面。

3.FreeMarkerView，支持FreeMarker框架生成的页面。

4.JstlView，支持生成jstl视图。

5.RedirectView，支持生成页面跳转视图。

6.MappingJackson2JsonView，输出Json的视图，使用Jackson库实现Json序列



> 视图的本质就是通过 `Response`对象，进行 `write()` 写出到客户端。



我们跟踪rander进入AbstractView的rander方法如下图所示，看到`renderMergedOutputModel`方法,然后随便找个实现，这里以`AbstractJackson2View`为列子,代码如下

```java
@Override
	protected void renderMergedOutputModel(Map<String, Object> model, HttpServletRequest request,
			HttpServletResponse response) throws Exception {

		ByteArrayOutputStream temporaryStream = null;
		OutputStream stream;

		if (this.updateContentLength) {
			temporaryStream = createTemporaryOutputStream();
			stream = temporaryStream;
		}
		else {java
			stream = response.getOutputStream();
		}

		Object value = filterAndWrapModel(model, request);
		writeContent(stream, value);

		if (temporaryStream != null) {
			writeToResponse(response, temporaryStream);
		}
	}
```

可以看到其实就是输出到客户端。

最后我们再次返回`DispatcherServlet`的`doDispatch()`方法,整体try-catch后，`finally` 代码块，调用拦截器进行最终通知

我们进入applyAfterConcurrentHandlingStarted方法，代码如下

```java
	void applyAfterConcurrentHandlingStarted(HttpServletRequest request, HttpServletResponse response) {
		for (int i = this.interceptorList.size() - 1; i >= 0; i--) {
			HandlerInterceptor interceptor = this.interceptorList.get(i);
			if (interceptor instanceof AsyncHandlerInterceptor asyncInterceptor) {
				try {
					asyncInterceptor.afterConcurrentHandlingStarted(request, response, this.handler);
				}
				catch (Throwable ex) {
					if (logger.isErrorEnabled()) {
						logger.error("Interceptor [" + interceptor + "] failed in afterConcurrentHandlingStarted", ex);
					}
				}
			}
		}
	}
```

遍历到的拦截器必须是 `AsyncHandlerInterceptor` 接口的实现类才行。

> 至此我们就能够比较清晰的了解sprimgmvc的整个处理流程了，我们可以看到基本都是围绕`DispatcherServlet`的`doDispatch`方法进行的



最后我们做个总结



初始化时

- DispatchServlet 初始化，调用 onRefresh 被调用调用，初始化 处理器映射HandlerMapping 、 HandlerAdapter适配器、 异常处理器、 视图解析器 等一系列组件。

请求到达时

- DispatchServlet 的 doService() 被调用，根据请求的Url查找对应的处理器，包含我们的Controller和拦截器。
- 根据处理器，找到对应的HandlerAdapter适配器，因为处理器的形式有很多，例如Servlet作为处理器、实现Controller接口，使用@Controller注解等，返回值都不一，就要使用适配器将结果都适配为ModelAndView。
- 执行适配器的处理方法前，先回调拦截器的前处理方法，这里可以进行拦截，如果拦截了，就不继续流程了。如果不拦截，则继续走，调用适配器的处理方法，在处理方法中，会调用处理器进行处理，就调用到我们的Controller。执行完后，再回调拦截器的后处理回调方法。
- 获取到 ModelAndView 后，交给视图解析器ViewResolver，进行解析视图名称为具体视图实例后，再进行视图和数据的渲染返回给客户端，并回调拦截器视图渲染完后的回调方法。



> 补充

我们稍微看下@Responsebody为啥不会返回视图解析器，还有入参的json格式化处理，这两个问题可能是平时开发中比较常见的疑惑

第一个问题,从头追踪比较长，我们直接写成果，我们debug直到下进入下图方法

```java
	@Nullable
	private HandlerMethodReturnValueHandler selectHandler(@Nullable Object value, MethodParameter returnType) {
		boolean isAsyncValue = isAsyncReturnValue(value, returnType);
		for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
			if (isAsyncValue && !(handler instanceof AsyncHandlerMethodReturnValueHandler)) {
				continue;
			}
			if (handler.supportsReturnType(returnType)) {
				return handler;
			}
		}
		return null;
	}
```

发现最后匹配到的handler是

RequestResponseBodyMethodProcessor,继续跟踪得到

```java
 public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType, ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
       //视图是否需要解析，如果是true就不需要解析
        mavContainer.setRequestHandled(true);
        //创建输入流
        ServletServerHttpRequest inputMessage = this.createInputMessage(webRequest);
        //创建输出流
        ServletServerHttpResponse outputMessage = this.createOutputMessage(webRequest);
        // 写出对应的消息
        this.writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
    }
```

核心方法就是

```java
protected <T> void writeWithMessageConverters(@Nullable T value, MethodParameter returnType,
		ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
		throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

	Object body;
	Class<?> valueType;
	Type targetType;

    //判断传进来的value的值是不是CharSequence类型
	if (value instanceof CharSequence) {
        //转成string
		body = value.toString();
		valueType = String.class;
		targetType = String.class;
	}
	else {//其他的类型，需要我们处理
		body = value;
        //获取返回值的类型并设置上去
		valueType = getReturnValueType(body, returnType);
        //获取目标的类型，包括泛型
		targetType = GenericTypeResolver.resolveType(getGenericType(returnType), returnType.getContainingClass());
	}

    //判断是否是Resource类型，明显这儿不是
	if (isResourceType(value, returnType)) {
		outputMessage.getHeaders().set(HttpHeaders.ACCEPT_RANGES, "bytes");
		if (value != null && inputMessage.getHeaders().getFirst(HttpHeaders.RANGE) != null &&
				outputMessage.getServletResponse().getStatus() == 200) {
			Resource resource = (Resource) value;
			try {
				List<HttpRange> httpRanges = inputMessage.getHeaders().getRange();
				outputMessage.getServletResponse().setStatus(HttpStatus.PARTIAL_CONTENT.value());
				body = HttpRange.toResourceRegions(httpRanges, resource);
				valueType = body.getClass();
				targetType = RESOURCE_REGION_LIST_TYPE;
			}
			catch (IllegalArgumentException ex) {
				outputMessage.getHeaders().set(HttpHeaders.CONTENT_RANGE, "bytes */" + resource.contentLength());
				outputMessage.getServletResponse().setStatus(HttpStatus.REQUESTED_RANGE_NOT_SATISFIABLE.value());
			}
		}
	}

	MediaType selectedMediaType = null;
    //获取媒体类型 这儿取出来的是null
	MediaType contentType = outputMessage.getHeaders().getContentType();
    //这儿也是false，因为我们没有媒体类型
	boolean isContentTypePreset = contentType != null && contentType.isConcrete();
	if (isContentTypePreset) {
		if (logger.isDebugEnabled()) {
			logger.debug("Found 'Content-Type:" + contentType + "' in response");
		}
		selectedMediaType = contentType;
	}
	else {
        //获取对应的request
		HttpServletRequest request = inputMessage.getServletRequest();
        //获取允许的类型
		List<MediaType> acceptableTypes = getAcceptableMediaTypes(request);
        //获取可生产的类型
		List<MediaType> producibleTypes = getProducibleMediaTypes(request, valueType, targetType);

		if (body != null && producibleTypes.isEmpty()) {
			throw new HttpMessageNotWritableException(
					"No converter found for return value of type: " + valueType);
		}
        //要使用的媒体类型
		List<MediaType> mediaTypesToUse = new ArrayList<>();
        //遍历找到要使用的媒体类型
		for (MediaType requestedType : acceptableTypes) {
			for (MediaType producibleType : producibleTypes) {
				if (requestedType.isCompatibleWith(producibleType)) {
					mediaTypesToUse.add(getMostSpecificMediaType(requestedType, producibleType));
				}
			}
		}
		if (mediaTypesToUse.isEmpty()) {
			if (body != null) {
				throw new HttpMediaTypeNotAcceptableException(producibleTypes);
			}
			if (logger.isDebugEnabled()) {
				logger.debug("No match for " + acceptableTypes + ", supported: " + producibleTypes);
			}
			return;
		}

        //进行排序
		MediaType.sortBySpecificityAndQuality(mediaTypesToUse);

        //遍历，找到对应的媒体类型
		for (MediaType mediaType : mediaTypesToUse) {
			if (mediaType.isConcrete()) {
				selectedMediaType = mediaType;
				break;
			}
			else if (mediaType.isPresentIn(ALL_APPLICATION_MEDIA_TYPES)) {
				selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
				break;
			}
		}

		if (logger.isDebugEnabled()) {
			logger.debug("Using '" + selectedMediaType + "', given " +
					acceptableTypes + " and supported " + producibleTypes);
		}
	}

	if (selectedMediaType != null) {
		selectedMediaType = selectedMediaType.removeQualityValue();
        //遍历消息转换器，这儿这个消息转换器，是我们人为添加的，至于什么时候添加的，笔者下面会讲到
		for (HttpMessageConverter<?> converter : this.messageConverters) {
			GenericHttpMessageConverter genericConverter = (converter instanceof GenericHttpMessageConverter ?
					(GenericHttpMessageConverter<?>) converter : null);
            //判断这个消息转换器能不能写这个消息
			if (genericConverter != null ?
					((GenericHttpMessageConverter) converter).canWrite(targetType, valueType, selectedMediaType) :
					converter.canWrite(valueType, selectedMediaType)) {
                //开始准备写
				body = getAdvice().beforeBodyWrite(body, returnType, selectedMediaType,
						(Class<? extends HttpMessageConverter<?>>) converter.getClass(),
						inputMessage, outputMessage);
				if (body != null) {
					Object theBody = body;
					LogFormatUtils.traceDebug(logger, traceOn ->
							"Writing [" + LogFormatUtils.formatValue(theBody, !traceOn) + "]");
                    //添加内容处置标题
					addContentDispositionHeader(inputMessage, outputMessage);
					if (genericConverter != null) {
                        //开始写出去
						genericConverter.write(body, targetType, selectedMediaType, outputMessage);
					}
					else {
						((HttpMessageConverter) converter).write(body, selectedMediaType, outputMessage);
					}
				}
				else {
					if (logger.isDebugEnabled()) {
						logger.debug("Nothing to write: null body");
					}
				}
				return;
			}
		}
	}

    //抛出异常
	if (body != null) {
		Set<MediaType> producibleMediaTypes =
				(Set<MediaType>) inputMessage.getServletRequest()
						.getAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);

		if (isContentTypePreset || !CollectionUtils.isEmpty(producibleMediaTypes)) {
			throw new HttpMessageNotWritableException(
					"No converter for [" + valueType + "] with preset Content-Type '" + contentType + "'");
		}
		throw new HttpMediaTypeNotAcceptableException(this.allSupportedMediaTypes);
	}
}

```

上面的处理逻辑就是先处理对应的类型，然后调用对应的消息转化器中的write方法，将内容写出去。那么这儿的消息转换器什么时候添加的？我们的配置类是继承了WebMvcConfigurationSupport，所以这个类也是会被解析的。这里还可以看出，一些具体比如返回头的设置之类的也会在这里处理见`getReturnValueType`


> 至此整个`@ResponseBody`注解的处理流程就讲完了,做个总结的话就是在handle处理这一步去做了具体的处理



下一个问题入参序列化的地方在哪

这个问题我们先来根据我们上述分析经验猜测下，肯定也是在handle这一步去处理的，我们跟进debug源码，很快来到了`RequestMappingHandlerAdapter`类，

```java
  protected boolean supportsInternal(HandlerMethod handlerMethod) {
        return true;
    }

    protected ModelAndView handleInternal(HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
        this.checkRequest(request);
        ModelAndView mav;
        if (this.synchronizeOnSession) {
            HttpSession session = request.getSession(false);
            if (session != null) {
                Object mutex = WebUtils.getSessionMutex(session);
                synchronized(mutex) {
                    mav = this.invokeHandlerMethod(request, response, handlerMethod);
                }
            } else {
                mav = this.invokeHandlerMethod(request, response, handlerMethod);
            }
        } else {
            mav = this.invokeHandlerMethod(request, response, handlerMethod);
        }

        if (!response.containsHeader("Cache-Control")) {
            if (this.getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
                this.applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
            } else {
                this.prepareResponse(response);
            }
        }

        return mav;
    }
```

发现当`invokeHandlerMethod`这个方法执行完后就有了参数我们进去看看

```java
@Nullable
	protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

		ServletWebRequest webRequest = new ServletWebRequest(request, response);
		try {
			WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
			ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

			ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
			if (this.argumentResolvers != null) {
				invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
			}
			if (this.returnValueHandlers != null) {
				invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
			}
			invocableMethod.setDataBinderFactory(binderFactory);
			invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

			ModelAndViewContainer mavContainer = new ModelAndViewContainer();
			mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
			modelFactory.initModel(webRequest, mavContainer, invocableMethod);
			mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

			AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
			asyncWebRequest.setTimeout(this.asyncRequestTimeout);

			WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
			asyncManager.setTaskExecutor(this.taskExecutor);
			asyncManager.setAsyncWebRequest(asyncWebRequest);
			asyncManager.registerCallableInterceptors(this.callableInterceptors);
			asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

			if (asyncManager.hasConcurrentResult()) {
				Object result = asyncManager.getConcurrentResult();
				mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
				asyncManager.clearConcurrentResult();
				LogFormatUtils.traceDebug(logger, traceOn -> {
					String formatted = LogFormatUtils.formatValue(result, !traceOn);
					return "Resume with async result [" + formatted + "]";
				});
				invocableMethod = invocableMethod.wrapConcurrentResult(result);
			}
            //此处
			invocableMethod.invokeAndHandle(webRequest, mavContainer);
			if (asyncManager.isConcurrentHandlingStarted()) {
				return null;
			}

			return getModelAndView(mavContainer, modelFactory, webRequest);
		}
		finally {
			webRequest.requestCompleted();
		}
	}
```

我们发现当以下方法执行后则有了参数，所以我们继续进去

```java
invocableMethod.invokeAndHandle(webRequest, mavContainer);
```

代码如下

```java
 public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
 //此处
        Object returnValue = this.invokeForRequest(webRequest, mavContainer, providedArgs);
        this.setResponseStatus(webRequest);
        if (returnValue == null) {
            if (this.isRequestNotModified(webRequest) || this.getResponseStatus() != null || mavContainer.isRequestHandled()) {
                mavContainer.setRequestHandled(true);
                return;
            }
        } else if (StringUtils.hasText(this.getResponseStatusReason())) {
            mavContainer.setRequestHandled(true);
            return;
        }

        mavContainer.setRequestHandled(false);
        Assert.state(this.returnValueHandlers != null, "No return value handlers");

        try {
            this.returnValueHandlers.handleReturnValue(returnValue, this.getReturnValueType(returnValue), mavContainer, webRequest);
        } catch (Exception var6) {
            if (this.logger.isTraceEnabled()) {
                this.logger.trace(this.getReturnValueHandlingErrorMessage("Error handling return value", returnValue), var6);
            }

            throw var6;
        }
    }
```

this.invokeForRequest执行后就有了参数，我们进去，代码如下

```java
@Nullable
	public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

		Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
		if (logger.isTraceEnabled()) {
			logger.trace("Invoking '" + ClassUtils.getQualifiedMethodName(getMethod(), getBeanType()) +
					"' with arguments " + Arrays.toString(args));
		}
		//此处
		Object returnValue = doInvoke(args);
		if (logger.isTraceEnabled()) {
			logger.trace("Method [" + ClassUtils.getQualifiedMethodName(getMethod(), getBeanType()) +
					"] returned [" + returnValue + "]");
		}
		return returnValue;
	}
```

我们可见关键方法doInvoke

```java
protected Object doInvoke(Object... args) throws Exception {
		ReflectionUtils.makeAccessible(getBridgedMethod());
		try {
		//此处生成实体类
			return getBridgedMethod().invoke(getBean(), args);
		}
		catch (IllegalArgumentException ex) {
			assertTargetBean(getBridgedMethod(), getBean(), args);
			String text = (ex.getMessage() != null ? ex.getMessage() : "Illegal argument");
			throw new IllegalStateException(getInvocationErrorMessage(text, args), ex);
		}
		catch (InvocationTargetException ex) {
			// Unwrap for HandlerExceptionResolvers ...
			Throwable targetException = ex.getTargetException();
			if (targetException instanceof RuntimeException) {
				throw (RuntimeException) targetException;
			}
			else if (targetException instanceof Error) {
				throw (Error) targetException;
			}
			else if (targetException instanceof Exception) {
				throw (Exception) targetException;
			}
			else {
				String text = getInvocationErrorMessage("Failed to invoke handler method", args);
				throw new IllegalStateException(text, targetException);
			}
		}
	}
```

进去后发现反射调用的地方参数已经被序列化完毕，如下图所示

![avatar](https://picture.zhanghong110.top/docsify/16408502551502.png)

可见在更上层就已经完成了参数序列化，我们重新跟踪发现是invokeForRequest的getMethodArgumentValues()处生成

```java
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

		MethodParameter[] parameters = getMethodParameters();
		if (ObjectUtils.isEmpty(parameters)) {
			return EMPTY_ARGS;
		}

		Object[] args = new Object[parameters.length];
		for (int i = 0; i < parameters.length; i++) {
			MethodParameter parameter = parameters[i];
			parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
			args[i] = findProvidedArgument(parameter, providedArgs);
			if (args[i] != null) {
				continue;
			}
			if (!this.resolvers.supportsParameter(parameter)) {
				throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
			}
			try {
				args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
			}
			catch (Exception ex) {
				// Leave stack trace for later, exception may actually be resolved and handled...
				if (logger.isDebugEnabled()) {
					String exMsg = ex.getMessage();
					if (exMsg != null && !exMsg.contains(parameter.getExecutable().toGenericString())) {
						logger.debug(formatArgumentError(parameter, exMsg));
					}
				}
				throw ex;
			}
		}
		return args;
	}
```

这边做下解释，总体的方法意思就是选择合适的MessageConverter来序列化,这边debug,进入 this.resolvers.resolveArgument，最终会来到

```java
protected <T> Object readWithMessageConverters(NativeWebRequest webRequest, MethodParameter parameter, Type paramType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {
        HttpServletRequest servletRequest = (HttpServletRequest)webRequest.getNativeRequest(HttpServletRequest.class);
        Assert.state(servletRequest != null, "No HttpServletRequest");
        ServletServerHttpRequest inputMessage = new ServletServerHttpRequest(servletRequest);
        Object arg = this.readWithMessageConverters(inputMessage, parameter, paramType);
        if (arg == null && this.checkRequired(parameter)) {
            throw new HttpMessageNotReadableException("Required request body is missing: " + parameter.getExecutable().toGenericString());
        } else {
            return arg;
        }
    }
```

这一步可能有点跳跃，感兴趣的自己debug看下，这里关键代码就是readWithMessageConverters，我们进去

```java
@Nullable
	protected <T> Object readWithMessageConverters(HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {

		MediaType contentType;
		boolean noContentType = false;
		try {
			contentType = inputMessage.getHeaders().getContentType();
		}
		catch (InvalidMediaTypeException ex) {
			throw new HttpMediaTypeNotSupportedException(ex.getMessage());
		}
		if (contentType == null) {
			noContentType = true;
			contentType = MediaType.APPLICATION_OCTET_STREAM;
		}

		Class<?> contextClass = parameter.getContainingClass();
		Class<T> targetClass = (targetType instanceof Class ? (Class<T>) targetType : null);
		if (targetClass == null) {
			ResolvableType resolvableType = ResolvableType.forMethodParameter(parameter);
			targetClass = (Class<T>) resolvableType.resolve();
		}

		HttpMethod httpMethod = (inputMessage instanceof HttpRequest ? ((HttpRequest) inputMessage).getMethod() : null);
		Object body = NO_VALUE;

		EmptyBodyCheckingHttpInputMessage message;
		try {
			message = new EmptyBodyCheckingHttpInputMessage(inputMessage);
            //这里
			for (HttpMessageConverter<?> converter : this.messageConverters) {
				Class<HttpMessageConverter<?>> converterType = (Class<HttpMessageConverter<?>>) converter.getClass();
				GenericHttpMessageConverter<?> genericConverter =
						(converter instanceof GenericHttpMessageConverter ? (GenericHttpMessageConverter<?>) converter : null);
				if (genericConverter != null ? genericConverter.canRead(targetType, contextClass, contentType) :
						(targetClass != null && converter.canRead(targetClass, contentType))) {
					if (message.hasBody()) {
						HttpInputMessage msgToUse =
								getAdvice().beforeBodyRead(message, parameter, targetType, converterType);
						body = (genericConverter != null ? genericConverter.read(targetType, contextClass, msgToUse) :
								((HttpMessageConverter<T>) converter).read(targetClass, msgToUse));
						body = getAdvice().afterBodyRead(body, msgToUse, parameter, targetType, converterType);
					}
					else {
						body = getAdvice().handleEmptyBody(null, message, parameter, targetType, converterType);
					}
					break;
				}
			}
		}
		catch (IOException ex) {
			throw new HttpMessageNotReadableException("I/O error while reading input message", ex, inputMessage);
		}

		if (body == NO_VALUE) {
			if (httpMethod == null || !SUPPORTED_METHODS.contains(httpMethod) ||
					(noContentType && !message.hasBody())) {
				return null;
			}
			throw new HttpMediaTypeNotSupportedException(contentType,
					getSupportedMediaTypes(targetClass != null ? targetClass : Object.class));
		}

		MediaType selectedContentType = contentType;
		Object theBody = body;
		LogFormatUtils.traceDebug(logger, traceOn -> {
			String formatted = LogFormatUtils.formatValue(theBody, !traceOn);
			return "Read \"" + selectedContentType + "\" to [" + formatted + "]";
		});

		return body;
	}
```

我们可以看到最后交由messageConverters数组中的元素处理，最终解析出我们的JSON

`org.springframework.http.converter.HttpMessageConverter` 是一个策略接口，接口说明如下

Strategy interface that specifies a converter that can convert from and to HTTP requests and responses. 简单说就是 HTTP request (请求)和response (响应)的转换器。该接口有只有5个方法，简单来说就是获取支持的 MediaType（application/json之类），接收到请求时判断是否能读（canRead），能读则读（read）；返回结果时判断是否能写（canWrite），能写则写（write）。

我们进入一个叫addDefaultHttpMessageConverters的类

```java
protected final void addDefaultHttpMessageConverters(List<HttpMessageConverter<?>> messageConverters) {
		messageConverters.add(new ByteArrayHttpMessageConverter());
		messageConverters.add(new StringHttpMessageConverter());
		messageConverters.add(new ResourceHttpMessageConverter());
		messageConverters.add(new ResourceRegionHttpMessageConverter());
		if (!shouldIgnoreXml) {
			try {
				messageConverters.add(new SourceHttpMessageConverter<>());
			}
			catch (Throwable ex) {
				// Ignore when no TransformerFactory implementation is available...
			}
		}
		messageConverters.add(new AllEncompassingFormHttpMessageConverter());

		if (romePresent) {
			messageConverters.add(new AtomFeedHttpMessageConverter());
			messageConverters.add(new RssChannelHttpMessageConverter());
		}

		if (!shouldIgnoreXml) {
			if (jackson2XmlPresent) {
				Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.xml();
				if (this.applicationContext != null) {
					builder.applicationContext(this.applicationContext);
				}
				messageConverters.add(new MappingJackson2XmlHttpMessageConverter(builder.build()));
			}
			else if (jaxb2Present) {
				messageConverters.add(new Jaxb2RootElementHttpMessageConverter());
			}
		}

		if (kotlinSerializationJsonPresent) {
			messageConverters.add(new KotlinSerializationJsonHttpMessageConverter());
		}
		if (jackson2Present) {
			Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.json();
			if (this.applicationContext != null) {
				builder.applicationContext(this.applicationContext);
			}
			messageConverters.add(new MappingJackson2HttpMessageConverter(builder.build()));
		}
		else if (gsonPresent) {
			messageConverters.add(new GsonHttpMessageConverter());
		}
		else if (jsonbPresent) {
			messageConverters.add(new JsonbHttpMessageConverter());
		}

		if (jackson2SmilePresent) {
			Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.smile();
			if (this.applicationContext != null) {
				builder.applicationContext(this.applicationContext);
			}
			messageConverters.add(new MappingJackson2SmileHttpMessageConverter(builder.build()));
		}
		if (jackson2CborPresent) {
			Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.cbor();
			if (this.applicationContext != null) {
				builder.applicationContext(this.applicationContext);
			}
			messageConverters.add(new MappingJackson2CborHttpMessageConverter(builder.build()));
		}
	}
```

可以看到，默认的情况下SpringMVC 启动时会自动配置一些HttpMessageConverter，我们导入了相应的依赖就会使用该包来解析，可以发现，这几种工具在springmvc下默认是有优先级之分的。



> 至此我们的springmvc源码分析就差不多了