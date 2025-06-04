---
title: 'Effective Java的理解与总结'
key: key-2024-10-1-EffectiveJava
date: 2024-10-01 14:10:26
tags: ["Java", "编码规范"]
comment: true
footer: true
show_edit_on_github: false
pageview: true
lightbox: true
aside:
toc: true
show_subscribe: false
---
### 1.写在前面
Effective Java这本书不推荐初学者去看，因为没有一定的经验基础，你在看这本书的时候不会产生共鸣，会让你觉得很多无法理解。这本书我读了两遍，分别是在工作三年和工作五年的时候，因为有了工作经验的积累，以及你在工作中遇到类似的问题，会和作者提出的观点不谋而合，所以更容易理解。
作者的有些条目，理论是正确的，但是在项目中不实用，我们也要选择性使用。有些条目我没有列出来是因为没有深刻的印象，也是没有完全理解。
全文均是小编结合ChatGPT的理解(嘿嘿，这个确实很有帮助)，如果还有更好的理解，可以相互交流。



### 2.主要内容
#### 1.考虑使用静态工厂方法替代构造方法
所谓静态工厂方法简单来说就是利用一个静态方法来创建一个对象的方式；
静态工厂方法的优点：

1. 静态工厂方法可以提供一个清晰的方法名用于返回对象；

2. 静态方法可以返回原类型的任何子类型(抽象的一种)； 与第四条结合

3. 与构造方法不同，它们不需要每次调用时都创建一个新对象;

4. 返回对象的类可以根据输入参数的不同而不同，其实这就是工厂模式的所在，但是难免会有很多if判断条件；
5. 在编写包含该方法的类时，返回的对象的类不需要存在；[没太理解]

> 例如A extends X, B extends X
> 使用静态方法可对返回类型进行抽象；
> public static X getX(String value){
> if("A".equals(value)){
>     return new A();
>    }
> if("B".equals(value)){
>   return new B();
>    }
> }

缺点：

1. 很难找到，通常使用of， from之类的方法，注意to方法通常应用成员方法。Date data = Date.from(); NewData newDate = date.to();
2. 无法子类化，静态方法不属于对象，谈不上继承了。

####  2.构造方法参数过多时使用 builder 模式
有个前提条件就是参数过多时用这种模式[作者提出超过四个参数的时候]，一旦使用了这种模式就得关闭通过new的方式来创建对象了。
作者认为build模式代码更简洁，易于阅读，但是会存在一个builder对象，影响性能。作者也提出JavaBeans模式，但是由于该模式构造出的对象违反了对象不可变性[构造后的对象可通过set方法对字段进行修改]，会存在安全隐患。

小编认为得看具体情况，实际开发中使用JavaBeans会给予很好的灵活性，如果只是局部变量使用创建一个对象，完全可以用JavaBeans，主打就是简单，当然这得还是看个人习惯。BTW，不建议大家在编码中使用copyProperties这种工具，不仅性能差(反射)而且不好维护（主要是get set方法调用不能很好的发现，对象的引用属性是浅拷贝）。


#### 3.使用私有构造方法或枚类实现 Singleton 属性
单例模式大家都不陌生，饥汉式和懒汉式，双检查锁，枚举，匿名内部类的方式，这里作者建议了私有构造方法和枚举，接下来看看实现：

**饥汉式：**

```java
private static final SingletonInstanceDemo INSTANCE = new SingletonInstanceDemo();
private SingletonInstanceDemo(){      //构造方法私有
    if(null != INSTANCE){
        throw new RuntimeException("Can not create singleton instance by reflection"); //抛异常的方式防止在本类中来调用。
    }
}
public static SingletonInstanceDemo getFirstInstance(){
    return INSTANCE;
}

```

**双检查锁懒汉式：**

```java
private static volatile SingletonSecondDemo INSTANCE = null; //必须要volatile修饰防止指令重排
public static SingletonSecondDemo getInstance_2(){
    if(null == INSTANCE){   //外层判断可以去掉，加上外层判断是为了提升性能
        synchronized (SingletonSecondDemo.class){           //如果没有内置检查 还是会存在创建新的对象的
            if(null == INSTANCE){
                INSTANCE = new SingletonSecondDemo();
            }
        }
    }
    return INSTANCE;
}
```

