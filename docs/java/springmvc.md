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



```
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

```
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

`getServletConfig`由servlet-api的`GenericServlet`提供

我们主要看`initServletBean`方法，回到`FrameworkServlet`类中

```
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

我们重点分析`initWebApplicationContext`方法，代码如下

```
protected WebApplicationContext initWebApplicationContext() {
		WebApplicationContext rootContext =
				WebApplicationContextUtils.getWebApplicationContext(getServletContext());
		WebApplicationContext wac = null;

		if (this.webApplicationContext != null) {
			// A context instance was injected at construction time -> use it
			wac = this.webApplicationContext;
			if (wac instanceof ConfigurableWebApplicationContext cwac && !cwac.isActive()) {
				// The context has not yet been refreshed -> provide services such as
				// setting the parent context, setting the application context id, etc
				if (cwac.getParent() == null) {
					// The context instance was injected without an explicit parent -> set
					// the root application context (if any; may be null) as the parent
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
			wac = findWebApplicationContext();
		}
		if (wac == null) {
			// No context instance is defined for this servlet -> create a local one
			wac = createWebApplicationContext(rootContext);
		}

		if (!this.refreshEventReceived) {
			// Either the context is not a ConfigurableApplicationContext with refresh
			// support or the context injected at construction time had already been
			// refreshed -> trigger initial onRefresh manually here.
			synchronized (this.onRefreshMonitor) {
				onRefresh(wac);
			}
		}

		if (this.publishContext) {
			// Publish the context as a servlet context attribute.
			String attrName = getServletContextAttributeName();
			getServletContext().setAttribute(attrName, wac);
		}

		return wac;
	}
```

