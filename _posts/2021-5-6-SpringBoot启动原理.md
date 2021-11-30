---
title: 'Spring Boot如何实现零配置'
key: key-2021-05-06SpringBoot-config
tags: [springboot, java]
comment: true
footer: true
show_edit_on_github: false
pageview: true
lightbox: true
aside:
toc: true
show_subscribe: false
lightbox: true
---
## **1. Web项目启动的过程**
- Servlet的容器：Tomcat,Jetty,Jboos等，其中Nginx、Apache是http容器;
- Web.xml配置（配置listener和servlet）
   1. listener需要配置`ContextLoaderListener`，再通过访问`webApplicationContext`加载spring上下文，实际就是为了加载spring.xml文件；
   2. 配置servlet是为了启动`spring-MVC`, 该过程是通过`DispatchServlet`访问spring-mvc.xml

web.xml 文件

```java
<context-param>
<param-name>contextConfigLocation</param-name>
<param-value>classpath:/config/spring.xml</param-value>
</context-param>

<listener>
<description>listener</description>
<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>

```

<font color = "blue">web项目的启动流程:</font>Tomcat启动时会解析web.xml文件 $\rightarrow$ 通过web.xml文件初始化listener和servlet      $\rightarrow$     执行`listener(ContextLoaderListener)` 加载spring    $\rightarrow$    执行`servlet(DispatchServlet)`加载spring-mvc

## **2.SpringBoot替换机制**
当使用ssm这种架构去完成一个web项目时，你需要去写配置很多内容。例如：web.xml, spring-mvc.xml, spring.xml等。但是在springboot无需要任何配置也能完成web项目，springboot采用约定大于配置的方式完成使用注解的方式实现配置，在内嵌的Tomcat中通过代码的方式代替了`web.xml`中servlet和listener的配置。
<font color = "orange">1.内嵌Tomcat代替原生去除web.xml:</font>
```java
<!-- 添加Tomcat的依赖 -->
<dependency>
<groupId>org.apache.tomcat.embed</groupId>
<artifactId>tomcat-embed-core</artifactId>
<version>8.0.48</version>
</dependency>
<dependency>
<groupId>org.apache.tomcat.embed</groupId>
<artifactId>tomcat-embed-jasper</artifactId>
<version>8.0.48</version>
</dependency>
 <dependency>
<groupId>org.apache.tomcat.embed</groupId>
<artifactId>tomcat-embed-logging-juli</artifactId>
<version>8.0.48</version>
</dependency>

public class MainApplication {
    public static void main(String[] args) throws ServletException, LifecycleException {
        Tomcat tomcat = new Tomcat();
        tomcat.setPort(80);
        //获取指定的绝对路径获取上下文
        Context context = tomcat.addWebapp("/", new File("sourceCode/src/main/webapp").getAbsolutePath());
        //此处可通过context设置servlet和listener.
        WebResourceRoot root = new StandardRoot(context);
        String path = MainApplication.class.getResource("").getPath();
        System.out.println(path);
        root.addPreResources(new DirResourceSet(root, "/WEB-INF/classes", path, "/"));
        context.setResources(root);
        tomcat.start();
        tomcat.getServer().await();
    }
}
```
```html
<!DOCTYPE html>
<html>
<head>Hello World</head>
</html>
```
运行后通过`localhost:80/index.html`访问到`Hello World`即表示成功
`Note:` 1. 在获取绝对路径并访问该绝对路径下的`index.html`文件时，会存在忽略**模块名**这一项导致找不到文件报错，后来我把模块名加上才成功启动

<font color = "orange">2.使用注解代替spring.xml</font>
采用`AnnotationConfigApplicationContext`替换`AbstractXmlApplicationContext`,主要的一些注解包括：<font color = "red">@Configuration, @Bean, @Service,@ComponentScan, @Component, @MapperScan, @Mapper</font>
完成一下伪代码：
```java
  AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        //将配置注册到context容器中
        context.register(xxxConfig.class);
        //容器刷新获取最新的所有bean
        context.refresh();
        //根据class对象获取该对象的bean
        XxxMapper xxxMapper = context.getBean(XxxMapper.class);
        //业务使用
        xxxMapper.getUser(userId);
```
<font color = "orange">3.代替spring-mvc.xml</font>
目前官网也推荐使用代码的方式配置`DispatchServlet`:[点击进入](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-servlet)
```java
<dependency>
<groupId>org.springframework</groupId>
<artifactId>spring-web</artifactId>
<version>5.2.12.RELEASE</version>
<scope>compile</scope>
</dependency>

public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext servletContext) {

        // Load Spring web application configuration
        AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
        context.register(AppConfig.class);

        // Create and register the DispatcherServlet 之前此处应该是配在xml文件中的
        DispatcherServlet servlet = new DispatcherServlet(context);
        ServletRegistration.Dynamic registration = servletContext.addServlet("app", servlet);
        registration.setLoadOnStartup(1);
        registration.addMapping("/");
    }
}
```
`onStartup`方法替换`web.xml`, 完成初始化IOC容器和初始化servlet,因此Tomcat启动总是会加载`MyWebApplicationInitializer`中的`onStartup`方法完成这两个重要的过程。

<font color = "red">Tomcat启动会加载MyWebApplicationInitializer原因：</font>由于在java的servlet中有个接口`ServletContainerInitializer`, Spring使用`SpringServletContainerInitializer`实现了该接口，然后又声明了`WebApplicationInitializer`接口供开发者使用，在通过`@HandlesTypes`注解将二者关联在一起，因此会调用开发者所有实现该接口中的onStartup方法。
ServletContainerInitializer 是 servlet3.0 提供的一个 SPI(Service Provider Interface)，可以通过 HandlesTypes 筛选出相关的 servlet 类。

## **3.SpringBoot启动**
在第二节的模拟Tomcat的启动过程, 通过调用Tomcat中的`addWebapp`方法表示这个项目是一个web项目，从而实现了对onStartup方法的调用。而在Spring Boot启动的时候内嵌了Tomcat容器，并没有调用`addWebapp`方法，源码如下：
```java
public WebServer getWebServer(ServletContextInitializer... initializers) {
		Tomcat tomcat = new Tomcat();
		File baseDir = (this.baseDirectory != null ? this.baseDirectory
				: createTempDir("tomcat"));
		tomcat.setBaseDir(baseDir.getAbsolutePath());
		Connector connector = new Connector(this.protocol);
		tomcat.getService().addConnector(connector);
		customizeConnector(connector);
		tomcat.setConnector(connector);
		tomcat.getHost().setAutoDeploy(false);
		configureEngine(tomcat.getEngine());
		for (Connector additionalConnector : this.additionalTomcatConnectors) {
			tomcat.getService().addConnector(additionalConnector);
		}
		prepareContext(tomcat.getHost(), initializers);
		return getTomcatWebServer(tomcat);
	}
```
上述代码没有调用`addWebapp`方法，所以springboot启动的时候并不是一个web的项目，因此不会通过调用`onStartup`去初始化`DispatchServlet`。当然如果该工程被打成war包的形式，则还是会调用`onStartup`。


