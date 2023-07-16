---
title: 'SpringCloud | OpenFeign原理'
key: key-2023-07-16-springcloud-openfeign
tags: ["SpringCloud"]
comment: true
footer: true
show_edit_on_github: false
pageview: true
lightbox: true
aside:
toc: true
show_subscribe: false
---


OpenFeign 是声明式的 HTTP 客户端，让远程调用更简单。
提供了HTTP请求的模板，编写简单的接口和插入注解，就可以定义好HTTP请求的参数、格式、地址等信息
整合了Ribbon(负载均衡组件)和 Hystix(服务熔断组件)，不需要显示使用这两个组件同时Spring Cloud Feign 在 Netflix Feign的基础上扩展了对SpringMVC注解的支持

对于该注解我会分成两部分去解析: 1. 开发使用；2.原理解析。其中原理解析会分为两部分分别是启动和加载的过程。
## 1. FeignClient的用法

1.定义一个接口，在接口上加@FeignClient，该注解提供了很多参数：
+ name/value：指定FeignClient的名称，如果项目使用了Ribbon，name属性会作为微服务的名称，用于服务发现；
+ url：可以用于域名的直接调用;(如何url有值，那么会导致负载均衡失效)
+ configuration: Feign配置类，可以自定义Feign的Encoder、Decoder、LogLevel、Contract;
+ fallback: 容错处理类，当接口调用发生错误会调用该类;
+ fallbackFactory: 工厂类，用于接口返回错误的统一处理;

2.在启动类上加@EnableFeignClients

+ basePackages可用于指定扫描哪些包；
+ defaultConfiguration可以提供一些默认的配置；

### 1.1全局配置和局部配置

a). 通过配置文件方式：

```yaml
logging:
  level:
    com:
      dev: debug #com.dev为包路径 此处配置 日志无法打印

feign:
  client:
    config:
      default:  #全局配置  如果想要对某个feign配置 将default改成对应的feign的name 就是注解中的name变量对应的值
        loggerLevel: NONE #提供集中类型
```

b).通过配置类方式：

```java
@Configuration
public class MyEurekaClientConfig {
    @Bean
    public Logger.Level feignLogLevel(){
        return Logger.Level.FULL;
    }
}

// 全局配置
@EnableFeignClients(defaultConfiguration = MyEurekaClientConfig.class)

//只针对当前的Feign配置
@FeignClient(value = "order", url = "http://127.0.0.1:81", configuration = MyEurekaClientConfig.class)

```

### 1.2 配置Feign的Http客户端

OpenFeign默认使用的是JDK自带的HttpURLConnection, 该客户端没有池化，所以性能不好，因此得采用Apache和OkHttp来替换JDK原生的Http客户端，具体使用方式如下：

a).OkHttp

引入依赖

```
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-okhttp</artifactId>
</dependency>
```

开启配置和配置连接池

```properties
feign.okhttp.enabled: true

feign.okhttp.max-connections: 200  # 最大连接数，默认：200
feign.okhttp.max-connections-per-route: 200  # 最大路由，默认：50
feign.okhttp.connection-timeout: 200  # 连接超时，默认：2000/毫秒
```

b).Apache Http

引入依赖

```properties
 <dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-httpclient</artifactId>
 </dependency>
```

配置和OkHttp类似

```properties
feign.httpclient.enabled: true

feign.httpclient.max-connections: 200  # 最大连接数，默认：200
feign.httpclient.max-connections-per-route: 200  # 最大路由，默认：50
feign.httpclient.connection-timeout: 200  # 连接超时，默认：2000/毫秒
```

### 1.3 配置Feign的超时重置

```yaml
feign:
  client:
    config:
      default:  #全局配置 如果针对某个服务的话 将对应的服务名替换default
        connectTimeout: 1000 # 连接时间
        readTimeout: 1000 # 读取超时时间 两者同时配置才可以生效
        retryer: com.dev.wizard.config.RetryerConfig #配置类的全路径
```

```java
@Configuration  //如果加了这个注解 上述的配置类就不用加了
public class RetryerConfig implements Retryer {
    @Override
    public void continueOrPropagate(RetryableException e) {
        throw e;
    }

    /**
    *period：周期，重试间隔时间
    *maxPeriod：最大周期，重试间隔时间按照一定的规则逐渐增大，但不能超过最大周期
    *maxAttempts：最大尝试次数，重试次数
    **/
    @Override
    public Retryer clone() {
        return new Default(100, TimeUnit.SECONDS.toMillis(1), 5);
    }
}
```

不建议在高并发场景中使用重试机制，失败直接返回错误，链路终止。尤其是在链路很长的多个微服务之间时，重试会导致重试风暴。例如A -> B-> C ->D, 如果D发生错误, C会重试, C重试失败 B继续重试

重试机制的原理见后文