**内部类的方式：**

```java
    public static SingletonSecondDemo getInstance_3() {
        return InnerClass.INNER_INSTANCE;  //只有在调用是才会触发内部类的加载。}
        private static class InnerClass {   //private修饰要保证该静态类只能被本类访问

            static SingletonSecondDemo INNER_INSTANCE = new SingletonSecondDemo();

            static {
                System.out.println("InnerClass 静态代码块");
            }
        }
    }
```
#### 4.使用私有构造方法强制不可实例化
就是使用private修饰构造方法来禁止对象的实例化。例如项目中的静态工具类需要使用private来修饰构造方法，防止其他人来创建一个静态类对象。如果避免反射来调用的话，在这个私有的构造函数中抛出一个异常；[可用于工具类中]


#### 5.使用依赖注入取代硬连接资源
同样是创建对象，当一个类依赖多个对象[其他资源]时，这些对象会影响该对象的创建，这时候我们可以通过将创建资源的方式交出去，而不是在该类内部直接创建。例如：
```java
class Product {
    private String name;
    public Product(String name) {
        this.name = name;
    }
    public String getName() {
        return name;
    }
}
class Service {
    private final Product product;

    // Supplier<Product> 作为构造方法参数
    public Service(Supplier<Product> productSupplier) {
        this.product = productSupplier.get();
    }
}

//调用方
// 使用 Supplier<Product> 注入 Product 的创建方式
Supplier<Product> productFactory = () -> new Product("Sample Product");
// 将 productFactory 传入 Service 构造函数
Service service = new Service(productFactory);
```

#### 7.消除过期的对象引用，手动释放方便回收
```java
public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        return elements[--size];
}
//正常返回后，但是elements[size]的对象还是会存在，按照pop的语义而言，pop后这个对象elements[size]应该remove，改正后如下：

public Object pop() {
    if (size == 0)
        throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null; // Eliminate obsolete reference
    return result;
}
```

需要有这种编码意识
#### 9.使用 try-with-resources 语句替代 try-finally 语句
通常我们会在finally块中关闭流资源，当存在流嵌套时，我们要关心流关闭的先后顺序问题。但在Java7后使用try-with-resources可以对于继承了 AutoCloseable 接口任何流资源进行自动关闭。这样就将整个关闭流的方式交给JVM了。

#### 15.使类和成员的可访问性最小化

1. 让每个类或成员尽可能地不可访问，尽可能设置低的访问级别。

> 之前产品leader认为我们需要对于Java对象中的方法通过修改权限修饰符来适配UT的编写。其实我是很反对这种做法的。
> 权限修饰符决定了类和方法的可见性，要慎重改变。单元测试的重点侧重于类的行为和暴露的方法。但是对于其测试的方法内部存在调用私有方法的复杂逻辑，这绝不是简单粗暴修改该私有方法的权限修饰符来适配UT，当然使用反射的方式也是不可取的。此时考虑的是该私有方法中复杂逻辑的合理性设计，是否需要将复杂逻辑提取到公用的逻辑中设计，是否需要将复杂逻辑提取到权限高的方法中，亦或者使用复杂的测试数据来覆盖。


2.类具有公共静态 final 数组属性，或返回这样一个属性的访问器是错误的

> 作者认为这种会存在安全漏洞，因为获取了该引用后，可随意对其修改，因此我们需要保证该引用的对象是不可变的。

第一点合理运用权限修饰符(private, default, protect, public), 减少成员，方法的暴露，隐藏细节和方法封装是软件设计的原则(可参考最小知识原则)。
第二点要做到真正的不可变：1.引用不可变；2.不可修改。
static final这种确实能保证引用不可变，但是当客户端获取到引用后可对容器对象进行add或者remove操作，这就是修改了该容器。为了做到不可修改，可以使用Collections.unmodifiableList，如果用的是高版本的JDK，使用List.of就是不可修改的。

