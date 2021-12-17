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

## 二.提出问题

网上有一个靓仔提出了如下几个问题，我们怼刚开始提出的疑问做下扩充

1. 一个 Http 请求和响应，是如何转变成 Servlet 的请求和相应的？
2. Servlet 是如何抽象 Http 请求的？
3. Tomcat 如何管理 tcp 连接？
4. Tomcat 如何管理输入输出流？
5. Tomcat 的生命周期，如何启动的？如何运行？
6. Session 如何管理？
7. 如何加载类？
8. Tomcat 如何管理 servlet 容器，如何连接？
9. 不同的浏览器连接上应用，如何判断是不同的客户端呢？是 Tomcat 负责判断的吗？
10. JSP 页面如何转为 java 代码，输出 html 文件？

### 三.源码解析

待续