![image-20230708094819776]({{"/assets/picture/2023-07/image-20230708094819776.png" | absolute_url}})


![image-20230708103631227]({{"/assets/picture/2023-07/image-20230708103631227.png" | absolute_url}})


## 2.原理解析

### 2.1 加载过程

1. EnableFeignClients注解中@Import(FeignClientsRegistrar.class)触发FeignClientsRegistrar初始化；[实际上Enable系列注解原理相同]

![image-20230704222006799]({{"/assets/picture/2023-07/image-20230704222006799.png" | absolute_url}})

2. 完成默认配置和为一个Feign的容器注入

![image-20230708222856246]({{"/assets/picture/2023-07/image-20230708222856246.png" | absolute_url}})

![image-20230708224628728]({{"/assets/picture/2023-07/image-20230708224628728.png" | absolute_url}})

3. 扫描路径下所有的FeignClient注解，为每个个Feign注入时，需要为其注入对应的配置；
4. 调用registerFeignClient方法基于FeignClientBeanFactory将每一个Feign的Bean注入容器[懒加载方式]；

![image-20230708230044229]({{"/assets/picture/2023-07/image-20230708230044229.png" | absolute_url}})

![image-20230708230213963]({{"/assets/picture/2023-07/image-20230708230213963.png" | absolute_url}})

`总结：`通过EnableFeignClients注解调用FeignClientsRegistrar的registerBeanDefinition的方法，用于默认配置初始化，另外对basePackage的路径来扫描所有加了FeignClient的注解的接口，也会扫描当前启动类路径下所有加了该注解的接口，(只允许FeignClient注解加在接口上 加在类上启动报错)为每一个OpenFeign接口创建一个FeignClientFactoryBean对象注入到Bean容器中，此时注入的只是包含OpenFeign接口的BeanDefinition。真正发挥作用的代理对象还没有生成。（OpenFeign的代理对象是一种懒加载的方式），真正初始化生成接口的代理对象是Supplier的表达式。源码见：org.springframework.cloud.openfeign.FeignClientsRegistrar#registerFeignClient

在加载配置的过程，FeignClientSpecification是配置类的Bean对象，该对象包含了Feign的配置信息。FeignClientFactoryBean是一个用来创建FeignClient代理对象的工厂，当我们使用@Autowired注入@FeignClient标记的接口时，会触发Spring的Bean实例化机制，则会调用该类对象的FactoryBean的getObject()方法，创建一个代理对象。

### 2.2 代理对象创建

OpenFeign接口代理对象：

1. 执行BeanDefinition中的InstanceSupplier，该lambda中存在factoryBean对getObject方法的调用，最终使用getTarget方法调用；
2. 获取FeignContext对象，FeignContext保存了每一个Feign对象的独立的上下文对象；
3. 在初始化过程中对于没有URL的  则会认为是基于Eureka去使用的，存在URL意味着独立使用。是否存在URL影响代理对象的构建，没有URL是基于注册中心配置去使用的，可实现调用时的负载均衡；
4. 获取对应的Client对象，例如OkHttp或者是Apache Http Client；

​        如果是引入了LoadBalance，两个Client的初始化见：

​        org.springframework.cloud.openfeign.ribbon.HttpClientFeignLoadBalancedConfiguration

​         org.springframework.cloud.openfeign.loadbalancer.OkHttpFeignLoadBalancerConfiguration用于初始化OkHttpFeign

5. 构建Feign对象，该对象包含了Encoder，Decoder，Contract等对象，作用如下；

   编码器（Encoder）：如果调用接口时，传递的参数是个对象，Feign会将这个对象进行编码，转换成JSON格式；

   解码器（Decoder）：接受到响应后，将JSON转换为一个对象；

   Logger：负责日志打印，即打印这个接口调用的详细请求，包含请求、响应等等；

   Contract：契约组件，这个组件就负责Feign的原生注解与SpringMVC注解之间的转化；

   Feign.Builder：FeignClient的一个实例构造器，这是Builder设计模式的典型实现；

   Client：Feign客户端，里面包含了上述的一系列组件；

