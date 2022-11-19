---
title: 其实parallelStream流真的没那么快
key: AAA-2022-11-13-jdkNewFeature
tags: [Java, parallelStream]
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



大家在使用parallelStream的初衷，我理解应该是为了程序更快的完成，实际上并非如此，甚至起反作用，这篇文章主要是通过实验的方式来说明parallelStream并没有大家想象的那么快。
## 1.简单介绍

说到parallelStream，我们就不得不提ForkJoinPool，因为并行流的内部使用的就是这种类型线程池。简单描述一下这个线程池：

+ 该线程池对计算型的任务很友好，对IO型的任务显得很无力；
+ 使用递归的方式将一个任务拆分成各个小任务；

现在很多开发者在吹嘘ForkJoinPool的性能多么牛逼，其实用对场景还是很可观的，用错了那简直就是灾难。我有理由怀疑那些使用并行流的作者是不是也是ForkJoinPool的吹嘘者。下面我将会通过实验结果来说明parallelStream流的性能。

我会通过三个维度去分析parallelStream的性能：

1. parallelStream和stream的性能；
2. parallelStream在不同结构的集合下的性能；
3. parallelStream在计算类型和IO类型两种任务下的性能；

对于计算类型的任务是通过计算1到n的和，项目中使用时注意包装类的拆装箱操作；对于IO类型的任务是打印1到n；文章中使用的是`System.out.println()`的方式，这影响测试可靠性，最好使用`log.info()`的方式。下述为实验的源码：

```java
public class ParallelStreamDemo {

    public static void main(String[] args) {
      //  List<Long> list = getArrayList(1_000_00L);
      //   testArrayList_output(list);
      //  testArrayList_sum(list);

        List<Long> list = getLinkedList(1_000_000L);
        testLinkedList_output(list);
        testLinkedList_sum(list);
    }

    public static void testArrayList_output(List<Long> list){

        long startTime = System.currentTimeMillis();
        list.stream().forEach(value -> System.out.println(value));
        long endTime = System.currentTimeMillis();

        long startTime2 = System.currentTimeMillis();
        list.parallelStream().forEach(value -> System.out.println(value));
        long endTime2 = System.currentTimeMillis();

        System.out.println("stream + output + ArrayList cost " + (endTime - startTime) + "ms");
        System.out.println("parallelStream + output + ArrayList cost " + (endTime2 - startTime2) + "ms");
    }

    public static void testArrayList_sum(List<Long> list){

        long startTime = System.currentTimeMillis();
        long result = list.stream().reduce(0L, Long::sum);
        long endTime = System.currentTimeMillis();

        long startTime2 = System.currentTimeMillis();
        long result2 = list.parallelStream().reduce(0L, Long::sum);
        long endTime2 = System.currentTimeMillis();

        System.out.println("stream + sum + ArrayList cost " + (endTime - startTime) + "ms");
        System.out.println("parallelStream + sum + ArrayList cost " + (endTime2 - startTime2) + "ms");
    }

    private static List<Long> getArrayList(long max){
       return LongStream.range(1, max).boxed().collect(Collectors.toList());
    }

    private static List<Long> getLinkedList(long max){
        List<Long> list = new LinkedList<>();
        for(long value = 1; value <= max; value ++){
            list.add(value);
        }
        return list;
    }

    public static void testLinkedList_output(List<Long> list){

        long startTime = System.currentTimeMillis();
        list.stream().forEach(value -> System.out.println(value));
        long endTime = System.currentTimeMillis();

        long startTime2 = System.currentTimeMillis();
        list.parallelStream().forEach(value -> System.out.println(value));
        long endTime2 = System.currentTimeMillis();

        System.out.println("stream + output + LinkedList cost " + (endTime - startTime) + "ms");
        System.out.println("parallelStream + output + LinkedList cost " + (endTime2 - startTime2) + "ms");
    }

    public static void testLinkedList_sum(List<Long> list){

        long startTime = System.currentTimeMillis();
        long result = list.stream().reduce(0L, Long::sum);
        long endTime = System.currentTimeMillis();

        long startTime2 = System.currentTimeMillis();
        long result2 = list.stream().reduce(0L, Long::sum);
        long endTime2 = System.currentTimeMillis();

        System.out.println("stream + sum + LinkedList cost " + (endTime - startTime) + "ms");
        System.out.println("parallelStream + sum + LinkedList cost " + (endTime2 - startTime2) + "ms");
    }
}
```

