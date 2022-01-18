---
title: Lambda表达式中的变量只能用final
key: AAA-2022-01-18-Lambdabiaodashizhongdebianliangzhinengyongfinal
tags: [Lambda]
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

这个和方法传参数一个道理，方法传参时采用的是值传递的方式，类似做一个副本拷贝，lambda表达式同理也是通过这样的方式。但是众所周知method不关心你是不是final类型的，因为代码执行有先后顺序，完全不担心后面对变量的修改影响了之前的数据。 而lambda表达式就不一样了，开发者可以带着当前的变量副本声明着，但是执行却很晚。 如果执行改表达式前改变了实例变量，这时当前的变量和lambda表达式中变量副本就不一致了，这就是问题所在了。



Case1: 需要根据条件判断，然后根据判断后的结果在lambada表达式中用到。

```java
这种情况不要使用if else来判断，要使用三目运算符。
  如果使用if else后需要将获取的结果重新赋给一个变量，比较庸余；
String name;
if(flag){
  name = "zhangsan";
}else{
  name = "lisi";
}
final String tempName = name;

----------------------------分隔符----------------------------------------

String name = flag? "zhangsan": "lisi";
```

Case2: 有数据在Lambda表达式中需要累加等计算

```java
# 1.需要统计的数据很多建议重新建一个Class， 然后基于该类去做加减操作；
# 2.使用数组的方式， 但是数组的index对可读性来说不友好；
# 3.当数据只有一个的情况下，可以使用AtomicInteger 或者Holder这个类。
  eg: Holder<Integer> integerHolder = new Holder<>(5);

```

Case3:其他搞不定的情况

当然没有反射搞不定的， 用了希望不要被领导发现。
