---

title: 代理对象中的Spring的Bean为Null引发的惨案
key: AAA-2022-03-21-proxyObjectCausesTheNullOfSpring
tags: [动态代理, Spring]
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


### 1. 背景介绍

这次事件是发生在我司的开发环境中，然后也有其他小伙伴去总结了，我看了一下，总感觉少了点什么，然后专门研究到底是怎么回事。

惨案的起因：

1. 我们通过google的EventBus来订阅jvm级别的事件，就是在每个方法上加了一个@Subscribe，但是有哥们会把该方法写成私有的方式，大概内容如下：

```java
@Component
public class UserService {

    @Autowired
    private OrderService orderService;

    public void testProxy(){
        System.out.println("userService");
    }

    @Subscribe
    public void testProxyPublic(User user){
        orderService.order();
        System.out.println("public orderService.toString()");
    }

    @Subscribe
    private void testProxyPrivate(User user){
       orderService.order();
       System.out.println("private orderService.toString()");
    }
}
```

2. 其实上述的代码中如果不生成代理类是没有任何问题的，但是如果一旦加了切面形成了代理类，那就会导致私有方法中orderService为null， 所以会产生NPE，我司的生产环境也是产生了大量的NPE异常，后来发现定位到该问题。

其实很多case和上述相似，例如加了@Transactional的私有方法，controller中的私有方法，或者是被final修饰了方法或者类都会相同的情况。

### 2. 原因分析

2.1 orderService为啥为Null

2.2 如果基于继承的方式为啥私有的方法还是会执行？

2.3 子类通过反射执行父类的方法会成功吗？

对此，带着以上几个问题作为出发点一起去探个究竟。

![image-20220319194009605]({{"/assets/picture/image-20220319194009605.png" | absolute_url}})

由上图可知，代理对象中是包含了目标类，但是并没有注入对应的spring的bean，对此，我们得从spring的bean的生命周期说起。

我想大部分人都应该知道生成Bean的整个过程，大致如下：

1. 通过BeanDefinition获取到该bean的class文件；

2. 使用反射对推断出的构造方法进行实例化对象；

3. 对实例化后的对象进行属性填充(bean依赖注入)；

4. 判断是否需要生成代理对象，如果需要则为其生成代理对象；

5. 放入spring容器；

由此可知，在生成代理对象后，spring并不会对代理对象进行属性的依赖注入，因此上述的orderService自然为null，但是spring会把目标类赋给代理的。最后在通过反射执行的时候是通过目标类的对象去执行的，因此你会发现不为null的orderService的bean是来自userService，该类似的过程如下：

```java
class UserServiceProxy extends UserService{
    private UserService target; 
    public void testProxyPublic(){
               //执行@before切面逻辑
              this.target.testProxyPublic();
              //执行@After逻辑
   }
}
```

上述是cglib代理执行的基本过程，其是通过继承的方式实现代理的。因为如果类被final修饰，或者方法被final或者private修饰时，是不能实现继承关系的，因此在该情况下代理失败，但是会报错吗？在网上看了几篇文章，然后发现有一篇文章下面提出了和我很相似的疑问，而博主在文中并没有给予解释，贴出问题如下：

![]({{"/assets/picture/2022-03-20-10-04-36-image.png" | absolute_url}})

对于问题一：对于反射本身而言，都是传入当前的对象也就是代理对象实现反射，但是在代理对象中存在回调机制，对于私有方法是不会执行代理中的方法拦截器的回调，因此执行私有方法是基于代理对象去执行的而不是目标对象；而对于public等其他能够被代理的方法是可以进入回调方法中，在该方法中使用`this.target`这个目标对象去执行反射方法的，而这个`this.target`是来源于Spring中的Bean对象，所有的依赖属性都被填充了。如下图所示：

![]({{"/assets/picture/2022-03-20-09-52-39-image.png" | absolute_url}})

![image-20220319194230836]({{"assets/picture/image-20220319194230836.png" | absolute_url}})

对于问题二：正常情况下，通过反射是拿不到父类的私有方法的，我自己还做了实验输出该子类的所有方法，硬是没有发现父类的私有方法。后来使用EventBus中的`@Subscribe`

注解发现这里面原来是可以获取父类的私有方法的，如下图所示：

![image-20220319200800998]({{"assets/picture/image-20220319200800998.png" | absolute_url}})

上述的代码是获取了当前类的所有父类中的打上`@Subscriber`的方法，包括了私有方法。我自己使用代码实验了获取父类的私有方法并用子类对象执行的方式，因此不会报`NoSuchMethodException`,该方法是来源于父类的，这也证实一点:<font color=red>子类是可以通过反射的方式获取父类的任何属性和方法.</font>

```java
public class People {

    private String name;
    @Subscribe
    public void sleep(){
        System.out.println("public People sleep");
    }

    @Subscribe
    private void eat(){
        System.out.println("private People eat");
    }
}
public class Teacher extends People {

    @Override
    public void sleep() {
        System.out.println("Public Teacher sleep");
    }

    private void play(String name) {
        System.out.println(name + "在玩耍");
    }
}
 public static void main(String[] args) throws InvocationTargetException, IllegalAccessException {
        Teacher teacher = new Teacher();
        // 此处也可以使用getSuperClass的方式获取父类的class，我这里就借用了EventBus中的源码
        Set<? extends Class<?>> supertypes = TypeToken.of(teacher.getClass()).getTypes().rawTypes();
        teacher.getClass().getSuperclass();
        for (Class<?> supertype : supertypes) {
            for (Method method : supertype.getDeclaredMethods()) {
                if (method.isAnnotationPresent(Subscribe.class)) {
                    method.setAccessible(true);
                    method.invoke(teacher);
                }
            }
        }
    }
```

然后我把public的修饰改成了protect了，发现也是可以成功实现回调的，唯一不同是在`invokeJoinpoint()`中走的是else的逻辑，public方法走的是if的逻辑。
![]("/assets/picture/2022-03-20-09-56-05-image.png" | absolte_url}})

### 3.结果总结

3.1在形成切面之前，userService的bean不是代理对象，是一个普通的bean对象，然后通过该对象的反射方式自然可以执行该类中的私有方法的；

3.2 orderService为null是因为此时的对象是代理对象，代理对象并没有对任何属性进行依赖注入，因此自然为null；

3.3 子类不能直接通过反射的方式执行父类的中的私有方法，但是可以间接的获取。先通过getSuperClass的方式获取父类中的Class，然后获取父类中的私有方法或者属性；

3.4 产生代理对象后还是会执行目标类中的私有方法。 此时执行私有方法不同public等一些成功代理的方法，私有的方法没有走到方法拦截器中执行方法的回调，而是直接使用代理对象对反射的方法执行，因此此时如果该方法中有用到spring的bean自然会报NPE。