## 2. stream和parallelStream的比较

### 2.1基于ArrayList输出1-n耗时(单位: ms)


<img src="\assets\picture\2022-11\image-20221111163817737.png" alt="image-20221111163817737" style="zoom: 25%;"/>

由上述的实验结果显示，parallelStream在输出耗时比stream要差，这主要原因是因为IO类型的任务发生阻塞概率以及时长都是比较高，这对于ForkJoinPool而言的话就很无力，不能发挥其优势。

### 2.2基于ArrayList计算n总和耗时(单位: ms)

<img src="\assets\picture\2022-11\image-20221111170423291.png" alt="image-20221111170423291" style="zoom:25%;" />

由上述的实验结果显示，parallelStream在求和耗时比stream要好，这主要原因还是归属于ForkJoinPool的对计算类型的任务十分友好。

### 2.3 基于LinkedList输出1-n耗时

<img src="\assets\picture\2022-11\image-20221111170610954.png" alt="image-20221111170610954" style="zoom:25%;" />

在LinkedList的集合中，stream和parallelStream的耗时表现和ArrayList中类似，同样parallelStream表现不是很乐观。

### 2.4 基于LinkedList计算n总和耗时

<img src="\assets\picture\2022-11\image-20221111170633374.png" alt="image-20221111170633374" style="zoom:25%;" />

在LinkedList的集合中，stream和parallelStream的耗时表现和ArrayList有很大的不一样。ArrayList中的求和过程中，parallelStream的耗时明显要小于stream的耗时。这是由于stream流在求和时不需要拆分子任务，而parallelStream在求和时需要拆分很多个子任务，同时主要耗时都在拆分任务的过程，实际求和并没有花太多的时间。如果想提高并行流性能，那就得提高计算过程的复杂度而不再是求和这么简单，换句话说我们得把计算的过程时间超过拆分任务的时间，这才能体现并行流的价值，但这并不是开发者能够准确掌握的。

## 3. parallelStream在ArrayList/LinkedList的比较

### 3.1 parallelStream输出1-n

<img src="\assets\picture\2022-11\image-20221111171029733.png" alt="image-20221111171029733" style="zoom:25%;" />

### 3.2 parallelStream求和n

<img src="\assets\picture\2022-11\image-20221111170800683.png" alt="image-20221111170800683" style="zoom:25%;" />

在上述两个图中，对于求和的耗时来看，基于ArrayList的parallelStream求和耗时很大程度低于基于LinkedList的。主要原因是来自于两个集合的结构，这和2.4节表现的结果的原因一致。我们知道ArrayList是基于数组的，LinkedList是基于双向链表的。parallelStream在对任务拆分成小任务是，对ArrayList可直接使用Index进行拆分，但是对于LinkedList需要遍历进行拆分。比如集合中都有100个数，分成四段。ArrayList可直接使用Index：0-24, 25-49, 50-74, 75-99。但是对于LinkedList而言就需要遍历到相应的位置才能进行一个有效的拆分。如果`总时间 = 拆分任务 + 任务执行`来分析的话，任务执行的时间越小于拆分任务的时间，这就显得parallelStream性能越差。

## 4.总结

基于上述的实验结果，你会发现parallelStream真的没有你想象的那么快。另外总结一下并行流的注意点：

1. 线程不安全；
2. parallelStream不是顺序执行的；
3. 整个项目中直接使用并行流都会使用系统默认配置的ForkJoin线程池，一旦有IO类型的任务就显得无力；
4. 使用并行流注意集合的结构，LinkedList就不适合使用并行流；
5. 对于并行流中存在call远程API或者DB获取数据的这种实现，尽量避免，一旦出现问题后果很严重；

希望看完文章的小伙伴使用时一定要慎重，不要为了快上来就用，其实不一定能达到你想要的效果，同时在评审同事代码的时候提出自己的建议，为公司的项目减少不必要的问题。