#### 16.在公共类中使用访问方法而不是公共属性
这个就不用多讲了，get set方法就是很好的例子了。
公共类不应该暴露可变属性，属性都要求final修饰；
工具类的构造方法要私有，类final修饰，因为工具类中的所有方法都是静态的；

#### 17.最小化可变性
和上述类似，作者认为设计一个不可变的类是对于线程安全的负责，并提供了五条规则设计不可变的类：
1. 不要提供修改对象状态的方法；
2. 确保这个类不能被继承，需要使用final修饰；
3. 把所有属性设置为 final；[目的是一旦赋值后就不可以修改]
4. 把所有的属性设置为 private；
5. 确保对任何可变组件的互斥访问确保对任何可变组件的互斥访问； 如果对象中存在可变对象的引用，不能够将该引用暴露出来。
   作者提出了不可变对象的很多优势，最主要的就是线程安全，可以自由共享。但是也会存在最大的缺点，就是修改任何一个属性是都需要一个单独的对象。
> 在实际业务开发的过程中，大家写的Java Bean基本不会设计成不可变的方式，一点是多线程场景执行少，另外就是不可变的话，发现非常难用。例如想要对某个属性进行修改，通常使用set方法方法就好了，但是因为不可变，我要重新创建一个对象，在对象的构造过程中设置上这个属性值。
> 此外，作者提供了一种伙伴类思想来支持这种情况，例如String和StringBuilder和StringBuffer。

#### 18.组合优于继承

| 场景         | 继承                           | 组合                                       | 场景         |
| ------------ | ------------------------------ | ------------------------------------------ | ------------ |
| 关系类型     | is-a 关系                      | has-a 关系                                 | 关系类型     |
| 灵活性       | 编译时确定，行为不可变         | 运行时组合，行为可变                       | 灵活性       |
| 适用结构     | 稳定的层次结构                 | 频繁变化、多维度组合                       | 适用结构     |
| 代码复用     | 子类重用父类的大部分行为和属性 | 类独立实现功能，通过组合实现复用           | 代码复用     |
| 设计模式     | 模板模式、框架设计             | 策略模式、装饰器模式、依赖注入             | 设计模式     |
| 典型应用场景 | 动物、几何图形、框架的模板类   | 支付系统、订单折扣、角色扮演游戏等多维组合 | 典型应用场景 |

典型应用场景	动物、几何图形、框架的模板类	支付系统、订单折扣、角色扮演游戏等多维组合
is-a: Bird is Annimal， 所以Bird可以继承Annimal；
has-a: Car has Engine， 所以引擎需要和车是组合关系；
我们都知道继承是怎么使用的，主要是extends的这个关键字就可以了。那对于组合的话就是在一个类中定义另一个类作为其字段（属性）来使用。这种方式让一个类可以通过持有其他类的实例来获得额外的行为或属性。在一个类中定义另一个类作为其字段（属性）来使用，这种方式让一个类可以通过持有其他类的实例来获得额外的行为或属性。

```java
class Engine {
    void start() {
        System.out.println("Engine started");
    }
}

class Car {
    private Engine engine;  // 组合关系
    public Car() {
        this.engine = new Engine();  // Car 拥有一个 Engine 对象
    }
    public void start() {
        engine.start();  // 通过组合来使用 Engine 的方法
    }
}
```

> 之前产品Leader说过继承会产生耦合，我印象很深刻。文章作者也提出，对于跨包的情况尽量少用继承，优先考虑组合方式。

#### 22.接口仅用来定义类型
1. 常量接口模式是对接口的糟糕使用。因为这些常量会污染实现类甚至是子类。
>所谓常量接口就是这个接口中定义的全是常量，没有实际的方法。接口的设计是用来定义规范和约定，作者指出可以使用枚举或者常量类来使用。
2. 数字文字中使用下划线字符
> 9.109_383_56e-31
> 100_000
3. 使用类名来限定常量名
> Constans.MAX_VALUE 这种形式来对常量限定，作者还表示当同一个常量类中使用比较多的时候，我们可以静态导入包的路径。
> import static com.xXXX.Constant.*;

最佳实践：
1.常量与现有的类或者接口有关，可将其添加到该类或者接口中。例如在使用cache时，可将cacheKey和ttl定义在接口中；
2.使用枚举成员；
3.使用一个不可以实例化的工具类；