6. 基于methodName[OrderFeignClient#getOrderById(String)]和MethodHandler之间的映射关系构建出method对象和MethodHandler[SynchronousMethodHandler]之间的映射关系，主要用于调用的过程；

```java
Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();

//在创建代理对象前将方法和方法处理器之间的映射关系赋值给了ReflectiveFeign.FeignInvocationHandle对象
//再将该对象赋到代理对象中
InvocationHandler handler = factory.create(target, methodToHandler);
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),
        new Class<?>[] {target.type()}, handler);

 public InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch) {
      return new ReflectiveFeign.FeignInvocationHandler(target, dispatch);
    }

//关注此处的dispatch，在后续执行的流程会使用到
FeignInvocationHandler(Target target, Map<Method, MethodHandler> dispatch) {
      this.target = checkNotNull(target, "target");
      this.dispatch = checkNotNull(dispatch, "dispatch for %s", target);
    }

```

`Note：`

对于没有在FeignClient中使用url时，可以使用loadBalance方式，主要就是通过微服务名字通过负载均衡的策略替换成对应的ip和端口号。

![image-20230709084023577]({{"/assets/picture/2023-07/image-20230709084023577.png" | absolute_url}})

如果项目中引入了Hystrix依赖，则Targeter为HystrixTargeter，创建代理对象是通过它的target方法：
![image-20230705063404702]({{"/assets/picture/2023-07/image-20230705063404702.png" | absolute_url}})

### 2.3执行过程

在2.2中讲述了代理对象创建过程，在该代理对象中主要是所有的信息元数据都放到了MethodHandler中，同时在代理对象中会维护一个map，接口每个方法就会有对应的一个 MethodHandler，当我们调用接口方法时，其实是调用动态代理对象的 MethodHandler 来发送远程调用请求的。代理对象的代码如下：

```java
public final class $Proxy81 extends Proxy implements OrderFeignClient {
    private static final Method m0;
    private static final Method m1;
    private static final Method m2;
    private static final Method m3;

    public $Proxy81(InvocationHandler var1) {
        super(var1);
    }

    //此处的h就是来自ReflectiveFeign.FeignInvocationHandler生成的对象，该静态类中的dispatch维护方法对象和方法处理器之间的     //映射关系，见2.2中的第六步
    public final OrderDTO getOrderById(String var1) {
        try {
            return (OrderDTO)super.h.invoke(this, m3, new Object[]{var1});
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("com.dev.wizard.feign.OrderFeignClient").getMethod("getOrderById", Class.forName("java.lang.String"));
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}

```

```java
//上述通过代理对象调用时，实际上会调用ReflectiveFeign.FeignInvocationHandle中的invoke方法
//如果是开启了hystrix的fallback机制的话生成FeignCircuitBreakerInvocationHandler
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      if ("equals".equals(method.getName())) {
        try {
          Object otherHandler =
              args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
          return equals(otherHandler);
        } catch (IllegalArgumentException e) {
          return false;
        }
      } else if ("hashCode".equals(method.getName())) {
        return hashCode();
      } else if ("toString".equals(method.getName())) {
        return toString();
      }
    //看到这里 基本上明白最终是根据method对象拿到SynchronousMethodHandler对象，所有的元数据都保存在这个对象中
      return dispatch.get(method).invoke(args);
    }
```

feign.SynchronousMethodHandler#invoke

```java
final class SynchronousMethodHandler implements MethodHandler
FeignCircuitBreakerInvocationHandler

public Object invoke(Object[] argv) throws Throwable {
    //根据参数构建请求模板，这些参数主要用于和请求参数之间绑定关系
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    Options options = findOptions(argv);
    Retryer retryer = this.retryer.clone();
    //重试机制
    while (true) {
      try {
          //执行请求并对反序列化
        return executeAndDecode(template, options);
      } catch (RetryableException e) {
        try {
          //如果存在重试机制 并满足重试要求就会continue执行下一次调用
          //直到重试次数用完会抛出重试异常
          retryer.continueOrPropagate(e);
        } catch (RetryableException th) {
          Throwable cause = th.getCause();
          if (propagationPolicy == UNWRAP && cause != null) {
            throw cause;
          } else {
            throw th;
          }
        }
        if (logLevel != Logger.Level.NONE) {
          logger.logRetry(metadata.configKey(), logLevel);
        }
        continue;
      }
    }
  }
```

### 3.OpenFeign的最佳实践

方式一：定义公用接口

1. 定义一个公用的接口；
2. 调用方继承该接口，然后在该接口中声明这是一个FeignClient；
3. 被调用方实现该接口然后实现该接口具体的逻辑返回数据；

![image-20230703220746907]({{"/assets/picture/2023-07/image-20230703220746907.png" | absolute_url}})

方式二：抽取出独立模块

![image-20230703220955690]({{"/assets/picture/2023-07/image-20230703220955690.png" | absolute_url}})

## 4.总结

本文主要介绍了FeignClient接口从启动加载到代理对象创建，再到代理对象执行的过程，整体过程如下：

1. 服务在启动时根据EnableFeignClients的注解扫描对应路径下的所有加了FeignClient的接口；
2. 创建全局配置信息，为每一个接口创建一个FeignClientBeanFactory对象放到Spring容器中；
3. 存在Feign对象注入时会初始化FeignClientBeanFactory对象，调用getObject方法中的getTarget方法创建代理对象；
4. 创建代理对象主要是将接口中的方法和对应的方法处理器构建映射关系，将这种关系提供给代理对象；
5. 请求在调用某个方法是，根据上述的映射关系获取到对应的方法处理器，该处理器中包含了所有该方法的元数据；
6. 构建出API发送请求；
