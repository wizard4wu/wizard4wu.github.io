---
title: SpringCloud的FeignClient不能将header设置为Null值
key: AAA-2022-09-30-springcloud-feign
tags: [SpringCloud,FeignClient]
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
### 1.问题产生

这是发生在最近的一个bug，同事将幂等Id设置到API请求的header里面，然后被调用方通过判断header中的该值是否为空进行处理。不为空走幂等逻辑，为空走重新创建新数据(非幂等)的逻辑。现在有个业务需要一直走非幂等逻辑，于是将Header的key设置成null值后不能走非幂等逻辑。

### 2.问题分析

模拟当时的代码：

```java
SpringBoot 和 SpringCloud的依赖：
    版本较老
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-parent</artifactId>
<version>1.5.3.RELEASE</version>

<dependency>
 <groupId>org.springframework.cloud</groupId>
 <artifactId>spring-cloud-dependencies</artifactId>
 <version>Dalston.RELEASE</version>
 <type>pom</type>
 <scope>import</scope>
</dependency>

 //上述版本的SpingCloud是来自NetFlix


//接收方
@RestController
public class SpringCloudController {
    @GetMapping("order/byId")
    public void byId(@RequestParam("id") String id, @RequestHeader("null_header") String nullHeader, @RequestHeader("order_key") String orderKey, HttpServletRequest request){
        String result = request.getHeder("order_key");
        System.out.println("From request:" + result);
        System.out.println("order_key:" + orderKey);
        System.out.println("null_header:" + nullHeader);
    }
}


//发请求方
//先发请求到这个API，这个API内会调用Feign的
@RestController
@RequestMapping("/springcloud")
public class SpringCloudController {
    @Autowired
    private OrderInterface orderInterface;
    @GetMapping("/feign")
    public void testFeign(){
        orderInterface.getOrderById("orderId", null, "encodeOrderKey");
    }
}

//定义这个Feign 本地调用
@FeignClient(value = "ORDER", url = "http://127.0.0.1:80", fallback = OrderClient.class)
public interface OrderInterface {
    @GetMapping("/order/byId")
    OrderDTO getOrderById(@RequestParam("id") String id, @RequestHeader("null_header") String nullableHeader, @RequestHeader(value = "order_key", required = false) String orderKey);
}

@Component
class OrderClient implements OrderInterface{

    @Override
    public OrderDTO getOrderById(String id, String nullableHeader, String orderKey) {
        return null;
    }
}

```
![image-20220918093916066]({{"/assets/picture/2022-09/image-20220918093916066.png" | absolute_url}})

![image-20220918094128921]({{"/assets/picture/2022-09/image-20220918094128921.png" | absolute_url}})


1. 当收到请求需要调用feign时，他会把参数按顺序收集到一个数组里面，数组叫做Object[] argv;
2. 根据启动时注解加载的元数据创建一个RequestTemplate，该对象内部的请求参数请求头都是key -->{key}形式;
3. 根据argv数组构建一个LinkedHashMap，构建时会跳过Null值，这个map时为了后期根据参数的key装载数据的;
4. 先装载query数据到RequestTemplate，构建URL地址连接，装载header数据
5. 装载header数据的时候调用feign.RequestTemplate#expand，下述这段代码让为null的header产生一个另一个{key}

```java
          String key = var.toString();
          Object value = variables.get(var.toString());
          if (value != null) {
            builder.append(value);
          } else {
            builder.append('{').append(key).append('}');
          }
```

```java
换成新版本： 新版本是Spring用的自家的openFeign
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-parent</artifactId>
<version>2.6.3</version>



<dependency>
 <groupId>org.springframework.cloud</groupId>
 <artifactId>spring-cloud-dependencies</artifactId>
 <spring.cloud-version>2021.0.4</spring.cloud-version>
 <type>pom</type>
 <scope>import</scope>
</dependency>

```

新版本和老版本的主要流程差不多，主要区别是expend方法 feign.template.HeaderTemplate#expand(新版本)

```java
  public String expand(Map<String, ?> variables) {
    List<String> expanded = new ArrayList<>();
    if (!this.values.isEmpty()) {
      for (Template template : this.values) {
        String result = template.expand(variables);

        if (result == null) {     //如果获取不到对应的value时 不对该header的key进行塞值
          /* ignore unresolved values */
          continue;
        }

        expanded.add(result);
      }
    }

    StringBuilder result = new StringBuilder();
    if (!expanded.isEmpty()) {
      result.append(String.join(", ", expanded));
    }

    return result.toString();
  }
```
![image-20220918093026657]({{"/assets/picture/2022-09/image-20220918093026657.png" | absolute_url}})

虽然新版本对为null的header的不设置值，但是接收方如果使用@RequestHeader接收时一定要要把required的属性设置成false，不然会报错。或者通过HttpServletRequest的getHeader("key")的方法来获取。