```java
public class PhysicalConstants {
private PhysicalConstants() { }  // Prevents instantiation
public static final double AVOGADROS_NUMBER = 6.022_140_857e23;
public static final double BOLTZMANN_CONST =1.380_648_52e-23;
public static final double ELECTRON_MASS = 9.109_383_56e-31;
}
```
#### 24. 优先考虑静态成员类
这篇作者是为了推荐使用静态类。
嵌套类分为四类，总体概括为两大类：
* 静态类
* 内部类
  * 非静态类
  * 匿名类
  * 局部类
```java
class OuterClass {
    private static int staticValue = 10;


    //内部类
   class InnerClass {
        void display() {
            System.out.println("Accessing outer class field: " + outerField);
        }
    }

    // 静态成员类
    static class StaticNestedClass {
        void display() {
            System.out.println("Static Value from OuterClass: " + staticValue);
        }
    }
}
```

静态类和非静态类最大的区别就是前者可以独立new出对象，后者得依靠外部类new出对象后才能创建对象，对外部的依赖是比较大。当内部类和外部类的属性具有关联性的时候，使用非静态内部类，当内部类和外部类属性没有任何关联性的时候，使用静态成员类。
`OuterClass outerClass = new OuterClass();`InnerClass innerClass = outerClass.new InnerClass();`

Note: 内部内对象会有个外部类的隐藏指针，如果内部类对象一直存在会影响外部类对象的回收。
匿名内部类比较好理解：用于将方法作为参数传递实现回调，JDK8有了lambda表达式了，尽量摒弃使用匿名内部类。
局部内部类：定义在方法中，使用较少。
局部内部类是定义在函数的内部,不可以用访问修饰符修饰,只能在函数内部使用,随着函数的调用而使用,只能在该函数中实例化对象,和局部变量差不多。
JDK源码可查看AQS有很多关于静态内部类的使用，总的来看是为了代码更加内聚。

#### 26.不要使用原始类型

作者主要想表达对于集合的使用，提倡使用泛型(`List<?>` 或者`List<T>`)而不是原始类型List的方式
提出使用原始类型List的缺点：

+ 编译器不能检查，运行时会报ClassCastException错；
* 不能确定集合中的对象类型，可读性差；
* List和List<Object>虽然都会接收所有类型，但是对于List可以接收List<String>,List<Object>就不能接收；
* 泛型有子类型的规则， List 的子类型，但不是参数化类型的子类型
* 虽然原始类型List和List< ?>看起来都是能接受任何类型的，但是通配符类型是安全的。

List是元素类型未知的列表，可以包含任何类型元素；
List<?>只能读取不能添加元素，会提示编译错误；安全性的体现在于：明确集合中元素的不确定性，但是会阻止添加元素。

因为泛型类型信息在运行时被删除，所以在无限制通配符类型以外的参数化类型上使用instanceof运算符是非法的。 使用无限制通配符类型代替原始类型不会以任何方式影响instanceof运算符的行为。一旦确定对象是一个 Set ，则必须将其转换为通配符 Set<?> ，而不是原始类型 Set 。 这是 一个强制转换，所以不会导致编译器警告。

| 术语             | 示例                              |
| ---------------- | --------------------------------- |
| 参数化的类型     | List<String>                      |
| 实际类型参数     | String                            |
| 泛型             | List<E>                           |
| 形式类型参数     | E                                 |
| 无限制通配符类型 | List<?>                           |
| 原生态类型       | List                              |
| 有限制类型参数   | <E extends Number>                |
| 递归类型限制     | <T extends Comparable>            |
| 有限制通配符类型 | List<? extends Number>            |
| 泛型方法         | static <E> List <E> asList(E[] a) |
| 类型令牌         | String.class                      |

通配符（Wildcard）
通配符是泛型的一种特殊用法，可以使方法在不确定参数类型的情况下仍然能够处理多种类型。常见的通配符有：
? extends T：表示该类型是 T 的子类或 T 本身，常用于读取类型数据，上边界。
? super T：表示该类型是 T 的父类或 T 本身，常用于写入类型数据，下边界。
?：表示任意类型，适用于我们不关心具体类型时。
泛型方法：
泛型类：一般是存在使用泛型定义了该属性

```java
//只读
public void addElement(List<? extends Number> list) {
    // 以下代码会编译失败
    list.add(new Integer(10));  // 编译错误
    list.add(10);                // 编译错误
}
//可写
public static void addIntegers(List<? super Integer> list) {
    // 由于 List<? super Integer> 可以是 List<Integer> 或 List<Number> 或 List<Object>
    list.add(10);  // 可以安全地添加 Integer 类型的元素
}
```

泛型会存在类型擦除，那么运行期间就不知道是哪种类型，那么集合的反序列化的怎么做的？
#### 28. 列表优于数组
>  Arrays differ from generic types in two important ways. First, arrays are covariant. This scary-sounding word means simply that if Sub is a subtype of Super, then the array type Sub[] is a subtype of the array type Super[]. Generics, by contrast, are invariant: for any two distinct types Type1 and Type2, List<Type1> is neither a subtype nor a supertype of List<Type2> .
>
>  Either way you can’t put a String into a Long container, but with an array you find out that you’ve made a mistake at runtime; with a list, you find out at compile time. Of course, you’d rather find out at compile time.

泛型是通过擦除来实现的
数组提供了运行时的类型安全，但是没有编译时的类型安全，泛型与之相反。一般来说，数组和泛型不能很好地混合使用。
作者认为数组是协变的，泛型是不变的。通过一个例子表明数组的缺点：
```java
// Fails at runtime!
Object[] objectArray = new Long[1];
objectArray[0] = "I don't fit in"; // Throws ArrayStoreException
but this one is not:
// Won't compile!
List<Object> ol = new ArrayList<Long>(); // Incompatible types
ol.add("I don't fit in");
```
上述使用数组的话会在运行期抛错，但是使用列表编译器就会提示错误。
> In summary, arrays and generics have very different type rules. Arrays are covariant and reified; generics are invariant and erased. As a consequence, arrays provide runtime type safety but not compile-time type safety, and vice versa for generics. As a rule, arrays and generics don’t mix well. If you find yourself mixing them and getting compile-time errors or warnings, your first impulse should be to replace the arrays with lists.

#### 31.利用有限制通配符来提升API的灵活性
为了获得最大限度的灵活性 要在表示生产者或者消费者的输入参数上使用通配符类型
如果参数化类型表示一个T生产者 就使用<? extends T> 如果它表示一个T消费者 就使用<? super T>
PECS 代表: producer-extends，consumer-super。
List<? extends T>作为方法的入参
<? extends T> 用于返回值 —— 生产者
生产者场景
当方法参数使用 <? extends T> 时，它表示可以接受 T 类型的子类的集合，并且方法可以从这些集合中读取数据，但不能向其中写入数据。

不要用通配符类型作为返回类型
如果使用得当 通配符类型对于类的用户来说几乎是无形的 它们使方法能够接受它们应该接受的参数 并拒绝那些应该拒绝的参数 如果类的用户必须考虑通配符类型 类的API或许就会出错
如果类型参数只在方法声明中出现一次 就可以用通配符取代它

 Note: https://stackoverflow.com/questions/2723397/what-is-pecs-producer-extends-consumer-super


#### 32. 合理地结合泛型和可变参数
可变参数和泛型不能很好地交互，因为可变参数机制是在数组上面构建的脆弱的抽象，并且数组具有 与泛型不同的类型规则。 虽然泛型可变参数不是类型安全的，但它们是合法的。 如果选择使用泛型(或参数化)可 变参数编写方法，请首先确保该方法是类型安全的，然后使用 @SafeVarargs 注解对其进行标注，以免造成使用不愉快。

#### 34.使用枚举来代替整型常量

例如在接口方法的参数中可使用枚举。你会发现很多人对于int直接作为参数传递，阅读的时候都很懵逼，这个int值代表的意思。另外对于分页的方法，pageSize和pageNumber我们最好也用枚举，尤其是pageSize。用了枚举会固定使用枚举类型定义的数量，因为会有很多人不考虑性能上来就塞一个很大的值导致数据库压力大增，如果使用了枚举就会限制住，那有人会问，那如果那个人在枚举中定义一个很大值怎么办？实际上他定义枚举值和他拍脑袋传入一个int，明显前者的成本会比后者大。另外通过枚举类型找到了这个pageSize，大概率会根据现有参数来的。
枚举的优点：
可读性好；
存在编译器的安全检查：例如如果使用int这种常量的话随便塞个int值，但是如果使用了枚举的话就必须使用对应枚举中的值来传递。

#### 36.使用 EnumSet 替代位属性
写过一篇[博客]({{"/_posts/2021-11-16-Bit位用于option的总结.md" | absolute_url}})可以参考一下用法。
#### 38.使用接口模拟可扩展的枚举
通过定义一个接口，然后使用枚举实现该接口。
作者通过接口的方式来扩展枚举，小编曾经这么写过，实现后你会发现每个枚举都会实现对应的方法，这时候你会发现就是一个策略模式。每一个枚举就是一个策略对象，每个策略对象执行对应的方法。例如你的项目中有很多的handler的话可以应用一下。(小编之前这么用过被喷了，但是个人觉得这种写法有优势的，起码是符合了开闭原则)

作者举了下面的例子，用到你的项目中你会发现，apply方法会需要使用来自spring的bean对象，这时就很犯难了。

```java
// Emulated extensible enum using an interface
public interface Operation {
    double apply(double x, double y);
}
public enum BasicOperation implements Operation {
    PLUS("+") {
        public double apply(double x, double y) { return x + y; }
    },
    MINUS("-") {
        public double apply(double x, double y) { return x - y; }
    },
    TIMES("*") {
        public double apply(double x, double y) { return x * y; }
    },
    DIVIDE("/") {
        public double apply(double x, double y) { return x / y; }
    };
    private final String symbol;
    BasicOperation(String symbol) {
        this.symbol = symbol;
}
    @Override public String toString() {
        return symbol;
    }
}
```

#### 41.使用标记接口定义类型

 如果要标记除类和接口以外的程序元素，或者将标记符合到已经大量使用注解类型的框架中，那么标记注解是正确的选择。 如果发现自己正在编写目标为 ElementType.TYPE 的标记注解类型，请花点时间确定它是否应该是注释类型，是不是标记接口是否更合适。 （常用的标记接口就是Serializable）
•	标记接口定义了一个由标记类实例实现的类型，而标记注解不会
•	标记接口会在编译时就捕获错误，但是标记注解直到运行时才能捕获错误
•	标记接口相对于标记注解来说可以更加精确的定位目标

#### 42.lambda 表达式优于匿名类

这里大家平时用的比较多，不做过多讲述，作者提出合理使用Stream流。

> If you find this code hard to read, don’t worry; you’re not alone. It is shorter, but it is also less readable, especially to programmers who are not experts in the use of streams. Overusing streams makes programs
> hard to read and maintain.
> Luckily, there is a happy medium. The following program solves the same problem, using streams without overusing them. The result is a program that’s both shorter and clearer than the original

看到有一处我笑了，作者自己也认为过度使用lambda会影响可读性、维护性和问题排查，因此要适度使用lambda。另外作者也提供了一些增加可读性的方法论，例如lambda中的代码块过长时，需要重新定义一个方法，那么lambda中只需要获取到该方法的引用就可以了。
 lambda优于匿名内部类，在lambda中使用方法引用提高程序的可读性；

#### 49.检查参数有效性

作者提出对于一些方式对于方法参数进行校验，例如使用Nullable注解,Objects.requireNonNUll(..)和使用断言等的方式。
对于非public或者private的方法可以使用断言的方式，因为是自己的调用。
构造函数也需要对于参数进行检查，以保证存储的是一个有效的参数。

#### 50. 必要时进行防御性拷贝
我理解作者是为了保证对象的安全，避免外部对其内部的对象数据进行任意修改。其实这主要发生在对象嵌套的情况，对于基本类型和包装类还是String类型是没有这类问题的(反射除外)，因为在对象嵌套时，客户端在使用get方法获取到内部对象属性的引用，基于该引用修改内部的数据。另外作者还举出了两个例子来说明：
>
>// Attack the internals of a Period instance
>Date start = new Date();
>Date end = new Date();
>Period p = new Period(start, end);
>end.setYear(78); // Modifies internals of p!
>
>// Second attack on the internals of a Period instance
>Date start = new Date();
>Date end = new Date();
>Period p = new Period(start, end); p.end().setYear(78); // Modifies internals of p!

第一个例子说明在构造对象时，但是客户端存在对象中属性的引用，基于该引用在客户端修改是会导致对象内部的数据发生改变；第二个例子是由于该对象提供了属性的get方法，返回的是对象属性的引用，再基于该引用调用set方法从而修改了其内部的对象。
>// Repaired constructor - makes defensive copies of parameters
>public Period(Date start, Date end) {
>this.start = new Date(start.getTime());
>this.end   = new Date(end.getTime());
>...
>}
>// Repaired accessors - make defensive copies of internal fields
>public Date start() {
>return new Date(start.getTime());
>}


这种方式确实能保证对象的安全性，但是在实际开发中慎用。正如标题所示，在必要时才使用，不要在任何情况下都去使用，用不好就是坑。我理解如果你是依赖包的提供者或者你写一些公用的组件之类的可以使用，如果是单纯的业务功能最好不要使用这类，过度的创建对象会增加对象回收的频率。

#### 53.明智审慎地使用可变参数
作者认为使用可变参数会很方便，但是会有性能担忧，因为在方法内部处理时使用数组来接收，因此每次方法调用都会new一个数组对象，为了解决这样的问题，使用不同参数个数的重载方式，当参数超过三个时使用可变参数，因为作者认为超过三个参数的概率比较低。在高版本JDK9中，你会发现创建不可变的List可以使用List.of的方式，这个of方法就是在超过十个参数的时候才使用了可变参数。

#### 54.返回空的数组或集合，不要返回 null
这个我相信大家都经历过，对于数组返回list.toArray(new String[0])这种，对于集合返回emptyList之类的，因为返回这类就不用判空直接使用，另外不要在集合中存储null值数据。
我们将一个长度为零的数组传递给 toArray 方法时，作者认为会存在性能问题，于是定义一个常量的数组对象行进返回。
private static final Cheese[] EMPTY_CHEESE_ARRAY = new Cheese[0];

#### 55. 明智的返回Optional
1.不要在方法返回类型是Optional时候返回null值;
2.容器类不要用Optional包装返回；
3.不要用Optional返回需要装箱的基本类型，例如int，long之类，涉及两次拆装箱int->Integer->Optional<Integer>；
4.除了作为返回值之外，不应该在任何其他地方中使用 Optional。

> Optional我个人认为是一个很鸡肋的类，如果是写基建的话，我理解没有人会用这玩意的。小编在业务代码中使用也只局限于对象嵌套比较深的情况。因为if判断的话太恶心了。另外我发现项目中使用Optional的方式也有问题，Optional的增加在某种程度上可以避免if判断的，可以通过ifPresent的方式实现链式编程，但是很多开发依旧是使用isPresent方法来判断，这样的我还不如返回null，还能减少内存开销；
> 下述是Optional作者对其理解:
> https://stackoverflow.com/questions/26327957/should-java-8-getters-return-optional-type

#### 57.最小化局部变量的作用域

1. 第一次使用局部变量是声明，不要把变量声明在上面，避免if-else逻辑后该变量都没有使用。

2. 每个局部变量的声明的同时，都应该被赋值，如果没有合适的初始化值，应该推迟声明；

3. 使用一个局部变量来获取一个方法的返回结果，避免在一个循环中频繁调用；

   ```java
   for (int i = 0, n = expensiveComputation(); i < n; i++) {
      ... // Do something with i;
   }
   ```


   作者推荐上述的n的变量定义方式，这样i和n保持相同的作用域。同时认为for循环优于while，因为此处定义的i和n不会对后续的代码误用，会发生编译错误提示，但是while不会。

#### 58.for-each 循环优于传统 for 循环
作者强烈推荐for-each循环也就是我们所说的增强for循环，因为和传统的for循环相比更清晰，灵活和避免错误。
但是会有三种情况for-each循环无法实现：
1.存在数据过滤，例如删除某个值，因为size会改变，推荐用迭代器或者removeIf;
2.数据更新，例如改变容器中某个index的元素值；
3.并行集合，如果需要通过所迭代的下标控制另外的容器的迭代下标，这个时候需要使用传统for循环；
使用迭代器循环需要注意嵌套的情况，如下例子：
```java

for (Iterator<Face> i = faces.iterator(); i.hasNext(); )
    for (Iterator<Face> j = faces.iterator(); j.hasNext(); )
        System.out.println(i.next() + " " + j.next());

```
上述的i.next会在第二次循环的时候发生位置移动，如果两个循环都是6个元素的话，预期打印36次，但是实际上只会打印6次。

#### 60. 若需要精确答案就应避免使用 float 和 double 类型

float 和 double 类型特别不适合进行货币计算，因为基于二进制的。
使用 BigDecimal、int 或 long 进行货币计算
如果数值不超过 9 位小数，可以使用 int;如果不超过 18 位，可以使用 long。如果数量可能超过 18 位，则使用 BigDecimal。 在使用int和long时，需要开发者自己手动的处理小数点。

#### 61. 基本数据类型优于包装类

 将 == 操作符应用于包装类型几乎都是错误的

```java
public static void main(String[] args) {
    Long sum = 0L;
    for (long i = 0; i < Integer.MAX_VALUE; i++) {
         sum += i;
    }
    System.out.println(sum);
}
```


它意外地声明了一个局部变量 (sum) ，它是包装类型 Long，而不是基 本类型 long。程序在没有错误或警告的情况下编译，变量被反复装箱和拆箱，导致产生明显的性能下降

#### 64. 通过接口引用对象
如果存在合适的接口类型，那么应该使用接口类型声明参数、返回值、变量和字段;
List<String> list = new ArrayList<>(); 这是一呃很简单的例子，这样的定义方式会让程序更加的灵活。
如果没有合适的接口存在，那么用类引用对象是完全合适的;
在程序中由于使用接口来引用对象，这会导致对象本身有的方法无法调用，由于抽象化后失去了该方法调用，这时就得往其子类查看(类层次结构中提供所需功能的最底层的类).

#### 78同步访问共享的可变数据

为了在线程之间进行可靠的通信，也为了互斥访问，同步是必要的。
千万不要使用 Thread.stop 方法
作者最后提出“当多个线程共享可变数据的时候，每个读或者写数据的线程都必须执行同步“，其实看情况，我们要完全保证写数据同步执行，避免数据脏写，但是对于读的话看具体需求，如果要求强一致性的话那么这时读也是要同步的，但是如果对于读要求弱一致性，也可以不对读加同步的。例如CopyOnWrite这种集合没有对读加锁。

#### 79. 避免过度同步

通常来说，应该在同步区域内做尽可能少的工作 你不确定的时候，就不要同步你的类，而是应该建立文档，说明他不是线程安全的。

####  81. 相比 wait 和 notify 优先使用并发工具
应该优先使用 ConcurrentHashMap ，而不是使用 Collections.synchronizedMap 。
对于间歇式的定~~时~~，始终应该优先使用 System.nanoTime ，而不是使用 System.currentTimeMillis 。因为 System.nanoTime 更准确，也更精确，它不受系统的实时时钟的调整所影响
没有理由在新代码中使用 wait 方法和 notify 方法，即使有，也是极少的 使用while 而不是if来调用wait方法

#### 83. 明智审慎的使用延迟初始化[懒加载]
如果您使用延迟初始化来取代初始化的循环(circularity)，请使用同步访问器
双重检查模式可以严格保证字段只会初始化一次，但是看起来很复杂。
如果没有严格的要求可以使用单检查模式，但是得用volatile 来修饰字段。
对于静态字段的懒加载可以使用如下的加载方式：

```java
private static class FieldHolder {
    static final FieldType field = computeFieldValue();
}
private static FieldType getField() { return FieldHolder.field; }
```
